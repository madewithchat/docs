# Scenario: Research & Draft a Marketing Brief

This document walks through a detailed, end-to-end scenario to illustrate how the various components of the AI agent system work together.

**Goal:** The user wants the agent to research a topic on the web and then create a new document based on that research.

---

## The Scenario

**User:** *"Hey, can you research the latest trends in AI-powered vacuum cleaners and then create a marketing brief for me?"*

We will follow this request through the entire system, step by step.

---

### Step 1: Session Initiation and Idle State

*Before the user speaks, they open the application.*

#### Component Breakdown

*   **Frontend:**
    *   **Action:** The React application initializes.
    *   **Communication:** It establishes a connection to the LiveKit server, joining a specific room. When joining, it passes metadata in the connection options:
        *   `metadata: { "user_id": "user-abc-123", "project_id": "proj-xyz-789" }`
    *   **UI State:** The application is in its default idle state, waiting for user interaction. No documents are open.

*   **Backend (Agent):**
    *   **Concerned Functions:**
        *   `main.entrypoint(ctx)`: The initial function called by the LiveKit worker when the agent process starts.
        *   `AgentSessionManager.start_session(ctx)`: The primary function that orchestrates the entire session setup.
        *   `AgentSessionManager.create_agent_session(ctx)`: Handles the core logic of creating and configuring the agent.
        *   `user_preferences_service.get_user_preferences(user_id)`: Fetches user-specific settings.
        *   `summarized_conversation_messages_service.find_all_summarized_messages(...)`: Loads past conversation context.
        *   `FunctionAgent.__init__(...)`: The agent's constructor.
    *   **Workflow:**
        1.  The LiveKit worker calls `start_session`, which receives the `JobContext` (`ctx`) containing the frontend's metadata.
        2.  `create_agent_session` is called. It extracts `user_id` and `project_id` from the metadata.
        3.  It calls the backend API via `ServiceAPIClient` to fetch user preferences (like custom instructions) and the 20 most recent conversation summaries for this project.
        4.  A `ChatContext` object is created and populated with the loaded conversation summaries to give the LLM long-term memory.
        5.  An instance of `FunctionAgent` is created. Its constructor merges multiple prompt sources (base prompts, user-specific instructions, and profile data from preferences) into a single master prompt.
        6.  Event handlers (`setup_event_handlers`) and RPC handlers (`setup_rpc_handlers`) are attached to the session.
    *   **State & Context Evolution:**
        *   **`StateManager` State:** The agent's state is initialized.
            *   `active_document_id`: `None`
            *   `documents`: Populated with a list of document references (`id`, `title`) from the backend.
            *   `hover_history`: `[]` (empty)
            *   `current_user_id`: `"user-abc-123"`
        *   **`ChatContext`:**
            *   Contains a system message with the large, merged master prompt.
            *   Contains a series of `user` and `assistant` messages representing the summaries from past conversations.
            *   A final message is added: `"The user has just started a new conversation. Briefly greet the user and tell him where you stopped last time..."`

*   **External Services:**
    *   **LiveKit:** A new participant (the agent) joins the room, ready to receive audio and send data.
    *   **Backend API:** Receives GET requests from the `ServiceAPIClient` to fetch user preferences and conversation history.

---

### Step 2: User Speaks and Request is Transcribed

*The user says their request out loud.*

#### Component Breakdown

*   **Frontend:**
    *   **Action:** The user speaks into their microphone. The browser captures the audio.
    *   **UI State:** The UI might show a "listening" indicator.

*   **Backend (Agent):**
    *   **Concerned Functions:**
        *   `livekit.agents.vad.VAD`: Voice Activity Detection.
        *   `livekit.plugins.deepgram.STT`: Speech-To-Text.
        *   `event_handlers.on_conversation_item_added(event)`: Event handler for new messages.
        *   `conversation.manage_conversation_message(...)`: Filters, summarizes, and stores the message.
        *   `FunctionAgent.on_user_turn_completed(turn_ctx, ...)`: Hook that runs after the user finishes speaking but before the LLM is called.
    *   **Workflow:**
        1.  Raw audio is streamed from the frontend to the agent via LiveKit.
        2.  The VAD detects that the user is speaking.
        3.  The audio is fed to the Deepgram STT plugin, which transcribes it in real-time.
        4.  When the user stops speaking, the STT plugin finalizes the transcript, and LiveKit emits a `conversation_item_added` event.
        5.  The `on_conversation_item_added` handler fires. It calls `manage_conversation_message` in the background to process and persist the user's message.
        6.  The agent's `on_user_turn_completed` hook is triggered. This function is crucial for context.
            *   It calls `self.state.format_context()` to get a string representation of the current state (active document, file list, etc.).
            *   It adds this context string as an `assistant` message to the `ChatContext`.
            *   It also adds the current timestamp.
    *   **State & Context Evolution:**
        *   **`StateManager` State:** Unchanged in this step.
        *   **`ChatContext`:**
            *   **Input:** The context from Step 1.
            *   **Evolution:**
                1. A new `user` message is added with the final transcript: `"Hey, can you research the latest trends in AI-powered vacuum cleaners and then create a marketing brief for me?"`
                2. A new `assistant` message is added by `on_user_turn_completed` containing the output of `format_context()` (e.g., "Current Context: Document ID: None, Available Documents: ...").
                3. Another `assistant` message is added with the current time.

*   **External Services:**
    *   **LiveKit:** Streams audio from the user to the agent. Transports the event data.
    *   **Deepgram:** Receives audio stream, returns transcribed text.
    *   **OpenAI LLM:** The fully populated `ChatContext` is now sent to the main LLM for processing.

---

### Step 3: LLM Decides to Create a Plan

*The main LLM determines the request is too complex for a single action.*

#### Component Breakdown

*   **Frontend:**
    *   **UI State:** The UI might show a "thinking" indicator. The thinking sound starts playing.

*   **Backend (Agent):**
    *   **Concerned Functions:**
        *   `FunctionToolsMixin.create_plan(...)`: The function tool that the LLM calls.
        *   `plan.PlanManager.create_plan(...)`: The core logic for creating a new `ExecutionPlan`.
        *   `_send_plan_update_to_frontend(...)`: An RPC utility function.
        *   `_trigger_automatic_plan_execution(...)`: A function to kick off the plan.
    *   **Workflow:**
        1.  The LLM response arrives. It's not a text reply, but a **function call** to `create_plan`.
        2.  The agent's `FunctionToolsMixin` executes the `create_plan` method.
            *   **Input:**
                *   `user_request`: "research latest trends in AI-powered vacuum cleaners and create a marketing brief"
                *   `plan_steps`: A JSON string from the LLM like `[{"type": "WEB_SEARCH", "description": "Research AI vacuum trends"}, {"type": "DOCUMENT_CREATE", "description": "Draft marketing brief from research"}]`
        3.  Inside `create_plan`, it calls `self.plan_manager.create_plan`. This creates a new `ExecutionPlan` object and stores it in `self._current_plan`. The steps are parsed into `PlanStep` models, each with a unique ID and a status of `PENDING`.
        4.  An RPC call (`plan_created`) is sent to the frontend with the plan's data, so the UI can render a checklist of the steps.
        5.  `_trigger_automatic_plan_execution` is called as a background task to immediately start working on the first step.
    *   **State & Context Evolution:**
        *   **`PlanManager` State:**
            *   A new `ExecutionPlan` is created and becomes the `_current_plan`.
            *   `_current_plan.steps` now holds two `PlanStep` objects, both `PENDING`.
        *   **`StateManager` State:** Unchanged.
        *   **`ChatContext`:** A confirmation message is added, like `"Plan created with 2 steps. Plan ID: <uuid>"`. This will be part of the context for future LLM calls.

*   **External Services:**
    *   **OpenAI LLM:** Provides the structured function call response.

---

### Step 4: Plan Execution - The Web Search

*The agent automatically starts the first step of the plan.*

#### Component Breakdown

*   **Frontend:**
    *   **UI State:** The UI updates to show the "Research AI vacuum trends" step is now "In Progress".

*   **Backend (Agent):**
    *   **Concerned Functions:**
        *   `_trigger_automatic_plan_execution`: Identifies the next `PENDING` step.
        *   `core.plan.step_executor.WebSearchExecutor.execute(...)`: The executor for `WEB_SEARCH` steps in plans.
        *   `infrastructure.api_clients.FirecrawlAPIClient.search_and_scrape(...)`: Makes the external API call for advanced content extraction.
        *   `plan.PlanManager.update_step_by_id(...)`: Updates the status of the plan step.
    *   **Workflow:**
        1.  The execution logic finds the first step ("Research AI vacuum trends") and calls the plan-specific web search executor.
            *   **Input:** `query="latest trends in AI-powered vacuum cleaners"`
        2.  The executor calls `search_web_firecrawl`, which calls `interrupt_agent` to notify the user: "Gathering detailed content for: latest trends...".
        3.  It then instantiates `FirecrawlAPIClient` and calls its `search_and_scrape` method for enhanced content extraction.
        4.  The `FirecrawlAPIClient` uses Firecrawl's search and scraping capabilities to extract detailed content from relevant pages.
        5.  Once the response is received, `format_search_results` is called to create a detailed, markdown-formatted string of the scraped content.
        6.  The agent calls `self.plan_manager.update_step_by_id` to change the step's status.
            *   **Input:** `step_id`, `status=StepStatus.COMPLETED`, `result={"summary": "Found 5 articles...", "full_text": "..."}`.
        7.  An RPC call (`plan_updated`) is sent to the frontend to update the UI (e.g., show a checkmark next to the step).
        8.  The search results are added to the conversation history for the next step's context.
    *   **State & Context Evolution:**
        *   **`PlanManager` State:**
            *   The `WEB_SEARCH` `PlanStep`'s status is updated from `PENDING` to `IN_PROGRESS`, and then to `COMPLETED`.
            *   Its `result` attribute is populated with the formatted search results.
            *   `started_at` and `completed_at` timestamps are set.
        *   **`ChatContext`:** The formatted search results are added as an `assistant` message.

*   **External Services:**
    *   **Firecrawl API:** Receives the search query and performs advanced web scraping to extract detailed, structured content from relevant pages.

---

### Step 5: Plan Execution - Document Creation

*With the research complete, the agent moves to the second step.*

#### Component Breakdown

*   **Frontend:**
    *   **UI State:** The "Draft marketing brief" step becomes "In Progress". The UI shows a global "Creating document..." loading state.

*   **Backend (Agent):**
    *   **Concerned Functions:**
        *   `_trigger_automatic_plan_execution`: Finds the `DOCUMENT_CREATE` step.
        *   `FunctionToolsMixin.create_document(...)`: The tool for this step.
        *   `core.document.create_service.DocumentCreator.create_new_document(...)`: The main creation logic.
        *   `DocumentCreator._try_template_creation(...)`: Tries to find a matching template first.
        *   `DocumentCreator._generate_raw_document_json(...)`: Falls back to LLM generation.
        *   `infrastructure.livekit.rpc.perform_agent_rpc(...)`: Utility to call frontend methods.
    *   **Workflow:**
        1.  The execution logic proceeds to the `DOCUMENT_CREATE` step and calls `create_document`.
        2.  This delegates to `DocumentCreator.create_new_document`.
        3.  `_try_template_creation` is called first. It performs a vector search via the backend API. Let's assume **no matching template is found**.
        4.  The code then falls back to `_generate_raw_document_json`.
            *   It constructs a `ChatContext` for the **Execution LLM (a different, more precise instance of GPT-4o)**.
            *   The context includes the `generate_document.txt` prompt, the user's request, and crucially, the **search results from the previous step**.
            *   It calls the LLM with `response_format: {"type": "json_object"}` to ensure structured output.
        5.  The LLM returns a large JSON string representing the document structure and content.
        6.  `_create_document_via_rpc` is called, which uses `perform_agent_rpc` to call the `createDocument` method on the frontend.
            *   **Payload:** `{ "title": "Marketing Brief...", "document": "<the raw json string>" }`
        7.  Finally, `self.plan_manager.update_step_by_id` is called to mark the `DOCUMENT_CREATE` step as `COMPLETED`. The plan is now finished.
    *   **State & Context Evolution:**
        *   **`PlanManager` State:** The `DOCUMENT_CREATE` step is marked `COMPLETED`. The entire plan is now complete. The `_current_plan` might be cleared or archived.
        *   **`StateManager` State:** After the frontend creates the document, it might send a state update, causing the new document's ID to be added to the `documents` list and set as the `active_document_id`.

*   **External Services:**
    *   **Backend API:** Queried for document templates.
    *   **OpenAI LLM (Execution):** Receives the generation prompt and research context, returns the document as a JSON object.

*   **Frontend (Receiving the RPC):**
    *   The `createDocument` RPC handler is invoked.
    *   It parses the JSON document content and renders it in the application's editor.
    *   The user now sees their fully drafted marketing brief on the screen. 