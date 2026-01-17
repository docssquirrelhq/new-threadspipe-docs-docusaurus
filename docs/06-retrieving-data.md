---
title: Retrieving Data and Insights
slug: retrieving-data
description: Learn how to fetch posts, profiles, replies, and insights using ThreadsPipe-py. Covers API methods, parameters, responses, pagination, and permissions.
sidebar_label: Retrieving Data and Insights
sidebar_position: 6
---

# Retrieving Data and Insights

ThreadsPipe-py provides a robust set of methods for retrieving data from the Threads platform, including posts, profiles, replies, and detailed analytics insights. These methods interact directly with the Meta Threads Graph API (version 1.0 by default), allowing you to query your account's content and engagement metrics programmatically. This is essential for tasks like content auditing, audience analysis, and performance tracking.

All retrieval methods require a valid, long-lived access token with appropriate scopes (e.g., `threads_basic` for basic data, `threads_manage_insights` for analytics). Initialize a `ThreadsPipe` instance with your credentials before use:

```python
from threadspipepy import ThreadsPipe

api = ThreadsPipe(
    user_id="your_user_id",
    access_token="your_long_lived_access_token"
)
```

Responses are returned as raw JSON dictionaries from the API, which you can parse for specific fields. For large datasets, pagination is supported via cursor-based fetching. We'll cover each method below, including parameters, response formats, code examples, and best practices.

## Basic Data Retrieval

These methods focus on fetching core content like posts, individual post details, replies, and user profiles. They use GET requests to endpoints like `/me/threads` or `/{post_id}`.

### get_posts() - Fetching Account Posts

The `get_posts` method retrieves all posts from your Threads account, including replies and quotes. It supports date-range filtering and limits to control the scope of results. This is ideal for pulling a history of your content.

#### Parameters
- `since_date` (str | None, default: None): ISO 8601 timestamp (e.g., "2024-01-01T00:00:00+0000") for the earliest post to include. Omitting this fetches from the account's earliest post.
- `until_date` (str | None, default: None): ISO 8601 timestamp (e.g., "2024-12-31T23:59:59+0000") for the latest post to include. Omitting this fetches up to the most recent post.
- `limit` (str | int | None, default: None): Maximum number of posts (e.g., 50). The API defaults to ~50 per page if unspecified.

The method requests a fixed set of fields: `id`, `is_quote_post`, `media_product_type`, `media_type`, `media_url`, `permalink`, `owner`, `username`, `text`, `timestamp`, `shortcode`, `thumbnail_url`, `alt_text`, `children`, and `link_attachment_url`. These provide comprehensive post details, including media URLs for images/videos.

#### Response Format
Returns a JSON dict with:
- `data`: List of post objects (e.g., `{"id": "123", "text": "Hello Threads!", "media_url": "https://example.com/image.jpg", "timestamp": "2024-09-01T10:00:00+0000"}`).
- `paging`: Dict with `cursors` (`before`/`after`) and `next` URL for pagination.

Empty `data` indicates no matching posts. Errors (e.g., invalid token) appear under an `error` key.

#### Code Example: Querying and Parsing
```python
# Fetch recent posts with limit
posts_response = api.get_posts(
    since_date="2024-06-01T00:00:00+0000",
    until_date="2024-09-01T00:00:00+0000",
    limit=10
)

# Parse JSON response
if "data" in posts_response:
    posts = posts_response["data"]
    for post in posts:
        print(f"ID: {post['id']}, Text: {post['text'][:50]}..., Media: {post.get('media_url', 'None')}")
else:
    print(f"Error: {posts_response.get('error', {}).get('message', 'Unknown error')}")
```

This example filters posts from June to September 2024 and prints key fields. For media-heavy posts, access `media_url` to download assets via `requests.get(post['media_url'])`.

#### Pagination
The API paginates results. Use the `paging` cursors to fetch more:

```python
def paginate_posts(api, since_date=None, until_date=None, limit=50):
    all_posts = []
    response = api.get_posts(since_date=since_date, until_date=until_date, limit=limit)
    
    while "data" in response:
        all_posts.extend(response["data"])
        
        if "paging" in response and "next" in response["paging"]:
            # Manually construct next URL or use cursors (API supports ?after=CURSOR)
            next_url = response["paging"]["next"]
            response = requests.get(next_url, headers={"Authorization": f"Bearer {api.access_token}"}).json()
        else:
            break
    
    return all_posts

# Usage
complete_posts = paginate_posts(api, since_date="2024-01-01T00:00:00+0000")
print(f"Retrieved {len(complete_posts)} posts total.")
```

This helper function handles full pagination, useful for accounts with thousands of posts. Note: Rate limits apply (~200 calls/hour), so add delays with `time.sleep(1)`.

### get_post() - Single Post Details

Use `get_post` to fetch detailed info for a specific post by ID. This is efficient for deep dives into individual content.

#### Parameters
- `post_id` (str, required): The post's unique ID (e.g., from `get_posts`).

It requests the same fixed fields as `get_posts`.

#### Response Format
A JSON dict with a single post object (e.g., `{"id": "123", "text": "...", "media_url": "..."}`) or an `error` on failure.

#### Code Example
```python
post_id = "your_post_id_here"
post_details = api.get_post(post_id)

if "id" in post_details:
    print(f"Permalink: {post_details['permalink']}")
    print(f"Owner: {post_details['username']}")
    print(f"Children (replies): {len(post_details.get('children', []))}")
else:
    print(f"Error fetching post: {post_details.get('error', {}).get('message')}")
```

Parse for specifics like `children` (nested replies) or `alt_text` for accessibility.

### get_post_replies() - Fetching Replies

This method retrieves replies to a post, with options for depth and ordering. Great for engagement analysis.

#### Parameters
- `post_id` (str, required): Target post ID.
- `top_levels` (bool, default: True): If True, fetch only direct replies; if False, include nested replies (full thread).
- `reverse` (bool, default: False): If True, newest replies first; else, oldest first.

Fields include: `id`, `text`, `timestamp`, `media_type`, `media_url`, `shortcode`, `thumbnail_url`, `children`, `has_replies`, `root_post`, `replied_to`, `is_reply`, `hide_status`.

#### Response Format
JSON with `data`: List of reply objects (e.g., `{"id": "reply1", "text": "Reply text", "children": [...]}` for nesting).

#### Code Example: Parsing Replies
```python
replies = api.get_post_replies(post_id="post_id", top_levels=False, reverse=True)

def flatten_replies(replies_data):
    flat_list = []
    for reply in replies_data.get("data", []):
        flat_list.append(reply)
        if "children" in reply:
            flat_list.extend(flatten_replies({"data": reply["children"]}))  # Recursive
    return flat_list

all_replies = flatten_replies(replies)
print(f"Total replies: {len(all_replies)}")
for reply in all_replies[:5]:  # Top 5 newest
    print(f"Reply: {reply['text'][:30]}... at {reply['timestamp']}")
```

This flattens nested replies for easier analysis. Use `top_levels=True` for shallow fetches to avoid deep recursion.

### get_profile() - User Profile Info

Fetches basic profile details for the authenticated account.

#### Parameters
None. Uses the connected account.

#### Response Format
JSON with: `id`, `username`, `name`, `threads_profile_picture_url`, `threads_biography`.

#### Code Example
```python
profile = api.get_profile()
print(f"Username: @{profile['username']}")
print(f"Bio: {profile['threads_biography']}")
print(f"Profile Pic: {profile['threads_profile_picture_url']}")

# Save profile pic
import requests
if "threads_profile_picture_url" in profile:
    img_data = requests.get(profile["threads_profile_picture_url"]).content
    with open("profile.jpg", "wb") as f:
        f.write(img_data)
```

Ideal for personalizing apps or verifying account setup. Note: This is for your own profile; other users require separate API calls with their IDs.

## Insights and Analytics

Insights methods provide metrics like views and demographics, requiring the `threads_manage_insights` permission. Data availability starts from account creation, but filters have limits (see below).

### get_post_insights() - Post Engagement Metrics

Retrieves lifetime metrics for a single post.

#### Parameters
- `post_id` (str, required): Post ID.
- `metrics` (str | list[str] | 'all', default: 'all'): Comma-separated or list of metrics from `threads_post_insight_metrics`: `views`, `likes`, `replies`, `reposts`, `quotes`, `shares`.

#### Response Format
JSON with `data`: List of metric objects (e.g., `{"name": "views", "period": "lifetime", "values": [{"value": 1234, "end_time": "2024-11-01T00:00:00+0000"}]}`).

#### Code Example: Parsing Metrics
```python
insights = api.get_post_insights(post_id="post_id", metrics=["views", "likes"])

metrics_dict = {item["name"]: item["values"][0]["value"] for item in insights["data"]}
print(f"Views: {metrics_dict.get('views', 0)}, Likes: {metrics_dict.get('likes', 0)}")

# For all metrics
all_insights = api.get_post_insights(post_id="post_id")
for metric in all_insights["data"]:
    value = metric["values"][0]["value"]
    print(f"{metric['name'].title()}: {value}")
```

This aggregates values into a dict for easy charting (e.g., with Matplotlib).

### get_user_insights() - Account-Wide Insights

Fetches aggregated metrics and follower demographics for an account.

#### Parameters
- `user_id` (str | None, default: None): Target user ID (defaults to "me").
- `since_date` / `until_date` (str | None): ISO 8601 dates for filtering (restrictions apply).
- `follower_demographic_breakdown` (str, default: 'country'): Breakdown for demographics: from `threads_follower_demographic_breakdown_list` (`country`, `city`, `age`, `gender`).
- `metrics` (str | list[str] | 'all', default: 'all'): From `threads_user_insight_metrics`: `views`, `likes`, `replies`, `reposts`, `quotes`, `followers_count`, `follower_demographics`.

`follower_demographics` requires a breakdown and provides segmented data (e.g., top countries by follower count).

#### Response Format
JSON with `data`: List of metric objects. For demographics: `{"name": "follower_demographics", "values": [{"value": 5000, "breakdown": [{"name": "US", "value": 2000}, ...]}]}`.

#### Code Example: Demographics and Parsing
```python
insights = api.get_user_insights(
    since_date="2024-06-01",
    until_date="2024-09-01",
    follower_demographic_breakdown="country",
    metrics=["followers_count", "follower_demographics"]
)

# Parse followers
followers_count = next((m["values"][0]["value"] for m in insights["data"] if m["name"] == "followers_count"), 0)
print(f"Total Followers: {followers_count}")

# Parse demographics
demo_metric = next((m for m in insights["data"] if m["name"] == "follower_demographics"), None)
if demo_metric:
    breakdowns = demo_metric["values"][0]["breakdown"]
    for bd in breakdowns[:5]:  # Top 5 countries
        print(f"{bd['name']}: {bd['value']} followers")
```

This extracts totals and top demographics. For age/gender, adapt the parsing to bucket labels.

## Date Filtering Notes

Date parameters (`since_date`, `until_date`) use ISO 8601 format and apply to `get_posts` and `get_user_insights`. However:
- Post-April 2024 limits: Filters do not support dates before April 13, 2024. Earlier data may be incomplete or unavailable.
- Insights availability: User insights (e.g., demographics) are not guaranteed before June 1, 2024. Post metrics are lifetime but filtered within bounds.
- Best practice: Always validate `end_time` in responses to confirm coverage. For historical data pre-2024, rely on unfiltered calls and client-side filtering.

Example warning for invalid dates:
```python
# This will error or return empty
old_insights = api.get_user_insights(since_date="2023-01-01")  # Ignored, use post-April
```

## Permissions and Best Practices

- **Basic Retrieval**: Requires `threads_basic` scope for posts/profiles/replies.
- **Insights**: Mandatory `threads_manage_insights` scope. Without it, methods raise API errors (e.g., OAuthException). Verify scopes during [setup authentication](../03-setup-authentication.md).
- **Rate Limits**: ~200 calls/hour. Implement exponential backoff for retries.
- **Error Handling**: Always check for `error` in responses. Common issues: Invalid IDs (code 100), permissions (code 200).
- **Pagination Tips**: For `get_posts` and `get_user_insights`, use cursors to avoid redundant fetches. Store `after` tokens for incremental syncs.
- **Data Privacy**: Insights include aggregated demographics; comply with Meta's terms. Avoid storing raw PII.
- **Integration**: Combine with [posting methods](../05-advanced-posting.md) for full workflows, like auto-analyzing new posts.

For CLI usage, see [CLI Usage](../07-cli-usage.md). Troubleshoot errors in [Errors & Best Practices](../09-errors-best-practices.md).

This covers the core of data retrieval in ThreadsPipe-py, enabling powerful analytics. Experiment with small queries first to understand your data volume.

(Word count: 1624)