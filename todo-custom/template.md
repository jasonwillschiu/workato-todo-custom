# Template: Workato-only web app (agent instructions)

This file is for an AI agent. It has two jobs:

1. **Part A – Map of the moving parts.** Every asset the reference app (`Todo App`) uses, with IDs, so you never need Workato Airo or the dev API just to *find* something.
2. **Part B – Build a new app from this template.** Ordered steps, with the Workato Airo / dev API calls to use.

The reference app is a todo list. To build a different app, keep the wiring and swap the data model, the CRUD recipes and the HTML.

> Last verified against the live workspace (ANZ Presales, Development) on 2026-09-30. IDs below are for the reference app only. A new app gets its own IDs.
>
> Where a fact was tested, it is tagged *(dev API)* or *(Airo)* to show which tool it came from. See "Provenance" at the end.

---

## Part A – Map of the reference app

### The shape

```
Browser
  │  GET  https://apim.workato.com/anz-presales/todo-app/ui?api_token=…      → HTML page
  │  PATCH/POST/PUT/DELETE  …same URL…                                      → JSON CRUD
  ▼
Workato API collection "Todo App" (slug: todo-app)  ── one path: /ui, five methods
  ├─ GET    → recipe "Todo App - Serve UI"  → reads HTML from Workato File Storage
  ├─ PATCH  → recipe "Todos - List"         ┐
  ├─ POST   → recipe "Todos - Create"       │ CRUD against the
  ├─ PUT    → recipe "Todos - Update"       │ Data Table "Todos"
  └─ DELETE → recipe "Todos - Delete"       ┘
```

The UI and the API live in the same collection on the same URL, split by HTTP method. The page's own JS calls `fetch()` back to its own URL, so there is no CORS to configure.

### Asset inventory

| Piece | Reference-app value | Where to see it |
|---|---|---|
| Workspace / team | ANZ Presales, environment `Development` | – |
| Project (root-level folder) | `Todo App Custom UI`, folder id `35033143`, project id `17815807` | <https://app.workato.com/recipes?fid=35033143> |
| Data table | `Todos`, numeric id `141793`, UUID `482a0443-acce-49bb-9322-d001bd8e15ca` | <https://app.workato.com/data_tables/141793> |
| API collection | `Todo App`, id `1393729`, handle/slug `todo-app` | <https://app.workato.com/api_groups/1393729/endpoints> |
| Base URL | `https://apim.workato.com/anz-presales/todo-app/ui` | – |
| API client | `Todo Public Website`, id `1270569` (legacy client type) | API Platform → Clients |
| Access profile | `Public UI only`, id `1356466`, scoped to collection `1393729` only, auth type `token` | API Platform → Clients → the client |
| File Storage folder | `/todo-app` (created by hand once) | <https://app.workato.com/files/management?directory_path=%2Ftodo-app> |
| UI files | `/todo-app/todo-ui-v1.html`, `/todo-app/todo-ui-v2.html`, … | same as above |
| Local copies of UI files | `todo-ui-v1.html`, `todo-ui-v2.html` in the repo root | this repo |
| Full project export | `manifests/todo-app-v2.0/` (unzipped) and `manifests/todo-app-v2.0.zip` | this repo |

Recipes (all in folder `35033143`, all trigger `workato_api_platform`):

| Endpoint (id) | Method | Recipe (id) | Actions | Request | Response |
|---|---|---|---|---|---|
| `Serve UI` (`23943310`) | `GET` | `Todo App - Serve UI` (`81419527`) | `workato_files` + `workato_custom_code` (Ruby) + `return_response` | query `v` (optional string) | raw HTML, header `Content-Type: text/html; charset=utf-8` |
| `List Todos` (`23943311`) | `PATCH` | `Todos - List` (`81412451`) | `workato_db_table.get_records` + `return_response` | none | `{"records_json": "<string>"}` |
| `Create Todo` (`23943312`) | `POST` | `Todos - Create` (`81416563`) | `workato_db_table.add_record` + `return_response` | JSON `title` (required), `notes`, `completed` (boolean), `due_date` | `{"record_json": "<string>"}` |
| `Update Todo` (`23943313`) | `PUT` | `Todos - Update` (`81416601`) | `workato_db_table.update_record` + `return_response` | JSON `id` (required), `title`, `notes`, `completed`, `due_date` | `{"record_json": "<string>"}` |
| `Delete Todo` (`23943314`) | `DELETE` | `Todos - Delete` (`81416713`) | `workato_db_table.delete_record` + `return_response` | `id` (sent as a `?id=` query param by the UI) | `{"status": "deleted"}` |

Why the CRUD verbs are unusual: `GET` is taken by the UI, so **list uses `PATCH`**. Do not "fix" this.

Data table `Todos` columns (field ids are what the UI and recipes reference):

| Column | Type | Required | Field id (UUID; underscores in recipe/UI output) |
|---|---|---|---|
| Record ID | short-text (system) | – | `11fbe9a6-a16d-4d7e-86ea-afe42ec03005` |
| title | short-text | yes | `3c688cb9-f069-420d-99a9-62180137b81e` |
| notes | short-text | no | `3f29d34b-7d94-4fcb-a6a2-f3e2896642d5` |
| completed | boolean | yes | `ddc9a141-bb20-46d3-9eec-17cbe8ba9ed5` |
| due_date | date | no | `365a868f-9e51-42bd-a02a-e66ba9ced868` |
| Created time / Last modified time | date-time (system) | – | `a5612739-…` / `61aae604-…` |

The list response keys each record by field id with underscores (for example `3c688cb9_f069_420d_99a9_62180137b81e`). The UI maps them in its `F` constant. **A new data table gets new field ids, so the UI's `F` map and every recipe's field mapping must be regenerated.**

### How "Serve UI" picks a file

Recipe `81419527` has one trigger input, `v`. It has two independent `if` blocks (no `else`, see gotchas):

1. `if_latest`: runs when `v` is blank **or** equals `latest`.
   1. `workato_files.search_files` in `/todo-app` with name filter `.html`.
   2. Custom Ruby step `get_latest_file_version` takes the file list and returns the highest `vN` number found in the filenames.
   3. `workato_files.get_file_contents` reads `/todo-app/todo-ui-v{version}.html`.
   4. `return_response` 200, body = file content, `Content-Type: text/html; charset=utf-8`.
2. `if_explicit`: runs when `v` is present **and** not `latest`.
   1. `get_file_contents` reads `/todo-app/todo-ui-{v}.html` (so `?v=v1` reads `todo-ui-v1.html`).
   2. `return_response`, same as above.

The Ruby step (input `files_array` of `{name}`):

```ruby
files = input["files_array"] || []
sorted = files.sort_by { |f| (f["name"].to_s[/v(\d+)/, 1] || "0").to_i }.reverse
latest_name = sorted.first && sorted.first["name"]
version = latest_name.to_s[/v(\d+)/, 1]
{ version_integer: version }
```

There is **no** manifest or "latest" copy file. The highest `todo-ui-vN.html` in `/todo-app` is the latest. To publish a new UI version, put `/todo-app/todo-ui-v{N+1}.html` in File Storage, either by uploading in the File Storage UI (how v3 was published) or with a recipe using `workato_files.store_file` (tested, see gotcha 5). An agent can therefore automate publishing by building and running such a recipe; it cannot do it through the dev API alone. Nothing else to update. To roll back, either serve `?v=vN` or store a higher version with the old content.

The exact recipe JSON is in `manifests/todo-app-v2.0/Todo App/*.recipe.json`. Use it as the source when recreating recipes.

### Auth model (current approach)

- The collection is not anonymous: Workato API Platform authenticates every call. To make the app openable by anyone with a link, a dedicated client + access profile + token is used, and the token travels in the URL as `?api_token=…`.
- A browser navigating to a link cannot set headers, so the **initial page load must carry the token in the query string**. The same token is then needed on every `fetch()` the page makes.
- The page's JS reads `api_token` from `location.search` and appends it to each API call, so the served HTML file contains no secret. `todo-ui-v3.html` does this. **`todo-ui-v1.html` / `todo-ui-v2.html` still hard-code `TOKEN`.** Base new apps on v3:

  ```js
  const TOKEN = new URLSearchParams(location.search).get("api_token");
  ```
- Scope the access profile to this one collection only. Treat the token as a revocable site key. Never reuse it for other APIs, and never commit a real token to a public repo.
- This is the approach used today, not a fixed rule. OAuth or another method may replace it. Keep auth in one place (the token constant and the access profile) so it is easy to swap.

### Conventions the UI relies on

- One HTML file, inline CSS + JS, no build step, no external hosting.
- `BASE` = the collection URL + `/ui`; `PATCH` list, `POST` create, `PUT` update, `DELETE` with `?id=`.
- The API returns `records_json` / `record_json` as **Ruby `Hash#inspect` strings**, not JSON. The UI's `parseRubyHash()` converts `"key"=>value`, `nil` and unquoted timestamps into JSON before `JSON.parse()`. Copy it. (Alternatively, fix it server-side by returning a real JSON array from the recipe.)
- `PUT` must always send `title` **and** `completed`, because the columns are required.

### Gotchas (learned the hard way)

1. **`else` blocks fail validation** in API-Platform-triggered recipes (`"unexpected line"`). Use two `if` blocks with complementary conditions. Condition operands are Workato's vocabulary: `equals_to`, `not_equals_to`, `blank`, `present`, not `equals`.
2. **Trigger fields are nested under `request`** at runtime (`request.v`, not `v`). Recipes written via API need a matching `extended_output_schema` on the trigger or the pill will not validate.
3. **Booleans need `boolean_conversion`** (`render_input` / `parse_output`) on the pill, or the data table rejects the value (`expected type boolean, got: string`).
4. **The collection handle/slug and each endpoint's path cannot be set through the management API** (the collection PUT only takes `name` / `proxy_connection_id` / `public_access`; the endpoint PUT has no path field). Set both by hand in the UI (see B7). Endpoints created through the API land on an auto-generated path.
5. **File Storage has no dev (management) API.** Only the File Storage UI and `workato_files` recipe actions touch it. Tested on 2026-09-30 through the dev API MCP by building a throwaway recipe (Scheduler trigger, `put_recipes_start`, `post_recipes_force_run`, then `get_recipes_jobs_`):
   - **Works** *(dev API, via test recipe)*: `store_file` writes a file into an **existing** folder. Input keys: `file_path` (the folder, for example `/todo-app`), `file_name`, `content`. The job succeeded, the output was `{name, path, size}`, and `get_file_contents` read back the same size (28 bytes written, 28 read).
   - **Works** *(dev API, via the live Serve UI recipe `81419527`)*: `search_files` (lists a folder, used by Serve UI) and `get_file_contents`.
   - **Fails** *(dev API, via test recipe)*: `store_file` into a folder that doesn't exist. The job errors with `Directory not found`, so it does not create folders.
   - **Unknown:** a recipe action to create a folder. *(Airo)* says one exists in the UI ("Create directory", fields "Directory name" and "Directory path"), but its internal name could not be found. *(dev API, via test recipe)* `create_directory`, `create_folder`, `make_directory`, `create_dir` and `store_directory` were all rejected as `is invalid`. Treat folder creation as manual: create the folder in the File Storage UI. If you need it automated, build a recipe in the Workato UI with that action and read its code with `get_recipes_` to get the real name.
   - There is no confirmed delete-file action either, so test files written to File Storage stay there unless removed in the UI.
   - *(dev API)* `workato_files` is not in the dev API connector list (`get_workato_cli_v1_connectors`), so its actions can't be looked up there.
   - **Airo is unreliable for this** *(Airo, cross-checked against the dev API)*. It said there is no `store_file` (there is; it calls it "Create file") and that no recipe in the workspace uses `workato_files` (Serve UI does). Use it for hints only, and verify with a test recipe.
6. **Recipes must be started** (`put_recipes_start`) and endpoints must be active before the URL works.
7. **The data-table API needs the UUID** for `get_data_tables_` (numeric id `141793` is rejected). Recipes use the numeric id as `table_id`.

---

## Part B – Build a new app from this template

Do these in order. Each step says what to call and what to verify.

### B0. Decide the new app's inputs

Write down before touching Workato:

- App name and handle (for example `Habit Tracker` → handle `habit-tracker`).
- Data model: columns, types, which are required.
- Whether five methods on one path is enough (it is for simple CRUD). If you need more operations, add more methods or a request field like `?action=`.
- How the token will be delivered (default: `?api_token=` in the URL).

### B1. Tool conventions (Workato dev API MCP)

The `workato-dev-api-anz-presales` MCP wraps the Developer API. Quirks found on 2026-09-30:

- Every tool needs both `verb` and `path`. Their descriptions are swapped in the schema: `verb` = HTTP method (`get`), `path` = URL path.
- Paths have **no `/api` prefix**: `/users/me` works, `/api/users/me` returns "API not found".
- Tools with a trailing `_` in the name take an extra id argument (for example `get_folders_` takes `id`, `get_data_tables_` takes `data_table_id` as a UUID).
- `get_data_tables` returns a very large payload. Filter it (for example `jq` on the saved output) rather than reading it whole.
- Health check: `get_users_me` with `verb=get`, `path=/users/me`. Airo tools (`get_airo_*`, `post_airo_chat`) live on the same server.

### B2. Create the project folder

`post_folders` (a folder with no parent, or the root folder id, becomes a project). Note the returned `id` (folder) and project id.

### B3. Create the data table

`post_data_tables` in the new folder with your schema. Note the UUID **and** numeric id, and each column's `field_id`. Confirm with `get_data_tables_`.

### B4. Create the five recipes

Model each on the matching file in `manifests/todo-app-v2.0/Todo App/` (`todos_list`, `todos_create`, `todos_update`, `todos_delete`, `todo_app_serve_ui`). Use `post_recipes` with the recipe `code` JSON, replacing:

- `table_id` → new numeric id.
- Field ids in every `parameters` map and `extended_*_schema` → new field ids.
- Trigger request schema → your columns.
- Names → `<App> - List`, `<App> - Create`, and so on.

Keep gotchas 1 to 3. The **Serve UI** recipe needs no data-model change; only the folder path (`/todo-app` → `/<your-folder>`) and filename prefix (`todo-ui-v` → `<your-prefix>-v`). Its `workato_files` and custom-code actions need no connection.

Then `put_recipes_start` for each. Check `get_recipes_` for `running: true` and no `stop_cause`.

Shortcut: `post_recipes_copy` on the reference recipes (`81412451`, `81416563`, `81416601`, `81416713`, `81419527`) into the new folder, then `put_recipes_` to change the table id and field ids. This is not tested end to end; the export in `manifests/` is the safer source of truth.

### B5. Create the File Storage folder and first UI file (manual + recipe)

1. **By hand:** in <https://app.workato.com/files/management>, create the folder (for example `/habit-tracker`). Neither the dev API nor `store_file` can create a folder (tested; see gotcha 5).
2. Build the HTML from `todo-ui-v3.html`: change `BASE`, the `F` field-id map, the form and rendering for your columns, and read the token from `location.search`.
3. Put it at `<folder>/<prefix>-v1.html`: upload in the File Storage UI, or run a small recipe with `workato_files.store_file` (`file_path` = folder, `file_name`, `content` = the HTML). The folder must already exist. Recipe skeleton that worked on 2026-09-30:

   ```json
   config: [{"keyword":"application","name":"clock","provider":"clock"},
            {"keyword":"application","name":"workato_files","provider":"workato_files"}]
   code:   {"number":0,"provider":"clock","name":"scheduled_event","as":"trigger1","keyword":"trigger",
            "input":{"time_unit":"days","trigger_every":"1","timezone":"Australia/Sydney"},
            "block":[{"number":1,"provider":"workato_files","name":"store_file","as":"store1","keyword":"action",
                      "input":{"file_path":"/<folder>","file_name":"<prefix>-v1.html","content":"<html…>"}}]}
   ```

   Config entries need the `name` key or start fails with `missing adapter configuration`. Run it once with `post_recipes_force_run`, then check the job with `get_recipes_jobs_`. Stop the recipe afterwards so it doesn't run daily.
4. Keep a copy in this repo as `<prefix>-v1.html` so the repo stays the source of truth.

### B6. Create the API collection and endpoints

1. `post_api_collections` with a name and version. It gets an auto-generated slug like `habit-tracker-v1`.
2. `post_api_collections_api_endpoints` five times, one per method, all with `path: "ui"`:
   `GET` → Serve UI, `PATCH` → List, `POST` → Create, `PUT` → Update, `DELETE` → Delete. Use each recipe's `flow_id`.
3. Enable endpoints if needed (`put_api_endpoints_enable`).

### B7. Manual UI step: slug and path

In the Workato UI (<https://app.workato.com/api_groups/{id}/endpoints>) open the collection settings and change the URL slug to something short (`todo-app-v1` → `todo-app`). If the endpoint paths are not `ui`, edit each endpoint's path to `ui`. Neither field is editable via API (gotcha 4).

### B8. Create the client, token and access profile

1. Create an API client (`post_v2_api_clients` or `post_api_clients`) for the app.
2. Create an access profile scoped to **only** the new collection (`post_api_access_profiles`, `auth_type: token`). Capture the token when it is issued; it may not be retrievable later.
3. Build the shareable link: `https://apim.workato.com/<team>/<slug>/ui?api_token=<token>`.

### B9. Verify

```sh
TOKEN="…"; URL="https://apim.workato.com/<team>/<slug>/ui"
curl -i "$URL?api_token=$TOKEN"           # 200, text/html
curl -s -X PATCH "$URL?api_token=$TOKEN"  # {"records_json":"[...]"}
curl -s -X POST  "$URL?api_token=$TOKEN" -H 'Content-Type: application/json' -d '{"title":"test","completed":false}'
```

Then open the link in a browser and add, complete and delete a row. Check `get_recipes_jobs` if anything fails.

### B10. Record what you built

Update `README.md` with the new asset ids and link, and add an entry to `changelog.md`. If you built a new app rather than editing this one, put it in its own repo or folder. Do not overwrite the reference app's files.

## Checklist of things that need a human

- [ ] Create the File Storage folder
- [ ] Set the collection slug and endpoint paths
- [ ] Copy the API token when it is issued
- [ ] Decide who gets the link

## Provenance: which tool each fact came from

Both tools are on the same MCP server (`workato-dev-api-anz-presales`). "Dev API" means the management-API tools (`get_*`, `post_*`, `put_*`, `delete_*`). "Airo" means `post_airo_chat` only.

| Fact | Source |
|---|---|
| Server health and identity (`get_users_me`), Airo health (`get_airo_knowledge_bases`, returned no knowledge bases) | dev API |
| Asset IDs: recipes, endpoints, collections, folders/project, API clients, access profile, data table (numeric id and UUID) | dev API (`get_recipes`, `get_api_endpoints`, `get_api_collections`, `get_folders_`, `get_v2_api_clients`, `get_api_access_profiles`, `get_data_tables`) |
| Serve UI recipe logic (`search_files` + Ruby + `get_file_contents`) | dev API (`get_recipes` code) |
| `store_file` works into an existing folder; fails with `Directory not found` otherwise; input keys `file_path`, `file_name`, `content` | dev API (throwaway recipe: `post_recipes`, `put_recipes_`, `put_recipes_start`, `post_recipes_force_run`, `get_recipes_jobs_`). The recipe was deleted afterwards with `delete_recipes_`. |
| Folder-creating action name guesses all rejected | dev API (recipe start validation) |
| A "Create directory" action exists in the UI with fields "Directory name" and "Directory path" | Airo only. **Not verified.** |
| Airo claims no `store_file` and no `workato_files` recipes in the workspace | Airo. **Both wrong**, disproved by the dev API results above. |
| Gotchas 1 to 4 (`else` blocks, `request` nesting, boolean conversion, slug/path not API-editable) | Earlier README, from earlier testing. Not re-tested on 2026-09-30. |

Test residue: two files, `zz-temp-store-file-test.txt` and `zz-temp-store-file-test2.txt`, remain in `/todo-app` (no delete action is confirmed; remove them in the File Storage UI). They don't affect the app because Serve UI only searches for `.html`.

## Sources

- Workato dev API tools on the MCP server, checked 2026-09-30: `get_recipes`, `get_api_endpoints`, `get_api_collections`, `get_folders_`, `get_v2_api_clients`, `get_api_access_profiles`, `get_data_tables`, plus the throwaway-recipe tests for `store_file`
- Workato Airo (`post_airo_chat`), used only for File Storage questions; its answers are tagged and cross-checked in "Provenance"
- `manifests/todo-app-v2.0/` project export
- `todo-ui-v3.html`
- The previous `README.md` for gotchas 1 to 4 (recorded there from earlier testing; not re-tested on 2026-09-30)
