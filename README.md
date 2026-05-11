#Important notice: Please check portfolio https://portfolio-8ge.pages.dev/ for updates and project details before complete files or demos are uploaded here.
# Local Agent System

A local FastAPI backend plus a WhatsApp bridge that forwards allowlisted commands to a bounded manager/worker pipeline.

## Operating modes

- **Chat**: the manager answers directly, with no worker and no tool call.
- **Plan**: the manager returns a structured plan and the orchestrator dispatches specialist workers.
- **Refuse**: the manager returns a short safe refusal or alternative.

## Architecture

- `prompts/soul.md`: permanent behavior contract.
- `prompts/manager.json`: editable manager scenarios for routing, refusal, and final response composition.
- `prompts/worker.json`: editable worker scenarios for `news_agent`, `mail_agent`, `calendar_agent`, and `task_agent`.
- `app/core/prompts.py`: prompt loader with live file reloading.
- `app/core/context.py`: request-scoped context with `request_id`, `source_id`, `channel`, `route_mode`, and branch metadata.
- `app/utils/logger.py`: structured logging with request/source/mode/branch scoping.
- `app/orchestrator.py`: control plane that routes, plans, dispatches, validates, persists, and finalizes.
- `app/memory/sqlite_store.py`: exact state plus graph-style run nodes.
- `app/memory/vector_store.py`: semantic memory with Chroma fallback.
- `app/llm/manager.py`: route manager and final response composer.
- `app/llm/worker.py`: narrow worker executor with bounded tool loops.

## Memory model

- **SQLite** stores exact facts, deadlines, tasks, calendar events, runs, run events, and graph nodes.
- **Chroma** stores semantic summaries for retrieval.
- Each WhatsApp chat id is treated as a separate `source_id`.

## Agent behavior

- `news_agent`: fetches current YouTube + web tech news and returns compressed summaries.
- `mail_agent`: reads Gmail through the Gmail API, summarizes the last 24 or 48 hours, flags Google Classroom mail, and extracts deadlines.
- `calendar_agent`: reads today/tomorrow calendar events and creates new events only when requested.
- `task_agent`: manages local tasks and stored deadlines.

## Security boundaries

- Gmail access is read-only through `https://www.googleapis.com/auth/gmail.readonly`.
- Calendar access is limited to `https://www.googleapis.com/auth/calendar.events`.
- The code only calls `messages.list` / `messages.get` for Gmail, and `events.list` / `events.insert` for Calendar.
- There is no code path for Gmail send, edit, or delete.
- There is no code path for Calendar edit or delete.
- Logs do not store raw model outputs; they store request-scoped summaries, digests, and event metadata.

## WhatsApp scaling

- The bridge forwards `source_id` as the WhatsApp chat id.
- Each chat is a separate memory boundary.
- The backend is ready for multiple allowlisted chats on the same local network.

## Requirements

- Python 3.11+
- Node.js 18+
- LM Studio running locally with an OpenAI-compatible endpoint at `http://127.0.0.1:1234/v1`
- A Google Cloud OAuth Desktop client JSON saved as `credentials.json` in the project root

## Google Cloud setup

1. Create a Google Cloud project.
2. Enable the Gmail API and the Google Calendar API.
3. Create an OAuth client of type **Desktop app**.
4. Download the JSON client file and save it as `credentials.json` in the project root.
5. Start the backend once; the browser-based OAuth flow will open and save a local token.

## Run order

1. Start LM Studio and load the models.
2. Activate the Python virtual environment.
3. Run `python -m uvicorn app.main:app --host 127.0.0.1 --port 8000`.
4. Start the WhatsApp bridge from `whatsapp_bridge/server.js`.
5. Scan the QR code.
6. Set the allowlisted chat id.
7. Send a prefixed command such as `!give me today's news summary`.
