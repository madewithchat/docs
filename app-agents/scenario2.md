# Scenario 2: Research & Draft a Marketing Brief (In-Depth)

This scenario provides a comprehensive, step-by-step walkthrough, detailing every function, input/output, state change, and external interaction.

---
## Step 1: Agent Startup & Session Initialization

1. **Frontend (React App)**
   - Starts LiveKit CLI via `cli.run_app(...)` in **`src/main.py`**.
   - Joins room with metadata:
     ```json
     { "user_id": "user-abc-123", "project_id": "proj-xyz-789" }
     ```

2. **LiveKit**
   - Invokes entrypoint:
     ```1:1:src/main.py
     async def entrypoint(ctx: JobContext):
         logger.info("Starting agent application")
         session_manager = AgentSessionManager()
         await session_manager.start_session(ctx)
     ```

3. **AgentSessionManager.start_session** (`src/agent/agent.py` 441-462)
   - **Input:** `ctx: JobContext`
   - **Actions:**
     1. `await ctx.connect()` → establishes audio connection.
     2. Calls `await self.create_agent_session(ctx)`.
     3. Waits: `await ctx.wait_for_participant()`.
     4. Starts session: `await session.start(agent=agent, room=ctx.room)`.
   - **Output:** Returns `(session, agent)`.

4. **AgentSessionManager.create_agent_session** (`src/agent/agent.py` 361-403)
   - **Input:** `ctx`, `metadata = safe_json_loads(ctx.job.metadata, "job metadata")`.
   - **Actions:**
     1. Extract `user_id`, `project_id`, `is_stateless_sessions`.
     2. Fetch preferences:
        ```python
        preferences = await user_preferences_service.get_user_preferences(user_id)
        ```
     3. Load history:
        ```python
        recent_summaries = await summarized_conversation_messages_service.find_all_summarized_messages(
            project_id, sort_by="createdAt", sort_order="DESC", limit=20
        )
        ```
     4. Build `ChatContext` and seed with summaries and greeting stub.
     5. Instantiate `FunctionAgent`:
        ```python
        agent = FunctionAgent(
          preferences["mainInstructions"],
          preferences["userInstructions"],
          chat_ctx,
          preferences
        )
        ```
     6. Setup event & RPC handlers.
     7. Start `BackgroundAudioPlayer` with sounds from `get_thinking_sounds_config()`.
   - **Output:** `(session: AgentSession, agent: FunctionAgent)`.

5. **FunctionAgent.__init__** (`src/agent/agent.py` 55-141)
   - **Input:** Prompt templates, `chat_ctx`, `user_preferences`.
   - **Actions:**
     1. Load static prompt files via `load_prompt_from_file`:
        ```python
        main_prompt = load_prompt_from_file(MAIN_PROMPT_PATH)
        ```
     2. Format `instructions` by injecting user/project profiles.
     3. Initialize `StateManager(max_hover_history=5)`.
     4. Load function tool templates for document, analyze, diff.
   - **State:** `agent.state` initialized.

6. **Setup Handlers**
   - **Event Handlers** (`src/agent/event_handlers.py`):
     ```python
     setup_event_handlers(session, agent, ctx, ...)  # lines 1-107
     ```
   - **RPC Handlers** (`src/agent/rpc_handlers.py`):
     ```python
     setup_rpc_handlers(ctx, agent, ...)  # lines 1-105
     ```

7. **Thinking Sounds**
   - `get_thinking_sounds_config()` scans `ASSETS_DIR` and returns list of `AudioConfig`.
   - Background audio starts.

**State after Step 1:**
- **StateManager:**
  - `active_document_id = None`
  - `documents = []` (or from preferences)
  - `hover_history = []`
  - `current_user_id = "user-abc-123"`
- **ChatContext.messages:** Contains system prompt & past summaries.

---
## Step 2: User Speech → Transcription → Message Processing

1. **Frontend:** Captures audio, shows "listening" indicator.
2. **LiveKit:** Streams to:
   - **VAD** (`silero.VAD`) → detects speech start/end.
   - **STT** (`deepgram.STT`) → emits `UserInputTranscribedEvent`.

3. **Event Handler** (`src/agent/event_handlers.py` 29-50)
   ```python
   @session.on("conversation_item_added")
   def on_conversation_item_added(event: ConversationItemAddedEvent):
       asyncio.create_task(
         manage_conversation_message(ctx, ..., message_data, project_id)
       )
   ```
   - **Input:** `event.item.text_content`.
   - **Action:** Calls `manage_conversation_message` asynchronously.

4. **Conversation Processing** (`src/core/conversation/summarization_service.py` 1-50)
   - **filter_content(text)**: LLM filters filler words.
   - **summarize_content(text, role)**: Creates concise summary.
   - **create_message** via API client.
   - **Result:** Message stored in DB and broadcasted.

5. **Agent Hook**: `FunctionAgent.on_user_turn_completed` (`src/agent/agent.py` 209-239)
   - **Calls:**
     ```python
     context_info = await self.state.format_context()
     turn_ctx.add_message(role="assistant", content=context_info)
     plan_ctx = await self._get_formatted_plan_context()
     turn_ctx.add_message(role="assistant", content=plan_ctx)
     turn_ctx.add_message(role="assistant", content=current_time)
     ```
   - **Effect:** Enriches `ChatContext` with live state & plan data.

**State & Context after Step 2:**
- ChatContext now includes:
  1. `user` message: transcript.
  2. `assistant` message: state snapshot:
     ```text
     Current Context:
       Document ID: None
       Project ID: proj-xyz-789
     Available Documents (0):
     ```
  3. `assistant` message: plan context (none yet).
  4. `assistant` message: timestamp.

---
## Step 3: LLM Function Call → Create Plan

1. **LLM Input**: The enriched `ChatContext` is sent to `openai.LLM.chat(...)`.
2. **LLM Output**: A function call JSON:
   ```json
   {
     "name": "create_plan",
     "arguments": {
       "user_request": "research ... vacuum cleaners and create marketing brief",
       "plan_steps": "[{\"type\": \"WEB_SEARCH\", ... }]"
     }
   }
   ```

3. **FunctionToolsMixin.create_plan** (`src/agent/function_tools.py` 15-65)
   - **Input:** `user_request`, `plan_steps`.
   - **Validation:** `validate_required_params` ensures presence.
   - **Parsing:** `parsed_steps = safe_json_loads(plan_steps)`.
   - **Calls:** `plan = await self.plan_manager.create_plan(user_request, parsed_steps, context)`.

4. **PlanManager.create_plan** (`src/core/plan/plan_manager.py` 15-60)
   - **Actions:**
     - Constructs `PlanStep` objects with UUIDs.
     - Initializes `ExecutionPlan` with `steps`, `user_request`.
     - Saves to `self._current_plan`.

5. **Feedback to Frontend:**
   - `_send_plan_update_to_frontend("plan_created")` calls:
     ```python
     await perform_agent_rpc(method="planCreated", payload=plan_dict)
     ```
   - UI renders a checklist:
     ```text
     [ ] Research trends
     [ ] Draft marketing brief
     ```

6. **Automatic Execution:**
   - `safe_async_task(self._trigger_automatic_plan_execution(...))` schedules step 4.

**State after Step 3:**
- **PlanManager:**
  - `_current_plan.id = 'uuid-0001'`
  - `steps = [PlanStep(type=WEB_SEARCH, status=PENDING), PlanStep(type=DOCUMENT_CREATE, status=PENDING)]`
- **ChatContext:** An assistant message: "Plan created with 2 steps. Plan ID: uuid-0001".

---
## Step 4: Execute Step 1 → Web Search

1. **_trigger_automatic_plan_execution** (background) finds first `PENDING` step.
2. Calls **WebSearchExecutor.execute** (`src/core/plan/step_executor.py` 111-140):
   - **Input:** `query = "latest trends in AI-powered vacuum cleaners"`.
   - **Action:** `await search_web_firecrawl(...)`.

3. **FirecrawlAPIClient.search_and_scrape** (`src/infrastructure/api_clients/firecrawl_api_client.py`)
   - Performs advanced web scraping using Firecrawl API.
   - Receives structured content with markdown formatting and LLM-extracted information.

4. **format_search_results** processes top results into markdown.
5. **PlanManager.update_step_by_id** (`src/core/plan/plan_manager.py` 60-120)
   - Updates step status to `COMPLETED`.
   - Sets `result` with summary + details.
   - Logs `started_at`, `completed_at`.

6. **Feedback:**
   - `perform_agent_rpc(method="planUpdated", payload=updated_plan_dict)` → UI updates checkmark.
   - Agent adds search results as an `assistant` message.

**State after Step 4:**
- **PlanManager:**
  - Step 1: status=COMPLETED, result={...}
  - Step 2: status=PENDING
- **ChatContext:** Contains search results summary.

---
## Step 5: Execute Step 2 → Document Creation

1. **Trigger**: Next `PENDING` step is `DOCUMENT_CREATE`.
2. **Function Call**: `create_document(title, description)` in `FunctionToolsMixin`.
3. **DocumentCreator.create_new_document** (`src/core/document/create_service.py` 1-60)
   - **Stage 1:** `_try_template_creation` searches templates via backend API.
     - If found: `template_content` returned.
   - **Stage 2:** Fallback to LLM JSON generation:
     ```2:50:src/core/document/create_service.py
     response_stream = self.llm.chat(chat_ctx, extra_kwargs={"response_format": {"type": "json_object"}})
     ```
   - **Extract JSON** via `extract_json_from_llm_response`.
4. **RPC to Frontend:**
   ```python
   await perform_agent_rpc(method="createDocument", payload={"title": title, "document": raw_json})
   ```
5. **PlanManager.update_step_by_id** marks creation step as `COMPLETED`.
6. **Frontend:** Renders document in editor, sets `active_document_id`.
   - UI triggers `updateState` RPC with new document list.

7. **RPC Handler** (`src/agent/rpc_handlers.py` 30-60) catches `update_state` calls.
   - Calls `agent.state.update_from_dict(payload)` → updates `StateManager`.

**State after Step 5:**
- **PlanManager:** All steps COMPLETED.
- **StateManager:**
  - `active_document_id = "doc-1234"`
  - `documents = [{ id: "doc-1234", title: "Marketing Brief..." }]`

---
## Sub-Scenario A: Template Found

If `_try_template_creation` returns a template:
1. Step 5.1: `template_content` is non-null.
2. Skips LLM generation.
3. Proceeds directly to `perform_agent_rpc(createDocument)`.

## Sub-Scenario B: Error Handling

If any function raises an exception, decorated with `@handle_errors` (`src/common/error_handler.py`):
1. Logs error details.
2. Hides loading UI: `perform_agent_rpc(method="setLoadingState", payload={ action: "hide" })`.
3. Interrupts agent with apology via `interrupt_agent`.

---

This detailed scenario maps each action to its code implementation, inputs/outputs, and shows how the system state and context evolve across the entire flow. 