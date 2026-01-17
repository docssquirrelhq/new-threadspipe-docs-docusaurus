---
title: Basic Posting and Replying
slug: basic-posting
description: Learn the fundamentals of posting text, media, and replies using the pipe() method in ThreadsPipe-py. Covers parameters, examples, auto-splitting, error handling, and rate limits.
sidebar_label: Basic Posting
sidebar_position: 4
tags: [posting, replying, pipe, media, threads]
---

# Basic Posting and Replying

Welcome to the basic guide on posting and replying with ThreadsPipe-py. This section focuses on the core functionality of the library: creating posts, attaching media, and building threaded conversations on Instagram Threads. Whether you're sharing simple text updates, replying to existing posts, or uploading images and videos, the `pipe()` method serves as your primary tool.

ThreadsPipe-py simplifies interactions with the Threads API by handling authentication, media uploads, content validation, and even automatic threading for long-form content. By the end of this guide, you'll understand how to use `pipe()` effectively, manage attachments from various sources (local files, URLs, bytes, or base64), handle replies, and navigate common pitfalls like rate limits and errors.

This page assumes you've completed the [setup and authentication steps](../03-setup-authentication.md). If you're new to the library, start with the [introduction](../01-introduction.md) for an overview.

We'll cover the `pipe()` method in detail, including its parameters, examples, auto-splitting for oversized content, error handling, and light internals. Step-by-step tutorials will walk you through practical scenarios. Expect around 1,800 words of in-depth explanation to get you posting confidently.

## The pipe() Method: Your Gateway to Posting

At the heart of ThreadsPipe-py is the `pipe()` method, defined in `threadspipe.py` within the `ThreadsPipe` class. This method abstracts the complexities of the Threads Graph API, allowing you to create posts or replies with minimal boilerplate. It supports text, media (images and videos), replies, reply restrictions, and more.

### Signature and Core Parameters

The `pipe()` method is flexible, accepting optional parameters for customization. Here's the relevant signature focusing on the key ones you specified:

```python
def pipe(
    self,
    post: str | "" = "",
    files: List[Any] | [] = [],
    file_captions: List[str | None] | [] = [],
    reply_to_id: str | None = None,
    who_can_reply: str | None = None,
    chained_post: bool = True,
    # Other params like tags, allowed_country_codes, etc., omitted for brevity
) -> dict | requests.Response
```

- **post** (`str | ""`, optional, defaults to empty string): The main text content of your post. This can be a short message or a long essay—up to 500 characters per post segment. If longer, and `chained_post=True` (default), it auto-splits into a thread. Whitespace is stripped, and hashtags can be auto-handled if enabled.

- **files** (`List[Any] | []`, optional, defaults to empty list): A list of media attachments. Supports up to 20 per post (excess splits into threads). File sources include:
  - Local paths (e.g., `"/path/to/image.jpg"`—requires GitHub setup for temporary uploads).
  - URLs (e.g., `"https://example.com/video.mp4"`).
  - Bytes (e.g., `open('file.jpg', 'rb').read()`).
  - Base64-encoded strings (handled similarly to bytes).

- **file_captions** (`List[str | None] | []`, optional): Alt-text descriptions for accessibility, matching the `files` list by index. Use `None` for no caption. The list length doesn't need to match exactly—unmatched files get no caption.

- **reply_to_id** (`str | None`, optional): The ID of the post you're replying to. Turns your content into a reply, using reply-specific quotas.

- **who_can_reply** (`str | None`, optional): Restricts who can reply to your post. Valid options from `ThreadsPipe.who_can_reply_list` include `"everyone"`, `"accounts_you_follow"`, or `"mentioned_only"`. Defaults to everyone if omitted.

Other parameters like `chained_post=True` enable threading, while class-level flags (e.g., `check_rate_limit_before_post=True`) control behaviors like quota checks.

The method returns a dictionary on success (with post IDs) or a `requests.Response` (or wrapped dict) on errors. More on outputs later.

### Why Use pipe()?

Unlike raw API calls, `pipe()` handles:
- Media normalization and uploads (e.g., local files to temporary GitHub storage).
- Content splitting for threads.
- Status polling (waits for media to process before publishing).
- Cleanup of temporary files.
- Validation and error raising for invalid inputs.

This makes it ideal for beginners while scalable for advanced use cases covered in [advanced posting](../05-advanced-posting.md).

## Handling Media Attachments

Media is a key feature of Threads, and `pipe()` supports diverse input types. All files are validated for MIME types (images: JPEG, PNG, etc.; videos: MP4, etc.) and processed into API-compatible formats.

### Supported Input Types

1. **Local File Paths**: Provide a string path to a file on your system. ThreadsPipe-py uploads it temporarily to a GitHub repository (requires `gh_bearer_token`, `gh_username`, and `gh_repo_name` in your `ThreadsPipe` instance). This is useful for private or local assets.

2. **URLs**: Direct links to publicly accessible files. No upload needed—Threads fetches them directly. Ensure URLs are stable and CORS-friendly.

3. **Bytes**: Raw binary data, e.g., from `open('file.jpg', 'rb').read()`. Uploaded to GitHub like local paths.

4. **Base64-Encoded Strings**: Strings like `base64.b64encode(open('file.jpg', 'rb').read()).decode()`. Treated as bytes for upload.

For videos, expect longer processing times (up to 35 seconds default wait). Images are faster.

### Example: Posting with Mixed Media

Here's a basic example mixing input types. Initialize your `ThreadsPipe` instance first (see [setup](../03-setup-authentication.md)):

```python
from threadspipe import ThreadsPipe
import base64

# Assuming authenticated instance
api = ThreadsPipe(
    user_id="your_user_id",
    access_token="your_access_token",
    gh_bearer_token="your_github_token",  # For local/bytes uploads
    gh_username="your_github_username",
    gh_repo_name="your-upload-repo"
)

# Mixed media post
result = api.pipe(
    post="Check out these amazing images and videos! #ThreadsPipe",
    files=[
        "/home/user/photos/vacation.jpg",  # Local path
        "https://example.com/public-image.png",  # URL
        open("/home/user/video.mp4", "rb").read(),  # Bytes (video)
        base64.b64encode(open("/home/user/art.jpg", "rb").read()).decode()  # Base64
    ],
    file_captions=[
        "A beautiful beach sunset",  # For files[0]
        None,  # No caption for URL
        "Fun family video",  # For bytes video
        "Abstract art piece"  # For base64
    ],
    who_can_reply="accounts_you_follow"  # Restrict replies
)

print(result)  # {'info': 'success', 'data': {'media_ids': ['post_id']}}
```

:::tip
For base64, ensure it's a valid encoding without prefixes (e.g., no `data:image/jpeg;base64,`—strip if needed). Test uploads separately if dealing with large files (>5MB may timeout).
:::

If you have more than 20 files, set `chained_post=True` (default) to auto-split into a carousel thread.

## Creating Simple Text Posts and Replies

Text posts are the simplest use case. For replies, just add `reply_to_id`.

### Step-by-Step: Simple Text Post

1. **Prepare Content**: Write your text (under 500 chars for single post).

2. **Call pipe()**: No files needed.

3. **Handle Response**: Check for success.

Example:

```python
# Simple text post
result = api.pipe(
    post="Hello, Threads! This is my first post with ThreadsPipe-py. Excited to share more.",
    who_can_reply="everyone"
)

if result.get('info') == 'success':
    print(f"Posted successfully! ID: {result['data']['media_ids'][0]}")
else:
    print(f"Error: {result['message']}")
```

For longer text (>500 chars), it auto-splits:

```python
long_post = "This is a very long message that exceeds 500 characters. " * 10  # ~600 chars
result = api.pipe(post=long_post, chained_post=True)
# Creates a thread: root post + reply(ies)
```

### Step-by-Step: Replying to a Post

1. **Obtain reply_to_id**: Fetch it via [retrieving data](../06-retrieving-data.md), e.g., from a previous post.

2. **Craft Reply**: Use `reply_to_id` and optional media.

3. **Publish**: Replies use a separate quota (1000/day vs. 250 for posts).

Example:

```python
# Reply to an existing post
reply_result = api.pipe(
    post="Great point! I agree and want to add more details here.",
    reply_to_id="1234567890abcdef",  # Target post ID
    files=["https://example.com/reply-image.jpg"],  # Optional media
    file_captions=["Supporting image for my reply"],
    who_can_reply="mentioned_only"  # Only mentioned users can reply further
)

print(reply_result)  # Chained if long, with reply threading
```

Replies inherit the original post's context but can form their own threads if content is oversized.

:::caution
Replies count toward the reply quota. Always check with `get_quota_usage(for_reply=True)` before posting.
:::

## Auto-Splitting for Threads (chained_post=True)

ThreadsPipe-py mimics Twitter-style threads by auto-splitting oversized content. This is enabled by default with `chained_post=True`.

### How It Works

- **Text Limits**: Single post max 500 characters. If exceeded, split into ~500-char chunks. The first becomes the root; others reply to the previous (chaining via `reply_to_id`).

- **File Limits**: Max 20 media per post. Excess files attach to subsequent text chunks or form media-only replies.

- **Combined Splitting**: Aligns text and file batches. E.g., 600-char text + 25 files → Root (500 chars + 20 files) → Reply (100 chars + 5 files).

- **Continuity**: Adds "..." to splits for readability. Hashtags distribute across posts if `handle_hashtags=True`.

- **Edge Cases**: If no text but >20 files, creates media-only thread. Quoting/links persist if configured.

Example of auto-splitting:

```python
# Long text + many files
long_text = "Detailed tutorial on ThreadsPipe-py. Step 1: Install via pip. " * 20  # >500 chars
many_files = ["https://example.com/img" + str(i) + ".jpg" for i in range(25)]

result = api.pipe(
    post=long_text,
    files=many_files,
    file_captions=[f"Caption for image {i}" for i in range(25)],
    chained_post=True
)
# Response: {'data': {'media_ids': ['root_id', 'reply1_id', 'reply2_id']}}
```

Disable with `chained_post=False` to truncate instead—useful for strict single-post needs.

This feature shines for storytelling or tutorials, creating natural conversation flows.

## Rate Limiting and Quota Management

Threads enforces quotas: 250 posts/day and 1000 replies/day (24-hour windows). `pipe()` integrates checks via `get_quota_usage()`.

### Using get_quota_usage()

Call this before posting to avoid errors:

```python
# Check post quota
post_quota = api.get_quota_usage(for_reply=False)
if post_quota:
    usage = post_quota['data'][0]['quota_usage']
    total = post_quota['data'][0]['config']['quota_total']
    print(f"Posts used: {usage}/{total}")
    if usage >= total:
        print("Quota exceeded—wait 24 hours.")
else:
    print("No quota data available.")

# Check reply quota
reply_quota = api.get_quota_usage(for_reply=True)
```

`pipe()` auto-checks if `check_rate_limit_before_post=True` (default). If exceeded and `wait_on_rate_limit=True`, it sleeps for the quota duration (e.g., 24 hours). Otherwise, returns an error dict.

:::info
Quotas reset daily. Monitor via the API; errors return HTTP 429 with details.
:::

## Error Handling

Robust error handling prevents invalid posts. Key checks:

- **Empty Posts**: Raises an exception if no text and no files:

```python
try:
    result = api.pipe(post="", files=[])  # Empty
except Exception as e:
    print(f"Error: {e}")  # "Either text or at least 1 media... must be provided"
```

- **Invalid Files**: MIME errors or upload failures return dicts with `'info': 'error'`.

- **Rate Limits**: As above, early return with quota details.

- **Other**: Geo-gating ineligibility, invalid `who_can_reply`, etc., yield descriptive messages.

Always wrap calls in try-except for production use. Logs (via Python's `logging`) provide debug info—disable with `disable_logging=True`.

## Light Internals of __send_post__

Under the hood, `pipe()` loops over splits, calling the private `__send_post__()` for each. Lightly:

- **Quota Check**: Calls `get_quota_usage()`; waits or errors.

- **Media Prep**: `__handle_media__()` normalizes files (uploads locals/bytes/base64 to GitHub), detects types, returns URLs.

- **Container Build**: Creates API containers (single media or carousels with children). Adds params like text, `reply_to_id`, `who_can_reply`.

- **Polling & Publish**: Waits for 'FINISHED' status (35s timeout default), then publishes. Chains by passing prior ID as `reply_to_id`.

- **Cleanup**: Deletes GitHub temps on completion.

This orchestration ensures reliability without exposing you to API minutiae. For deeper dives, see the [API reference](../10-changelog-api-reference.md).

## Step-by-Step Tutorials

### Tutorial 1: Simple Text Post (200 words)

1. Import and initialize:

```python
from threadspipe import ThreadsPipe
api = ThreadsPipe(user_id="...", access_token="...")
```

2. Define content:

```python
post_text = "My first Threads post! #ThreadsPipe"
```

3. Post:

```python
result = api.pipe(post=post_text)
```

4. Verify:

```python
if result['info'] == 'success':
    print("Posted! Check Threads app.")
```

Output: Success dict with post ID.

### Tutorial 2: Media Post with Local and URL (300 words)

1. Setup GitHub for locals (if needed).

2. Prepare files:

```python
local_img = "/path/to/local.jpg"
url_video = "https://example.com/video.mp4"
bytes_img = open("another.jpg", "rb").read()
base64_img = base64.b64encode(bytes_img).decode()
```

3. Call with captions:

```python
result = api.pipe(
    post="Media showcase:",
    files=[local_img, url_video, bytes_img, base64_img],
    file_captions=["Local beach", "Online video", "Bytes image", "Base64 art"]
)
```

4. Handle chaining if >20 files (add more to list).

5. Check quota pre-post:

```python
quota = api.get_quota_usage()
if quota and quota['data'][0]['quota_usage'] < quota['data'][0]['config']['quota_total']:
    # Proceed
    pass
```

Errors? Inspect `result['error']`.

### Tutorial 3: Threaded Reply with Splitting (300 words)

1. Get target ID (e.g., from retrieval).

2. Long reply content:

```python
reply_text = "In-depth response... " * 15  # >500 chars
files = ["https://example.com/img" + str(i) + ".jpg" for i in range(22)]  # >20
```

3. Post as reply:

```python
result = api.pipe(
    post=reply_text,
    files=files,
    reply_to_id="target_id",
    file_captions=[f"Reply img {i}" for i in range(22)],
    who_can_reply="accounts_you_follow",
    chained_post=True
)
```

4. Inspect chain:

```python
media_ids = result['data']['media_ids']
print(f"Thread: {media_ids[0]} -> {media_ids[1:]}")
```

This creates a reply thread to the original post.

### Tutorial 4: Error Handling and Quota Check (200 words)

1. Wrap in try:

```python
try:
    quota = api.get_quota_usage(for_reply=True)
    if quota['data'][0]['quota_usage'] >= 1000:
        raise Exception("Reply quota exceeded")
    
    result = api.pipe(post="Test reply", reply_to_id="id")
except Exception as e:
    print(f"Handled error: {e}")
```

2. For empty: Explicit check or let it raise.

These tutorials build progressively—experiment in a script!

## Output Formats

Success:

```python
{
    'info': 'success',
    'message': 'Post piped to Instagram Threads successfully!',
    'data': {'media_ids': ['id1', 'id2']},  # For chains
    'response': <requests.Response>  # Raw, optional
}
```

Error (e.g., empty post):

```python
{
    'info': 'error',
    'message': 'Either text or at least 1 media must be provided',
    'error': {...},  # API body
    'response': <requests.Response>
}
```

Rate limit example:

```python
{
    'message': 'Rate limit exceeded!',
    'is_error': True,
    'body': {'quota_usage': 251, 'quota_total': 250}
}
```

Always parse `result['info']` to branch logic.

## Next Steps

Mastered basics? Explore [advanced posting](../05-advanced-posting.md) for tags, geo-gating, and quoting. For data retrieval, see [retrieving data](../06-retrieving-data.md). Report issues in the [changelog and API reference](../10-changelog-api-reference.md).

This guide equips you to post and reply effectively—happy threading!

(Word count: ~1,850)