---
name: x-publishing
description: Set up and use X/Twitter publishing from an agent or terminal with the official xurl CLI, including posts, threads, media, and X Articles drafted from Markdown/MDX. Use when the user wants to configure X API access, publish or delete posts, create threads, upload media, validate Markdown for X Articles, create Article drafts, or publish a reviewed Article.
---

# X Publishing

Use the official X CLI as the credential-bearing boundary, and keep third-party Markdown conversion code credentialless.

## Core invariant

```text
posts / threads / media ───────────────→ official xurl ─→ api.x.com

Markdown / MDX
    ↓
credentialless validator / converter
    ↓
X Article content_state JSON ──────────→ official xurl ─→ api.x.com
```

Never give a third-party converter the user's X OAuth tokens, X browser cookies, Bitwarden access token, or browser session. Never use `auth_token` / `ct0` cookie automation when the official API can perform the operation.

## Start every X task with discovery

Do not redo setup when the machine is already configured. First run:

```bash
command -v xurl
xurl --version
xurl auth status
xurl whoami
```

Interpretation:

- `xurl whoami` succeeds: use the existing authenticated account.
- `xurl` is missing or OAuth is not configured: read [Setup and security](references/setup-and-security.md).
- Do not print, copy, or ask the user to paste secrets into chat.

## Posts and threads

Before any public write, show or otherwise establish the exact final text and require the user's explicit instruction to publish it. A request such as “post it” after reviewing the draft is sufficient authorization for that content.

Create one post:

```bash
xurl post "TEXT"
```

Create a thread by capturing each returned post ID and replying to the immediately previous post:

```bash
xurl post "1/ ..."
xurl reply <POST_ID_1> "2/ ..."
xurl reply <POST_ID_2> "3/ ..."
```

Other common operations:

```bash
xurl whoami
xurl read <POST_ID_OR_URL>
xurl delete <POST_ID_OR_URL>
xurl media upload <FILE>
```

Use `xurl <command> --help` before inventing flags.

## X Articles

For Markdown/MDX Article work, read [Article pipeline and compatibility](references/article-pipeline.md).

Default workflow:

1. Author or receive Markdown/MDX.
2. Validate locally and surface every lossy/unsupported construct.
3. Resolve or explicitly accept warnings before creating an X draft.
4. Convert to X `content_state` without credentials.
5. Upload local media with official `xurl`.
6. Create an **unpublished** Article draft through the official X API.
7. Let the user review the draft in X.
8. Publish only after a separate explicit instruction to publish that Article.

Creating a draft changes external state but does not make the Article public. Do not create repeated probe drafts casually: X can enforce a low user-level daily draft-create limit.

## Security rules

- `xurl` is the only component allowed to hold/use X credentials.
- Store durable secrets in a secrets manager or OS credential store, not plaintext shell configuration.
- Never paste OAuth client secrets, refresh/access tokens, Bitwarden machine tokens, X cookies, or browser-session material into chat or command output.
- Never run `xurl -v` on authenticated requests: tested `xurl` versions print the `Authorization` header to the terminal.
- Do not use XMaster or browser-cookie tooling as the authenticated publishing path for this workflow.
- Third-party code may be reused for pure conversion only after source review, version pinning, and dependency installation with lifecycle scripts disabled when supported.
- Treat HTTP `429` as a stop condition; do not retry blindly.
- Treat HTTP `5xx` as potentially transient, but isolate the payload before repeated writes.

## Current converter choice

The tested converter is `mgcrea/mcp-x`, used only for its pure Markdown → Article conversion/validation code. Keep it credentialless.

Use the pinned source procedure in [Article pipeline and compatibility](references/article-pipeline.md). Do not silently upgrade the pin: audit the new revision first.

## Handoff

After any write, report:

- what was created/deleted;
- the returned post/article IDs and public URL when one exists;
- whether the object is public or only a draft;
- any formatting warnings or unresolved compatibility limits.
