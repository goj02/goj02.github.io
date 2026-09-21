# Chat Image Generator API Documentation

This document describes the Flask application in `app.py` and the local services and workflows called by it. It is based on the current source code, so it documents the behavior that exists today rather than an idealized or future API.

## Overview

The application is a Slack-controlled image-generation service. A Slack event is received by Flask, authorized by checking the Slack user's name, and dispatched to a background thread. The workflows use Dropbox for catalog and image storage and use ImaginePro for the image-generation path currently reached by Slack commands.

The application has one HTTP endpoint:

```text
POST /slack/webhook
```

The Flask object is exported as `app` and can be served with Gunicorn:

```text
gunicorn --bind 0.0.0.0:$PORT --workers 2 app:app
```

## Runtime Architecture

`app.py` initializes the following objects at import time:

| Object | Class | Responsibility |
| --- | --- | --- |
| `slack_svc` | `SlackService` | Slack user lookup, messages, and thread reads |
| `dropbox_svc` | `DropboxService` | Dropbox downloads, uploads, folders, links, and moves |
| `imgpro_svc` | `ImgProService` | ImaginePro image generation and polling |
| `catalog_wf` | `CatalogWorkflow` | Reads the catalog and generates images through ImaginePro |
| `generation_wf` | `GenerationWorkflow` | Alternate image-generation workflow that uploads generated images |
| `review_wf` | `ReviewWorkflow` | Posts review messages and moves approved/rejected files |

The current webhook dispatches to `catalog_wf` and `review_wf`. `generation_wf` is initialized but is not called by the current webhook.

## HTTP API

### POST `/slack/webhook`

Receives Slack Events API requests. The handler always returns HTTP `200` for the implemented branches, including ignored events and most malformed or unsupported requests.

#### Request headers

| Header | Required | Description |
| --- | --- | --- |
| `Content-Type: application/json` | For event processing | Required for Flask's `request.is_json` branch to run. |
| `X-Slack-Retry-Num` | No | Any value causes the request to be ignored as a Slack retry. |

The application does not currently verify Slack request signatures. Network-level protection or signature verification should be added before exposing this endpoint to untrusted traffic.

#### Slack URL verification request

Request:

```json
{
  "type": "url_verification",
  "challenge": "challenge-value"
}
```

Response: HTTP `200`

```json
{
  "challenge": "challenge-value"
}
```

#### Slack event request

The handler reads these fields from `event`:

| Field | Use |
| --- | --- |
| `user` | Passed to `SlackService.is_user_ellie`. |
| `text` | Compared with the supported commands. Missing text becomes an empty string. |
| `channel` | Passed to the selected workflow. |
| `thread_ts` | Distinguishes a channel command from a thread reply. |
| `bot_id` / `subtype` | Bot messages are ignored. |

Example event envelope:

```json
{
  "type": "event_callback",
  "event": {
    "type": "message",
    "user": "U0123456789",
    "text": "Generate Images",
    "channel": "C0123456789"
  }
}
```

#### Response values

| Condition | HTTP | JSON response |
| --- | ---: | --- |
| `X-Slack-Retry-Num` is present | 200 | `{"status":"ignored_retry"}` |
| Event has `bot_id` or `subtype == "bot_message"` | 200 | `{"status":"ignored_bot"}` |
| Authorized user starts image generation | 200 | `{"status":"processing_catalog"}` |
| Authorized user starts a review | 200 | `{"status":"processing_review"}` |
| Authorized user replies `Yes` or `No` in a thread | 200 | `{"status":"processing_reply"}` |
| Any other request | 200 | `{"status":"ok"}` |

The processing responses acknowledge Slack immediately. The actual workflow runs in a newly created Python thread, so a `processing_*` response means that work was scheduled, not that it succeeded.

## Supported Slack Commands

Commands are only accepted when `SlackService.is_user_ellie` returns `True`. Matching is case-insensitive for command text.

### Generate Images

Send `Generate Images` as a top-level channel message. The handler starts:

```python
catalog_wf.run(channel_id, "generate images")
```

The workflow downloads `/catalog.csv`, builds a prompt for each catalog row, submits the product image and prompt to ImaginePro, waits for completion, and saves each result in Dropbox's `/Generated` folder.

The source also attempts to parse the final word of the trigger as a product count. The webhook currently passes the literal string `generate images`, so the parse fails and the default limit is used.

### Review

Send `Review` as a top-level channel message. The handler starts:

```python
review_wf.start_review(channel_id)
```

The workflow lists files in `/Generated` and posts one Slack message per file containing a temporary Dropbox link. Each message asks the reviewer to reply in its thread with `Yes` or `No`.

### Yes / No

Reply to a review message in its Slack thread with `Yes` or `No`. The handler starts:

```python
review_wf.handle_reply(channel_id, thread_ts, text)
```

The workflow extracts the filename from backticks in the parent message and moves the file from `/Generated` to `/Approved` for `Yes`, or `/Rejected` for `No`.

## Configuration API

`Config` loads environment variables with `python-dotenv` when `config.py` is imported.

### Environment variables

| Variable | Required by code | Use |
| --- | --- | --- |
| `SLACK_BOT_TOKEN` | Yes for Slack calls | Slack Web API bot token. A warning is printed unless it starts with `xoxb-`. |
| `DROPBOX_ACCESS_TOKEN` | Yes for Dropbox calls | Dropbox access token. |
| `IMAGEPRO_KEY` | Yes for the active catalog workflow | ImaginePro bearer token. |
| `SLACK_CHANNEL` | No | Defaults to `#imageteam`; currently stored but not used by the webhook. |
| `PORT` | No | Listening port in local execution; defaults to `3000`. |

### Dropbox paths

```python
Config.CATALOG_PATH = "/catalog.csv"
Config.GENERATED_FOLDER = "/Generated"
Config.APPROVED_FOLDER = "/Approved"
Config.REJECTED_FOLDER = "/Rejected"
```

`Config.validate()` prints warnings for missing or malformed critical tokens. It does not raise an exception, so the application can start with invalid credentials and fail later when a service call is made.

## Service APIs

### `SlackService`

Module: `services/slack_service.py`

#### `SlackService(token: str)`

Creates a Slack SDK `WebClient` using `token`.

#### `is_user_ellie(user_id: str) -> bool`

Calls Slack `users.info` and checks the user's display name, real name, and username. Returns `True` if any non-empty value contains `ellie`, case-insensitively. Returns `False` on `SlackApiError`.

#### `send_message(channel: str, text: str, thread_ts: str = None)`

Posts `text` to `channel`. If `thread_ts` is provided, posts as a reply in that thread. Returns the Slack SDK response.

#### `get_parent_message_text(channel: str, thread_ts: str) -> str`

Calls `conversations.replies` with `limit=1` and returns the first message's text. Returns an empty string if no messages are returned.

### `DropboxService`

Module: `services/dropbox_service.py`

#### `DropboxService(access_token: str)`

Creates a Dropbox SDK client.

#### `download_csv(file_path: str) -> str`

Downloads a Dropbox file and decodes its contents as UTF-8 text.

#### `upload_file(content_bytes: bytes, dropbox_path: str)`

Uploads bytes to `dropbox_path` using overwrite mode. Returns the Dropbox SDK response.

#### `ensure_folder(folder_path: str)`

Creates a Dropbox folder. An existing-folder conflict is ignored; other Dropbox API errors are re-raised.

#### `list_files(folder_path: str) -> list`

Lists one page of entries and returns only `dropbox.files.FileMetadata` entries. Pagination is not implemented.

#### `get_temporary_link(file_path: str) -> str`

Returns a temporary Dropbox link for a file.

#### `move_file(src_path: str, dest_path: str)`

Moves a file with Dropbox `autorename=True`. Returns the Dropbox SDK response.

#### `save_image_from_url(image_url: str, file_name: str, folder_path: str = "/Generated") -> str`

Downloads an image with `requests`, ensures the target folder exists, uploads the bytes, and returns the destination Dropbox path. HTTP errors from the image download are raised.

### `ImgProService`

Module: `services/imagepro_service.py`

#### `ImgProService(imgkey, dropbox_service)`

Stores the ImaginePro key and a Dropbox service used by `getResult`.

#### `startImgGen(imgurl, prompt) -> str`

Submits the source image URL and prompt to ImaginePro's universal image endpoint using model `nano-banana-2`. On success, returns the provider `messageId`. If the response cannot provide a message ID, returns a string beginning with `IMAGE GEN FOR ... FAILED:` instead of raising the provider error.

#### `checkProcessing(proccode)`

Polls ImaginePro's message endpoint for up to 60 seconds at five-second intervals. Returns the result URI when the response contains `uri`; raises `TimeoutError` when the operation fails or times out. A provider response with a non-`PROCESSING` status and no URI exits the loop and becomes a timeout error.

#### `getResult(proccode, productname)`

Calls `checkProcessing`, creates a timestamped JPG filename, downloads the result indirectly through `DropboxService.save_image_from_url`, and saves it in `/Generated`. The method has no explicit return value.

## Workflow APIs

### `CatalogWorkflow`

Module: `workflows/catalog_workflow.py`

#### `CatalogWorkflow(slack_service, dropbox_service, imgpro_service)`

Stores the injected Slack, Dropbox, and ImaginePro services.

#### `run(channel: str, triggerMessage: str)`

1. Sends a progress message to Slack.
2. Downloads `/catalog.csv`.
3. Requires a `Category` CSV column; otherwise sends an error and returns.
4. Builds a prompt for each row with `prompt_builder.build_prompt`.
5. Submits the row's `Photo` URL and prompt to ImaginePro.
6. Waits 30 seconds, polls for the result, and saves it to `/Generated`.
7. Sends a Dropbox folder link when processing completes.

The CSV row is expected to contain at least `Product Name` and `Photo`. Prompt construction also recognizes `Color / Finish`, `Material`, `Shot Idea`, and `Notes`.

The outer exception handler catches failures and sends `Catalog extraction failed: ...` to Slack. It does not re-raise the error to the Flask request because the workflow runs in a background thread.

### `GenerationWorkflow`

Module: `workflows/generation_workflow.py`

#### `GenerationWorkflow(slack_service, dropbox_service, image_generation_service, generated_folder: str)`

Stores the injected Slack, Dropbox, and image-generation services, along with the target Dropbox folder.

#### `run(channel: str, records: list)`

Ensures the target folder exists, generates an image for each record through the injected image-generation service, and uploads it as `{generated_folder}/{record_id}.jpg`. `record_id` is taken from `record["id"]` or defaults to `item_N`. Per-record exceptions are logged and skipped. A completion summary is sent to Slack.

This workflow is not connected to the current `/slack/webhook` command dispatch. The active command path uses `ImgProService` through `CatalogWorkflow`.

### `ReviewWorkflow`

Module: `workflows/review_workflow.py`

#### `ReviewWorkflow(slack_service, dropbox_service, config)`

Stores the injected services and the configuration class/object containing Dropbox folder constants.

#### `start_review(channel: str)`

Ensures `/Approved` and `/Rejected` exist, lists files in `/Generated`, and sends a review message with a temporary link for each file. If no files exist, sends a no-images message. Lookup failures are reported to Slack.

#### `handle_reply(channel: str, thread_ts: str, reply_text: str)`

Accepts only `yes` and `no`. Reads the parent Slack message, extracts the first backtick-delimited filename, moves the corresponding `/Generated/{filename}` file to `/Approved/{filename}` or `/Rejected/{filename}`, and replies in the thread. Invalid replies, missing parent text, or missing filenames return without moving a file.

## Prompt Builder API

Module: `workflows/prompt_builder.py`

### `build_prompt(record: Dict[str, Any]) -> str`

Builds an editorial product-photography prompt from a catalog row.

| Record key | Behavior |
| --- | --- |
| `Product Name` | Product name; defaults to `product`. |
| `Color / Finish` | Optional product attribute. |
| `Material` | Optional product attribute. |
| `Shot Idea` | Replaces the default scene description when non-empty. |
| `Notes` | Added as an additional design cue unless it contains `Ellie:`. |

The returned prompt always includes the product context, scene, soft natural light, warm modern styling, realistic materials, crisp detail, an in-context product presentation, and ecommerce/social-use guidance.

Example:

```python
build_prompt({
    "Product Name": "Serving Bowl Large",
    "Color / Finish": "Forest",
    "Material": "Stoneware",
    "Shot Idea": "on a set dinner table with food in it",
    "Notes": "top seller"
})
```

## Catalog Contract

The catalog is downloaded from Dropbox as `/catalog.csv` and parsed with Python's `csv.DictReader`. The required header is `Category`; the following fields drive the current prompt and generation path:

```csv
Category,Product Name,Photo,Color / Finish,Material,Shot Idea,Notes
Serveware,Serving Bowl Large,https://example.com/bowl.jpg,Forest,Stoneware,on a set dinner table,top seller
```

`Photo` must be a URL accessible to the ImaginePro service. The application does not validate URL format, image type, dimensions, or row completeness before submitting the request.

## End-to-End Flow

```text
Slack event
  -> POST /slack/webhook
  -> retry/bot filtering
  -> Slack users.info authorization check
  -> background thread
     -> Generate Images: Dropbox catalog -> prompt builder -> ImaginePro -> Dropbox /Generated
     -> Review: Dropbox /Generated -> Slack review messages
     -> Yes/No reply: parent Slack message -> Dropbox move -> Slack confirmation
```

## Error and Operational Behavior

- Service objects are initialized while `app.py` is imported. Initialization errors are printed, but the module does not stop startup or return a health-check failure.
- Workflow work is started with `threading.Thread(...).start()` and is not tracked, joined, retried, or persisted.
- Slack receives an immediate acknowledgement before image generation or Dropbox work completes.
- Slack signature verification is absent.
- Dropbox folder listing is not paginated.
- The catalog limit check is `if i == numproducts`, so a numeric limit currently processes `numproducts + 1` rows. The webhook's literal `generate images` command falls back to the default value of `1`.
- `CatalogWorkflow` sends a hard-coded Dropbox folder URL after processing rather than deriving a link from the configured folder.
- `ImgProService.getResult` saves the image but returns `None`.
- `GenerationWorkflow` is available as a module API but is not reachable through the current Slack endpoint; the active image-generation path is `CatalogWorkflow` with `ImgProService`.

## Local Execution

Install the dependencies listed in `requirements.txt`, provide the environment variables above, and run:

```text
python app.py
```

The local server binds to `0.0.0.0` and uses `PORT`, defaulting to `3000`. For deployment, use the Gunicorn command in the overview or the included `Procfile`.
