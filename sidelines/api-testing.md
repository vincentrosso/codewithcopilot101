# Testing the Layer Under the Buttons — API Testing
*v1.0.0*

This is a sideline reference, not a paced lesson. Read it after QA06, or the moment you notice that most of your browser smoke suite is spending three seconds and a browser process to check something the page merely *relayed* from somewhere else. There's no time budget and nothing to commit. Everything here can be tried with `curl` against the same Toolshop demo the QA track uses, and there's a reflection prompt at the bottom.

The QA track's Lesson 6 called this "the highest-value next step" and then didn't explain why. This is the explanation.

---

## 1. The thing under the page

Open the Toolshop, search for "hammer", and watch the Network tab (QA03 §4). The page doesn't go and fetch a catalog itself. It asks another program:

```
GET https://api.practicesoftwaretesting.com/products/search?q=hammer
```

and gets back a lump of JSON — a list of products, each with a name, a price, an id. Everything you then see on screen is that lump, arranged. The prices, the ordering, the "no results found", the pagination: decided elsewhere, delivered over HTTP, and merely *drawn* by the browser.

That "elsewhere" is the **API** — an interface for programs rather than people. You've already used two, though nobody called them that at the time: Week 1's `requests.get()` against a brand's website, and Week 7's calls to GitHub Models. Same shape. A request goes out, a response comes back, and the response is data rather than a page.

So the app you've been testing through a browser is really two things stacked:

- **The API** — which decides what's true. Prices, stock, who's allowed to see what, whether an order exists.
- **The UI** — which decides what it looks like. Layout, wording, whether the button is reachable by keyboard.

Almost every bug lives clearly in one or the other. Testing through the browser means testing them fused together, and paying UI prices for API answers.

## 2. Why bother going one layer down

**Speed, and it isn't close.** A Playwright test starts a browser, loads an Angular app, waits for it to render, clicks. Seconds. The equivalent HTTP request is tens of milliseconds. A suite that runs in four seconds gets run on every commit; one that takes eleven minutes gets run nightly, and a nightly suite tells you what broke sometime in the last twenty-four hours.

**Stability.** API tests don't have selectors. Nobody restyles a JSON field, and a designer moving a button breaks nothing. Most of the flakiness you were warned about in QA04 §4 is a browser-layer disease — waiting for elements, animation timing, network races in the page. Down here, you send bytes and read bytes.

**Precision.** When a browser test fails, the honest first question is *which* of the seven things involved broke. When `GET /products?sort=price,asc` returns the list out of order, the defect is in exactly one program, and your bug report gets to say so with a copy-pasteable request instead of a five-step repro.

**Reach.** Some states are miserable to set up by clicking and trivial to set up by request: a cart with forty items, an expired session, an order in a half-finished state, the 400th product. Even when you *test* through the UI, driving the setup through the API is often the difference between a case you run and a case you mark BLOCKED.

This is the test pyramid from W05D3 and QA04, one step further out: cheap fast checks underneath, expensive slow ones on top, and as few of the expensive ones as will do the job. Your browser suite shouldn't be checking that the price maths is right — it should be checking that the price the API returned actually reaches the screen.

**And the blindness, stated plainly:** a fully green API suite tells you nothing about whether a human being can complete a purchase. The API can be flawless while the checkout button sits behind a modal that won't close. API tests replace the *checks* in your browser tests, never the browser tests themselves, and never exploration.

## 3. What a request actually is

You've been sending these all course; here are the parts with their names.

**Request:**

| Part | Example | What it's for |
|------|---------|---------------|
| **Method** | `GET`, `POST`, `PUT`, `PATCH`, `DELETE` | The verb. Read, create, replace, modify, remove. |
| **URL path** | `/products/01M0WXVAQ3ZAWQMPHM94AM9BYF` | Which thing. |
| **Query params** | `?q=hammer&sort=price,asc` | Options on a read: filter, sort, paginate. |
| **Headers** | `Content-Type: application/json`, `Authorization: Bearer …` | Metadata — the format, who you are. |
| **Body** | `{"email":"…","password":"…"}` | The payload, on writes. |

**Response:** a **status code**, headers, and usually a JSON body. Those three are your entire observation surface — which is a much smaller surface than a rendered page, and that's the point.

Try it. `curl` is already on your Mac:

```
curl -s https://api.practicesoftwaretesting.com/categories | python3 -m json.tool | head -30
```

`-s` silences the progress meter; piping through `json.tool` pretty-prints it. Look at what comes back: each category has an `id`, a `name`, a `slug`, and a `parent_id` that's either `null` or another category's id. That's a parent/child tree — the same foreign-key idea as W06D3, arriving over the wire. The left-hand filter menu in the UI is that tree, drawn.

Now the catalog, with options:

```
curl -s "https://api.practicesoftwaretesting.com/products?sort=price,asc&between=price,1,20" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['total']); [print(p['price'], p['name']) for p in d['data'][:5]]"
```

Note the response shape: not a bare list, but an object with `current_page`, `data`, `total`. Pagination, made explicit. Two test cases fall straight out of that and neither is visible from the UI: *does `total` agree with the number of rows you can actually page through?* and *what happens on `?page=0`?*

To see the headers and status without the body:

```
curl -s -o /dev/null -D - "https://api.practicesoftwaretesting.com/products"
```

## 4. Authentication — the part with the interesting bugs

Most of an API is behind a login. The Toolshop hands out a token:

```
curl -s -X POST https://api.practicesoftwaretesting.com/users/login \
  -H "Content-Type: application/json" \
  -d '{"email":"customer@practicesoftwaretesting.com","password":"welcome01"}'
```

Back comes an `access_token` — a long dot-separated string, a **JWT**. It's not encrypted, just encoded; the middle section is base64 JSON you can read:

```
TOKEN=$(curl -s -X POST https://api.practicesoftwaretesting.com/users/login \
  -H "Content-Type: application/json" \
  -d '{"email":"customer@practicesoftwaretesting.com","password":"welcome01"}' \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")

echo "$TOKEN" | cut -d. -f2 | python3 -c "
import sys, base64, json
s = sys.stdin.read().strip()
print(json.dumps(json.loads(base64.urlsafe_b64decode(s + '=' * (-len(s) % 4))), indent=2))
"
```

(The padding arithmetic is there because JWTs use base64**url** without `=` padding, so a plain `base64 -d` chokes partway through with a JSON error — a small, extremely common papercut, and a decent reminder that "it decoded to garbage" usually means an encoding detail rather than bad data.)

Read the `iat` and `exp` fields — issued-at and expiry, in Unix seconds. Subtract them. When I looked, the gap was **300 seconds**: this token dies five minutes after it's born.

That single number is a boundary (QA02 §4), and it's one you'd never find by clicking, because a browsing human refreshes their token constantly without noticing. It generates real cases: *what does the app do when a request fails mid-checkout because the token expired?* Does it re-authenticate silently, or drop the user on a login page having lost their cart? Sit on a checkout page for six minutes and find out — that's a bug hunt the API handed you.

Read the rest of the payload too, because one field should stop you: it carries a **`role`**. Your own authorization level, sitting in a string that arrived in your browser and that you can trivially edit. What stops you promoting yourself is the signature — the third dot-separated section — which the server recomputes and checks. So there's a test, and on a practice target it's a fair one to run: change a character in the payload, re-send, and confirm you get a refusal rather than an admin. A server that accepts a tampered token, or accepts one whose header says the algorithm is `none`, has a hole big enough to end a company. Rare in a maintained framework, catastrophic when present, and thirty seconds to check.

Use the token by putting it in a header:

```
curl -s https://api.practicesoftwaretesting.com/products \
  -H "Authorization: Bearer $TOKEN" | head -c 200
```

**Authorization is the highest-yield bug class down here**, because the UI hides the problem so well. The customer account has no admin *menu*, so through the browser it's invisible — but the menu was never the lock. The questions worth asking:

- Does the customer token work on an endpoint only an admin should reach?
- Can you read a record belonging to a *different* user by putting their id in the URL?
- Does a request with **no** token get refused, or quietly treated as a guest with more rights than it should have?

That third one is where real systems fail: not "the check is wrong" but "there was no check, because the UI never offered the button."

**A word on where you point this.** The Toolshop exists to be tested, which is why it's the practice target — but this class of probing is exactly what you must never aim at a system you haven't been asked to test. On a real product, poking at other people's records is a security engagement, and security engagements happen with a scope and permission *in writing*. The technique is ordinary; the authorization is what makes it professional rather than a résumé-limiting event. Same courtesy as the rest of the track applies too: this is someone's free demo server, so a handful of requests, not a flood.

## 5. Everything from QA02 transfers

The design techniques don't change layer; only the inputs do.

**Equivalence partitioning** on `?page=`: a valid page, page zero, a negative page, a page past the end, a non-numeric page. Five requests, five partitions, about ninety seconds. Doing that through the UI means finding a way to make the browser send a page number that its own controls can't produce.

**Boundary values** on `between=price,X,Y`: the exact price of the cheapest product, one cent under, one cent over. On `?q=`: the empty string, one character, a very long string, a term matching nothing.

**Decision tables** for auth, which is where they earn their keep — the conditions are *authenticated?* × *role* × *owns this record?*, and every combination has an expected status code. That table is small, mechanical, and catches things nobody thinks to click.

**State transitions** on the token lifecycle: no token → valid token → expired token → logged out. And the ugly one worth checking on any real system: after logging out, does the token you already have still work? (Server-side sessions get revoked. Stateless JWTs often just… keep working until they expire. Whether that's acceptable is a product decision — which makes it an excellent question to raise, per QA02 §6.)

## 6. What to assert, in order

A response has a lot in it, and asserting on all of it makes a brittle test that fails every time someone adds a field. Work in layers, most stable first:

1. **The status code.** Cheapest, most stable, most informative. Very often the whole test.
2. **The shape.** The fields you depend on exist and are the right type. `price` is a number, not the string `"14.15"`. `data` is a list.
3. **Specific values** — but only the ones your case is actually about. If the case is "search returns matching products", assert every returned name contains the search term; do *not* assert the list has exactly six items, because someone will add a seventh hammer and your test will fail for the wrong reason.
4. **Side effects.** The most-skipped one. A `200` means the request was accepted, not that anything happened. If a `POST` claims to have created an order, go and `GET` the order. **"It returned 200 and did nothing" is a real, common, and completely invisible defect** if you only ever look at the response you were handed.

### The status codes worth knowing by name

You met these in W02D3; here's the tester's cut, because arguing about the *right* code is half of API bug triage.

| Code | Means | Where it goes wrong |
|------|-------|---------------------|
| `200` / `201` / `204` | OK / created / done, nothing to say | `200` returned alongside an error message in the body — pick a lane |
| `400` | Malformed request | Used as a catch-all for everything below |
| `401` | Not authenticated — *who are you?* | Returned when the real answer is 403 |
| `403` | Authenticated but not allowed — *not for you* | Returned as `404` to hide existence; defensible, must be deliberate |
| `404` | No such thing | Returned for a valid resource the caller merely can't see |
| `409` | Conflict — it already exists, or state won't allow it | Rarely used where it should be |
| `422` | Well-formed but semantically invalid — *this email isn't an email* | The one people skip, collapsing validation into 400 or 401 |
| `429` | Too many requests | Test it deliberately only where you're allowed to |
| `500` | The server broke | **Never acceptable as a response to bad input.** A 500 on a malformed payload is a bug every time — it means something threw instead of validating |

Now go and try one for real. Send a login with a value that isn't an email address at all and an empty password:

```
curl -s -o /dev/null -w "%{http_code}\n" -X POST https://api.practicesoftwaretesting.com/users/login \
  -H "Content-Type: application/json" -d '{"email":"nope","password":""}'
```

Read the code you get, and the body. Then decide, with the table above and the **oracles from QA01 §4** in hand: is that the right answer? There's a genuine argument on both sides — "the credentials didn't authenticate, so `401`" versus "`nope` isn't an email address and an empty password isn't a password, so this never should have reached authentication at all; that's `422`". Whichever way you come down, notice what you're doing: you're judging behavior against a *standard* (oracle #6), not against your taste. That's a fileable finding, and the argument in the report is what makes it survive triage.

## 7. Automating it

You already know the tools. `pytest` from Week 5, `requests` from Week 1:

```python
import requests

BASE = "https://api.practicesoftwaretesting.com"


def test_search_returns_only_matching_products():
    r = requests.get(f"{BASE}/products/search", params={"q": "hammer"}, timeout=10)
    assert r.status_code == 200
    names = [p["name"].lower() for p in r.json()["data"]]
    assert names, "search returned no products"
    assert all("hammer" in n for n in names)
```

That's the whole harness. No browser, no `playwright install`, no selectors — and it runs in about the time it takes a Playwright test to open a tab. Put a logged-in token behind a `@pytest.fixture` (W05D2) so each test that needs auth just asks for it, and use `conftest.py` for the base URL so pointing the suite at a different environment is one line.

In CI it's the same workflow file as QA06, minus the browser install step, which also removes the most common reason CI runs are red. A reasonable split once you have both: **API tests for the rules, browser tests for the journey.** Ten API tests covering pricing, filtering, auth, and validation, plus three browser tests proving a human can log in, add to cart, and check out.

## 8. Where this layer lies to you

Every technique has a characteristic way of producing false confidence. This one has four:

- **Green API, broken app.** The data was right and the page didn't render it, or rendered it in the wrong currency. Only a human or a browser test catches that.
- **Testing your own mock.** The moment you replace the real service with a fake to make tests fast, you're testing your fake. Useful for *your* code's behavior; worthless as evidence about *theirs*. (W05D4 taught the technique; this is the caveat.)
- **Contract drift.** The API adds a required field, your tests still send the old payload and still pass because the old shape is still accepted — until the day it isn't. Contract testing exists for exactly this.
- **Endpoint coverage as a metric.** "We test all 40 endpoints" is QA04's coverage lie in a new suit. One `200`-check per endpoint is not coverage; it's a pulse.

---

## Reflection

No commit, no checklist. Two questions, and they're best answered with your QA04 automation list open:

```
## API testing — second look
Which cases in cases/smoke.md are actually checking a rule rather than a journey?
(A rule = the API decided it. A journey = a human moved through screens.)
Which of them would be faster, less flaky, and just as meaningful one layer down?

The token expiry:
Sit on the checkout page longer than the token's lifetime, then try to pay.
What happened, what *should* have happened, and which oracle says so?
```

The first question is the one that changes how you build suites. Most people write every check at the top of the pyramid because that's the layer they can see — and then spend the next year maintaining browser tests that are really arithmetic tests wearing a browser.

---

## Where to next

- **A better client than curl.** [httpie](https://httpie.io) for a friendlier command line; Bruno or Postman when you want a saved, shareable collection of requests. Whatever you use, keep the raw `curl` skill — it's the one that's present on every machine you'll ever ssh into, and it's what you paste into a bug report.
- **OpenAPI / contract testing.** Many APIs publish a machine-readable description of their endpoints and shapes. When one exists, it's the strongest oracle you'll ever get: a written specification (QA01 §4, oracle #1) that a test can check *automatically*.
- **Schema assertions.** `jsonschema` or `pydantic` turn "the shape is right" from a pile of `assert isinstance` lines into one declaration.
- **The security-shaped part.** Authorization bugs are where API testing shades into security work. Read OWASP's API Security Top 10 for the vocabulary — and remember §4: on anything you don't own, scope and written permission come first.

Companions: [qa/QA06.md](../qa/QA06.md) — the browser suite this sits underneath. [qa/QA02.md](../qa/QA02.md) — the design techniques being reapplied here. [W02D3.md](../W02D3.md) — HTTP, methods, and status codes, from the main course.
