# CMM_streaming

A Flask based prototype that ingests uploaded videos, generates multiple bitrate variants, slices them into HTTP friendly segments, and delivers an adaptive playback experience in the browser. This document explains the full logic path, from HTTP request handling to FFmpeg processing, manifest authoring, and the browser side adaptive player.

---

## High level architecture

- **Client (browser)**: Renders `templates/index.html`, loads styles from `static/css/style.css`, executes behaviour in `static/js/main.js`. Users interact through an upload modal and a custom player.
- **Server (Flask)**: `app.py` defines routes, orchestrates FFmpeg, writes manifests, and serves media assets.
- **Processing layer (FFmpeg)**: Transcodes the uploaded source into target resolutions and produces fixed length segments.
- **Storage layout**: Every upload is stored under `media/<playlist_id>/` with `source/`, `variants/`, `segments/`, and `manifest.json` describing the asset.

The system is a classic HTTP client server application. Browsers send HTTP requests (GET, POST) to Flask, Flask responds with HTML, JSON (embedded), media files, or redirects. No persistent database is required; JSON manifests act as the catalog.

---

## HTTP routes and protocol flow

### `GET /`
1. Browser requests the home page over HTTP GET.
2. Flask reads all manifests under `media/`, sorts them by timestamp, and determines which playlist should be active (query string `?playlist=<id>` or latest upload).
3. If an active manifest exists, Flask builds a playback payload containing URLs to every variant and segment. The payload is serialized into JSON and embedded in the HTML via a `<script type="application/json" id="playerData">` tag.
4. Flask renders the `index.html` template, injecting sidebar data, summary stats, and any flashed messages.
5. The browser receives an HTTP 200 response with HTML. While parsing, it triggers additional GET requests for CSS, JS, fonts, and later for media files only when the player loads them.

### `POST /upload`
1. The upload form submits a multipart/form-data POST request containing the binary video file plus selected resolutions and segment length.
2. Flask validates the request (file presence, allowed extension, at least one resolution, numeric segment duration). If validation fails, Flask flashes an error and returns an HTTP 302 redirect back to `/`.
3. On success, Flask saves the file to `media/<playlist_id>/source/`. The playlist id contains the sanitized filename plus a UTC timestamp, ensuring uniqueness.
4. Flask calls FFprobe (through `probe_video`) to extract the source width, height, and duration.
5. For each target resolution (including the original height) Flask launches FFmpeg with the following important flags:
   - `-map 0:v:0` and `-map 0:a:0?` select only the primary video stream and the first audio stream (if present), ignoring data or subtitle tracks.
   - `-vf scale=-2:<height>` rescales while preserving aspect ratio (width is automatically computed and divisible by two).
   - `-force_key_frames expr:gte(t,n_forced*segment_duration)` enforces key frames at segment boundaries so every segment starts cleanly.
   - `-pix_fmt yuv420p` ensures broad codec compatibility.
   - `-map_metadata -1 -dn -sn` strips timecode, data, and subtitle tracks to prevent invalid streams from propagating.
6. When the variant file is ready, Flask runs a second FFmpeg command to segment it:
   - Again maps only video and audio streams, copies them (`-c:v copy -c:a copy`), resets timestamps per chunk, and writes numbered MP4 files of the chosen length.
7. Flask records metadata (sizes, segment paths, bitrate estimate) in an in memory structure.
8. Finally, Flask writes `manifest.json` with:
   - Source details (filename, dimensions, duration).
   - Segment duration, requested vs skipped resolutions.
   - Every variant plus an ordered list of segment objects (`path`, `duration`, `label`, `size_bytes`).
   - An ISO 8601 `created_at` and a numeric `created_at_ts` for sorting.
9. Flask flashes success messages, stores the new playlist id in the session, and returns `HTTP 302 Found` pointing to `/`. The redirect pattern avoids form resubmission on refresh.

### `GET /media/<path>`
- Simple static file delivery. Flask locates the requested path under `media/` and streams the file over HTTP. The browser uses standard range requests when necessary, allowing seeking.

---

## HTTP, APIs, and data exchange explained

- **HTTP methods**: GET retrieves resources (HTML, JSON, CSS, media). POST uploads data that modifies server state.
- **Request structure**: Includes verb, path, headers (content type, cookies, user agent), and optional body (the video file during upload).
- **Response structure**: Contains status codes (200, 302, 404), headers (content type, caching hints), and body payload (HTML, JSON, or binary media).
- **APIs vs pages**: The only machine readable payload is the embedded JSON. The browser never calls a dedicated JSON endpoint; instead the payload is delivered inline with HTML to keep the prototype simple.
- **Sessions and cookies**: Flask uses a secure cookie to remember `last_upload_id`. After redirect, the home route can auto select the freshly processed playlist.
- **Static assets**: Served from `/static/` (Flask built in). The browser caches CSS/JS based on standard HTTP caching semantics.

---

## Frontend logic (static/js/main.js)

1. **Modal and form management**
   - Open/close the upload dialog, update the displayed filename, and prevent submission when no resolutions are checked.
   - Upon form submission, disables the submit button to signal processing.
2. **Sidebar behaviour**
   - In responsive layouts the menu button toggles the sidebar class `sidebar--open`. When the upload modal opens on mobile, the sidebar closes to avoid overlap.
3. **Adaptive player initialization**
   - Reads the JSON payload in `playerData` and sets up player state.
   - Builds resolution chips with bitrate hints and a segment list (including a "Full video" synthetic entry).
   - Keeps track of the currently selected variant and segment index.
4. **Variant switching**
   - `setVariant` replaces the `<video>` source while attempting to maintain playback position. When a user manually chooses a resolution, auto adapt mode is disabled.
   - Segment seeking uses the `currentTime` API. Each segment button sets `adaptivePlayer.src` to the segment URL and calls `play()`.
5. **Auto adaptation**
   - Controlled by a toggle (checkbox). When active, a timer samples download throughput using the Resource Timing API and, if available, Network Information (`navigator.connection.effectiveType`, `downlink`).
   - Logic maintains counters to avoid flapping. If measured bandwidth falls below the current variant by a margin, it downgrades; if sustained above a higher quality threshold, it upgrades.
   - Resolution chips indicate state with CSS classes (`chip--active` for manual selection, `chip--auto-active` to show which variant auto mode picked).
6. **Segment continuity**
   - Listens to `timeupdate` and `ended` events. When a segment finishes, it queues the next one automatically, tracking whether the user is in full video mode or segmented mode.
7. **Playback status UI**
   - Updates `currentResolutionLabel` and `currentSourceLabel` to reflect the active stream, helping users understand what is playing.

---

## Template and layout reminders

- `templates/index.html` uses Jinja2 to inject available playlists, active summary, and the player payload script.
- The page is structured into two main columns: a sidebar with upload history and a content column containing the intro, player, and summary card.
- `static/css/style.css` provides the dark theme, responsive layout rules, chip styling, and the toggle design.

---

## Processing pipeline summary

1. User clicks "Upload a Video" → modal opens.
2. Form submit triggers an HTTP POST to `/upload`.
3. Flask validates, saves the file, probes metadata.
4. For each selected resolution:
   - FFmpeg transcodes to MP4 (H.264 + AAC) with key frames inserted.
   - FFmpeg segments the MP4 into ~`segment_duration` second chunks.
   - Metadata gets appended to the manifest list.
5. Manifest JSON is written to disk alongside the generated assets.
6. Flask flashes a success message, records the playlist id in the session, and redirects to `/`.
7. On the redirected GET, Flask loads all manifests, chooses the active one, serializes the playback payload, and renders the template.
8. Browser executes `main.js`, initializes the adaptive player, fetches variant/segment files on demand over HTTP.
9. During playback the `<video>` element issues HTTP GET requests for the selected media file. Range requests allow scrubbing without re downloading the entire resource.

---

## Key protocols and media considerations

- **HTTP/1.1**: Browser and server communicate via stateless requests. Each segment is just another HTTP response, enabling CDN proxying or caching if deployed.
- **MIME types**: Flask relies on Werkzeug to infer `video/mp4` when serving segment files, which allows browsers to play them.
- **Transport**: Even though the player uses simple MP4 files, serving them in short segments mimics adaptive streaming approaches like HLS or DASH, keeping compatibility with the native `<video>` tag.
- **Caching strategy**: Static assets can be cached aggressively. Media responses could also include `Cache-Control` headers in a production setup. Currently the prototype uses defaults.
- **Error handling**: `ProcessingError` wraps FFmpeg failures and surfaces the stdout/stderr tail to the user. The cleanup routine removes partially processed playlists to avoid corrupt state.

---

## API surface summary

| Method | Path | Description |
| ------ | ---- | ----------- |
| GET    | `/` | Render the dashboard, summary, and player payload. |
| POST   | `/upload` | Accept a multipart form upload, generate variants/segments, write manifest, redirect. |
| GET    | `/media/<path>` | Serve variant or segment files via HTTP for playback. |

Extended browser APIs used on the client side:

- `Resource Timing API` (`performance.getEntriesByType('resource')`) to measure download durations for adaptive logic.
- `Network Information API` (`navigator.connection`) to inspect downlink and effective network type when available.
- `HTMLMediaElement` API for controlling the video player (`currentTime`, `play`, `pause`, event listeners).
- `IntersectionObserver` is not currently used but can be added for lazy loading in future iterations.

---

## Local development checklist

1. Install Python 3.11+ and FFmpeg/FFprobe, ensure they are on PATH.
2. Set `FLASK_SECRET_KEY` for production runs (default is `dev-secret-key`).
3. Install dependencies (only Flask is required for this prototype):
   ```bash
   python -m pip install flask
   ```
4. Run the server:
   ```bash
   python app.py
   ```
5. Visit `http://localhost:5000`, upload a video, and test adaptive playback.

---

## Future enhancements

- Swap MP4 segments for HLS/DASH playlists to leverage native adaptive streaming implementations.
- Persist manifests in a database, attach user identities, and add authentication.
- Attach a background task queue for long running transcodes so uploads return immediately.
- Integrate actual thumbnails or preview frames for the sidebar instead of letter avatars.
- Add metrics collection for bandwidth decisions and error reporting.
