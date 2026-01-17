---
title: CLI Usage Guide
slug: cli-usage
description: A comprehensive guide to using the ThreadsPipe CLI for generating, refreshing, and managing access tokens for the Threads API.
sidebar_label: CLI Usage
sidebar_position: 7
tags: [cli, tokens, authentication]
---

# CLI Usage Guide

The ThreadsPipe CLI provides a straightforward command-line interface for managing access tokens in the Threads API. This tool is essential for developers who want to quickly obtain short-lived (1-hour) and long-lived (60-day) access tokens from an authorization code, or refresh long-lived tokens before they expire. Built on Python's `argparse` module, the CLI abstracts the complexities of OAuth token exchange and refresh requests, making it ideal for initial setup, token maintenance, or automation scripts.

This guide covers installation, command details, flags, examples, integration with Python scripts, and troubleshooting. For background on the authentication flow, refer to the [Setup Authentication](../03-setup-authentication.md) page. The CLI leverages the `run()` function from `cli.py`, which serves as the entry point: it parses arguments, handles subcommands (internally mapped to `access_token` and `refresh_token`), performs API calls to Threads endpoints (e.g., `https://graph.threads.net/oauth/access_token` for token exchange), and optionally updates `.env` files using `python-dotenv`. Output is enhanced with colored logging via the `colorama` dependency, which adds visual cues like green for success and red for errors.

The CLI ensures secure handling—sensitive inputs like app secrets are not echoed in logs—and supports silent mode to suppress verbose output. All commands are invoked via the `threadspipepy` executable after installation.

## Installation

To use the CLI, install ThreadsPipe with the optional `[cli]` extras group. This includes `colorama` for terminal color support, which improves readability by formatting messages (e.g., bold green for successful token generation). If you already have the core library installed, the extras will add only CLI-specific dependencies without reinstalling the base package.

Run the following in your terminal:

```bash
pip install threadspipepy[cli]
```

This command installs:
- Core dependencies: `filetype` (for media handling), `python-dotenv` (for `.env` management).
- CLI extras: `colorama` (for colored output).

Verify installation by checking the help menu:

```bash
threadspipepy --help
```

Output will display available commands (`access_token` and `refresh_token`), global flags, and usage syntax. The `run()` function in `cli.py` initializes logging at DEBUG level and uses `colorama.init()` to enable cross-platform color support. If you're in a virtual environment (recommended), activate it first with `source venv/bin/activate` (Linux/macOS) or `venv\Scripts\activate` (Windows).

:::tip
The `[cli]` extras are lightweight—`colorama` is only ~50KB—and ensure the CLI works seamlessly in terminals like iTerm, PowerShell, or Command Prompt.
:::

## Generating Access Tokens

The `access_token` command swaps a one-time authorization code (obtained from the Threads Authorization Window) for both short-lived and long-lived tokens. This mirrors the `ThreadsPipe.get_access_tokens()` method but via CLI. Short-lived tokens are valid for 1 hour and useful for testing; long-lived ones last 60 days and support production use.

### Command Syntax

The command uses a positional argument (`access_token`) followed by required flags. All inputs must match your Threads app settings from the Meta Developer Dashboard.

```bash
threadspipepy access_token --app_id=YOUR_APP_ID --app_secret=YOUR_APP_SECRET --auth_code=YOUR_AUTH_CODE --redirect_uri=YOUR_REDIRECT_URI [--env_path=PATH_TO_ENV] [--env_variable=VAR_NAME] [--silent]
```

- **Positional Argument**: `access_token` – Specifies the token generation subcommand. No value needed; it's the command trigger.
- **Required Flags**:
  - `--app_id` (short: `-id`): Your Threads app ID (e.g., `123456789`).
  - `--app_secret` (short: `-secret`): The app secret from the dashboard (under *Use cases > Customize > Settings*).
  - `--auth_code` (short: `-code`): The authorization code from the redirect URL after user approval (e.g., `AQC...` – valid for ~5 minutes).
  - `--redirect_uri` (short: `-r`): The exact redirect URI registered in your app (e.g., `https://example.com/redirect`). Mismatches cause failures.
- **Optional Flags**:
  - `--env_path` (short: `-p`): Path to a `.env` file (defaults to `./.env`). If provided, updates it with the long-lived token.
  - `--env_variable` (short: `-v`): Name of the environment variable to set (e.g., `THREADS_LONG_LIVED_TOKEN`). Defaults to `THREADS_ACCESS_TOKEN`.
  - `--silent` (short: `-s`): Disables logging and colored output. Pass as `--silent` or `-s true` to suppress DEBUG/INFO messages from `run()`.

### Outputs

- **Console**: Prints a formatted dictionary with `user_id`, `short_lived_token`, and `long_lived_token` using `pprint` for readability. Example:
  ```
  {'user_id': '178414...',
   'short_lived_token': 'short-lived-string-here',
   'long_lived_token': 'long-lived-string-here (expires in 60 days)'}
  ```
  Success is highlighted in green via `colorama.Fore.GREEN`.
- **Environment Update**: If `--env_path` and `--env_variable` are specified, `dotenv.set_key()` appends/updates the `.env` file (e.g., `THREADS_LONG_LIVED_TOKEN=long-lived-string-here`). No overwrite of existing vars unless matching.
- **Errors**: Red error messages (e.g., via `colorama.Fore.RED`) for issues like invalid codes. The process exits with `sys.exit(1)`.

The `run()` function internally calls `__get_access_token__`: It POSTs to the OAuth endpoint for the short token, then exchanges it via GET for the long-lived one (using `grant_type=th_exchange_token`). Logs track each step at DEBUG level.

## Refreshing Access Tokens

Long-lived tokens can be refreshed after 24 hours but before 60-day expiration; short-lived ones cannot. The `refresh_token` command handles this, equivalent to `ThreadsPipe.refresh_token()`.

### Command Syntax

```bash
threadspipepy refresh_token --access_token=YOUR_LONG_LIVED_TOKEN [--env_path=PATH_TO_ENV] [--env_variable=VAR_NAME] [--auto_mode] [--silent]
```

- **Positional Argument**: `refresh_token` – Triggers the refresh subcommand.
- **Required Flags** (unless `--auto_mode`):
  - `--access_token` (short: `-token`): The unexpired long-lived token to refresh.
- **Optional Flags**:
  - `--env_path` (short: `-p`): Path to `.env` (required with `--auto_mode`).
  - `--env_variable` (short: `-v`): Variable name (required with `--auto_mode`).
  - `--auto_mode` (short: `-auto`): If `true`, reads the token from the env variable (ignores `--access_token`), refreshes it, and auto-updates the `.env`. Ideal for scripts.
  - `--silent` (short: `-s`): Suppresses logging.

### Outputs

- **Console**: Prints the new long-lived token and expiration info in green.
- **Environment Update**: Similar to generation—updates `.env` if paths/vars provided.
- **Errors**: Red alerts for expired tokens or API failures.

Under the hood, `run()` invokes `__refresh_token__`, sending a request to `https://graph.threads.net/refresh_access_token`. It enforces the 24-hour rule via API response checks.

## Common Flags and Options

Both commands share flags for consistency:
- Short forms (e.g., `-id` for `--app_id`) reduce typing.
- `--silent` or `-s` mutes `logging` output from `run()`, hiding DEBUG traces while still printing essential results. Useful in CI/CD pipelines.
- No global config file; flags override defaults (e.g., env path `./.env`).

For full help: `threadspipepy access_token --help` or `threadspipepy refresh_token --help`.

## Examples

### Token Generation

1. Basic generation (prints tokens, no env update):
   ```bash
   threadspipepy access_token -id 123456 -secret abcdef -code AQD... -r https://example.com/redirect
   ```
   Output: Colored dict with tokens.

2. With env update (silent mode):
   ```bash
   threadspipepy access_token --app_id=123456 --app_secret=abcdef --auth_code=AQD... --redirect_uri=https://example.com/redirect -p .env -v THREADS_TOKEN -s
   ```
   Updates `.env` silently; tokens printed minimally.

### Token Refresh

1. Manual refresh:
   ```bash
   threadspipepy refresh_token -token eyJ...long-lived-token-here
   ```
   Prints new token.

2. Auto-mode (from env):
   ```bash
   threadspipepy refresh_token --auto_mode=true -p .env -v THREADS_TOKEN
   ```
   Reads from `THREADS_TOKEN` in `.env`, refreshes, and updates it.

In all cases, `colorama` ensures outputs are vivid: `Style.BRIGHT + Fore.GREEN` for success, `Fore.RED` for failures.

## Integration with Python Scripts

The CLI complements the library for token management, but you can integrate tokens into Python scripts seamlessly. After generating tokens via CLI, load them in code using `python-dotenv`.

Example script (`post_thread.py`) using a CLI-generated token:

```python
import os
from dotenv import load_dotenv
from threadspipepy import ThreadsPipe

load_dotenv('.env')  # Loads THREADS_LONG_LIVED_TOKEN from CLI update

tp = ThreadsPipe(access_token=os.getenv('THREADS_LONG_LIVED_TOKEN'))
response = tp.create_thread(text="Hello from CLI-integrated script!")
print(response)
```

Run: `python post_thread.py`. For automation, chain CLI in shell scripts:

```bash
#!/bin/bash
# refresh_and_post.sh
threadspipepy refresh_token --auto_mode=true -p .env -v THREADS_TOKEN -s
python post_thread.py
```

This workflow—CLI for tokens, library for API calls—streamlines development. Refer to [Basic Posting](../04-basic-posting.md) for more script examples. The `run()` function's env updates ensure scripts always use fresh tokens without hardcoding.

:::caution
Never commit `.env` files to version control; use `.env.example` templates.
:::

## Troubleshooting

Common issues stem from OAuth nuances:

- **Invalid Authorization Code**: Error like "Code expired" (red output). Auth codes last ~5 minutes—regenerate via Authorization Window ([Setup Authentication](../03-setup-authentication.md)). Ensure no reuse; each is one-time.
  
- **Redirect URI Mismatch**: API rejects with 400 Bad Request. Double-check dashboard settings match `--redirect_uri` exactly (including trailing slashes).

- **Expired Long-Lived Token**: Refresh fails post-60 days. Generate new via `access_token` command. For refreshes, wait 24+ hours—API enforces this.

- **No Color Output**: If `colorama` fails (e.g., non-ANSI terminal), install manually or use `--silent`. Test with `python -c "from colorama import init; init(); print('\033[92mGreen\033[0m')"`—should print green.

- **Env Update Fails**: Permissions issue? Run with `sudo` or check path. Verify with `cat .env` post-command.

- **CLI Not Found**: Ensure `pip` installs to PATH; use `python -m threadspipepy access_token ...` as fallback.

- **Network/Rate Limits**: Proxy errors? Set `HTTP_PROXY` env var. Threads API limits: 200 calls/hour—CLI respects this via single requests.

For persistent errors, enable DEBUG logging (omit `--silent`) and check `run()` traces. If issues persist, consult [Errors & Best Practices](../09-errors-best-practices.md) or the [Changelog & API Reference](../10-changelog-api-reference.md).

The CLI's design—via `run()`'s robust error handling—minimizes downtime. With practice, token management becomes routine, enabling reliable Threads integrations.

(Word count: 1128)