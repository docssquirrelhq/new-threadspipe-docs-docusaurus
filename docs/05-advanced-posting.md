---
title: Advanced Posting Features
slug: advanced-posting
description: Dive deep into ThreadsPipe-py's advanced posting capabilities, including geo-gating, quoting, hashtags, link attachments, file uploads, splitting logic, captions, wait times, and more. Learn to handle carousels, long threads, and edge cases with practical examples.
sidebar_label: Advanced Posting
sidebar_position: 5
---

# Advanced Posting Features

ThreadsPipe-py is a powerful Python library for interacting with the Meta Threads API, offering not just basic posting but a suite of advanced features designed to handle complex scenarios like geo-restricted content, threaded conversations, media carousels, and intelligent content splitting. These features are particularly useful for developers building social media automation tools, content management systems, or bots that need to post long-form content, multimedia threads, or targeted posts while respecting API limitations such as 500-character text limits per post, up to 20 media items per post, and daily quotas.

In this guide, we'll explore the advanced posting functionalities in depth, drawing from the core implementation in `threadspipe.py`. We'll cover geo-gating for country-specific visibility, quoting and reposting existing posts, sophisticated hashtag handling, link attachments for text-only posts, local file uploads via temporary GitHub repositories, content splitting logic for long threads and media, captions indexing, and configurable wait times to manage rate limits and processing delays. Examples will include carousels with multiple media, long threads exceeding character limits, and edge cases like rate limit handling. We'll also reference relevant checks from `tests/test.py` for validation, such as splitting algorithms and base64 media encoding.

By the end of this page, you'll be equipped to leverage these features for robust, production-ready Threads posting workflows. For foundational concepts, refer to the [Basic Posting](/docs/04-basic-posting) guide. If you're new to setup, start with [Installation](/docs/02-installation) and [Setup & Authentication](/docs/03-setup-authentication).

## Geo-Gating: Restricting Posts by Country

Geo-gating allows you to limit post visibility to specific countries, enhancing privacy and compliance for global audiences. This feature relies on the Threads API's `allowlisted_country_codes` parameter, which accepts ISO 3166-1 alpha-2 country codes (e.g., "US" for United States, "CA" for Canada). However, not all accounts are eligible—eligibility requires the `threads_manage_replies` permission and is verified via the user's profile.

### Eligibility Check: `is_eligible_for_geo_gating`

Before applying geo-gating, ThreadsPipe-py performs an eligibility check using the `is_eligible_for_geo_gating` method. This queries the Threads Graph API endpoint:

```
GET https://graph.threads.net/{api_version}/me?fields=id,is_eligible_for_geo_gating&access_token={access_token}
```

The response is a JSON object like:

```json
{
  "id": "your_user_id",
  "is_eligible_for_geo_gating": true
}
```

If ineligible (or an error occurs), posting with geo-gating will fail gracefully. In the `pipe` method, this is enforced as follows:

```python
if allowed_country_codes is not None:
    is_eligible = self.is_eligible_for_geo_gating()
    if 'error' in is_eligible or not is_eligible.get('is_eligible_for_geo_gating', False):
        return self.__tp_response_msg__(
            message="You are attempting to send a geo-gated content but you are not eligible for geo-gating",
            body=is_eligible,
            is_error=True
        )
```

To use this, initialize your `ThreadsPipe` instance with a valid access token that includes the necessary scopes (e.g., `threads_basic`, `threads_manage_replies`). You can fetch supported country codes via `get_allowlisted_country_codes`, which paginates through:

```
GET https://graph.threads.net/{api_version}/me/threads?fields=allowlisted_country_codes&limit=100
```

This returns a list of eligible codes for your account.

### Applying Geo-Gating in Posts

The `allowed_country_codes` parameter in the `pipe` method accepts either a comma-separated string (e.g., "US,CA,NG") or a list of strings (e.g., `["US", "CA", "NG"]`). Internally, it's normalized to a comma-separated string and appended to the post creation and publishing endpoints:

```
&allowlisted_country_codes=US,CA,NG
```

For threaded posts (chained content), geo-gating applies uniformly to all segments. Example usage:

```python
from threadspipe import ThreadsPipe

api = ThreadsPipe(
    access_token="your_access_token",
    threads_api_version="v1.0"
)

response = api.pipe(
    post="This post is only visible in select countries! 🌍",
    allowed_country_codes=["US", "CA"],
    chained_post=True
)

print(response)  # {'info': 'success', 'id': 'post_id', ...}
```

If ineligible, the response will include an error message and the eligibility check body. Edge case: Mixing geo-gating with replies—use `for_reply=True` in quota checks to ensure compliance. In `tests/test.py`, eligibility is mocked for unit tests, simulating both eligible and ineligible scenarios to verify error handling.

Geo-gating is ideal for region-specific campaigns but adds a small latency due to the pre-check. Always test eligibility in development to avoid runtime surprises.

(Word count so far: ~450)

## Quoting and Reposting Posts

Quoting allows you to reference an existing Threads post in your new content, creating a linked conversation. This is powered by the `quote_post_id` parameter, which attaches the quoted post as a contextual element. ThreadsPipe-py also supports full reposts via a dedicated method.

### Quoting with `quote_post_id` and Persistence

The `quote_post_id` parameter accepts a string or integer representing the Threads post ID (e.g., "1234567890111213"). It's appended to the creation endpoint:

```
&quote_post_id=1234567890111213
```

For threaded posts, the `persist_quoted_post` flag (default: `False`) controls propagation:

- If `False`, the quote attaches only to the root (first) post.
- If `True`, it attaches to every chained segment, ensuring the reference persists across the thread.

Implementation in the chaining loop of `pipe`:

```python
_quote_post_id = quote_post_id
for index, s_post in enumerate(splitted_post):
    # Prepare post data with _quote_post_id
    # ...
    if not persist_quoted_post:
        _quote_post_id = None  # Reset for subsequent chains
```

This prevents redundant quoting in long threads while allowing full-thread references when needed. Example for a quoted thread:

```python
response = api.pipe(
    post="Building on this great discussion...",
    quote_post_id="1234567890111213",
    persist_quoted_post=True,  # Quote every segment
    files=["https://example.com/image.jpg"]  # Optional media
)
```

### Reposting Functionality

For direct reposts (sharing without new content), use the `repost_post` method:

```python
response = api.repost_post(post_id="1234567890111213")
```

This hits:

```
POST https://graph.threads.net/v1.0/{post_id}/repost?access_token={access_token}
```

Returns the repost ID on success or an error dict. Reposts count toward daily quotas (250/day for posts, 1000/day for replies). In edge cases, like quoting a non-existent ID, the API returns a 404—ThreadsPipe-py wraps this in its unified error response.

From `tests/test.py`, quoting is tested with mock API responses, verifying ID validation and persistence logic across splits. This ensures quotes don't break during content truncation.

Quoting enhances engagement by linking conversations, making it a staple for community-driven apps.

(Word count so far: ~750)

## Hashtags Handling: Extraction, Splitting, and Persistence

Hashtags in Threads are limited to one per post, but users often include multiple (e.g., "#Python #Threads #API"). ThreadsPipe-py automates extraction, distribution, and splitting to maximize visibility without manual intervention.

### Configuration Parameters

In the `ThreadsPipe` initializer or `update_param` method:

- `handle_hashtags: bool = True`: Enables basic extraction and appending of hashtags to post ends.
- `auto_handle_hashtags: bool = False`: Advanced mode—skips processing if the post already contains hashtags (detected via regex).

In `pipe`:

- `tags: Optional[List[str]] = []`: Explicit hashtags (e.g., `["#Python", "#Automation"]`), overriding extraction.
- `persist_tags_multipost: bool = False`: Repeats the full tags list across all chained posts (useful for single tags).

### Extraction and Splitting Logic

Hashtags are extracted using regex: `r"(?<=\s)(#[\w\d]+(?:\s#[\w\d]+)*)$"`, targeting end-of-post tags. The `__should_handle_hash_tags__` method decides processing:

```python
def __should_handle_hash_tags__(self, post: Union[None, str]) -> bool:
    if post is None:
        return False
    if self.__auto_handle_hashtags__:
        # Check if post already has hashtags
        return len(re.compile(r"([\w\s]+)?(\#\w+)\s?\w+").findall(post)) == 0
    return self.__handle_hashtags__
```

For splitting, `__split_post__` (detailed later) integrates tags:

- For short posts (<=500 chars): Appends the first tag; if overflow, truncates text and chains.
- For long posts: Distributes tags across chains (one per segment, up to chain count). Uses `math.ceil(len(post) / 500)` for segments.
- Overflow: Adds "..." connectors (e.g., "Text chunk... #Tag1" → "Continued... #Tag2").
- Persist mode: Repeats tags (e.g., "#Python" on every chain).
- Auto mode: Skips if a segment already has a tag.

In `pipe`, tags are stripped from the original post before splitting:

```python
if tags:
    _post = _post.rstrip()  # Remove trailing tags
splitted_post = self.__split_post__(_post, tags)
```

Example for a post with multiple tags:

```python
post = "Exciting updates on ThreadsPipe! #Python #API #Automation"
response = api.pipe(
    post=post,
    handle_hashtags=True,
    auto_handle_hashtags=True,
    persist_tags_multipost=False
)
# Result: Chains like "Exciting updates on ThreadsPipe! #Python", "... #API", "... #Automation"
```

Changelog notes fixes for tag stripping (v0.4.1–0.4.4), ensuring no double-tags in splits. In `tests/test.py`, regex-based extraction is unit-tested with cases like embedded vs. trailing hashtags, validating split outputs for accuracy.

This logic prevents hashtag loss in long content, boosting discoverability.

(Word count so far: ~1100)

## Link Attachments for Text-Only Posts

For purely textual posts, ThreadsPipe-py supports attaching external links as rich previews, limited to one per post by the API.

### Parameter and Logic

- `link_attachments: List[str] = []` in `pipe`: URLs like `["https://example.com/article"]`.

Logic in `__send_post__`:

- Applies only if no media (`len(files) == 0`).
- Distributes across chains: First link to first segment, cycles if more links than chains.
- Appended as `&attached_link={url}`.

Example:

```python
response = api.pipe(
    post="Check out this amazing resource!",
    link_attachments=["https://docs.threads.net"],
    chained_post=True  # For long text with link on root
)
```

Excess links are ignored in chains to avoid API errors. No media? Links enhance text posts with previews. Edge case: Invalid URLs trigger validation failures—use `tests/test.py`'s URL regex checks for local testing.

(Word count so far: ~1200)

## Local File Uploads via GitHub Temporary Repo

The Threads API requires public URLs for media, so ThreadsPipe-py uses GitHub as a temporary host for local files, bytes, or base64 data. This is configured via init parameters and handles uploads securely.

### Configuration

In `__init__` or `update_param`:

- `gh_bearer_token: str = None`: GitHub fine-grained token (scopes: contents:write, repo).
- `gh_repo_name: str = None`: Public repo for temp files (create one manually).
- `gh_username: str = None`: Your GitHub username.
- `gh_api_version: str = '2022-11-28'`: GitHub API version.
- `gh_upload_timeout: int = 300`: Timeout in seconds.

### Upload Logic in `__handle_media__`

Supports inputs: URLs, local paths (e.g., "/path/to/img.jpg"), bytes (e.g., `open('file', 'rb').read()`), base64 strings.

- URL validation: Regex `__file_url_reg__` (matches HTTP/HTTPS, queries like "?q=80").
- Base64 check (`__is_base64__`):

```python
def __is_base64__(self, o_str: str) -> bool:
    if not o_str or len(o_str) % 4 != 0:
        return False
    base64_regex = re.compile(r'^[A-Za-z0-9+/]*={0,2}$')
    if not base64_regex.match(o_str):
        return False
    try:
        base64.b64decode(o_str, validate=True)
        return True
    except (base64.binascii.Error, ValueError):
        return False
```

For non-URLs: Generates random filename (via `filetype` lib for extension), base64-encodes content, PUTs to:

```
PUT /repos/{username}/{repo}/contents/{filename}
```

Headers: `Authorization: Bearer {token}`, body: `{"message": "ThreadsPipe upload...", "content": base64_content}`.

Returns: `{'url': download_url, 'sha': sha, '_link': api_link}`. Tracks in `self.__handled_media__` for cleanup.

Post-posting, `__delete_uploaded_files__` DELETEs via `{_link}?sha={sha}` with message. Failures log warnings.

Example:

```python
files = [
    "/local/path/to/image.jpg",
    open("video.mp4", "rb").read(),
    "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEAYABgAAD..."  # Base64
]
response = api.pipe(
    post="Uploading local media!",
    files=files,
    gh_bearer_token="ghp_your_token",
    gh_repo_name="temp-threads-media",
    gh_username="your_username"
)
```

In `tests/test.py`, base64 validation and upload mocks test edge cases like invalid encodings or timeouts, ensuring robust handling. Cleanup runs on success/error, preventing repo clutter.

This feature bridges local development with API requirements seamlessly.

(Word count so far: ~1550)

## Splitting Logic: Handling Long Threads and Media

Long content exceeds API limits, so ThreadsPipe-py splits text and media into chained posts (replies to the root).

### Text Splitting: `__split_post__`

- Chunks text into ~500-char segments using `math.ceil(len(post) / 500)`.
- Adds "..." connectors for continuity.
- Integrates tags (as above): Distributes one per chain, truncates if needed.

Pseudocode:

```python
def __split_post__(self, post: str, tags: List[str]) -> List[str]:
    if len(post) <= 500:
        # Append tag or handle overflow
        return [post + " " + tags[0] if tags else post]
    # Split logic: chunk, add "...", distribute tags
    chunks = []
    for i, chunk in enumerate(text_chunks(post, 500)):
        tag = tags[i] if i < len(tags) else ""
        connector = "..." if i > 0 else ""
        chunks.append(f"{connector}{chunk} {tag}".strip())
    return chunks
```

### Media Splitting: `__handle_media__`

- Batches into 20s: `splitted_files = [files[20*i:20*(i+1)] for i in range(math.ceil(len(files)/20))]`.
- Aligns with text splits: First batch to root, others to replies.
- Supports carousels (below).

In `pipe`, if `chained_post=False` and content overflows, raises `Exception("Content exceeds limits; enable chaining")`.

Example for long thread:

```python
long_post = "This is a very long post that needs splitting... " * 10  # >500 chars
response = api.pipe(
    post=long_post,
    tags=["#LongThread", "#ThreadsPipe"],
    chained_post=True
)
# Creates root + reply chain with distributed tags
```

From `tests/test.py`, splitting is rigorously tested: text chunks, tag distribution, and overflow with "..."—ensuring no char limit violations.

### Carousel Support (>1 Media)

For multi-media posts, `__send_post__` creates child containers:

- Images: `&is_carousel_item=true&image_url={url}&alt_text={caption}`.
- Videos: Similar with `video_url`.
- Parent: `&media_type=CAROUSEL&children={child_ids}`.

Up to 20 items; excess splits into chains. Example carousel:

```python
files = ["img1.jpg", "img2.jpg", "img3.jpg"]  # Local or URLs
captions = [None, "Caption for img2", None]
response = api.pipe(
    post="My photo carousel!",
    files=files,
    file_captions=captions,
    is_carousel=True  # Inferred from len(files) > 1
)
```

Handles mixed media (images + videos). Edge case: 21+ files → root carousel (20) + reply post (1).

(Word count so far: ~1850)

## Captions Indexing

Captions provide alt text for accessibility. `file_captions: List[Union[str, None]]` in `pipe` aligns by index.

- Pads short lists with `None`.
- Splits: `[_captions[20*i:20*(i+1)] for i in ...]`.
- In carousels: Per-child `&alt_text={caption}`.

Example:

```python
file_captions = [None, "First image desc", None, "Third image", None]  # For 5 files
# Applies to indices 0:None, 1:"First...", etc.
```

`tests/test.py` validates indexing with mismatched lengths, ensuring no index errors.

## Wait Times and Rate Limits

### Wait Times

To ensure media processing, configurable waits poll status:

- `wait_before_post_publish: bool = True`, `post_publish_wait_time: int = 35`: Polls container until "FINISHED" (min 30s for videos).
- `wait_before_media_item_publish: bool = True`, `media_item_publish_wait_time: int = 35`: For individual items.

In `__get_uploaded_post_status__`:

```python
while status == "IN_PROGRESS":
    time.sleep(wait_time)
    # Poll endpoint
if status == "ERROR":
    raise Exception(error_message)
```

### Rate Limits

- `wait_on_rate_limit: bool = False`, `check_rate_limit_before_post: bool = True`.
- Queries `get_quota_usage`: `https://graph.threads.net/{version}/{user_id}/threads_publishing_limit?fields=quota_usage,config`.
- Limits: 250 posts/day, 1000 replies/day.
- If exceeded and `wait_on_rate_limit=True`, sleeps (note: spawns processes, monitor memory).
- `for_reply=True` for reply-specific checks.

Example edge case handling:

```python
response = api.pipe(
    post="Rate limit test",
    wait_on_rate_limit=True,  # Wait instead of fail
    check_rate_limit_before_post=True
)
```

In long threads (e.g., 100+ segments), intersperses waits to avoid bursts. `tests/test.py` mocks quota responses for limit-hit scenarios, testing wait logic.

For best practices, monitor quotas via [Errors & Best Practices](/docs/09-errors-best-practices). These features make ThreadsPipe-py resilient for high-volume posting.

(Word count: ~2100)

For more on API details, see [Changelog & API Reference](/docs/10-changelog-api-reference). Contribute tests or features via GitHub!

---