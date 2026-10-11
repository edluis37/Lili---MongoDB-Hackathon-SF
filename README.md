# LILI - MongoDB 
# Lili · the Explorer!
https://sentiai.pro/mongo_h/
An adaptive discovery assistant for finding interesting places, local communities,
and professional opportunities. Built by Ed Amaya with Python and MongoDB Atlas,
and a browser interface for text and realtime voice.

**Example:** “I have two hours in Lisbon. Help me discover somewhere quiet and
meet people working in design.”

Lili’s main experience is a conversation. Her admin dashboard supports operations;
it is not the core discovery product.

## Recursive harnessing

Lili adapts her runtime configuration from feedback instead of only saving a
user’s preferences. MongoDB holds the policy versions and their outcomes.

```text
Recommendation → feedback → validated policy change → trial recommendation
                                                        ↓
                                             retain or roll back
```

| Feedback | Harness adaptation |
| --- | --- |
| Quieter places | Prioritise quieter venues and off-peak visits |
| Lower cost | Prioritise free or inexpensive options |
| More unusual | Seek distinctive local discoveries |
| Verify current details | Enable existing text-chat web-search routing, when globally allowed |
| Shorter answers | Request at most three short options and reduce recent context to six messages |
| Helpful | Retain the trial after a recommendation has used it |
| Worse | Restore the preceding policy |

Changes use an allowlisted mutation library, fixed guardrails, and revision-checked
MongoDB updates. Only one trial runs at a time; stale feedback cannot overwrite a
newer policy. The primary Lili prompt in `app.py` remains authoritative.

This is **bounded configuration adaptation**, not arbitrary self-rewriting code.
Validation checks policy settings; user feedback measures perceived usefulness.
Retention counts are not benchmark scores or proof of improved recommendation
quality. Automated A/B evaluation is not implemented.

## Current capabilities

- Conversational place and opportunity discovery.
- Realtime voice using instructions supplied by the Python backend.
- Three discovery starters in the Lili interface.
- MongoDB-backed conversation snapshots, explicit preferences, and voice archives.
- Per-session policy trials, audit events, retention, and rollback.
- Read-only MongoDB diagnostics and command-line monitoring.
- Existing account, document, billing, and admin features backed by MySQL.

## Architecture

| Component | Responsibility |
| --- | --- |
| `app.py` | Flask routes, primary Lili prompt, AI requests, legacy integrations |
| `explorer_mongo.py` | MongoDB client, session memory, preferences, voice records |
| `explorer_harness.py` | Versioned policies, feedback, validation, rollback |
| `static/js/ajax-chat.js` | Text chat and response revision events |
| `static/js/explorer-feedback.js` | Feedback controls beneath recommendations |
| `static/js/realtime-voice.js` | Browser voice connection and transcripts |
| `templates/index.html` | Main discovery experience |
| `static/css/lili-explorer.css` | Lili’s explorer theme |
| `check_mongo.py` | Connectivity, authentication, read-access and transport diagnostics |
| `monitor_mongo.py` | Counts, latency, and recent activity metadata |
| `passenger_wsgi.py` | cPanel Passenger entry point |

### MongoDB collections

| Collection | Stored data |
| --- | --- |
| `explorer_memory` | Conversation snapshot, explicit preferences, update time |
| `explorer_voice_turns` | Session-scoped voice transcripts with deduplicated turn IDs |
| `explorer_harness` | Active policy, revision, trial, last 50 events, outcome counters |

Memory is keyed by a server-generated identifier in the Flask browser session,
not by a submitted display name or email. Clearing the session or logging out
starts a new explorer identity. Cross-device account memory is not implemented.

The memory store keeps at most 20 messages, while current chat routes cap their
session history at 10. The harness has a default ceiling of 12 messages, therefore
receives at most the available 10; the short-answer policy reduces that to six.

MongoDB supplements the existing MySQL application. It does not replace its
accounts, billing, admin reports, analytics, or document tables.

## Setup

Use a Python environment compatible with the dependencies in `requirements.txt`.

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Create a private `.env` alongside `app.py`, or configure equivalent cPanel
environment variables. Use `.env.example` as a starting point; preserve any
existing configuration.

```dotenv
MONGO_URI=mongodb+srv://USERNAME:PASSWORD@YOUR_CLUSTER.mongodb.net/?appName=AtlasExplorer
MONGO_DATABASE=atlas_explorer
```

`MONGODB_URI` and `MONGODB_DATABASE` are also supported and take precedence over
`MONGO_URI` and `MONGO_DATABASE`. Existing process environment variables take
precedence over `.env`. Avoid stale duplicate settings. URI-encode special
characters in credentials.

In Atlas, allow the hosting server’s outbound IP and grant the database user the
required read/write access. For the hackathon, use the provided Atlas Sandbox.

### Other application configuration

The full app also requires its existing MySQL schema, AI credentials, and the
configuration for any mail or billing features you use. MongoDB setup alone does
not provision those services.

**Before deployment or making the repository public:** remove and rotate embedded
credentials in `app.py` and the connection-test script `mdb.py`. The current code
still contains hardcoded service/authentication settings; setting corresponding
environment variables does not automatically replace assignments that do not read
them. Move these settings to private configuration, including a strong Flask
session-signing secret. Do not publish `.env`.

Optional settings already read from the environment include `OPENAI_MODEL`,
`OPENAI_MAX_OUTPUT_TOKENS`, `OPENAI_PROMPT_CACHE_KEY`,
`OPENAI_WEB_SEARCH_ENABLED`, `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, and
`STRIPE_WEBHOOK_SECRET`.

For local development after configuration:

```bash
python app.py
```

The development server listens on port 5000. On cPanel, use the configured Python
application with `passenger_wsgi.py` as the startup file and `application` as the
entry point. Install dependencies in that application’s environment, upload the
Python modules plus `templates/` and `static/`, and restart the application.
The current Passenger file imports `imp`; it must be modernised before using a
Python runtime that no longer provides that module.

## Verify MongoDB

Run inside the same Python environment as the deployed application:

```bash
python -u check_mongo.py
```

This checks connection/authentication and collection read access. It does not
verify writes. On failure it reports a safe diagnostic category and probes Atlas
DNS, TCP, and TLS connectivity. Credentials and document contents are not printed.

For monitoring:

```bash
python monitor_mongo.py
python -u monitor_mongo.py --watch 30
python monitor_mongo.py --limit 10
```

Monitoring reports ping latency, conversation-record and voice-turn counts,
activity during the previous 24 hours, and recent timestamps. It does not display
message content or policy outcomes. Stop watch mode with Ctrl+C.

## Hackathon demo

1. Start a text conversation with Lili and request a discovery recommendation.
2. Choose **Quieter places** beneath the response.
3. Ask for a fresh recommendation; the revised policy now prioritises quiet options.
4. Choose **Helpful — keep change** to retain it.
5. Try **Shorter answers**, request another recommendation, then choose
   **Worse — undo change** to demonstrate rollback.
6. Inspect `/api/explorer/harness` in the same browser session to see policy
   revisions, before/after changes, and recorded outcomes.

New voice calls inherit the adaptive discovery instructions. Existing calls do
not refresh their policy automatically. Voice has no live web-search execution,
uses a separate context window, and does not replay archived transcripts. Use
text chat for the complete feedback/evaluation demo.

## Explorer API

All endpoints use the current browser’s Flask session.

| Method | Route | Purpose |
| --- | --- | --- |
| GET / POST | `/api/explorer/preferences` | Read or update explicit preferences |
| POST | `/api/explorer/voice-turn` | Archive a voice turn |
| GET | `/api/explorer/harness` | Inspect the current policy and events |
| POST | `/api/explorer/feedback` | Submit a feedback signal and response revision |

Preference fields are `destination`, `dates`, `interests`, `field`, `budget`,
`accessibility`, and `pace`, each a text value of at most 500 characters. Preferences
are not automatically extracted from conversations and have no dedicated editor yet.

Example feedback body:

```json
{"signal": "too_crowded", "revision": 0}
```

Use the actual revision from the recommendation, not a fixed value. Supported
signals are `too_crowded`, `too_expensive`, `too_generic`, `outdated`, `too_long`,
`helpful`, and `worse`.

## Tests

```bash
python -m unittest discover -s tests -v
node --check static/js/ajax-chat.js
node --check static/js/realtime-voice.js
node --check static/js/explorer-feedback.js
```

Tests cover memory/session isolation, error handling, worker client reuse,
policy validation, context/tool routing, stale feedback, retention, and rollback.
They do not call paid AI services or establish end-to-end browser correctness.

## Known limitations

- Discovery-policy mutations are deterministic and feedback-driven.
- The primary prompt still includes placeholder links and contact fields to resolve.
- MySQL remains required for legacy features; its admin dashboard is not a MongoDB
  health monitor.
- MongoDB outages let ordinary chat continue with fallback behaviour; memory may
  not be saved. Explicit memory requests return an error.
- Persistent records contain personal conversation data. Configure appropriate
  access, retention, and deletion controls before production use.

More detail: [MongoDB setup](MONGODB_SETUP.md) ·
[Recursive harness demo and scope](RECURSIVE_HARNESS.md).

## License

This project is proprietary software. All rights reserved by Ed Amaya.
