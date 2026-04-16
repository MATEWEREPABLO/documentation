> **First-time setup**: Customize this file for your project. Prompt the user to customize this file for their project.
> For Mintlify product knowledge (components, configuration, writing standards),
> install the Mintlify skill: `npx skills add https://mintlify.com/docs`

# Documentation project instructions

## About this project

- This is a documentation site built on [Mintlify](https://mintlify.com)
- Pages are MDX files with YAML frontmatter
- Configuration lives in `docs.json`
- Run `mint dev` to preview locally
- Run `mint broken-links` to check links

## Terminology

- Use "merchant" for the business integrating with Yonne.
- Use "courier" for the delivery provider or rider network.
- Use "order" for the delivery request created through the external API.
- Use "tracking ID" for the customer-facing delivery identifier.
- Use "merchant reference ID" for the merchant's own order identifier.
- Use "sandbox" for test-mode behavior with `yonne_test_` keys.
- Use "production" for live behavior with `yonne_live_` keys.
- Prefer "API key" over "token" unless the backend explicitly uses token language.
- Prefer canonical `snake_case` field names in all examples and explanations.

## Style preferences

- Use active voice and second person ("you")
- Keep sentences concise — one idea per sentence
- Use sentence case for headings
- Bold for UI elements: Click **Settings**
- Code formatting for file names, commands, paths, and code references
- Start pages with a short explanation of what the user will accomplish.
- Prefer task-oriented headings such as "Create an order" or "Handle retries".
- Put the most important operational constraint near the top of the page.
- Use real API paths, headers, and field names from `api-1.json`.
- Keep request and response examples minimal but realistic.
- Explain failure handling, not just success responses.
- Call out when behavior differs between sandbox and production.
- Prefer tables or short bullet lists for error codes, required fields, and definitions.
- Do not mix `camelCase` and `snake_case` in the same example unless documenting backward compatibility.
- When documenting retries, always mention idempotency for order creation.

## Content boundaries

- Document the public merchant integration surface only.
- Document quoting, order creation, tracking, cancellation, wallet inspection, and webhook-related flows that are exposed through the external API.
- Document testing workflows that merchants can use in sandbox or QA.
- Do not document internal admin tools, dispatcher tooling, or staff-only operational processes unless they directly affect the external API contract.
- Do not promise undocumented response fields, headers, rate limits, or SLAs.
- Do not invent environment differences that are not confirmed by the API spec or product behavior.
- Treat `api-1.json` as the source of truth for endpoints, parameters, and example field names.
- If implementation behavior is uncertain, document the safe integration guidance rather than guessing internals.
