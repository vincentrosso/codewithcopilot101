# Field Notes

Case studies from real NIT America working days, written for someone who has finished the twelve weeks and the QA track. Each one is a single day or a single build, with the decisions left in: what was asked, what the vendor would and would not allow, where the code ended up living and why, what broke, and how the AI coding agent was actually used along the way (the case studies were worked with Claude Code; each note maps the habits onto Copilot agent mode).

They are not paced and they do not advance Brand Lens. Read one when the topic comes up at work, or when you want to see the course's habits — diff review, tests first, secrets in the shell, a human on the production button — applied under real constraints.

Each note ends with an optional hands-on that reproduces the vendor half in a free sandbox, and a journal prompt. Journal entries go in `journal/F0N.md` in whichever repo you did the hands-on in.

| # | Topic |
|---|-------|
| [Field Notes 1](F01.md) | **A phone-first login for a Shopify store, in one day** — why it could not be a Shopify login (Multipass), the five-piece map (theme via Shopify CLI, a Lambda members API, a Dev Dashboard app, webhooks, SMS referrals), the schema-before-values push trap, the three states of a permission (declared / granted / in-hand), handling your own webhook echo, Smile.io vs building referrals ourselves, the honest list of where Shopify said no, and a free dev-store hands-on that reproduces the scope lesson |
