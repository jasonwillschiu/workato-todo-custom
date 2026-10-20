# 0.5.1 - Update: Ads pipeline uses the OpenRouter custom connector
- New OpenRouter custom connector (Generate image, Send chat message, List models, custom action, live model dropdowns).
- Models Refresh now uses List models and Creative Generate uses Generate image, so neither needs the HTTP connector. Removed the leftover chat test step from Models Refresh.
- README updated. API collections, endpoints, the site key and the Google redirect URI are still manual steps.

# 0.5.0 - Add: Ads pipeline (Meta Ads x OpenRouter)
- New /ads-pipeline-meta-openrouter app: generate 3 ads through OpenRouter, simulate Meta Ads results, pick the winner on CPA (CTR explains it), then generate 3 variations of the winner.
- Workato project, 4 data tables and 9 recipes built; UI v1 tested against a mock API. Collections, endpoints, the OpenRouter connection and the Google redirect URI are still manual steps (see the README).

# 0.4.4 - Fix: Recipes would not start
- Trigger response codes in the four API recipes were set to "200 - OK" text, so the 200 return step was invalid.
- Set them back to numeric codes and documented the rule.

# 0.4.3 - Add: Browser-side domain check in UI v4
- UI v4 blocks accounts that are not verified @workato.com addresses before any API call.
- Look-alike domains such as notworkato.com and workato.com.evil.io are rejected.

# 0.4.2 - Add: Access limited page in UI v3
- Show a full-screen page when Google sign-in works but the app refuses the account (403, or 401 on a live token).
- Expired tokens still return to the sign-in screen.

# 0.4.1 - Update: Limit todo-oauth to workato.com
- API recipes return 403 unless the token email is a verified @workato.com address.
- Recipes read the caller from the Authorization header instead of the jwt_payload pill.

# 0.4.0 - Add: Google sign-in todo app
- New /todo-oauth app: per-user todos, Google login via OpenID Connect, separate Workato project, table, recipes and collections.
- Ledger-style UI with a sign-in screen; the page is its own OAuth callback.
- Recipes read the caller's email from the API trigger context and filter every read and write by owner.

# 0.3.0 - Update: Moved todo-custom to folder
- Moving to a /todo-custom folder for the initial example app

# 0.2.0 - Add: Versioned UI publishing
- Serve the highest numbered UI file or pin a version in the URL.
- Add a reusable Workato app template and project export.
- Read the API token from the page URL in UI v3.

# 0.1.0 - Initital commit
- first of some
