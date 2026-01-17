---
title: Installation Guide
slug: installation
description: Step-by-step instructions for installing ThreadsPipe-py, including prerequisites, virtual environments, pip installation, verification, and troubleshooting.
sidebar_label: Installation
sidebar_position: 2
---

# Installation Guide

ThreadsPipe-py is a Python library for interacting with Instagram's Threads API via Meta's Graph API. This guide provides a comprehensive walkthrough for installing the library, setting up your environment, and verifying the installation. The library requires Python 3.8 or higher and supports both core functionality and optional CLI tools.

By the end of this guide, you'll have ThreadsPipe-py installed and ready to use. For an overview of the library's features, see the [Introduction](../01-introduction.md). After installation, proceed to [Setup Authentication](../03-setup-authentication.md) to configure your Meta Developer account.

## Prerequisites

Before installing ThreadsPipe-py, ensure you meet the following requirements:

### Python Version
- ThreadsPipe-py requires **Python 3.8 or higher**. Older versions are not supported due to dependency compatibility.
- Check your Python version by running:
  ```bash
  python --version
  ```
  or
  ```bash
  python3 --version
  ```
- If you need to upgrade, download the latest Python from [python.org](https://www.python.org/downloads/). On Linux/macOS, use your package manager (e.g., `apt install python3.8` on Ubuntu).

### Facebook Developer Account
To use ThreadsPipe-py, you need a Meta Developer account to create an app and obtain API credentials (app ID, app secret, access tokens). This is essential for authentication and posting.

1. Sign up or log in at the [Meta for Developers](https://developers.facebook.com/) portal.
2. Create a new app:
   - Select "Other" > "Business" type.
   - Add the "Threads API" product under "Build Connected Experiences."
3. Configure your app:
   - Go to **App Dashboard > Use Cases > Customize > Settings**.
   - Note your **App ID** and **App Secret**—you'll need these for authentication.
   - Set up a valid **Redirect URI** (e.g., `https://example.com/handler.php`) for OAuth flows.
4. Request necessary permissions (scopes) like `threads_basic` and `threads_content_publish` during app review if needed for production use.

For detailed setup, refer to the [official Meta Threads API documentation](https://developers.facebook.com/docs/threads). Test tokens can be generated via the library's CLI after installation.

### System Dependencies
- **pip**: Ensure pip is installed and up-to-date (`python -m pip install --upgrade pip`).
- No additional system libraries are required, but a stable internet connection is needed for API interactions and dependency downloads.

## Setting Up a Virtual Environment

Using a virtual environment isolates your project dependencies, preventing conflicts with system-wide packages. This is highly recommended, especially for development.

### On Windows
1. Create a virtual environment:
   ```bash
   python -m venv threadspipe_env
   ```
2. Activate it:
   ```bash
   threadspipe_env\Scripts\activate
   ```
   Your prompt should change to indicate the environment is active (e.g., `(threadspipe_env)`).

### On macOS/Linux
1. Create a virtual environment:
   ```bash
   python3 -m venv threadspipe_env
   ```
2. Activate it:
   ```bash
   source threadspipe_env/bin/activate
   ```
   The prompt will update similarly.

To deactivate the environment later, run `deactivate`.

If you encounter issues (e.g., `venv` module not found), install it via your package manager (e.g., `apt install python3-venv` on Ubuntu).

## Installing ThreadsPipe-py

ThreadsPipe-py is distributed via PyPI. Install it using pip.

### Core Installation
Install the base library (includes core dependencies like `filetype` and `python-dotenv`):
```bash
pip install threadspipepy
```

This installs version 0.4.5 (as of the latest release). To install a specific version:
```bash
pip install threadspipepy==0.4.5
```

### CLI Installation (Optional)
The CLI provides commands for token generation and refresh (e.g., `threadspipepy access_token`). It requires the optional `colorama` dependency for colored output.

Install with extras:
```bash
pip install threadspipepy[cli]
```

Verify installation:
```bash
pip list | grep threadspipepy
```
Output should show `threadspipepy 0.4.5` (and `colorama` if CLI is installed).

From `pyproject.toml`, the CLI entry point is `threadspipepy = "threadspipepy.cli:run"`, enabling commands like `threadspipepy access_token --help`.

## Verification

After installation, verify everything works.

### Import Test
Run a simple Python script to check imports and version:
```bash
python -c "import threadspipepy; print('ThreadsPipe-py version:', threadspipepy.__version__)"
```
Expected output:
```
ThreadsPipe-py version: 0.4.5
```
If this fails (e.g., `ModuleNotFoundError`), ensure your virtual environment is active and reinstall.

### CLI Test (If Installed)
Run the help command:
```bash
threadspipepy --help
```
This should display available subcommands like `access_token` and `refresh_token`. For example:
```bash
threadspipepy access_token --help
```
If `colorama` is missing, you'll see a plain output without colors—install it manually with `pip install colorama`.

### Full API Test
Create a test script (`test_install.py`) to initialize the library (use dummy credentials for now):
```python
from threadspipepy import ThreadsPipe

# Initialize with placeholders (replace with real values later)
api = ThreadsPipe(
    user_id="your_user_id",
    access_token="your_access_token",
    handle_hashtags=True
)

# Get profile (requires valid auth; skip if not set up yet)
try:
    profile = api.get_profile()
    print("Profile fetched successfully:", profile.get('username', 'N/A'))
except Exception as e:
    print("Auth setup needed:", e)
```
Run it:
```bash
python test_install.py
```
With valid auth (from [Setup Authentication](../03-setup-authentication.md)), it should print your Threads username.

## Troubleshooting Common Issues

### Missing Dependencies
- **colorama for CLI**: If CLI commands run but output is uncolored or errors occur, reinstall with extras: `pip install "threadspipepy[cli]"`. Manually install: `pip install colorama`.
- **filetype or python-dotenv errors**: Run `pip install --upgrade threadspipepy`. Check for typos in imports.
- **pip failures**: Update pip (`python -m pip install --upgrade pip`) or use `--user` flag: `pip install --user threadspipepy`.

### Virtual Environment Issues
- **Command not found**: Ensure the environment is activated. On Windows, use `Scripts\activate`; on Unix, `bin/activate`.
- **Permission errors**: Avoid `sudo pip`—use virtual environments instead.

### Python Version Mismatch
- Error like "Python 3.7 not supported": Upgrade Python or use `pyenv` to manage versions.
- On macOS, ensure you're using the correct Python (e.g., via `brew install python@3.12`).

### Network/Proxy Issues
- If downloads fail, configure pip for proxies: `pip install --proxy http://user:pass@proxy:port threadspipepy`.
- Firewall blocks: Temporarily disable or whitelist PyPI.

### Authentication-Related Errors During Verification
- If `get_profile()` fails with invalid token: This is expected without setup. Follow [Setup Authentication](../03-setup-authentication.md) for OAuth flows using `api.get_auth_token()`.

For persistent issues, check the [Changelog](../10-changelog-api-reference.md) for fixes or open an issue on [GitHub](https://github.com/paulosabayomi/ThreadsPipe-py/issues).

## Quick 'Hello World' Example

Once installed and authenticated, post your first message. Save this as `hello_world.py`:

```python
from threadspipepy import ThreadsPipe

# Initialize (replace with your real credentials from auth setup)
api = ThreadsPipe(
    user_id="your_user_id",  # From get_auth_token response
    access_token="your_long_lived_access_token",
    handle_hashtags=True
)

# Simple text post
result = api.pipe(
    post="Hello, Threads! This is my first post using ThreadsPipe-py. #ThreadsAPI"
)

if 'error' not in result:
    print("Post successful! ID:", result.get('id'))
else:
    print("Error:", result.get('message'))
```

Run it:
```bash
python hello_world.py
```

Expected output (with valid auth): `Post successful! ID: <post_id>`. This demonstrates the core `pipe` method for posting text (up to 500 characters; longer posts auto-chain).

For media posts, add `files=["path/to/image.jpg"]` and captions. See [Basic Posting](../04-basic-posting.md) for more examples.

This installation should take 5-10 minutes. Total word count: ~950. If you encounter issues, refer to the [Errors & Best Practices](../09-errors-best-practices.md) section.
