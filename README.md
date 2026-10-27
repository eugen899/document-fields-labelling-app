# Foliolab Document Studio

A local-first browser app for defining scalar document fields and document-classification guidance, labelling PDF/image training samples, and exporting versioned JSON manifests. It does not classify or extract production documents.

Private source repository: [Eugen-Skurtov_Swisscom/document-fields-labelling-app](https://github.com/Eugen-Skurtov_Swisscom/document-fields-labelling-app).

Git tracks application source, tests, contracts, and deployment configuration, not local databases, uploaded documents, credentials, logs, or generated build/dependency folders. A repository clone is not a backup of your document data or encryption key; preserve those separately.

## Start with Docker Compose

1. Install Docker Desktop with Compose and copy `.env.example` to `.env`.
2. To save an LLM provider token, configure the separate server encryption key as described in **LLM token encryption** below.
3. Start the app from the repository root:

   ```powershell
   docker compose --file deploy/compose.yaml up --build
   ```
4. Open `http://localhost:8080`. Only the web port is published, bound to loopback. PostgreSQL, API, and worker ports remain on the private Compose network.

Set `APP_PORT` to change the local port. Change the default local PostgreSQL password in `.env` before storing anything beyond disposable local data. Keep the app bound to localhost; it has no user authentication.

The first startup applies the Alembic migration, then starts the API and background worker. The worker needs internet access only when a user explicitly configures and requests an external LLM provider. OCR runs locally in the worker with Tesseract English/German language packs.

### Repairing a Docker log-volume permission error

If startup fails with `PermissionError: [Errno 13] Permission denied: '/var/log/foliolab/api.log'` (or `worker.log`), the API/worker user cannot write to the persistent log volume. This can happen with volumes created by an earlier image that did not prepare the log directory. The image now creates `/var/log/foliolab` with mode `0750`, owned by `appuser` (UID 10001), before switching to that non-root user.

An image rebuild alone is not a reliable repair for an existing volume's permissions. From the project root on the Docker host, stop the log writers and run a one-off ownership repair:

```powershell
docker compose --file deploy\compose.yaml stop api worker
docker compose --file deploy\compose.yaml run --rm --no-deps --user 0:0 --entrypoint sh api -c "chown -R appuser /var/log/foliolab && chmod 0750 /var/log/foliolab"
```

The repair works with the existing image, preserves log contents, and changes only the log directory and its files; it does not touch the PostgreSQL data volume. Only this one-off helper runs as root. The API and worker continue running as `appuser`; do not change their normal runtime user or use world-writable permissions.

With the API and worker still stopped, verify that their normal non-root user can open both log files:

```powershell
docker compose --file deploy\compose.yaml run --rm --no-deps --entrypoint python api -c "from app.diagnostics import configure_logging, shutdown_logging; configure_logging('api'); configure_logging('worker'); shutdown_logging()"
```

If either command fails, keep the services stopped and investigate its error before proceeding. After the check succeeds, restart the application (rebuild when using the updated Dockerfile):

```powershell
docker compose --file deploy\compose.yaml up --build -d
docker compose --file deploy\compose.yaml ps
```

Do **not** run `docker compose down --volumes` to fix this error: it would also delete the document database.

### Repairing nginx HTTP 500 on the homepage

If the web log reports `rewrite or internal redirection cycle while internally redirecting to "/index.html"`, nginx is falling back to an index file it cannot locate. An earlier configuration omitted the static document root. The corrected `frontend\nginx.conf` explicitly sets `root /usr/share/nginx/html;` and `index index.html;`, matching the frontend Dockerfile's build-output destination.

After updating the VM's source files, rebuild and recreate only the web service. This does not restart the API/worker or change database/log volumes:

```powershell
docker compose --file deploy\compose.yaml up --build -d --no-deps --wait web
docker compose --file deploy\compose.yaml ps
curl.exe --fail --head http://127.0.0.1:8080/
curl.exe --fail --head http://localhost:8080/
```

Use your configured port if it differs from 8080. The web service now has its own HTTP homepage health check; a healthy API alone does not establish that the frontend can be served. If the rebuild or health check fails, inspect `docker compose --file deploy\compose.yaml logs --tail 60 web` rather than deleting volumes.

## Local development without Docker

Python 3.12 or newer and Node 24 are recommended. PostgreSQL is the production/local-container database; the API defaults to SQLite for convenient single-user development and tests.

In separate PowerShell terminals:

```powershell
python -m pip install -r backend\requirements.txt
Set-Location backend
python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

```powershell
Set-Location backend
python -m app.worker
```

```powershell
Set-Location frontend
npm ci
npm run dev -- --host 127.0.0.1
```

The development frontend proxies `/api` to `127.0.0.1:8000`. The API trusts the default Vite origins `http://localhost:5173` and `http://127.0.0.1:5173`; if Vite uses another port, set `ALLOWED_ORIGINS` to that exact frontend origin before starting the API. Install Tesseract separately and ensure English (`eng`) and German (`deu`) language data are available; if OCR cannot run, rendered pages remain available for manual labelling and show the OCR error.

### Upgrading for document classification

Restart the API and worker, then refresh the browser. Keep the existing database and encryption key. The local development startup creates the three new classification tables without altering existing extraction tables or reprocessing documents. Restart the worker in its existing configured terminal to preserve its Tesseract PATH and database settings.

Alembic-managed deployments use the additive `c102_classification` migration. Back up the database, stop the old API/worker, and apply `python -m alembic upgrade head` from `backend` before starting the updated services. Compose applies migrations in the API startup command before starting the API and worker. Do not initialize a new database or replace `.env` to activate this feature.

## LLM token encryption

Your provider **API key** goes into the browser's Authentication token field. `LLM_ENCRYPTION_KEY` is a **different, server-only key** used to encrypt that token in the database. Saving the provider token requires both; entering the provider key alone does not configure server encryption.

Generate the encryption key **once**, after installing the backend dependencies:

```powershell
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

Put the generated value in `LLM_ENCRYPTION_KEY=...` in `.env` at the project root (`document-fields-labelling-app\.env`), not in `backend\.env` or a `VITE_` variable. Preserve any existing key and other settings. Both the API and worker read this key from the root `.env` even when started from `backend`; a process environment variable takes precedence. Only the encryption key uses this local `.env` reader; other local server configuration still uses process environment variables. Compose supplies the key to both containers through their environments.

Restart the API and worker after configuring them (or recreate the Compose services), then save/test the connection again. If saving fails, the UI keeps the entered provider token so you can retry. Back up `.env` securely and preserve this same encryption key across restarts and database restores. Never commit or share it. Generating a replacement key will not decrypt tokens saved with the previous key; restore the original key instead. Local annotation does not need this key.

## LLM provider transport

Both **Test connection** and description generation append `/chat/completions` to the saved API base URL, send bearer authorization and `Accept: application/json`, and allow **600 seconds for the complete HTTP request**, including uploading the image and reading the response. The endpoint convention, saved URL/model, and encrypted token are unchanged.

No provider-specific API-version header is added. For the Swisscom `/internal-all-models/v1/chat/completions` endpoint, a controlled comparison returned HTTP 404 with `SCS-Version: v1` and HTTP 200 without it, with the same saved credentials, model, payload, and Windows transport. The earlier hard-coded version header has therefore been removed from both connection testing and generation.

On Windows, provider calls use .NET `HttpClient` through the Windows-only `pythonnet` dependency. This matches the UiPath activity's `WebRequest.DefaultWebProxy` with `CredentialCache.DefaultNetworkCredentials`: current-user Windows proxy settings (including automatic proxy configuration when configured) and Windows credentials for proxy authentication. The provider itself receives only the configured bearer token, not Windows credentials. HTTP redirects are not followed automatically. Windows uses its default proxy/bypass settings rather than `HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY` environment overrides.

Install updated backend requirements and restart **both the API and worker** to load this transport. Windows needs .NET Framework 4.7.2 or newer (normally already installed); no .NET SDK is required. A missing dependency/runtime produces an explicit error, not a silent fallback to a different proxy. Run under the Windows account whose proxy configuration works in UiPath.

Linux/Docker keeps HTTPX, its normal proxy environment configuration, and the same 600-second overall deadline. A Linux container does not inherit the host's Windows proxy discovery or credentials. The web container's API response timeout is 660 seconds so it does not interrupt the connection test before the provider deadline.

## Workflow

1. Create a project, then a document type and choose OCR languages.
2. Upload PDF, PNG, or JPEG samples. Samples receive monotonically increasing numbers. PDF pages are rendered upright and OCR is run in the background.
3. Open a sample, create shared field definitions, choose Value/Anchor/Label mode, and draw regions on any page. At **100% zoom**, the page is centered at **75% of the viewer's available width**, including rotated pages, and its aspect ratio determines the viewer height. Zoom ranges from **50% to 400%** of this baseline size. In Select mode, drag the blank page area to pan when zoomed in; dragging an existing annotation moves it instead. **Fit page** restores the 100% view and recenters the document. These controls also work when OCR is unavailable. Review OCR text and correct it; mark each absent field explicitly.
4. Finish a sample after every current field is marked present or not present. Required missing fields need a reason. A change to field type/required status reopens coverage; evidence changes invalidate the sample's completion.
5. Download the document-type JSON schema after at least one current sample is finished. The contract and an example are in `contracts`.
6. Optionally configure an OpenAI-compatible vision model in Settings. Test connection saves the current settings first, including a newly entered token. Manual Refresh sends the full page containing the field's saved value annotation, regardless of the page currently displayed. The field panel identifies the page that will be sent; Refresh is disabled until a value is annotated. Automatic suggestions use the same annotated-page rule and are off until opted in. Suggestions are review-only and cannot overwrite descriptions without approval.

The viewer is no longer constrained by browser height: at 100%, its height accommodates the whole current page plus padding. Scroll the browser to see the bottom of a tall page. Page changes, rotation, and viewer-width changes recalculate this baseline; zooming keeps that baseline height and lets you scroll or pan inside the viewer instead. Padding and reserved scrollbar space are excluded from the available width.

On desktop, the fields sidebar stays pinned below the app header as the browser scrolls the document. Its field list scrolls independently, so expanding or scrolling fields does not move the document. Sidebar controls and save status remain visible while the field list scrolls. Both scroll areas are keyboard-focusable. Narrow screens retain stacked panels: the viewer still uses page-derived height, and the field list has its own bounded scroll area.

Only the **selected field's value, anchors, and labels** are overlaid on the current page; no overlays appear until a field is selected. Click a field's name to select and open it without collapsing other fields. The selected card has a teal border and tinted header, and interacting with a field's controls also selects it (including keyboard focus). Use the separate chevron to expand or collapse details. Several fields can remain expanded, but only one is selected. Selecting a field does not change the current page; its value and region links navigate to their annotated pages.

Annotation rectangles have no permanent captions. Hover over a rectangle or its resize handle to see only its type: **value**, **label**, or **anchor**. These browser-native tooltips do not scale with document zoom; field IDs remain in the sidebar rather than appearing on the image.

Uploads are limited to 50 MiB, 100 pages per document, and 40 megapixels per decoded page/image by default; the server limits can be adjusted through environment variables. The app stores original uploads and rendered pages as PostgreSQL binary data. Back up the database, including its binary data, and preserve the Fernet key separately. Restoring the database without the key leaves document data intact but requires re-entering the LLM token.

### Document classification

Open a sample and choose **Classification** in the right pane. **Extraction** retains the existing field editor and its expanded cards. The tabs have independent overlays; classification cues do not need extraction fields and do not change a sample's extraction completion status.

1. Write a **Classification description** (up to 100 words), enter negative classification cues, and choose **Add negative cue** to build the list. Use the primary **Save classification** button in the fixed sidebar footer to persist the description/list. Saving also includes any text still in **New negative cue**, so the last item does not need a separate Add click. Up to 20 unique negative cues of 500 characters each are supported. These settings are shared across all samples of the document type, separately from its overview description.
2. Choose **Mark positive cue**, then draw around identifying text, a logo, or another visual feature on any page. **Stop marking cues** returns to moving/resizing existing rectangles. These geometry changes save immediately. Cue links navigate to their page; **Edit cue** lets you correct OCR text and add a visual-feature note, followed by the same **Save classification** button. Remove cues from the list when needed. Hovering a classification rectangle shows `positive cue` without scaling its tooltip.
3. Optionally choose **Generate classification description**. It sends each distinct page containing positive cues in this sample once, together with the saved classification description, negative cues, cue text/notes, and normalized regions. Pages without cues and images of other samples are not sent. The disclosure lists the source pages. No generation runs automatically on Finish.
4. Review the proposed description and choose **Use classification suggestion** or **Dismiss classification suggestion**. It never replaces your description without acceptance. Queued/processing/failed/cancelled status is durable; **Reload classification / check status** refreshes it without submitting a duplicate request. Source edits, page-image changes, or provider changes prevent stale suggestions from being applied.

AI classification uses the existing encrypted connection, native Windows proxy transport when applicable, 600-second deadline, and up to three worker attempts. The **combined encoded PNG bytes for all transmitted pages are capped at 8 MiB (8,388,608 bytes), before Base64 encoding**, or a lower configured `MAX_LLM_PAGE_BYTES`. Pages already within their allocated budget are unchanged; larger request copies are optimized and reduced in roughly 10% dimension steps. Saved images, OCR, and annotation coordinates are untouched. Base64 and JSON add overhead, so the complete HTTP request can exceed 8 MiB and a provider with a lower request limit may still reject it.

The save button, save status, and **Discard edits** stay outside the scrolling classification form. Description, negative-cue, and cue-text edits require explicit saving; rectangle changes save immediately. Save failures and validation errors appear beside the save controls, and drafts remain intact for correction or retry. Unsaved edits are guarded when leaving the tab or page. Save or discard them before drawing, generation, or suggestion acceptance. Revision conflicts require reloading classification; discard/reload only after preserving any desired local edits.

### Project classification export

On the **project overview listing its document types**, choose **Download classification JSON**. The download is `project-classification.json`, separate from each type's extraction-schema download.

The versioned contract is `contracts\classification-schema-v1.schema.json`. Each export is validated against it and includes the project, every document type's saved classification description and negative cues, and every sample's positive cues, OCR/corrected text, notes, page references/hashes/dimensions, and normalized rectangles. Empty types and samples without cues remain included. All saved samples are included regardless of extraction completion; only committed descriptions are exported, not pending AI suggestions.

The top-level `kind` is `document-classification`, `schema_version` is `1.0.0`, and `export_metadata.sample_scope` is `all-saved`. Coordinates use the same normalized, canonical-upright, top-left convention as extraction. Images, credentials, and background-job details are excluded. Filenames, cue text, and notes may still be sensitive. A downstream classifier needing reference images must obtain them separately.

### PDF rendering and page-image size

New PDF uploads and future processing retries target **300 DPI**. Each page is encoded as an optimized PNG and checked against `MAX_LLM_PAGE_BYTES`, which remains **8 MiB (8,388,608 bytes)** by default. If it is too large, the original PDF page is rendered again at 90% of the previous DPI: **300, 270, 243, 218.7, ...**, until the PNG fits. The byte limit applies to the encoded image, not the PDF file or the larger Base64/JSON request.

The existing resource safeguards still apply before bitmap allocation: `MAX_IMAGE_PIXELS` defaults to 40 megapixels and `MAX_RENDER_EDGE` now defaults to 10,000 pixels, allowing normal paper sizes to reach 300 DPI. Very large pages or stricter operator limits can start below 300 DPI. An impossible byte limit produces an explicit processing error rather than an endless reduction loop.

The final raster is stored and used for OCR, viewing, and LLM requests; its PNG metadata records the chosen DPI. Worker `page.rendered` events record the DPI, pixel dimensions, and encoded byte size without document content. Page image URLs include the image hash so an explicitly retried conversion does not leave an old browser-cached image visible.

**Existing stored pages are not regenerated automatically.** Their images and annotations remain unchanged unless processing is explicitly retried. Uploaded PNG/JPEG files retain their existing orientation/pixel normalization; the new DPI reduction policy applies to PDF conversion only.

### Description generation status

**Refresh description** queues work for the same background worker that processes documents. The API connection test runs separately and uses a tiny image; passing it does not guarantee a full-page generation request will succeed.

Each field shows its latest job's queued, generating, completed, failed, or cancelled state, including the attempt count and any error. Status survives leaving and reopening the sample, and automatic suggestions use the same status display. Active requests disable Refresh; **Check suggestion status** reloads the existing job without requesting another generation.

The workspace polls only while document processing or description jobs are active and stops when they finish. Successful suggestions remain subject to approval. Failed status requests show an error and can be retried with **Check suggestion status** or by reopening the sample.

A queued job needs a running `python -m app.worker` process connected to the same database as the API. A `WriteTimeout` means sending the request to the provider timed out: check the network/proxy and provider image upload limits. A `ReadTimeout` means the provider did not respond in time. A `TimeoutException` means the overall request deadline expired; it does not identify whether upload or response handling stalled. Generation allows 600 seconds per request and up to three worker attempts, so reaching a final result can take approximately 30 minutes plus queue/processing overhead. Description-job leases last 660 seconds, allowing the HTTP deadline plus cleanup time without premature recovery; document-processing leases retain their five-minute duration and per-page renewal. Retry only after addressing the reported error.

## Persistent diagnostic logs

The API and worker write JSON-line logs automatically on startup. On Windows the default location is `%LOCALAPPDATA%\Foliolab\Logs`, normally `C:\Users\<username>\AppData\Local\Foliolab\Logs`:

- `api.log`: startup/shutdown, request IDs, route templates, HTTP status codes, timings, and LLM connection-test traces.
- `worker.log`: job IDs, attempts, outcomes, exception chains, and description-generation HTTP traces.

Restart existing API and worker processes after installing this change; an API running with `--reload` reloads automatically, but the worker does not. Restart the worker in its existing configured terminal so it retains its Tesseract PATH and database settings. No LLM call is made merely to initialize logging.

```powershell
Get-ChildItem "$env:LOCALAPPDATA\Foliolab\Logs"
Get-Content "$env:LOCALAPPDATA\Foliolab\Logs\worker.log" -Tail 50
Get-Content "$env:LOCALAPPDATA\Foliolab\Logs\api.log" -Tail 50
```

Each file rotates at 5 MiB and retains three backups (`.log.1` through `.log.3`): approximately 40 MiB total across the API and worker. Logs append across restarts. This supports the app's single API process and single worker; do not run multiple instances writing to the same role file. An unwritable log directory is a startup error rather than a silent fallback.

HTTPX traces record `connection.connect_tcp`, TLS, request-header/body transmission, and response-header/body stages. Windows .NET traces instead record a combined `dotnet.send_request` stage covering upload and the complete response; they do not distinguish TCP/TLS/upload/response sub-stages. Each available stage has completion/failure events and timings. `llm.request.failed` records the failing stage and exception chain (including .NET inner exceptions and numeric error codes); `job.failed` links it to the worker attempt. Metadata includes the transport/proxy mode, provider hostname/port, configured timeout, serialized request/image sizes, and whether proxy environment settings are present. Request size is the intended payload size, not a count of successfully transmitted bytes. Proxy mode describes configuration, not the effective route; environment-proxy presence neither proves proxy use nor describes the Windows default proxy.

Persistent logs exclude authorization headers/tokens, full URLs and query strings, request/response bodies, filenames of uploaded documents, OCR text, image bytes, and raw exception messages. Exception details retain types, code-file/line/function frames, and available OS/TLS error codes, but not source text or local variables. Logs still contain hostnames, internal record IDs, and local log paths, so review them before sharing. Third-party debug logging is not enabled.

Previously, only job status and the latest error summary were stored in the database; HTTP access logs and worker tracebacks were console-only unless redirected externally. New traces cannot reconstruct an old failure. Uvicorn output from before application startup and browser-console messages are not copied into these application logs.

To override the directory, set an **absolute** `APP_LOG_DIR` in the process environment before starting the API and worker (it is not read from the local root `.env`). Linux defaults to `$XDG_STATE_HOME/foliolab/logs` or `~/.local/state/foliolab/logs`. Compose uses `/var/log/foliolab` in the shared `documentlab-logs` named volume so logs survive container recreation; it is separate from the PostgreSQL data volume. For example:

```powershell
docker compose --file deploy\compose.yaml exec worker tail -n 50 /var/log/foliolab/worker.log
```

## Development checks

```powershell
npm.cmd --prefix frontend run lint
npm.cmd --prefix frontend run build
python -m pytest
```

The frontend lint command uses Oxlint. Backend tests use a temporary SQLite database; run API and worker integration checks against PostgreSQL before release. Docker image builds, Tesseract-in-container OCR, and backup/restore require Docker Desktop.

For the workspace's Chromium browser regressions:

```powershell
Set-Location frontend
npx.cmd playwright install chromium
npm.cmd run test:browser
```

These tests start and stop an isolated Vite server at `http://127.0.0.1:5197` and mock API responses with synthetic documents; no running backend, real documents, database, or LLM connection is used. They cover opening/reopening a sample, delayed page processing, 75%-width sizing with page-derived height, viewport resizing, rotation, zoom up to 400% without changing the baseline viewer height, drag-panning to page edges, and annotation coordinates after zooming/rotating/panning. They also cover the sticky sidebar during browser scrolling, independent field/zoom scrolling, selected-field-only overlays with multiple expanded cards, keyboard selection, page navigation and editing overlapping annotations, late description success/failure, persisted status, cancellation, status-request recovery, and stopping idle polling without submitting duplicate generation requests. Classification scenarios cover keyboard tabs, creating/moving/resizing cues, multi-page evidence, text/negative-cue editing, AI failure and acceptance, unsaved-edit recovery, documents without extraction fields, and project JSON download. Backend classification checks cover shared persistence, export scope, image reduction without stored-image changes, stale/cancelled requests, and additive migration.

## Schema and privacy notes

- Schema coordinates are normalized `[0,1]` rectangles relative to canonical upright page images, with a top-left origin.
- Export contains field definitions and evidence from completed/current samples, including text and rectangles; it does not embed images. Filenames, corrected text, and annotations may be sensitive.
- Sample hashes and page IDs identify local evidence but do not provide remote image access.
- OCR runs locally. LLM requests send only the chosen page and relevant field context. Provider tokens are encrypted server-side and never returned from the settings API.
- JSON Schema manifest version starts at `1.0.0`; this is guidance for a future extraction agent, not an extraction API.
