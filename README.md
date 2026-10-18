# 🤖 Personal AI Assistant — Telegram Bot (n8n Workflow)

A fully automated personal AI assistant built with [n8n](https://n8n.io/), powered by GPT-4o-mini. It lives inside Telegram and connects to your everyday tools — Todoist, Google Calendar, Google Docs, and a personal knowledge base — so you can manage your life through a single chat interface.

---

## ✨ Features

| Capability | Description |
|---|---|
| 💬 Natural Language Chat | Conversational AI powered by OpenAI GPT-4o-mini |
| ✅ Task Management | Fetch, view, and close tasks from Todoist |
| 📅 Calendar Management | Schedule, update, and query events on Google Calendar |
| 📝 Document Creation | Create and populate Google Docs with structured content |
| 🧠 Persistent Memory | Remembers conversation history using PostgreSQL |
| 📚 Personal Knowledge Base | Retrieves information about you from a Supabase vector store |
| 📄 Document Ingestion | Indexes personal documents (e.g. resume) from Google Drive into the vector store |

---

## 🏗️ Architecture

The workflow is composed of two logical parts:

### Part 1 — Main Conversation Loop
```
Telegram Trigger → Edit Fields → AI Agent → Send Text Message
                                    ↑
                    ┌───────────────┼────────────────────┐
                    │               │                    │
              LLM (GPT-4o-mini)  Tools              Memory
                                    │              (PostgreSQL)
                    ┌───────────────┼────────────────────┐
                    │       │       │       │      │      │
               Todoist  Todoist  G.Docs  G.Cal  G.Cal  Vector
               (get)   (close) (create) (get) (create)  Store
                                (update)      (update) (retrieve)
```

### Part 2 — Document Ingestion Pipeline (one-time / on-demand)
```
Google Drive (Download PDF) → Supabase Vector Store
                                      ↑
                         Default Data Loader + Text Splitter
                         + OpenAI Embeddings
```

---

## 🧩 Nodes & Tools Breakdown

### Core Flow
| Node | Type | Purpose |
|---|---|---|
| **Telegram Trigger** | Trigger | Listens for incoming messages via webhook |
| **Edit Fields** | Set | Extracts `chat.id`, message text, and `sessionId` from the Telegram payload |
| **AI Agent** | LangChain Agent | Orchestrates all tool calls and generates responses |
| **OpenAI Chat Model** | LLM | GPT-4o-mini as the reasoning engine |
| **Send a text message** | Telegram | Sends the AI-generated reply back to the user |

### Memory
| Node | Type | Purpose |
|---|---|---|
| **Postgres Chat Memory** | Memory | Persists conversation history per user session using PostgreSQL |

### Tools Available to the Agent
| Tool Node | Service | What the Agent Can Do |
|---|---|---|
| `todoist_tool` | Todoist | Fetch all tasks |
| `close_tasks` | Todoist | Mark a specific task as complete |
| `google_docs` | Google Docs | Create a new document |
| `update_docs` | Google Docs | Write content into an existing document |
| `get_events` | Google Calendar | Retrieve events within a date range |
| `schedule_event` | Google Calendar | Create a new calendar event |
| `update_event` | Google Calendar | Modify an existing calendar event |
| `retrive_documents_info` | Supabase (Vector) | Semantic search over the personal knowledge base |

### Document Ingestion
| Node | Purpose |
|---|---|
| **Download file** | Downloads a PDF (resume) from Google Drive |
| **Default Data Loader** | Parses the binary file into text chunks |
| **Recursive Character Text Splitter** | Splits text for optimal embedding size |
| **Embeddings OpenAI (×2)** | Generates vector embeddings using OpenAI |
| **Supabase Vector Store** | Stores and retrieves embedded documents |

---

## 🔧 Prerequisites

Before importing this workflow, make sure you have the following accounts and credentials set up in your n8n instance:

- **n8n** (self-hosted or cloud) — v1.x or later
- **Telegram Bot** — created via [@BotFather](https://t.me/BotFather)
- **OpenAI** — API key with access to `gpt-4o-mini` and embeddings
- **Todoist** — OAuth2 connected app
- **Google Workspace** — OAuth2 credentials with scopes for Drive, Docs, and Calendar
- **PostgreSQL** — A running instance for chat memory storage
- **Supabase** — Project with a `documents` table configured for pgvector

---

## 🚀 Setup & Installation

### 1. Clone the repository
```bash
git clone https://github.com/<your-username>/n8n-telegram-ai-assistant.git
cd n8n-telegram-ai-assistant
```

### 2. Import the workflow into n8n
1. Open your n8n instance
2. Go to **Workflows** → **Import from file**
3. Select `workflow/telegram_bot.json`

### 3. Configure credentials
In n8n, go to **Credentials** and create the following:

| Credential Name | Type |
|---|---|
| Telegram account | Telegram API |
| OpenAI account | OpenAI API |
| Todoist OAuth2 API | OAuth2 |
| Google Docs OAuth2 API | OAuth2 |
| Google Calendar OAuth2 API | OAuth2 |
| Google Drive OAuth2 API | OAuth2 |
| Postgres account | PostgreSQL |
| Supabase account | Supabase API |

### 4. Set up the Supabase `documents` table
Run the following SQL in your Supabase project to enable vector search:

```sql
-- Enable the pgvector extension
create extension if not exists vector;

-- Create the documents table
create table documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);

-- Create an index for fast similarity search
create index on documents using ivfflat (embedding vector_cosine_ops)
  with (lists = 100);
```

### 5. Run the document ingestion pipeline
Trigger the **Download file** node manually once to index your personal documents (resume, notes, etc.) into the Supabase vector store.

### 6. Activate the workflow
Toggle the workflow to **Active** — the Telegram webhook registers automatically.

---

## 💬 Usage Examples

Once active, send any of these messages to your Telegram bot:

```
"What tasks do I have today?"
"Mark 'Submit assignment' as done"
"I have a Math exam on November 5th at 10 AM"
"Do I have anything scheduled this Friday?"
"Create a doc explaining the basics of neural networks"
"What's in my resume?"
```

---

## 🗂️ Project Structure

```
n8n-telegram-ai-assistant/
├── workflow/
│   └── telegram_bot.json      # The complete n8n workflow export
├── .gitignore
└── README.md
```

---

## 🔒 Security Notes

- **Never commit credentials.** All sensitive values (API keys, OAuth tokens) live inside n8n's encrypted credential store — they are not part of the exported JSON.
- The workflow JSON contains placeholder credential IDs that are specific to the original n8n instance. Importing on a new instance will prompt you to re-link credentials.
- Consider restricting your Telegram bot to your own user ID to prevent unauthorized access.

---

## 🛣️ Future Improvement Scopes

### 🧠 Intelligence & Memory
- [ ] **Long-term semantic memory** — Store conversation summaries in the vector store so the agent recalls context across sessions, not just within one.
- [ ] **User profile building** — Automatically extract and persist user preferences, habits, and frequently referenced information from conversations.
- [ ] **Multi-user support** — Isolate memory and context per Telegram user ID to serve multiple users from the same bot.

### 🛠️ Tool Expansion
- [ ] **Email integration** — Connect Gmail/Outlook to read, draft, and send emails via natural language.
- [ ] **Notion integration** — Create and update Notion pages and databases.
- [ ] **Weather & news** — Add real-time data tools for weather forecasts and news briefings.
- [ ] **Web search** — Integrate a search tool (e.g., SerpAPI, Tavily) so the agent can answer questions requiring up-to-date information.
- [ ] **GitHub integration** — Query issues, PRs, and repo stats directly from chat.

### 🎙️ Input / Output Modes
- [ ] **Voice message support** — Transcribe Telegram voice notes using OpenAI Whisper and process them as text.
- [ ] **Image understanding** — Accept image inputs and use GPT-4o vision capabilities for analysis.
- [ ] **Rich message formatting** — Use Telegram's MarkdownV2 or inline keyboards for more structured, interactive responses.

### 🏗️ Infrastructure & Reliability
- [ ] **Error handling nodes** — Add error branches to gracefully handle API failures and notify the user instead of silently failing.
- [ ] **Rate limiting** — Throttle requests per user to prevent abuse and control API costs.
- [ ] **Logging & observability** — Pipe execution logs to a monitoring tool (e.g., Datadog, Grafana) for production observability.
- [ ] **Model flexibility** — Make the LLM model configurable via an environment variable to easily swap between GPT-4o-mini, GPT-4o, or open-source models via Ollama.
- [ ] **Automated document re-indexing** — Trigger the ingestion pipeline automatically when files are updated in Google Drive.

### 🔐 Security
- [ ] **User allowlist** — Validate incoming Telegram `user.id` against an allowlist before processing any messages.
- [ ] **Secrets management** — Move all credential IDs to environment variables for cleaner, more portable deployments.

---

## 📄 License

This project is open-source under the [MIT License](LICENSE).

---

## 🙋 Author

Built by **Jahirul Islam Shifat**  
Feel free to fork, adapt, and build on top of this workflow.
