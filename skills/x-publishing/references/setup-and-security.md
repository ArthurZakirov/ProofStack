# X setup and security

Load this reference only when `xurl` is missing, authentication is broken, or the user asks how the X credentials are stored.

## Install the official CLI

Prefer X's official `xurl` project rather than a community posting CLI.

On macOS with Homebrew:

```bash
brew install --cask xdevplatform/tap/xurl
xurl --version
```

If that installation route has changed, verify the current installation method from the official `xdevplatform/xurl` repository before substituting another command.

## X developer app

The user needs an X developer application with user-context OAuth and write permission. For the local `xurl` browser flow, the established callback is:

```text
http://localhost:8080/callback
```

Register the app locally:

```bash
xurl auth apps add <APP_NAME> \
  --client-id "$X_CLIENT_ID" \
  --client-secret "$X_CLIENT_SECRET" \
  --redirect-uri http://localhost:8080/callback
```

Then authenticate and set the default:

```bash
xurl auth oauth2 --app <APP_NAME>
xurl auth default <APP_NAME> <X_USERNAME>
xurl auth status
xurl whoami
```

The browser authorization is a human checkpoint. Do not publish anything as part of authentication testing; `xurl whoami` is the safe verification.

## Bitwarden Secrets Manager pattern

A useful secure layout is:

```text
Project: X Developer API
  X_CLIENT_ID
  X_CLIENT_SECRET

Machine account:
  access to the project
```

Store the Bitwarden machine-account token in macOS Keychain rather than plaintext in `.zshrc`.

A shell function can retrieve the machine token only for the `bws` process:

```bash
bws() {
  BWS_ACCESS_TOKEN="$(
    security find-generic-password \
      -a "<KEYCHAIN_ACCOUNT>" \
      -s "BWS_ACCESS_TOKEN" \
      -w
  )" command bws "$@"
}
```

Do not hard-code the actual token in shell configuration.

Verify Bitwarden access without printing secret values:

```bash
bws project list --output table
bws secret list <PROJECT_ID> --output table
```

For a command that needs the X client values, prefer Bitwarden's injection mechanism rather than materializing them into chat or logs:

```bash
bws run --project-id <PROJECT_ID> -- <COMMAND>
```

## Credential boundary

Allowed to receive X secrets:

- official `xurl`;
- Bitwarden Secrets Manager / `bws`;
- macOS Keychain for the Bitwarden machine token.

Not allowed to receive X secrets:

- Markdown converters;
- downloaded GitHub utilities unless separately audited and explicitly trusted;
- browser automation scripts;
- prompts/chat messages;
- verbose terminal logs.

## Recovery checks

If authentication fails:

```bash
xurl auth status
xurl auth apps list
xurl whoami
```

If `bws` reports `invalid_client`, check whether the Bitwarden account uses the EU service and configure the appropriate Bitwarden server rather than replacing credentials blindly.
