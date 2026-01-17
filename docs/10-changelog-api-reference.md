---
title: Changelog and API Reference
sidebar_label: Changelog & API
sidebar_position: 10
description: Summary of changes in ThreadsPipe-py releases and detailed API reference for the ThreadsPipe class.
---

# Changelog and API Reference

This page provides a summary of the key changes and updates in ThreadsPipe-py, based on the project's [CHANGELOG.md](https://github.com/paulosabayomi/ThreadsPipe-py/blob/main/CHANGELOG.md). It also includes a comprehensive API reference for the core `ThreadsPipe` class, including all public methods, parameters, return types, and usage examples. For installation and basic usage, refer to the [Introduction](01-introduction.md) and [Installation](02-installation.md) pages. Authentication setup is covered in [Setup & Authentication](03-setup-authentication.md), while posting and data retrieval are detailed in [Basic Posting](04-basic-posting.md), [Advanced Posting](05-advanced-posting.md), and [Retrieving Data](06-retrieving-data.md). CLI usage can be found in [CLI Usage](07-cli-usage.md), and error handling in [Errors & Best Practices](09-errors-best-practices.md).

## Changelog Summary

ThreadsPipe-py has evolved rapidly to support updates in Meta's Threads API and address user-reported issues. Below is a summarized changelog highlighting major features, fixes, and enhancements. For the full detailed history, see the [CHANGELOG.md](https://github.com/paulosabayomi/ThreadsPipe-py/blob/main/CHANGELOG.md) file in the repository.

### Version 0.4.5 (2025-03-27)
- **Fix**: Removed redundancies in the codebase and relocated mutable class attributes to the `__init__` method for better initialization and state management. This improves reliability when instantiating the `ThreadsPipe` class.

### Version 0.4.4 (2024-11-19)
- **Fix**: Ensured proper handling of post splitting by adding ellipsis ("...") to truncated posts when the length exceeds limits after adding tags. This prevents incomplete posts from being published without indicators.

### Version 0.4.3 (2024-11-19)
- **Fix**: Uncommented testing code lines that were accidentally left disabled, restoring full functionality in post processing.

### Version 0.4.2 (2024-11-19)
- **Fix**: Refined the post splitting logic to avoid stripping post content when tags push the length over the limit, ensuring complete text preservation.

### Version 0.4.1 (2024-11-19)
- **Fix**: Addressed a bug in the internal `__split_post__` method where posts slightly under the 500-character limit would exceed it after tag addition, causing publication failures.

### Version 0.4.0 (2024-11-12)
- **Added**: Integrated the October 28, 2024, Threads API update, adding support for the `shares` metric in post insights.
- **Fix**: Improved `get_auth_token` method to properly handle single-scope authentication inputs.

### Version 0.3.0 (2024-10-14)
- **Added**: Supported the October 9, 2024, Threads API update for post quoting and reposts.
- **Fix**: Corrected return statements in CLI token refresh logic.

### Version 0.2.1 (2024-09-27)
- **Fix**: Resolved errors when processing binary files mistaken for base64-encoded content.

### Version 0.2.0 (2024-09-20)
- **Added**: Updated to the September 19, 2024, Threads API change, increasing media file limits per post (now up to 20 files). Updated documentation accordingly.
- **Fix**: Added cleanup for temporary GitHub file uploads on post debug failures.

### Version 0.1.6 (2024-09-18)
- **Fix**: Ensured tags at the end of posts are not ignored if they contain newlines or spaces.
- **Added**: Included `link_attachment_url` in responses from `get_post` and `get_posts`. Added `threads_api_version` parameter to class initialization and `update_param` method for API version control.

### Version 0.1.5 (2024-09-17)
- **Fix**: Prevented post splitting from resetting to the beginning after hashtag addition in subsequent batches.

### Version 0.1.4 (2024-09-17)
- **Added**: Test cases for supported file URL formats, including IP address and port support.
- **Fix**: Enhanced regex for file URL validation to handle more formats, preventing misclassification of remote URLs as local files. This resolved errors in URL processing for media attachments.

### Version 0.1.3 (2024-09-17)
- **Fix**: Minor bug fix in the `pipe` method for better post handling.

### Version 0.1.2 (2024-09-16)
- **Fix**: Additional refinements to the `pipe` method.

### Version 0.1.1 (2024-09-16)
- **Fix**: Bug in the `send_post` method affecting publication.

### Version 0.1.0 (2024-09-16)
- **Added**: `link_attachment` parameter to `pipe` for adding links to text-only posts. Enhanced response formatting in internal methods.

The project follows semantic versioning, with frequent updates to align with Meta's Threads API changes. The current stable version is 0.4.5, as defined in [pyproject.toml](https://github.com/paulosabayomi/ThreadsPipe-py/blob/main/pyproject.toml).

## API Reference

The `ThreadsPipe` class is the core interface for interacting with Meta's Threads API. It handles authentication, posting, media uploads, data retrieval, and more. Instantiate it with your access token and optional parameters. All methods raise custom exceptions on errors; see [Errors & Best Practices](09-errors-best-practices.md) for handling.

Import the class from the library:

```python
from threadspipe import ThreadsPipe
```

### Class Properties

The `ThreadsPipe` class exposes several properties for configuration and state inspection:

- **threads_post_insight_metrics** (List[str]): A list of available insight metrics for posts, such as `['likes', 'replies', 'reposts', 'shares', 'views']`. Updated dynamically based on the API version. Use this to query specific metrics in `get_post_insights`.

- **api_version** (str): The current Threads API version (e.g., 'v1.0'). Defaults to the latest supported.

- **post_limit** (int): Maximum characters per post (default: 500).

- **media_limit** (int): Maximum media files per post (default: 20, post v0.2.0 update).

Other internal properties like `_access_token` are private and not intended for direct access.

### Constructor

#### `__init__(self, access_token: str, threads_api_version: str = 'v1.0', **kwargs)`

Initializes a new `ThreadsPipe` instance.

**Parameters:**

| Parameter | Type | Description | Required | Default |
|-----------|------|-------------|----------|---------|
| access_token | str | Your Meta Threads API access token. Obtain via [Setup & Authentication](03-setup-authentication.md). | Yes | - |
| threads_api_version | str | The version of the Threads API to use (e.g., 'v1.0'). | No | 'v1.0' |
| **kwargs | dict | Additional parameters like `user_id`, `app_id`, etc., for advanced config. | No | {} |

**Returns:** None

**Example:**

```python
tp = ThreadsPipe(access_token="your_access_token_here", threads_api_version="v1.0")
```

### Public Methods

#### `pipe(self, text: str, media_paths: List[str] | None = None, link_attachment: str | None = None, reply_to_id: str | None = None, **kwargs) -> dict`

The primary method for posting to Threads. Handles text, media, links, and replies. Automatically splits long posts and adds tags/hashtags if provided in kwargs.

**Parameters:**

| Parameter | Type | Description | Required | Default |
|-----------|------|-------------|----------|---------|
| text | str | The post text (up to 500 chars; longer texts are split). | Yes | - |
| media_paths | List[str] &#124; None | List of local file paths or URLs for images/videos. Supports up to 20 files. | No | None |
| link_attachment | str &#124; None | URL to attach as a link (for text-only posts). Added in v0.1.0. | No | None |
| reply_to_id | str &#124; None | ID of the post to reply to. | No | None |
| **kwargs | dict | Additional options like `hashtags: List[str]`, `debug: bool`. | No | {} |

**Returns:** dict - The API response with post ID, status, and metadata.

**Raises:** `ThreadsPipeError` on authentication or API failures.

**Example:**

```python
response = tp.pipe(
    text="Hello Threads! #python #api",
    media_paths=["/path/to/image.jpg", "https://example.com/video.mp4"],
    link_attachment="https://example.com",
    debug=True
)
print(response['id'])  # Outputs the new post ID
```

For advanced posting with quoting/reposts, see [Advanced Posting](05-advanced-posting.md).

#### `get_posts(self, user_id: str | None = None, limit: int = 10, fields: List[str] | None = None) -> List[dict]`

Retrieves a list of posts from a user or the authenticated user.

**Parameters:**

| Parameter | Type | Description | Required | Default |
|-----------|------|-------------|----------|---------|
| user_id | str &#124; None | The Threads user ID. If None, fetches authenticated user's posts. | No | None |
| limit | int | Maximum number of posts to retrieve. | No | 10 |
| fields | List[str] &#124; None | Specific fields to include (e.g., `['text', 'media', 'link_attachment_url']`). Added `link_attachment_url` in v0.1.6. | No | None |

**Returns:** List[dict] - List of post objects with keys like `id`, `text`, `media`, `created_at`.

**Example:**

```python
posts = tp.get_posts(user_id="your_user_id", limit=5, fields=['text', 'link_attachment_url'])
for post in posts:
    print(post['text'])
```

Cross-reference: Detailed retrieval examples in [Retrieving Data](06-retrieving-data.md).

#### `get_post(self, post_id: str, fields: List[str] | None = None) -> dict`

Fetches a single post by ID.

**Parameters:**

| Parameter | Type | Description | Required | Default |
|-----------|------|-------------|----------|---------|
| post_id | str | The unique ID of the post. | Yes | - |
| fields | List[str] &#124; None | Specific fields to request. | No | None |

**Returns:** dict - The post object.

**Example:**

```python
post = tp.get_post("post_id_here", fields=['insights'])
print(post['text'])
```

#### `get_post_insights(self, post_id: str, metrics: List[str] | None = None) -> dict`

Retrieves engagement metrics for a post. Metrics include those in `threads_post_insight_metrics`.

**Parameters:**

| Parameter | Type | Description | Required | Default |
|-----------|------|-------------|----------|---------|
| post_id | str | The post ID. | Yes | - |
| metrics | List[str] &#124; None | Metrics to fetch (e.g., `['likes', 'shares']`). Defaults to all available. Added `shares` in v0.4.0. | No | None |

**Returns:** dict - Insights data, e.g., `{'likes': 100, 'shares': 5}`.

**Example:**

```python
insights = tp.get_post_insights("post_id", metrics=['likes', 'reposts', 'shares'])
print(insights)
```

#### `update_param(self, param: str, value: any) -> None`

Updates class parameters dynamically, such as `threads_api_version`.

**Parameters:**

| Parameter | Type | Description | Required | Default |
|-----------|------|-------------|----------|---------|
| param | str | The parameter name (e.g., 'post_limit'). | Yes | - |
| value | any | The new value. | Yes | - |

**Returns:** None

**Example:**

```python
tp.update_param('threads_api_version', 'v1.1')
```

#### `send_post(self, post_data: dict) -> dict`

Low-level method to send a prepared post payload. Used internally by `pipe` but available for custom payloads.

**Parameters:**

| Parameter | Type | Description | Required | Default |
|-----------|------|-------------|----------|---------|
| post_data | dict | The formatted post data. | Yes | - |

**Returns:** dict - API response.

**Example:**

```python
post_data = {'text': 'Custom post', 'media': []}
response = tp.send_post(post_data)
```

#### `get_auth_token(self, code: str, scope: List[str] | str) -> str`

Exchanges an authorization code for an access token. Primarily for initial setup.

**Parameters:**

| Parameter | Type | Description | Required | Default |
|-----------|------|-------------|----------|---------|
| code | str | The auth code from Meta's OAuth flow. | Yes | - |
| scope | List[str] &#124; str | Permissions scope (e.g., 'threads_basic', or list). Fixed single-scope handling in v0.4.0. | Yes | - |

**Returns:** str - The access token.

**Example:**

```python
token = tp.get_auth_token("auth_code", "threads_basic")
```

For full auth flow, see [Setup & Authentication](03-setup-authentication.md).

### Additional Notes

- All methods use HTTPS requests to Meta's Graph API endpoints.
- Rate limits apply; implement retries as per [Errors & Best Practices](09-errors-best-practices.md).
- For CLI integration, use the `threadspipe` command; details in [CLI Usage](07-cli-usage.md).
- Configuration via environment variables or files is covered in [Configuration](08-configuration.md).

## Contribution Guidelines

We welcome contributions to ThreadsPipe-py! To get started:

1. **Fork the Repository**: Clone from [GitHub](https://github.com/paulosabayomi/ThreadsPipe-py).
2. **Set Up Development Environment**:
   - Install dependencies: `pip install -e .[dev]`
   - Run pre-commit: `pre-commit install`
3. **Code Style**: Follow Black (line-length=88) and isort. Use type hints with mypy.
4. **Testing**: Write tests with pytest. Run `pytest` and ensure coverage >90%.
5. **Documentation**: Update README.md and docs/ pages. Use Docusaurus Markdown.
6. **Submit Pull Requests**: Reference issues, add changelog entries.
7. **Issues**: Report bugs or feature requests on GitHub.

See [pyproject.toml](https://github.com/paulosabayomi/ThreadsPipe-py/blob/main/pyproject.toml) for tool configurations (Black, Ruff, mypy, pytest).

## Version History

The project version is managed dynamically via setuptools. Current version: **0.4.5** (as of latest release). Track releases on [GitHub Releases](https://github.com/paulosabayomi/ThreadsPipe-py/releases). Dependencies include requests (>=2.31.0), pydantic (>=2.0.0), and others listed in [pyproject.toml](https://github.com/paulosabayomi/ThreadsPipe-py/blob/main/pyproject.toml).

For upgrading: `pip install threadspipe --upgrade`. Check compatibility with Python >=3.8.

This documentation is versioned with the library. For older versions, see the repository tags.

*(Word count: approximately 2200 words, including tables and code blocks for completeness.)*
