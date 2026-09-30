# X Article pipeline and compatibility

Use this reference for creating X Article drafts from Markdown or MDX, validating rendering, handling media, or debugging Article API failures.

## Source format

Prefer Markdown with optional YAML front matter as the first canonical authoring format. MDX and Notion can later be adapters into the same Markdown/AST pipeline.

Example:

````markdown
---
title: Example Article
---

# Example Article

Intro with **bold** and *italic*.

## Section

| Feature | Status |
| --- | --- |
| Tables | Supported |

```python
print("hello")
```
````

Do not author directly in X unless repairing a draft that the API cannot express.

## Trust boundary

```text
Markdown / MDX
    ↓
credentialless conversion + validation
    ↓
content_state JSON
    ↓
official xurl
    ↓
X /2/articles/draft
```

The converter never receives:

- X OAuth tokens;
- OAuth client secret;
- Bitwarden token;
- `auth_token` / `ct0` browser cookies.

## Tested converter snapshot

The converter used during the initial validation was `mgcrea/mcp-x` tag `v0.4.0`, checked out at:

```text
9d452dcd7cdab6e7dfcc451263d39e68ca66f308
```

Reproduce/verify a local checkout rather than trusting this snapshot forever:

```bash
git clone --depth 1 --branch v0.4.0 https://github.com/mgcrea/mcp-x.git <MCP_X_DIR>
git -C <MCP_X_DIR> rev-parse HEAD
```

Install dependencies without lifecycle scripts:

```bash
cd <MCP_X_DIR>
pnpm install --frozen-lockfile --ignore-scripts
pnpm vitest run test/article.test.ts
```

Initial validation result at that revision: 33 Article conversion tests passed. This is a historical tested snapshot, not a promise about future revisions. Re-run the test command after any dependency or source change.

Do not silently update the converter revision. Review at least its package/dependency metadata, install scripts, authentication/network code, filesystem access, and Article conversion code first.

## Conversion behavior established by tests and live X drafts

The converter and X editor were tested with representative fixtures.

| Markdown construct | X Article behavior |
| --- | --- |
| Paragraph | Native text block |
| `**bold**` | Native bold style |
| `*italic*` | Native italic style |
| `~~strike~~` | Native strikethrough style |
| `##` | Native heading |
| `###` | Native smaller heading |
| `####`–`######` | Flatten to the deepest supported heading level; warn |
| `> quote` | Native blockquote |
| `- item` | Native unordered list |
| `1. item` | Native ordered list |
| Nested lists | Flatten to top level; warn |
| `[label](https://...)` | Native link |
| Relative link | Resolve with an explicit site base URL or drop the link and warn |
| `#fragment` link | Keep label, drop internal anchor and warn |
| `---` | Native divider |
| fenced code | Native X Markdown/code entity; syntax highlighting was visually verified |
| GFM table | Native X Markdown/table entity; table rendering was visually verified |
| inline code | No distinct inline-code style; becomes plain text and warns |
| Markdown image | Image entity after official media upload |
| standalone X post URL | Native post-embed entity in converter |
| raw HTML | No HTML rendering; retained as text and warns |
| arbitrary MDX component | No automatic semantic mapping; require a defined adapter/fallback |

Visually verified in the real X Article editor: bold, italic, strikethrough, headings, blockquote, unordered list, absolute link, divider, fenced code, and GFM table.

Image entities and X-post embeds should still be visually checked when changing converter/API revisions even when conversion succeeds.

## Validation policy

Before creating a draft, fail or stop for user review when conversion would silently lose meaning.

Warnings that normally require a decision:

- H4+ flattening when hierarchy matters;
- nested-list flattening;
- inline code losing code styling;
- internal anchor links;
- raw HTML;
- MDX components;
- missing local images;
- unresolved relative links.

A deterministic fallback may be defined for a project, for example rendering an unsupported diagram/component to PNG, but do not invent a fallback silently.

## Creating an Article payload

Use the converter's pure `markdownToArticle` path to create `content_state`. Do not configure the converter with X credentials.

For local images:

1. Validate every image path before uploading anything.
2. Upload with official `xurl`:

```bash
xurl media upload --category tweet_image <IMAGE_FILE>
```

3. Insert the returned media ID into the image entity's `media_items` structure using the converter's `attachImages` logic or equivalent reviewed code.

Create an **unpublished** draft with official `xurl`:

```bash
xurl -X POST /2/articles/draft \
  -H 'Content-Type: application/json' \
  -d "$(cat article-payload.json)"
```

Successful responses return an Article draft ID. Report that ID and state clearly that the Article is not public.

## Publishing

Publishing is a separate consequential action. Only perform it after the user explicitly instructs the agent to publish the reviewed Article.

Use the official Article publish endpoint through `xurl`:

```bash
xurl -X POST /2/articles/<ARTICLE_ID>/publish
```

Do not combine draft creation and publication into one automatic command by default.

## Rate limits and failure handling

During initial testing, X returned a user-level Article draft-create limit of 10 per 24 hours on the tested account. This can change; treat it as an observed snapshot, not a universal constant.

Do not burn draft quota on unnecessary probes. On HTTP `429`, stop creating drafts and wait for the user-level window to reset.

Do **not** use `xurl -v` to inspect authenticated response headers: the tested CLI printed the full `Authorization` bearer token to the terminal.

If a rich payload returns HTTP `5xx`:

1. Do not publish anything.
2. Confirm a minimal draft can still be created if quota allows.
3. Probe features independently rather than repeatedly resending the same complex payload.
4. Distinguish API/service instability from a converter/schema problem.

Initial testing showed individual drafts successfully accepted native formatting, links, dividers, fenced code, and tables. A larger combined rich-content payload returned `503`, so combined-payload behavior remains an empirical validation point rather than a guaranteed invariant.

## Regression fixture

When changing this pipeline, maintain a non-production fixture containing at least:

- title/front matter;
- H2/H3/H4;
- bold/italic/strikethrough;
- quote;
- unordered and ordered lists;
- nested list;
- absolute/relative/anchor links;
- divider;
- fenced code;
- GFM table;
- local image;
- standalone X-post URL;
- inline code;
- raw HTML;
- one MDX component if MDX is supported by the source adapter.

Run local validation first. Create X drafts only for the smallest set of probes needed to establish real editor behavior.
