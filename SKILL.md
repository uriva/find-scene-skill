---
name: find-scene
description: Search movie and TV show scenes by dialog, time, or visual description. Download video clips, extract frames, find quotes, identify movies from quotes, and query IMDB data. Use when the user wants to find a specific scene, download a clip, search for a quote in a movie/show, extract a frame, or get movie information via the find-scene API.
---

# find-scene API

API for searching and downloading movie/TV show scenes by dialog, time, or visual description.

## Base URL

```
https://api.find-scene.com
```

All endpoints are `POST` with `Content-Type: application/json`, except
`GET /api/operation/{id}`.

## Authentication

Every API request requires a `_token` field in the JSON body.

```json
{ "_token": "user-api-token", ...other fields }
```

### How to get a token

1. Go to https://find-scene.com and sign in
2. Open https://find-scene.com/settings
3. Click "Generate new token" — the token is shown once, so copy it immediately
4. Include it as `_token` in every API request body

To revoke a token, go to https://find-scene.com/settings and click "Revoke" next
to the token you want to disable.

## Response Format

All successful responses are wrapped in `{ "result": <structured object> }`.
Each endpoint returns a typed JSON object (not a plain string). Error responses
use `{ "error": "message" }` at the top level (HTTP 4xx/5xx) or
`{ "result": { "error": "message" } }` for domain-level errors within a 200
response.

### Video Source Hash

An internal find-scene ID for a video file. Obtained from
`get_best_video_source`. This is NOT an IMDB ID or filename. Required for
downloads, frame extraction, and high-accuracy text source lookups.

### Text Source Hash

An internal find-scene ID for a subtitle/text file. Obtained from
`get_text_source` or `get_high_accuracy_text_source`. Required for phrase search
and subtitle retrieval. NOT a filename or IMDB ID.

### Async Operations

`get_best_video_source`, `get_text_source`, `get_high_accuracy_text_source`,
`download_by_time`, `extract_frame`, `stitch_videos`,
`stitch_videos_side_by_side` and `transcribe_by_time` return an operation ID
(not a direct result). You must poll `GET /api/operation/{id}` until status is
`completed`, then use the result fields from the response.

**Statuses:** `in_progress`, `completed`, `failed`, `cancelled`

### Video Query Object

Many endpoints accept a video query object with these optional fields:

| Field | Type | Description |
| --- | --- | --- |
| `movieOrTVShowName` | string/null | Movie or TV show name |
| `dubbed` | string/null | Dubbing language code, e.g. "en", "de", "pt-br" |
| `animated` | boolean/null | Is the video animated |
| `blackAndWhite` | boolean/null | Is the video black and white |
| `year` | number/null | Year (for distinguishing remakes) |
| `isSeries` | boolean/null | User explicitly said it's a series |
| `season` | number/null | Season number (series only) |
| `episode` | number/null | Episode number (series only) |

### Time Format

All time parameters use `HH:MM:SS` format, e.g. `"00:01:30"`.

### Display Params Object

Used by `download_by_time` and `extract_frame`:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `removeWatermark` | boolean | `false` | Pro users only |
| `gif` | boolean | `false` | Only if user explicitly asked for GIF |
| `mobile` | boolean | `false` | Crop to mobile format. Only if user asked |

## Typical Workflows

### Workflow 1: Find and download a scene by quote

```
1. quote_to_movie        -> identify which movie contains the quote
2. get_best_video_source -> returns operation ID, poll until completed to get videoHash
3. get_text_source       -> returns operation ID, poll until completed to get textSource hash
4. search_phrase         -> find exact timestamp of the quote
5. download_by_time      -> schedule clip download (returns operation ID)
6. GET /api/operation/id -> poll until completed, get download URL
```

### Workflow 2: Download a scene by time

```
1. get_best_video_source -> returns operation ID, poll until completed to get videoHash
2. download_by_time      -> schedule download with start/end times
3. GET /api/operation/id -> poll until completed
```

### Workflow 3: Search by visual scene description

```
1. find_by_scene_description -> search by what happens visually
2. get_best_video_source     -> returns operation ID, poll until completed to get videoHash
3. download_by_time          -> download the scene
4. GET /api/operation/id     -> poll until completed
```

### Workflow 4: Find which episode contains a quote (TV series)

```
1. find_episode_by_phrase -> find season/episode for a phrase
2. get_best_video_source  -> returns operation ID, poll until completed to get videoHash
3. get_text_source        -> returns operation ID, poll until completed to get textSource
4. search_phrase          -> get exact timestamp
5. download_by_time       -> download clip
6. GET /api/operation/id  -> poll until completed
```

### Workflow 5: Extract a frame / screenshot

```
1. get_best_video_source -> get videoHash
2. extract_frame         -> schedule frame extraction (returns operation ID)
3. GET /api/operation/id -> poll until completed, get image URL
```

## Tips

- Always get the video source hash first before attempting downloads or text
  source lookups.
- Use `get_high_accuracy_text_source` (with videoHash) over `get_text_source`
  when you have a video source, for better subtitle timing alignment.
- All async tools return operation IDs.
  Never return these to the user as download links. Always poll until you get the
  actual result.
- Keep clip durations reasonable (under 60 seconds) to avoid long processing
  times.
- For TV series, use `find_episode_by_phrase` first to identify the episode
  before searching within it.
- The `find_by_scene_description` endpoint requires the video to have been
  indexed. If it returns no results, use
  `request_indexing_for_scene_description` and try again later.
- OpenAPI spec is available at `https://api.find-scene.com/api/openapi.json` for
  machine-readable schema details.

## Error Handling

- **400**: Invalid parameters (check required fields)
- **401**: Invalid or missing `_token`
- **500**: Internal server error (retry or report)

## API Endpoints Reference

### `POST /api/get_best_video_source`

Schedules a task to get the best video source for a given video. The result will be automatically delivered when complete. You will be notified when the operation finishes — do not poll or check status.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `query` | object | required |  |
| `timeoutSeconds` | number | optional | Max time to wait for video sources to respond (in seconds). Leave empty unless the user gave a seconds value; avoid manual minute->seconds math. Default is 60 seconds. |

**Response:** `{ "operationId": string }` or `{ "error": string }`

This is an **async** endpoint. Returns `{ "result": { "operationId": "..." } }`. Poll `GET /api/operation/{id}` until completed.

### `POST /api/compute_running_time`

Compute the running time of a video.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `imdbId` | string | required |  |

**Response:** `{ "runtimeSeconds": number | null }`

### `POST /api/get_high_accuracy_text_source`

Schedules a task to get a text source for given video source. The result will be automatically delivered when complete. You will be notified when the operation finishes — do not poll or check status.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `language` | string | optional | The subs language we are interested in. e.g. pt-br, en, de etc' |
| `query` | object | required |  |
| `videoHash` | string | required | An internal find-scene.com id of the video source (not an imdb id or a filename). This will be in the output of the get_best_video_source tool. |

**Response:** `{ "operationId": string }` or `{ "error": string }`

This is an **async** endpoint. Returns `{ "result": { "operationId": "..." } }`. Poll `GET /api/operation/{id}` until completed.

### `POST /api/get_text_source`

Schedules a task to get a text source for given video details. If you have the video source use the other version, it will be more accurate with the timings. Note that this outputs a text source hash. NOT a video source hash. The result will be automatically delivered when complete. You will be notified when the operation finishes — do not poll or check status.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `language` | string | optional | The subs language we are interested in. e.g. pt-br, en, de etc' |
| `query` | object | required |  |
| `minDuration` | number | optional | If provided, only return subtitles where the final entry time end time is larger than this number of seconds. |

**Response:** `{ "operationId": string }` or `{ "error": string }`

This is an **async** endpoint. Returns `{ "result": { "operationId": "..." } }`. Poll `GET /api/operation/{id}` until completed.

### `POST /api/search_phrase`

Search phrases in video text.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `phraseSearchParams` | object | required |  |
| `textSource` | string | required |  |

**Response:** `{ "occurrences": object[] }` or `{ "error": string }`

### `POST /api/get_srt_entries_around_phrase`

Get subtitle entries (text and times) around a phrase, for a given video. Returns entries within a time window before and after the phrase.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `limit` | number | required | Max number of sections to return, if the phrase occurs many times. |
| `skip` | number | required | Skip this many phrase occurrences, e.g. if user asked for the next match. |
| `textSource` | string | required |  |
| `phrase` | string | required | Phrase to search for in subtitles |
| `secondsBefore` | number | required | Seconds before the phrase to include |
| `secondsAfter` | number | required | Seconds after the phrase to include |

**Response:** `{ "occurrences": object[] }` or `{ "error": string }`

### `POST /api/get_srt_entries_by_time_range`

Get subtitle entries (text and times) for a given video and time range.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `videoQuery` | object | required |  |
| `startTime` | string | required | e.g. 00:01:30 |
| `endTime` | string | required | e.g. 00:01:30 |
| `subsLanguage` | string | optional | Subtitle language, e.g. en, pt-br |

**Response:** `{ "srt": string }` or `{ "error": string }`

### `POST /api/download_by_time`

Schedules a task to download a video part by time. The result will be automatically delivered to the user (UI + email) and billed when complete. You will be notified when the operation finishes — do not poll or check status.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `startTime` | string | required | e.g. 00:01:30 |
| `endTime` | string | required | e.g. 00:01:30 |
| `srtOffset` | number | optional | If you need to correct for time in the subs file, give the offset here. |
| `videoHash` | string | required | An internal find-scene.com id of the video source (not an imdb id or a filename). This will be in the output of the get_best_video_source tool. |
| `textSource` | string | optional | Internal find-scene id of a text source. Not a filename or an imdb id. It can be got using the get_high_accuracy_text_source tool. |
| `displayParams` | object | optional |  |

**Response:** `{ "operationId": string }` or `{ "error": string }`

This is an **async** endpoint. Returns `{ "result": { "operationId": "..." } }`. Poll `GET /api/operation/{id}` until completed.

### `POST /api/extract_frame`

Extract a single frame from a video at a specific time. Useful for creating stickers or screenshots. The result will be automatically delivered to the user when complete. You will be notified when the operation finishes — do not poll or check status.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `videoHash` | string | required | An internal find-scene.com id of the video source (not an imdb id or a filename). This will be in the output of the get_best_video_source tool. |
| `time` | string | required | e.g. 00:01:30 |
| `textSource` | string | optional | Optional text source to overlay subtitles on the frame |
| `overrideTextTop` | string | optional | Optional custom text to overlay at the top of the frame (meme generator mode) |
| `overrideTextBottom` | string | optional | Optional custom text to overlay at the bottom of the frame (meme generator mode) |
| `displayParams` | object | optional |  |

**Response:** `{ "operationId": string }` or `{ "error": string }`

This is an **async** endpoint. Returns `{ "result": { "operationId": "..." } }`. Poll `GET /api/operation/{id}` until completed.

### `POST /api/stitch_videos`

Stitch multiple previously downloaded video clips into a single video. Accepts URLs from completed download_by_time operations. The result will be automatically delivered to the user (UI + email) and billed when complete. You will be notified when the operation finishes — do not poll or check status.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `urls` | string[] | required | Array of video URLs to stitch together in order. These should be URLs from completed download_by_time operations. |
| `displayParams` | object | required |  |

**Response:** `{ "operationId": string }` or `{ "error": string }`

This is an **async** endpoint. Returns `{ "result": { "operationId": "..." } }`. Poll `GET /api/operation/{id}` until completed.

### `POST /api/stitch_videos_side_by_side`

Place multiple previously downloaded video clips side by side in a single frame. All videos play simultaneously. Accepts URLs from completed download_by_time operations. The result will be automatically delivered to the user (UI + email) and billed when complete. You will be notified when the operation finishes — do not poll or check status.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `urls` | string[] | required | Array of video URLs to stitch together in order. These should be URLs from completed download_by_time operations. |
| `displayParams` | object | required |  |

**Response:** `{ "operationId": string }` or `{ "error": string }`

This is an **async** endpoint. Returns `{ "result": { "operationId": "..." } }`. Poll `GET /api/operation/{id}` until completed.

### `POST /api/cancel_operation`

Cancel a stuck async operation by its ID.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | string | required | The operation ID to cancel |

**Response:** `{ "cancelled": boolean, "operationId": string }` or `{ "cancelled": boolean, "reason": string }`

### `POST /api/is_string_a_movie_name`

Check if the string is a movie name

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `string` | string | required | The string to check |

**Response:** `{ "isMovieName": boolean }`

### `POST /api/quote_to_movie`

Get movie name from quote

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `quote` | string | required | The quote to check |

**Response:** `{ "candidates": string[] }`

### `POST /api/find_episode_by_phrase`

Find a series episode by a phrase. This is not for movies, but tv shows.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `phrase` | string | required | The quote or phrase to search for |
| `videoQuery` | object | required | The TV show to search in. Must include 'name' (the show title). |
| `season` | number | optional | Narrow search to a specific season |
| `limit` | number | required |  |

**Response:** `{ "episodes": object[] }`

### `POST /api/query_imdb`

Get movie information from IMDB, including the imdb id itself, if you only have the title.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `title` | string | required | The movie title to query |
| `year` | number | optional | The year of the movie |
| `imdbId` | string | optional | The IMDB ID of the movie |
| `season` | number | optional |  |
| `episode` | number | optional |  |

**Response:** `{ "name": string, "imdb": string, "year": number, "season": number, "episode": number }`

### `POST /api/popular_quotes_from_title`

Get popular quotes from a movie or TV show title

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `limit` | number | required |  |
| `name` | string | required |  |
| `imdb` | string | optional | The IMDB ID of the movie |

**Response:** `{ "quotes": string[] }`

### `POST /api/find_by_scene_description`

Search scene by non dialog description.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `description` | string | required | a description of what happens in the scene, if provided by the user. do not include the movie or tv show name here. |
| `video` | any | optional |  |
| `nResults` | number | required |  |
| `nSkip` | number | required | skip this amount of results, e.g. if user requested the next result, or another one. |
| `scoreThreshold` | number | optional | minimum similarity score (0-1) to include a result. use higher values (e.g. 0.6) for specific scenes, lower (e.g. 0.3) for vague descriptions. |

**Response:** `{ "results": object[], "warning": string }` or `{ "error": string }`

### `POST /api/request_indexing_for_scene_description`

Request indexing of a video. If it's a series we need to know the season and episode.

**Request body** (JSON, requires `_token`):

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `video` | object | required |  |

**Response:** `{ "requested": boolean }`

### `POST /api/check_quota`

Check how many search credits the current user has remaining. Credits work on a rolling 30-day window — each credit returns exactly 30 days after it was used. If the user is low on credits, tell them when their next credit returns.

**Response:** `{ "creditsRemaining": number, "nextCreditReturnMs": number }`

### `GET /api/operation/{id}`

Poll the status of an async operation (returned by download_by_time or extract_frame)

**Path parameters:**

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | string | yes | |

**Response:** object
