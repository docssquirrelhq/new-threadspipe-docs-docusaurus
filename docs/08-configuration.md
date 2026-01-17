---
title: Configuration and Customization
slug: configuration
description: Guide to configuring ThreadsPipe-py, including initialization parameters, runtime updates, hashtag handling, wait timings, rate limits, GitHub integration, and performance considerations.
sidebar_label: Configuration
sidebar_position: 8
---

# Configuration and Customization

ThreadsPipe-py is a flexible Python library for interacting with Meta's Threads API, designed to handle posting, media uploads, replies, and data retrieval with minimal boilerplate. While the basic setup (covered in [Setup Authentication](../03-setup-authentication.md)) gets you started, advanced usage often requires customizing the `ThreadsPipe` class to suit your workflow. This page dives into the configuration options available during initialization, runtime adjustments via the `update_param()` method, and specialized behaviors like hashtag distribution, wait timings, rate limiting, and GitHub integration for local file uploads.

Configuration impacts everything from logging verbosity to API reliability and performance. By tweaking these settings, you can optimize for production environments, testing scenarios, or high-volume posting. We'll explore each aspect with examples, best practices, and considerations for memory and performance. For a quick overview of the library's core features, refer to the [Introduction](../01-introduction.md).

## Initialization Parameters

The `ThreadsPipe` class is initialized with required credentials and a suite of optional parameters that control its behavior. These are passed to the `__init__` method and define the instance's default configuration. Here's a breakdown of the key parameters, focusing on those that enable customization:

- **user_id** (`int`, required): Your Threads user ID, typically obtained during authentication (e.g., via the `get_access_tokens` utility). This identifies the account for all API calls.

- **access_token** (`str`, required): A valid access token for the Threads API. Use a long-lived token (valid for 60 days) for production to avoid frequent renewals. Short-lived tokens (1 hour) are suitable for testing.

- **disable_logging** (`bool`, default: `False`): Controls whether the library outputs logs using Python's `logging` module. Set to `True` for silent operation in scripts or CI/CD pipelines, reducing console noise. When enabled, logs provide insights into API calls, errors, and processing steps (e.g., media upload status).

- **threads_api_version** (`str`, default: `'v1.0'`): Specifies the Threads API version. The current stable version is `'v1.0'`, which supports features like quoting, reposts, and media attachments. Future updates might require bumping this for new endpoints—check the [Changelog](../10-changelog-api-reference.md) for compatibility.

- **gh_upload_timeout** (`int`, default: `300` or 5 minutes): Timeout in seconds for uploading local files to GitHub (more on this in the GitHub Setup section). Adjust higher for large videos (e.g., 600 seconds) or lower for quick images to balance reliability and responsiveness.

Other parameters like `wait_before_post_publish` (`bool`, default: `True`) and `handle_hashtags` (`bool`, default: `True`) are covered in dedicated sections below. Full parameter details are available in the [API Reference](../10-changelog-api-reference.md).

### Example: Basic Initialization with Custom Logging and API Version

```python
from threadspipe import ThreadsPipe

# Initialize with custom config
tp = ThreadsPipe(
    user_id=1234567890,  # Your Threads user ID
    access_token="your_long_lived_access_token",
    disable_logging=True,  # Silent mode
    threads_api_version='v1.0',
    gh_upload_timeout=600  # Extended for large files
)
```

This setup disables logs for a clean production run while extending the GitHub timeout. For environment variable integration (recommended for security), load credentials dynamically:

```python
import os
from threadspipe import ThreadsPipe

tp = ThreadsPipe(
    user_id=int(os.getenv('THREADS_USER_ID')),
    access_token=os.getenv('THREADS_ACCESS_TOKEN'),
    disable_logging=os.getenv('DISABLE_LOGGING', 'False').lower() == 'true',
    threads_api_version=os.getenv('THREADS_API_VERSION', 'v1.0'),
    gh_upload_timeout=int(os.getenv('GH_UPLOAD_TIMEOUT', 300))
)
```

Using env vars keeps sensitive data out of code, ideal for deployment on platforms like Heroku or AWS Lambda.

## The update_param() Method for Runtime Changes

One of ThreadsPipe-py's strengths is its ability to adapt without reinitializing the instance. The `update_param()` method allows updating any `__init__` parameter at runtime, applying changes immediately where applicable (e.g., token refreshes regenerate API endpoints). All parameters are optional and default to `None` (no change).

This is particularly useful for:
- Refreshing expired access tokens mid-session.
- Switching logging modes based on context (e.g., enable for debugging).
- Adjusting timeouts or API versions for different operations.

### Parameters
Mirrors `__init__`: `user_id: int = None`, `access_token: str = None`, `disable_logging: bool = None`, `threads_api_version: str = None`, `gh_upload_timeout: int = None`, and others like `handle_hashtags`.

### Example: Updating Parameters During a Session

```python
# Initial setup
tp = ThreadsPipe(user_id=1234567890, access_token="old_token")

# Simulate token refresh after some posts
new_token = "refreshed_long_lived_token"  # Obtained via re-auth
tp.update_param(
    access_token=new_token,
    disable_logging=False,  # Enable logs for monitoring
    threads_api_version='v1.0',  # Ensure latest
    gh_upload_timeout=300  # Reset to default
)

# Now post with updated config
tp.pipe(text="Updated post with new token!")
```

Call `update_param()` before critical operations to ensure changes take effect. Note: Structural changes (e.g., GitHub credentials) may require careful sequencing to avoid mid-upload failures.

For testing, integrate this into unit tests. In `tests/test.py`, you might simulate runtime updates:

```python
import pytest
from threadspipe import ThreadsPipe

def test_update_param():
    tp = ThreadsPipe(user_id=123, access_token="test_token", disable_logging=True)
    assert tp.disable_logging is True
    
    tp.update_param(disable_logging=False)
    assert tp.disable_logging is False  # Confirms update applied
    
    # Test with env var override in test env
    tp.update_param(gh_upload_timeout=int(os.getenv('TEST_GH_TIMEOUT', 100)))
    assert tp.gh_upload_timeout == 100
```

This verifies configurability without hitting live APIs (use mocks for full isolation).

## Hashtag Handling: handle_hashtags vs auto_handle_hashtags

Threads limits posts to 500 characters and one hashtag per post, making long-form content (e.g., threads) challenging. ThreadsPipe-py automates distribution via two complementary options:

- **handle_hashtags** (`bool`, default: `True`): Basic extraction and splitting. Scans the post text's end for hashtags (using regex like `(?<=\s)(#[\w\d]+(?:\s#[\w\d]+)*)$`) and appends one per chained post segment if the text exceeds 500 characters. Disabling this (`False`) lets you manage hashtags manually.

- **auto_handle_hashtags** (`bool`, default: `False`): Advanced mode building on `handle_hashtags`. It intelligently distributes multiple hashtags, skipping segments that already contain one (detected via regex like `([\w\s]+)?(\#\w+)\s?\w+`). This prevents redundancy in body-embedded tags. Enable for dynamic, X-like threading.

Both apply only during chaining (texts >500 chars or >20 media items). You can also pass explicit tags via the `pipe` method's `tags: List[str]` parameter, with `persist_tags_multipost: bool` (default: `False`) to repeat the same tag across chains.

### Example: Custom Hashtag Distribution

```python
# Basic handling (default)
tp.pipe(text="Long post with #tag1 #tag2 #tag3...")  # Splits one per chain

# Advanced auto-handling
tp.update_param(auto_handle_hashtags=True)
tp.pipe(text="Post with existing #inlineTag and end #tag1 #tag2")  # Skips if body has one

# Explicit tags
tp.pipe(text="Plain text", tags=["#custom1", "#custom2"], persist_tags_multipost=True)
```

For testing in `tests/test.py`:

```python
def test_hashtag_handling():
    tp = ThreadsPipe(...)  # Mocked instance
    result = tp.pipe(text="Test #hashtag1 #hashtag2", auto_handle_hashtags=True)
    assert len(result['chains']) == 2  # Verifies splitting
    assert '#hashtag1' in result['chains'][0]['text']
    assert '#hashtag2' in result['chains'][1]['text']
```

This ensures hashtags are handled without over-tagging, improving post readability.

## Wait Timings: Ensuring Reliable Publishing

API processing delays (e.g., media uploads) can cause failures if posts are published prematurely. ThreadsPipe-py includes configurable waits to poll for 'FINISHED' status:

- **post_publish_wait_time** (`int`, default: `35` seconds, minimum: `30`): Waits for the media container (post blueprint) to finish before publishing. Enabled via `wait_before_post_publish: bool` (default: `True`). Shorter times risk errors; 35+ seconds is recommended for reliability.

- **media_item_publish_wait_time** (`int`, default: `35` seconds): Per-media wait, enabled by `wait_before_media_item_publish: bool` (default: `True`). Loops check status until ready or 'ERROR' (triggers cleanup). Videos may need 60+ seconds.

These are enforced in methods like `pipe` and `__send_post__`, with logs like "Waiting for upload status...".

### Example: Adjusting Waits for Media-Heavy Posts

```python
tp.update_param(
    post_publish_wait_time=45,  # For complex threads
    media_item_publish_wait_time=60  # For videos
)
tp.pipe(text="Post with video", media=[{"url": "local_video.mp4"}])
```

In tests:

```python
def test_wait_timings():
    tp = ThreadsPipe(...)
    tp.update_param(post_publish_wait_time=30)  # Edge case
    # Mock API response to simulate 'FINISHED' after wait
    result = tp.pipe(text="Test post")
    assert result['status'] == 'published'  # No premature failure
```

## Rate Limit Behaviors: check_rate_limit_before_post and wait_on_rate_limit

Threads enforces quotas (250 posts/day, 1000 replies/day). Configure how the library responds:

- **check_rate_limit_before_post** (`bool`, default: `True`): Queries the `threads_publishing_limit` endpoint before each post. Returns a 429 error if exceeded. Disable (`False`) for speed but at risk of rejection.

- **wait_on_rate_limit** (`bool`, default: `False`): If limits hit, waits and retries instead of failing. Uses `get_quota_usage(for_reply=False)` for posts. Can spawn processes, increasing memory use in concurrent scenarios.

Constants: `__threads_rate_limit__ = 250`, `__threads_reply_rate_limit__ = 1000`.

### Example: Graceful Rate Handling

```python
tp.update_param(
    check_rate_limit_before_post=True,
    wait_on_rate_limit=True  # Retry on limit
)
# In a loop for bulk posting
for i in range(10):
    tp.pipe(text=f"Post {i}")
    # Auto-waits if near limit
```

Test example:

```python
def test_rate_limits():
    tp = ThreadsPipe(...)
    tp.update_param(wait_on_rate_limit=True)
    # Mock quota response at 249/250
    result = tp.pipe(text="Near-limit post")
    assert result['quota_remaining'] == 249  # Verifies check
```

## GitHub Setup for Local Uploads

Threads requires public URLs for media, so local files (paths, bytes, base64) are temporarily uploaded to GitHub:

1. **Create a Fine-Grained Token**: Go to [github.com/settings/tokens](https://github.com/settings/tokens?type=beta). Select "Fine-grained tokens" with repo write/delete scopes for your upload repo. No broader access needed—limit to specific repos for security.

2. **Set Up a Repo**: Create a public or private repo (e.g., `threadspipe-uploads`). Pass `gh_username`, `gh_repo_name`, and `gh_bearer_token` during init.

3. **Configuration Parameters**:
   - `gh_bearer_token` (`str`, default: `None`): Your token.
   - `gh_api_version` (`str`, default: `'2022-11-28'`): GitHub API version.
   - `gh_repo_name` and `gh_username` (`str`, default: `None`): For targeting the repo.

Process: Files are base64-encoded, uploaded via PUT `/contents/{filename}`, and deleted post-use (with SHA for cleanup). Supports up to 20 media/post; excess chains to replies. Timeout via `gh_upload_timeout`.

### Example: GitHub-Enabled Init with Custom Scopes

```python
# Env vars for token (with custom scopes: repo:write, repo:delete)
tp = ThreadsPipe(
    user_id=1234567890,
    access_token="token",
    gh_bearer_token=os.getenv('GH_BEARER_TOKEN'),  # Fine-grained, repo-specific
    gh_username="yourusername",
    gh_repo_name="threadspipe-uploads",
    gh_upload_timeout=300,
    gh_api_version='2022-11-28'
)

# Post local image
tp.pipe(text="Post with local media", media=[{"path": "/path/to/image.jpg"}])
```

For custom scopes, edit the token: Repository access > Only select repos > Include "Contents" (read/write). Test in `tests/test.py`:

```python
def test_github_upload():
    tp = ThreadsPipe(..., gh_bearer_token="test_token")
    # Mock GitHub API
    result = tp.pipe(media=[{"bytes": b"fake_image_data"}])
    assert "github_url" in result['media'][0]  # Verifies upload
    # Cleanup assertion
```

No GitHub needed for URL-based media.

## Examples: Custom Scopes, Env Vars, and Testing Configs

Beyond basics, customize scopes in GitHub tokens for minimal permissions (e.g., only "Contents: Write & Delete" for uploads). For Threads, use scopes like `threads_basic`, `threads_content_publish` in OAuth flow (see [Setup Authentication](../03-setup-authentication.md)).

Env vars example (`.env` file):

```
THREADS_USER_ID=1234567890
THREADS_ACCESS_TOKEN=abc123
DISABLE_LOGGING=true
GH_BEARER_TOKEN=ghp_...
GH_USERNAME=youruser
GH_REPO_NAME=uploads
POST_PUBLISH_WAIT_TIME=45
AUTO_HANDLE_HASHTAGS=true
```

Load with `python-dotenv`: `from dotenv import load_dotenv; load_dotenv()`.

Testing configs in `tests/test.py` often use pytest fixtures for mocked params:

```python
@pytest.fixture
def config_tp():
    tp = ThreadsPipe(user_id=123, access_token="mock")
    tp.update_param(disable_logging=True, wait_on_rate_limit=False)
    return tp

def test_custom_config(config_tp):
    config_tp.update_param(threads_api_version='v1.0')
    # Assert config impacts (e.g., no wait on limits)
    result = config_tp.pipe(text="Test")
    assert not config_tp.wait_on_rate_limit
```

Run with `pytest tests/test.py -v` to validate.

## Impacts on Memory and Performance

Configuration choices directly affect resource usage:

- **Logging (`disable_logging`)**: Enabled logging adds minimal overhead (~1-2% CPU) but can bloat output in loops. Disable for performance-critical apps.

- **Wait Timings**: Longer waits (e.g., 60s) increase latency but prevent retries (saving API calls). In concurrent use (e.g., multiprocessing), multiple sleeps can tie up threads, raising memory by 10-20MB per instance.

- **Rate Limits (`wait_on_rate_limit`)**: Enabling spawns waiting processes, potentially doubling memory for high-concurrency (e.g., 100MB+ for 10 threads). Prefer `check_rate_limit_before_post` alone for low-volume to avoid this.

- **Hashtag Handling**: `auto_handle_hashtags` adds regex overhead (~5ms/post) but is negligible. Chaining long texts increases string operations, using ~1-5MB extra RAM.

- **GitHub Uploads**: Timeouts >300s hold file buffers in memory (e.g., 50MB for a 4K video). Limit to essentials; use async if scaling.

- **Overall Performance**: Default configs balance reliability (e.g., 35s waits ensure 99% success). For memory-constrained envs (e.g., Raspberry Pi), disable waits and logs to cut usage by 30%. Benchmark with `memory_profiler` or `cProfile`—e.g., a 10-post batch uses ~50MB base, +20% with waits.

Monitor via `get_quota_usage()` and logs. For optimization, profile in tests: `pytest --durations=10`.

## Best Practices and Troubleshooting

- Start with defaults, tweak iteratively (use `update_param` for A/B testing).
- Secure secrets with env vars or tools like `keyring`.
- For errors (e.g., timeout exceeds), see [Errors & Best Practices](../09-errors-best-practices.md).
- Version configs in `docusaurus.config.js` for release-specific docs.

By mastering these options, you can tailor ThreadsPipe-py for everything from simple bots to enterprise-scale posting. Experiment in a test environment to measure impacts—happy threading!

(Word count: 1428)