# Ads pipeline: Meta Ads x OpenRouter (ads-pipeline-meta-openrouter)

A demo app that runs entirely in Workato, with Google sign-in. A user writes an ad prompt, generates three images through OpenRouter, pulls simulated Meta Ads results from a data table, sees which ad won on CPA (and why, using CTR), then generates three variations of the winner. Sessions are saved per user and can be reopened.

It is a sibling of [`todo-oauth`](../todo-oauth/README.md): same Google client, same `@workato.com` rule, same page-is-its-own-callback pattern. It has its own project, tables, recipes and collections.

## Status

| Piece | State |
|---|---|
| Project, 4 data tables, 9 recipes, UI file | Built. Recipes are saved but **not started** |
| OpenRouter custom connector (v6, released) and connection | Built and connected (connection `22321145`). The connector's calls have **not been run** yet |
| `Models Refresh` and `Creative Generate` use the connector | Done. Neither uses the HTTP connector any more |
| UI logic | Tested in a browser against a mock of the API (both rounds, retry on a failed ad, saved sessions, branding) |
| File Storage folders `/ads-pipeline`, `/ads-pipeline/images` | Done |
| API collections, endpoints, site key, OIDC attachment | **Not created.** The dev-API tools cannot create them, so it is a GUI step |
| UI upload to `/ads-pipeline`, `API_BASE` set | **Not done** (needs the API collection's URL) |
| Google redirect URI | **Not added.** Do it last |
| Any recipe run against live OpenRouter | **Not run** |

## How it works

1. The page is served by a site-key collection (`Ads Pipeline Web`). The page signs the user in with Google and calls a second collection (`Ads Pipeline API`) with the Google ID token as a Bearer token.
2. Every API recipe decodes the token's payload in a Ruby step, allows only verified `@workato.com` addresses (401 with no identity, 403 for other accounts), and filters every read and write by that email.
3. **Round 1:** the UI creates a session, then calls `Creative Generate` three times in parallel (slots A, B, C). Each call runs the connector's **Generate image** action, saves the image to File Storage as a data URL text file, and records a row.
4. **Meta Ads:** `Meta Ads Simulate` writes one metrics row per ad into `Ads Meta Metrics`, re-randomised on every click. One ad is made to lead on both CPA and CTR, so the story is coherent.
5. **Winner:** the UI picks the lowest CPA and explains it with CTR. A winner needs no stored flag, it is computed from the rows.
6. **Round 2:** the UI pre-fills a prompt (keep the hook and visual, change one thing from the dropdown). `Creative Generate` passes the winner image to **Generate image** as a reference image when the model accepts references.
7. **Cheap models only:** the dropdown lists models where `enabled` is true in `Ads Image Models`, and `Creative Generate` rejects any other model. `Models Refresh` pre-enables a shortlist and never changes `enabled` afterwards.
8. **Limits:** 3 new sessions per user per day (Sydney time). 3 ads per round per session.

Latency: OpenRouter publishes no latency for image models (checked all 59). `Creative Generate` times each real call and keeps a running average in `Ads Image Models`, along with the cost the connector returns from OpenRouter's `usage`.

## The OpenRouter connector

Custom SDK connector `OpenRouter`, id `1365660`, [code](https://app.workato.com/custom_adapters/1365660/code). Currently v6, released.

- **Connection:** an API key sent as `Authorization: Bearer <key>` to `https://openrouter.ai/api/v1/`. The connection test calls `GET key`.
- **Generate image** (`POST images`): model, prompt, aspect ratio, resolution, output format, quality, seed, optional reference images. Returns the image as a data URL, media type, tokens and `cost_usd`.
- **Send chat message** (`POST chat/completions`, not streamed): model, prompt, optional system prompt, temperature, max tokens, top P, seed, response format. Returns the reply text, finish reason, model used, tokens and `cost_usd`.
- **List models:** the image or chat models OpenRouter lists, with prices and (image models) supported aspect ratios and reference-image support. `Models Refresh` uses it, and it is there for any other custom UI.
- **Custom action:** a free-form request for any other OpenRouter endpoint, using the same connection.
- **Model dropdowns:** the Model field on Generate image and Send chat message loads live from OpenRouter with the connection's key. A toggle lets you type a model ID instead. A recipe step that maps a model from a datapill (as `Creative Generate` does) uses the typed mode.

## Where everything lives

| Piece | Value |
|---|---|
| Project `Ads Pipeline - Meta Ads, Openrouter` | folder `35795104`, project `18253442`, [recipes](https://app.workato.com/recipes?fid=35795104) |
| Table `Ads Image Models` | id `159680`, UUID `29300ca0-69a8-4363-8abf-07ac5d3f0aea` |
| Table `Ads Sessions` | id `159681`, UUID `c7fe8d64-00ed-46dd-8523-87698706d92e` |
| Table `Ads Creatives` | id `159682`, UUID `deddf8d2-82dd-4719-944a-42116df4927f` |
| Table `Ads Meta Metrics` | id `159683`, UUID `a0bf85ea-b4c8-4abf-964f-bded68b69843` |
| OpenRouter connector | id `1365660`, adapter `open_router_connector_3905295_1792041222` |
| OpenRouter connection | `Jason's $5 / month Openrouter key`, id `22321145` |
| File Storage | `/ads-pipeline` (UI files), `/ads-pipeline/images/<user>/` (images) |
| Google OAuth client ID | same as todo-oauth: `62836436917-2drj9a39hhogs2idvfvmm7ip5tgmggun.apps.googleusercontent.com` |

### Endpoints to create

Web collection `Ads Pipeline Web` (site key):

| Method | Path | Recipe (id) |
|---|---|---|
| GET | `ui` | `Ads - Serve UI` (`82997003`) |

API collection `Ads Pipeline API` (OpenID Connect, Google token):

| Method | Path | Recipe (id) | Notes |
|---|---|---|---|
| GET | `models` | `Ads - Models List` (`82997004`) | enabled models for the dropdown |
| POST | `models/refresh` | `Ads - Models Refresh` (`82997023`) | uses the connector's List models |
| GET | `sessions` | `Ads - Sessions List` (`82997007`) | also returns runs today and the limit |
| POST | `sessions` | `Ads - Session Create` (`82997013`) | 429 at the daily cap |
| GET | `sessions/detail` | `Ads - Session Detail` (`82997015`) | query `session_id` |
| POST | `creatives/generate` | `Ads - Creative Generate` (`82997043`) | uses the connector's Generate image, set the endpoint timeout to the maximum (240s) |
| GET | `creatives/image` | `Ads - Creative Image` (`82997017`) | query `session_id`, `creative_id` |
| POST | `meta-ads/simulate` | `Ads - Meta Ads Simulate` (`82997021`) | body `session_id`, `round` |

### Data tables

- `Ads Image Models`: `model_id`, `model_name`, `price_summary`, `price_per_image`, `aspect_ratios`, `max_references`, `avg_latency_ms`, `latency_samples`, `avg_cost_usd`, `enabled`, `refreshed_at`. Global, not per user.
- `Ads Sessions`: `user_email`, `title`, `brief`, `model_id`, `aspect_ratio`, `session_day`, `stage`, `winner_creative_id`, `variable`. The Record ID is the session id.
- `Ads Creatives`: `session_id`, `user_email`, `round`, `slot`, `prompt`, `model_id`, `aspect_ratio`, `variable`, `parent_creative_id`, `image_path`, `status`, `error`, `latency_ms`, `cost_usd`.
- `Ads Meta Metrics`: `session_id`, `user_email`, `creative_id`, `round`, `slot`, `impressions`, `clicks`, `ctr`, `spend`, `conversions`, `cpa`, `is_winner` (left at its default; the winner is computed).

The workspace had 84 of 100 data tables before this project, so it now uses 88.

## Setup steps that need a person

Done: the OpenRouter API key and connection, and the File Storage folders. Still to do, in this order:

1. **Collections and endpoints** (API Platform): create the two collections and the endpoints in the tables above. Note the collection slug and the base URL of the API collection.
2. **Site key:** create a client and access profile (token) scoped to `Ads Pipeline Web` only. Copy the key when it is issued.
3. **OpenID Connect:** attach `Ads Pipeline API` to the existing OpenID Connect client from todo-oauth. Its Custom validation application (issuer `https://accounts.google.com`, `aud` is the Google client ID, `hd` is `workato.com`) then covers it.
4. **UI:** set `API_BASE` at the top of `ads-pipeline-ui-v1.html` to the API collection's base URL, upload the file to `/ads-pipeline` (File Storage never overwrites, so a change means `ads-pipeline-ui-v2.html`).
5. **Start the 9 recipes** (Airo has no start tool).
6. **Google console:** in the existing `Todo OAuth` Web client, add one authorized redirect URI, the exact page URL `https://apim.workato.com/anz-presales/<web-slug>/<path>?api_token=<SITE_KEY>`, no fragment. The JavaScript origin `https://apim.workato.com` is already there.
7. Open the page, sign in, click **Refresh models & prices**, and check that the 7 shortlisted models show in the dropdown. If the list is empty, open `Ads Image Models` and tick `enabled` on the cheap ones (see Unverified below).

## Verify

With a Google ID token from `sessionStorage` after signing in:

```sh
API="https://apim.workato.com/anz-presales/<api-slug>"
curl -s -H "Authorization: Bearer $ID_TOKEN" "$API/models"        # {"models_json":"[...]"}
curl -s "$API/models"                                               # 401
```

## Unverified (none of these has run against live Workato or OpenRouter)

- **OpenRouter calls through the connector:** the requests and responses follow OpenRouter's docs (`POST /api/v1/images` with `b64_json` in `data[0]` and `usage.cost`, `GET /api/v1/images/models`, `GET /api/v1/models`). Run **List models** and one **Generate image** and read the job output.
- **How long an SDK action can wait:** a single image can take a minute or more. If the connector action times out before OpenRouter answers, `Creative Generate` will fail, and it needs a different approach.
- **Pre-enabling the shortlist:** `Models Refresh` writes `enabled` as a constant true on new shortlist models. If the data table rejects it, tick `enabled` by hand.
- **Data table typed fields:** integers and numbers are mapped from typed Ruby output pills. Booleans are never written by the recipes except that one case.
- **Large responses:** `creatives/image` returns a base64 data URL of one image. If a model returns very large PNGs and the API platform rejects the response, ask the model for a smaller `resolution` or use a model that returns JPEG.
- **Sydney date:** the daily cap uses `now.in_time_zone('Sydney')`. If the recipe errors on it, change it to `now.strftime('%Y-%m-%d')`.
- **File Storage:** `store_file` is used with text content only (a data URL string). `ensure_dir_exists` is used for the per-user folder.

## Quirks worth knowing

- API recipes copy the todo-oauth pattern: numeric response codes (`200`, not `200 - OK`), `if` blocks instead of `else`, identity read from the `Authorization` header.
- Airo cannot start or stop recipes, create collections, or edit a running recipe. Stop a recipe in the GUI before changing it.
- The dev-API tools connected here cover MCP servers, static assets, on-prem agents and runtime connections only, so they cannot create API collections, endpoints, clients or access profiles.
- A custom connector action that fails raises an error, so a failed OpenRouter call fails the job and the endpoint returns an error. The `502` path only covers a response that has no image in it.
- Branding is browser-only: the title and logo are saved in `localStorage` and never sent to Workato.
