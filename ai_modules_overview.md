# AI Modules Overview

This file summarizes the AI-related addons in `C:\GIT\special`, what each one does, and how the stack works.

## Core Idea

The `ai` addon is the base layer. It provides:

- AI agents and sources
- Attachment chunking and embeddings
- LLM API access for OpenAI and Google
- Chat and livechat session handling
- Prompt rendering helpers
- Tool calling for server actions

Most other AI addons are thin integrations on top of that core.

## Base Addon: `ai`

### What it does

- Defines `ai.agent`, `ai.agent.source`, `ai.embedding`, `ai.topic`, `ai.composer`, and prompt helpers.
- Stores and retrieves embeddings for RAG-style retrieval.
- Sends requests to OpenAI or Google Gemini.
- Supports tool calling with JSON schema validation.
- Integrates AI into Discuss, mail templates, and livechat.

### How it works

- `LLMApiService` is the transport layer.
- `ai.agent` builds the system prompt, conversation history, and optional RAG context.
- `ir.attachment` extracts text from files and splits them into chunks.
- `ai.embedding` stores chunk vectors and runs similarity search with pgvector.
- `ai.composer` controls which AI prompt bundles are used on each UI surface.
- `ir.actions.server` can be executed by the LLM as tools.

### Important implementation points

- The module only installs if PostgreSQL `vector` is available.
- It uses `ai_session_identifier` from `ir.http.session_info()` to track UI sessions.
- It keeps AI chat data in `discuss.channel` with `channel_type = 'ai_chat'`.
- It supports inline citations in responses by replacing attachment id markers with links.

## UI Shell: `ai_app`

### What it does

- Adds the AI menu in the backend.
- Exposes agents, topics, and default prompt configuration screens.
- Adds the command palette entry for opening agent chats.
- Adds the AI settings panel for provider API keys.

### How it works

- Menu items link to the agent, topic, and composer actions.
- The command palette fetches `ai.agent` records and opens a chat action for the selected agent.
- Settings store OpenAI and Google keys in `ir.config_parameter`.

## AI Fields Stack: `ai_fields`

### What it does

- Adds AI-computed fields.
- Adds AI-backed properties on `fields.Properties`.
- Adds a cron job that fills empty AI fields and properties.
- Adds UI widgets and dialogs for configuring prompts, relations, and allowed values.

### How it works

- `ir.model.fields` gets:
  - `ai`
  - `ai_domain`
  - `system_prompt`
- When an AI field or AI property definition is created or changed, the cron is triggered.
- The cron finds records with empty AI fields and fills them by calling the LLM.
- `get_ai_value()` uses a structured JSON schema and validates the response before assigning it.

### Notable behavior

- It supports common field types such as char, text, html, boolean, selection, many2one, many2many, integer, float, monetary, and datetime.
- It can inject record context into the prompt safely.
- It sanitizes HTML prompt definitions.

## AI Server Actions: `ai_server_actions`

### What it does

- Adds a new server action evaluation mode: `ai_computed`.
- Lets server actions update a field value by asking the LLM.

### How it works

- The server action form shows an AI prompt editor when `evaluation_type = 'ai_computed'`.
- The AI prompt is used to compute a value for the selected stored field.
- When the action runs, it reuses the `ai_fields` value generation logic.

## Documents Integration: `ai_documents`

### What it does

- Lets document folders define an AI auto-sort prompt.
- Generates the server action and base automation rule that perform sorting.
- Runs AI-based folder moves and tag actions on documents.

### How it works

- A folder can store `ai_sort_prompt`.
- `action_ai_sort()` manually runs the AI sorter.
- `_ai_setup_sort_actions()` creates:
  - an `ir.actions.server` AI tool
  - a `base.automation` rule
- The generated prompt includes document name, content, tags, and mimetype.
- A cron can process queued documents later if there are many of them.

### Supporting addons

- `ai_documents_source` lets documents be used as AI sources.
- `ai_documents_account` adds accounting-specific document context and makes account record-create AI tools valid.

## Knowledge Integration: `ai_knowledge`

### What it does

- Lets knowledge articles be used as AI sources.
- Adds a knowledge-specific composer target for writing articles.

### How it works

- `ai.agent.source` gains a `knowledge_article` type.
- The article content is extracted from the article body and descendants.
- Access checks are done with the current user, not sudo, when evaluating source visibility.

## CRM Integration: `ai_crm` and `ai_crm_livechat`

### What they do

- `ai_crm` adds lead-creation tools and CRM-oriented agent behavior.
- `ai_crm_livechat` attaches livechat conversations to leads and exposes lead stats on agents.

### How it works

- The CRM addon defines two AI server actions:
  - create a lead
  - fetch available teams, tags, and location-specific IDs
- The topic instructs the model to gather required details, then call the lead creation tool once.
- The livechat addon stores the originating chat channel and visitor link on the created lead.

## Livechat Integration: `ai_livechat`

### What it does

- Allows livechat channels to use an AI agent as the operator.
- Supports forwarding from AI to a human operator.
- Tracks AI agent membership in livechat state.

### How it works

- `im_livechat.channel.rule` can point to `ai.agent`.
- `im_livechat.channel` computes the AI agent count and reports AI availability.
- `discuss.channel` stores `ai_agent_id` for AI/livechat channels.
- The browser can forward the chat to a human through `/ai_livechat/forward_operator`.
- The CORS controller exposes the same flow for public clients.

## Website Integration: `ai_website` and `ai_website_livechat`

### `ai_website`

- Generates website page HTML from template snippets using an AI agent.
- Extracts each snippet, builds a prompt, gets structured text back, and rehydrates the page HTML.

### `ai_website_livechat`

- Adds a website builder snippet for AI livechat.
- Stores the chosen agent on the snippet and updates the agent usage flag.
- Provides a public interaction component that mounts the livechat widget on the website.

## Voice / Phone Integration: `voip_ai`

### What it does

- Uploads VoIP recordings to the server.
- Queues transcription work.
- Stores the transcript and a short summary.

### How it works

- The controller accepts the audio upload, creates an `ir.attachment`, and marks the call as pending.
- A cron job picks the most recent pending call and sends audio to the transcription API.
- After transcription, it optionally asks an AI agent to summarize the transcript.
- `voip.provider` can force transcription for all users.

## Small Addons

### `ai_auto_install`

- Auto-installs `ai` only when pgvector support exists.

### `ai_account`

- Injects AI-powered email drafting into the invoice sending wizard.

### `sign_ai`

- Injects AI-powered email drafting into the signature request wizard.

### `web_studio_ai_fields`

- Adds an AI field type to Studio.
- Opens a custom configuration dialog to define prompt, type, and relation settings.

### `test_ai_fields`

- Provides lightweight test models for the AI field/property stack.

## End-to-End Flow

1. A UI surface creates an AI session or uses a composer rule.
2. The system builds a prompt from:
   - record context
   - composer prompt
   - topic instructions
   - source snippets or embeddings
3. `LLMApiService` sends the request to OpenAI or Google.
4. If the model returns tool calls, Odoo validates and executes them.
5. If embeddings are needed, attachments are chunked and indexed with pgvector.
6. The result is rendered back into the host feature:
   - chat message
   - field value
   - server action
   - document move
   - lead creation
   - webpage content
   - transcription summary

## Practical Notes

- `ai` is the root dependency for most of the stack.
- `ai_fields` and `ai_server_actions` are the main general-purpose automation layers.
- `ai_documents`, `ai_crm`, `ai_livechat`, `ai_knowledge`, `ai_website`, and `voip_ai` are vertical integrations.
- `ai_auto_install` is only there to gate installation by pgvector availability.

