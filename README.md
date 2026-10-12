# n8n Automations

Three workflows built in [n8n](https://n8n.io), each using Claude for the reasoning step and a human-in-the-loop approval before anything external happens (an email send, a LinkedIn post, a reply to a customer).

## Workflows

### [`lead-triage.json`](workflows/lead-triage.json) — Lead Triage

A contact-form triage agent for a small studio. A visitor submits a form (name, work email, company, what they need). From there:

1. **Normalize** the submission and flag free-email addresses (Gmail, Yahoo, etc.) as a weaker signal.
2. **Score the lead** — a Claude agent, with a `Tavily` web-search tool it calls only when a company is named, scores the enquiry 0-100 against an "ideal client" profile and returns a structured JSON object (score, tier, reasoning, a drafted reply) via a structured output parser.
3. **Log every lead** to an n8n Data Table first, before any routing decision, so nothing is lost if a step downstream fails.
4. **Route by tier**: hot (80+), warm (56-79), reject (under 35), or borderline (35-55) goes to a human.
5. **Hot leads** get a drafted reply from Claude, but it only reaches the customer after a one-click approval email (`Send this reply` / `Don't send`). No AI-written message reaches a real customer unverified.
6. **Borderline or failed-scoring leads** go to a human-decision email instead of being silently dropped or silently approved.

What this demonstrates: structured output parsing, tool-use (the agent decides whether to search), a data table as a durable log independent of routing, and a human-approval gate before any outbound message.

### [`job-hunter.json`](workflows/job-hunter.json) — Job Hunter

Watches for LinkedIn job-alert emails, and for each one:

1. Extracts every individual job listing from the alert email with Claude (not just the first one).
2. For each job, searches the web with Tavily to find the *actual* posting on the company's own site or ATS (Greenhouse, Lever, Workday, etc.) rather than relying on the LinkedIn listing, then verifies with a second Claude call that the result is genuinely the same vacancy (company, title, seniority) before trusting it.
3. Extracts the real page content and runs a second rejection pass (title/company word-overlap check, minimum content length) before treating it as a valid job description — catches cases where the "match" is actually a careers-listing page or an unrelated role.
4. Scores the job against a configurable candidate profile and returns a fit score, seniority classification, and matching/missing requirements.
5. Jobs above a threshold trigger an email with an approve/reject button; approving emails the user a day later-reviewed reply option and logs the decision to a Google Sheet (separate "applied" and "not interested" tabs).

What this demonstrates: multi-stage LLM verification (don't trust a single model call for a factual claim), defensive parsing of LLM output (regex + JSON recovery across different model response shapes), and idempotency (skips jobs already seen across multiple alert emails).

**Before using:** the `CANDIDATE PROFILE` block inside the `rating jobs` node is a placeholder — replace it with your own skills and background.

### [`linkedin-auto-poster.json`](workflows/linkedin-auto-poster.json) — LinkedIn Auto-Poster

A scheduled (roughly weekly) LinkedIn ghostwriter with a full draft/revise/approve loop:

1. On a schedule, emails asking what this week's post should be about (plus an optional picture).
2. Claude drafts a post against a fairly detailed style guide, written to avoid common "obviously AI-written" patterns (banned words, banned sentence structures, no em dashes, specific length band).
3. The draft is emailed back for review. Replying `POST` approves it; replying with feedback sends it through a revision pass that edits the *current* draft rather than restarting from scratch.
4. A final preview email (with the image attached, if any) requires a second explicit click — `Publish to LinkedIn` or `Cancel` — before the LinkedIn API is ever called.
5. Posts with or without an image depending on what was supplied.

What this demonstrates: a multi-turn approval loop over email (not just a single approve/reject), image-aware prompting (the model is told to reference the attached image only if relevant, never to invent details about it), and a style-guide prompt engineered against a specific, checkable list of failure patterns.

**Before using:** the `ABOUT ME` block inside the drafting/revision nodes is a placeholder — replace it with your own background.

## About these files

These are direct exports from my own n8n instance, with the following redacted before publishing:

- My real email address, replaced with `you@example.com`
- All credential IDs, replaced with `YOUR_CREDENTIAL_ID` (these were never API keys — n8n's API never exposes raw credential secrets, only internal references to them — but they're account-specific and not useful to a reader)
- Google Sheet/Doc IDs, replaced with `YOUR_GOOGLE_SHEET_ID`
- My LinkedIn member URN, replaced with `YOUR_LINKEDIN_PERSON_URN`
- Personal background text embedded in two prompts (a bio block in `job-hunter.json`, and an "about me" block in `linkedin-auto-poster.json`), replaced with a placeholder note

To actually run one of these, import the JSON into n8n, reconnect each node's credentials (Gmail, Anthropic, Tavily, Google Sheets, LinkedIn, as relevant), and fill in the placeholder blocks above with your own details.
