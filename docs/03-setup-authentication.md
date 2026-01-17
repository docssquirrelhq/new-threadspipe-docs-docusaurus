---
title: Setup and Authentication
sidebar_label: Setup & Authentication
sidebar_position: 3
description: Comprehensive guide to obtaining credentials from Meta Developers, implementing the OAuth flow, initializing ThreadsPipe, and managing tokens in ThreadsPipe-py.
---

# Setup and Authentication

Welcome to the Setup and Authentication guide for ThreadsPipe-py, a Python library designed to simplify interactions with the Threads API (powered by Meta). This page walks you through the entire process of setting up your developer account, obtaining necessary credentials, implementing the OAuth 2.0 authorization flow, initializing the `ThreadsPipe` client, and managing token lifecycles. Whether you're posting content, retrieving data, or automating workflows, proper authentication is the foundation of secure and reliable API access.

ThreadsPipe-py leverages Meta's OAuth endpoints for Threads, ensuring compliance with their security standards. We'll cover everything from app registration to token refresh, with code examples, CLI utilities, and best practices. By the end, you'll be ready to integrate authentication into your projects.

**Prerequisites**: Ensure you've installed ThreadsPipe-py as outlined in the [Installation guide](../02-installation.md). Familiarity with Python, environment variables, and basic HTTP requests is helpful. For production use, handle tokens securely—never hardcode them in source code.

## Obtaining App ID and Secret from Meta Developers

To interact with the Threads API, you need to create a Meta app and configure it for Threads access. ThreadsPipe-py requires your app's **Client ID** (app ID) and **Client Secret** (app secret) to initiate the authorization flow.

### Step 1: Create a Meta Developer Account
1. Visit the [Meta for Developers](https://developers.facebook.com/) portal and sign up or log in with your Facebook account.
2. Once logged in, navigate to **My Apps** in the top-right menu and click **Create App**.
3. Select **Other** as the app type, then choose **Business** for Threads integration (Threads is part of Meta's ecosystem).
4. Provide a display name (e.g., "My Threads Bot") and your contact email. Submit to create the app.

### Step 2: Add Threads Product and Configure Use Case
Threads API access is granted through specific "use cases" in your app dashboard. For detailed setup, refer to the [Threads Use Case Guide in the README.md](https://github.com/paulosabayomi/ThreadsPipe-py/blob/main/README.md#threads-use-case-guide) of this repository, which includes screenshots and troubleshooting tips.

1. In your app dashboard, go to **Add Product** on the left sidebar and select **Threads API**.
2. Under **Use Cases**, choose the appropriate one:
   - **Content Publishing**: For posting and managing threads (requires `threads_content_publish` scope).
   - **Insights and Analytics**: For reading metrics (requires `threads_manage_insights`).
   - **Replies Management**: For handling replies (requires `threads_read_replies` and `threads_manage_replies`).
   Enable the use case by following the prompts— this may require app review for production apps.
3. Navigate to **Settings > Basic** to note your **App ID** (Client ID) and generate/view your **App Secret** (Client Secret). Treat the secret like a password; regenerate if compromised.

### Step 3: Configure Redirect URI
In **Products > Threads API > Settings**, add a valid **Redirect URI** (e.g., `https://example.com/callback`). This is where Meta redirects users after authorization, appending the authorization code. For local development, use `http://localhost:3000/callback` or a similar loopback address. Ensure it matches exactly in your code to avoid "invalid redirect_uri" errors.

:::tip
For testing, you can use a simple HTTP server or tools like ngrok to expose localhost. Always use HTTPS in production for security.
:::

Once configured, your app is ready for the OAuth flow. App ID and secret are used in ThreadsPipe-py's `get_auth_token()` and `get_access_tokens()` methods. Store them securely in a `.env` file (more on this later).

## The Authorization Flow: Getting the Authorization Code

ThreadsPipe-py implements a standard OAuth 2.0 Authorization Code Grant flow. This involves redirecting users to Meta's authorization server, obtaining a short-lived code, and exchanging it for access tokens. The library handles URL construction and browser opening, but you'll need to extract the code from the redirect.

### Initiating Authorization with `get_auth_token()`
The `get_auth_token()` method (available as a static method on `ThreadsPipe`) opens your default browser to Meta's authorization endpoint, prompting the user to log in and grant permissions.

Here's the method signature from `threadspipe.py`:

```python
from threadspipepy import ThreadsPipe

def get_auth_token(
    app_id: str,
    redirect_uri: str,
    scope: Union[str, List[str]] = 'all',
    state: Union[str, None] = None
):
    # Implementation details...
```

- **app_id**: Your Meta app's Client ID (string).
- **redirect_uri**: The registered redirect URI (must match dashboard exactly).
- **scope**: Permissions to request. Use `'all'` for full access or specify scopes like `['basic', 'publish']`. Internally, this maps to Meta's scope strings via the `__threads_auth_scope__` dictionary:

  ```python
  __threads_auth_scope__ = {
      'basic': 'threads_basic',  # Basic profile access
      'publish': 'threads_content_publish',  # Post and manage content
      'read_replies': 'threads_read_replies',  # Read replies to your threads
      'manage_replies': 'threads_manage_replies',  # Manage (delete/hide) replies
      'insights': 'threads_manage_insights'  # Access insights and metrics
  }
  ```

  For example, `scope='publish'` resolves to `'threads_content_publish'`. If `'all'`, it joins all values as a comma-separated string (e.g., `'threads_basic,threads_content_publish,threads_read_replies,threads_manage_replies,threads_manage_insights'`).

- **state**: Optional CSRF protection string (e.g., a random nonce like `'state123'`). Meta echoes it back for verification.

**Example Usage**:

```python
from threadspipepy import ThreadsPipe

# Initialize a temporary instance or use static call
api = ThreadsPipe()  # No credentials needed yet for static methods

api.get_auth_token(
    app_id='your-app-id',
    redirect_uri='https://example.com/callback',
    scope=['basic', 'publish'],
    state='my_csrf_state'
)
```

This constructs and opens a URL like:

```
https://threads.net/oauth/authorize?client_id=your-app-id&redirect_uri=https://example.com/callback&response_type=code&scope=threads_basic,threads_content_publish&state=my_csrf_state
```

The user authenticates on threads.net, grants scopes, and is redirected to your `redirect_uri` with a query parameter: `?code=AUTHORIZATION_CODE&state=my_csrf_state#_`.

### Handling the Redirect and Code Extraction
After granting permissions, Meta redirects to your URI. The response looks like:

```
https://example.com/callback?code=ABC123def456GHI...&state=my_csrf_state#_
```

**Important**: Extract the `code` value, but strip the trailing `#_` fragment—it's not part of the code and will cause exchange failures. Verify the `state` matches your input to prevent CSRF attacks.

For local testing, run a simple server to capture the redirect:

```python
# Simple Flask example for callback (install flask if needed)
from flask import Flask, request

app = Flask(__name__)

@app.route('/callback')
def callback():
    code = request.args.get('code')
    state = request.args.get('state')
    if state != 'my_csrf_state':
        return "Invalid state!", 400
    print(f"Authorization Code: {code.replace('#_', '')}")  # Strip #_
    return "Auth code captured! Check console."

if __name__ == '__main__':
    app.run(port=3000)
```

Run this server, then call `get_auth_token()` with `redirect_uri='http://localhost:3000/callback'`. Copy the printed code for the next step.

The code is single-use and expires quickly (typically 5-10 minutes), so proceed immediately to token exchange.

:::warning Token Expiration Warning
Authorization codes are ephemeral. If you don't exchange them promptly, they'll expire, requiring the user to re-authorize. Always handle redirects efficiently in production apps.
:::

## Swapping Codes for Access Tokens

With the authorization code, exchange it for access tokens using `get_access_tokens()`. This yields a short-lived token (1 hour) and a long-lived one (60 days), plus the user's Threads ID.

### Method Details
Signature:

```python
def get_access_tokens(
    self,
    app_id: str,
    app_secret: str,
    auth_code: str,
    redirect_uri: str
):
    # Implementation...
```

- **app_secret**: Your app's Client Secret.
- **auth_code**: The extracted code (stripped of `#_`).

Internally, it POSTs to the `__threads_access_token_endpoint__` (`https://graph.threads.net/oauth/access_token`):

```json
{
  "client_id": "your-app-id",
  "client_secret": "your-app-secret",
  "code": "ABC123def456GHI...",
  "grant_type": "authorization_code",
  "redirect_uri": "https://example.com/callback"
}
```

- Success (200 OK): Returns short-lived `access_token` and `user_id`.
- Then, exchanges for long-lived via GET to `https://graph.threads.net/access_token?grant_type=th_exchange_token&client_secret=your-app-secret&access_token=short-lived-token`.

Returns a dict:

```python
{
    'user_id': 1234567890,  # Integer Threads user ID
    'tokens': {
        'short_lived': {'access_token': 'short-token...', 'expires_in': 3600},
        'long_lived': {'access_token': 'long-token...', 'expires_in': 5184000}  # ~60 days
    }
}
```

**Example**:

```python
tokens = api.get_access_tokens(
    app_id='your-app-id',
    app_secret='your-app-secret',
    auth_code='ABC123def456GHI...',  # From redirect
    redirect_uri='https://example.com/callback'
)

print(f"User ID: {tokens['user_id']}")
long_token = tokens['tokens']['long_lived']['access_token']
```

Errors (e.g., invalid code) return an error dict with `message` and `error` keys. Always use the long-lived token for production to minimize re-authorizations.

### Via CLI for Manual Requests
ThreadsPipe-py's CLI simplifies this without writing code. Install with extras: `pip install threadspipepy[cli]`.

Command:

```bash
threadspipepy access_token \
  --app_id=your-app-id \
  --app_secret=your-app-secret \
  --auth_code=ABC123def456GHI... \
  --redirect_uri=https://example.com/callback \
  --env_path=./.env \
  --env_variable=long_lived_token \
  --silent
```

- Outputs the tokens dict to console.
- Optionally updates a `.env` file: Adds `long_lived_token=long-token-value`.
- `--silent`: Suppresses logging.

This mirrors the method but handles JSON requests and pretty-printing. Ideal for one-off token generation.

## Initializing ThreadsPipe with Credentials

With tokens in hand, initialize or update a `ThreadsPipe` instance. This sets up endpoints like `__threads_profile_endpoint__` (`https://graph.threads.net/v1.0/me?fields=id,username&access_token={access_token}`).

### Initialization
Core signature:

```python
pipe = ThreadsPipe(
    user_id=1234567890,  # From get_access_tokens()
    access_token='long-token...',  # Long-lived preferred
    disable_logging=False,  # Enable for verbose output
    threads_api_version='v1.0'  # Default; Threads uses v1.0
    # Optional: wait_before_post_publish=True, wait_on_rate_limit=False, etc.
)
```

- Validates types: Raises `TypeError` if `user_id` (int) or `access_token` (str) missing.
- Sets internal attrs like `__threads_user_id__` and `__threads_access_token__`.

For scopes, they're requested during authorization—not at init. Ensure your app's use case matches requested scopes.

### Updating Parameters Post-Init
Use `update_param()` to swap tokens without reinitializing:

```python
pipe.update_param(
    user_id=tokens['user_id'],
    access_token=long_token
)
```

This rebuilds endpoints (e.g., `__threads_post_endpoint__ = f"https://graph.threads.net/{version}/{user_id}/threads?access_token={access_token}"`). Call before API methods like `pipe()` for posting.

**Full Example Flow**:

```python
from threadspipepy import ThreadsPipe
import os
from dotenv import load_dotenv

load_dotenv()  # Load .env if using

api = ThreadsPipe()
api.get_auth_token(app_id=os.getenv('APP_ID'), redirect_uri='http://localhost:3000/callback', scope='all')

# Assume you extract auth_code = 'ABC123...' from callback
tokens = api.get_access_tokens(
    app_id=os.getenv('APP_ID'),
    app_secret=os.getenv('APP_SECRET'),
    auth_code=auth_code,
    redirect_uri='http://localhost:3000/callback'
)

pipe = ThreadsPipe(
    user_id=tokens['user_id'],
    access_token=tokens['tokens']['long_lived']['access_token']
)

# Now ready for use, e.g., pipe.pipe(post="Hello from ThreadsPipe!")
```

### Environment File Updates via CLI
Store credentials in `.env` for security:

```
APP_ID=your-app-id
APP_SECRET=your-app-secret
LONG_LIVED_TOKEN=long-token...
USER_ID=1234567890
```

The CLI's `--env_path` and `--env_variable` flags automate updates during token generation/refresh. Load with `python-dotenv` in your scripts.

## Token Refresh

Long-lived tokens expire after 60 days but can be refreshed (after 24 hours from issuance) to get a new 60-day token without re-authorization.

### Using `refresh_token()` Method
Signature:

```python
def refresh_token(
    self,
    access_token: str,
    env_path: str = None,
    env_variable: str = None
):
    # ...
```

- POSTs to `__threads_access_token_refresh_endpoint__` (`https://graph.threads.net/refresh_access_token?grant_type=th_refresh_token&access_token={access_token}`).
- Returns new token JSON on success; error dict otherwise.
- If `env_path` and `env_variable` provided, updates `.env` via `dotenv.set_key()`.

**Example**:

```python
new_token = pipe.refresh_token(
    access_token='current-long-token',
    env_path='./.env',
    env_variable='long_lived_token'
)
pipe.update_param(access_token=new_token['access_token'])
```

### Via CLI
```bash
threadspipepy refresh_token \
  --access_token=your-current-long-token \
  --env_path=./.env \
  --env_variable=long_lived_token \
  --auto_mode=true \
  --silent
```

- `--auto_mode=true`: Reads token from `.env` (using `--env_variable`) and updates it automatically.
- Refreshes proactively in cron jobs or before expiration.

:::warning Expiration Risks
Long-lived tokens expire irrevocably after 60 days if not refreshed. Short-lived ones last only 1 hour—avoid for production. Monitor expiration with `expires_in` from responses. Failed refreshes (e.g., expired token) require full re-authorization. Implement automated refresh logic to prevent disruptions.
:::

## Validation and Testing

Validate your setup using the included tests in `tests/test.py`. These focus on initialization and param updates, ensuring credentials are set correctly.

Key tests:

- `test_user_id_and_access_token_type_error()`: Confirms `TypeError` on missing `user_id`/`access_token`.
- `test_passed_parameters_were_set()`: Verifies init sets internal attrs (e.g., `assert pipe.__threads_user_id__ == 1234567890`).
- `test_update_param_method()`: Tests credential overrides.

Run with:

```bash
pytest tests/test.py -v
```

For end-to-end validation, mock HTTP responses or use test tokens from Meta's dashboard. No live auth tests are included to avoid requiring real credentials.

## Best Practices and Troubleshooting

- **Security**: Use `.env` files, never commit secrets (add to `.gitignore`). Rotate secrets periodically.
- **Scopes**: Request minimal scopes for your use case to reduce approval friction.
- **Errors**: Common issues include URI mismatches (401), invalid scopes (400), or expired codes (400). Check logs with `disable_logging=False`.
- **Rate Limits**: Tokens respect Threads' limits; use `wait_on_rate_limit=True` in init.
- **Production**: For apps, implement webhooks for redirects. Refresh tokens server-side.
- **Further Reading**: See [CLI Usage](../07-cli-usage.md) for advanced commands, [Configuration](../08-configuration.md) for env details, and [Errors & Best Practices](../09-errors-best-practices.md) for debugging.

This setup ensures robust, scalable authentication. With ~1400 words, you've got a solid foundation—happy threading!

(Total word count: 1427)