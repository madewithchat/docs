# Detailed Codebase Architecture Guide

This repository implements a **sophisticated voice-first AI assistant** that orchestrates multi-step plans, creates & edits documents, manages conversation memory, and provides real-time collaboration features. The system is built on **[LiveKit Agents](https://docs.livekit.io/agents/)** for real-time audio processing and features a robust multi-layered architecture.

---

## 1. Architecture Overview

### 1.1 High-Level System Design

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (React)                     │
├─────────────────────────────────────────────────────────┤
│              LiveKit Real-time Communication            │
├─────────────────────────────────────────────────────────┤
│                    Agent Runtime                        │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐ │
│  │    Voice    │  │   Function   │  │   Planning     │ │
│  │  Pipeline   │  │    Tools     │  │   Engine       │ │
│  │ (STT→LLM→TTS)│  │ (~30 tools)  │  │ (Multi-step)   │ │
│  └─────────────┘  └──────────────┘  └────────────────┘ │
├─────────────────────────────────────────────────────────┤
│                Backend API (REST + JWT)                 │
├─────────────────────────────────────────────────────────┤
│             Database & Document Storage                 │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Directory Structure Deep Dive

| Path | Purpose | Key Files |
|------|---------|-----------|
| `src/main.py` | Minimal entrypoint - delegates to `AgentSessionManager` | Entry point with VAD pre-warming |
| `src/constants.py` | Centralized configuration | 69 constants including API keys, model settings |
| `src/agent/` | **Application orchestration layer** | `agent.py`, `function_tools.py`, event/RPC handlers |
| `src/core/` | **Business logic layer** | Plan management, document workflows, conversation processing |
| `src/infrastructure/` | **Integration layer** | API clients, LiveKit adapters, data persistence |
| `src/common/` | **Shared utilities** | Error handling, JSON parsing, type definitions |
| `prompts/` | **LLM prompt templates** | 12 specialized prompts for different AI operations |

---

## 2. Core Components Deep Dive

### 2.1 Agent Session Management (`src/agent/`)

#### `AgentSessionManager` - Session Lifecycle Controller
```python
class AgentSessionManager:
    async def start_session(self, ctx: JobContext):
        # 1. Parse metadata (user_id, project_id)
        # 2. Load user preferences & conversation history
        # 3. Create FunctionAgent with merged prompts
        # 4. Setup event/RPC handlers
        # 5. Start background audio & wait for participant
```

**Key Responsibilities:**
- Orchestrates entire session startup flow
- Loads 20 most recent conversation summaries for context
- Configures thinking sounds from `/assets` directory
- Manages authentication via `ServiceAPIClient`

#### `FunctionAgent` - The Core AI Agent
```python
class FunctionAgent(FunctionToolsMixin, Agent):
    def __init__(self, preferences_main_instructions, preferences_user_instructions, 
                 chat_ctx, user_preferences):
        # Merges 4 prompt sources:
        # 1. Static main prompt (prompts/main.txt)
        # 2. User preferences (main_instructions)
        # 3. User-specific instructions
        # 4. User/project profiles
```

**Special Features:**
- **Custom TTS Processing**: Overrides `tts_node()` to clean text for better pronunciation
- **Context Injection**: Automatically adds state, plan context, and timestamp to each LLM turn
- **State Management**: Universal `StateManager` with get/set/update patterns
- **Voice Configuration**: Supports male/female voice switching

#### `FunctionToolsMixin` - The Function Toolkit
Provides **~30 specialized function tools** including:

**Planning Tools:**
- `create_plan()` - Multi-step task orchestration
- `update_plan_step()` - Progress tracking with UUID-based steps
- `modify_plan()` - Dynamic plan adjustment
- `validate_plan_step()` - Quality assurance

**Document Tools:**
- `create_document()` - Template search → LLM generation fallback
- `edit_document()` - Semantic diff generation with hover context
- `delete_document()` - Safe document removal
- `get_document()` - Document retrieval

**Utility Tools:**
- `search_web_firecrawl()` - Firecrawl web scraping integration
- `update_agent_memory()` - LLM-powered memory management
- `switch_voice()` - Real-time voice preference changes
- `collect_user_input()` - Interactive question collection

### 2.2 Planning Engine (`src/core/plan/`)

#### `PlanManager` - Multi-Step Task Orchestration
```python
class PlanManager:
    def __init__(self):
        self._current_plan: ExecutionPlan | None = None
        self._lock = asyncio.Lock()  # Thread-safe operations
```

**Plan Lifecycle:**
1. **Creation**: User request → LLM generates step list → `ExecutionPlan` object
2. **Execution**: Steps progress through: `PENDING → IN_PROGRESS → COMPLETED/FAILED`
3. **Tracking**: UUID-based step identification, timestamps, results storage
4. **Context**: Plan state injected into every LLM conversation turn

**Example Plan Structure:**
```json
{
  "id": "uuid-123",
  "user_request": "Create a marketing brief for AI vacuum",
  "steps": [
    {
      "id": "step-uuid-1",
      "type": "WEB_SEARCH",
      "description": "Research AI vacuum market trends",
      "status": "COMPLETED",
      "started_at": "2024-01-01T10:00:00",
      "completed_at": "2024-01-01T10:02:00",
      "result": {"summary": "Found 5 relevant articles"}
    },
    {
      "id": "step-uuid-2", 
      "type": "DOCUMENT_CREATE",
      "description": "Draft marketing brief",
      "status": "IN_PROGRESS"
    }
  ]
}
```

### 2.3 Document Workflows (`src/core/document/`)

#### `DocumentCreator` - Intelligent Document Generation
**Two-Stage Creation Process:**
1. **Template Search**: Vector similarity search against existing templates
2. **LLM Generation**: Fallback to GPT-4o with JSON mode for structured output

```python
async def create_new_document(self, title: str, description: str):
    # Stage 1: Try template-based creation
    template_content = await self._try_template_creation(title, description)
    
    if not template_content:
        # Stage 2: LLM generation fallback
        template_content = await self._generate_raw_document_json(title, description)
    
    # Send to frontend via RPC
    await self._create_document_via_rpc(template_content, title)
```

#### `SimpleDocumentEditor` - Semantic Editing
**Context-Aware Editing Process:**
1. **Context Gathering**: Document content + hover history + text selection
2. **Semantic Diff Generation**: LLM produces structured changes
3. **Application**: Changes applied via RPC to frontend

**Hover Context Usage:**
- When user says "make this more concise" while hovering over text
- System captures hover events and uses them for precise editing
- Supports both full-document and selection-based edits

### 2.4 State Management (`src/core/state/`)

#### `StateManager` - Universal State Container
```python
class AppState(BaseModel):
    # Current context
    active_document_id: str | None = None
    current_user_id: str | None = None  
    active_project_id: str | None = None
    
    # UI state
    text_selection: TextSelection | None = None
    url: str | None = None
    page_title: str | None = None
    
    # References
    documents: list[DocumentRef] = []
    questions: list[QuestionRef] = []
    
    # Hover tracking (limited history)
    hover_history: list[HoverEvent] = []
```

**Key Features:**
- **Thread-safe**: All operations protected by `asyncio.Lock`
- **Hover Tracking**: Captures UI interactions for context-aware editing
- **Format Context**: Automatic context formatting for LLM consumption
- **Validation**: Pydantic models ensure data integrity

### 2.5 Conversation Memory (`src/core/conversation/`)

#### `SummarizationService` - Intelligent Message Processing
**Three-Stage Processing Pipeline:**

1. **Content Filtering**: Removes filler words, determines meaningful content
```python
async def filter_content(self, text: str) -> str:
    # LLM determines if "uhm, ok, let me see..." should be stored
    # Returns cleaned content or empty string to skip
```

2. **Role-Aware Summarization**: Different strategies for user vs assistant messages
```python
async def summarize_content(self, text: str, role: str) -> str:
    # User messages: Focus on intent and requests
    # Assistant messages: Focus on actions taken and results
```

3. **Storage & Context Loading**: Summaries stored in backend, loaded for new sessions
- Only summaries loaded (not full transcripts) for efficient context
- Chronological order maintained
- Configurable history limits

---

## 3. Real-Time Communication Layer

### 3.1 Event Handling (`src/agent/event_handlers.py`)

#### Event Pipeline
```python
def setup_event_handlers():
    # STT transcript processing for shortcuts
    session.on("user_input_transcribed", on_user_input_transcribed_for_shortcut)
    
    # Conversation item processing
    session.on("conversation_item_added", on_conversation_item_added)
    
    # Metrics collection
    session.on("metrics_collected", _on_metrics_collected)
```

**Event Processing:**
- **User Input**: Every transcript checked for shortcuts (accept/reject/select)
- **Conversation Items**: Automatic background summarization and storage
- **Hover Events**: UI interactions captured for editing context

### 3.2 RPC Communication (`src/agent/rpc_handlers.py`)

#### Frontend ↔ Agent Communication
```python
@ctx.room.local_participant.register_rpc_method("sendMessage")
async def send_message_handler(rpc_data):
    # Process direct messages from frontend
    
@ctx.room.local_participant.register_rpc_method("analyzeDocument") 
async def analyze_document_handler(rpc_data):
    # Background document analysis for question generation
```

**RPC Methods:**
- `sendMessage`: Direct text communication
- `analyzeDocument`: Generate clarifying questions
- `updateInstructions`: Dynamic prompt modification

### 3.3 Shortcut Processing (`src/core/shortcut/`)

#### Voice Shortcuts for UI Actions
```python
class ShortcutProcessor:
    async def process_shortcut_text(self, text: str, agent, session, is_final: bool):
        # Fast LLM classification of voice commands:
        # "accept all" → ACCEPT_ALL
        # "reject changes" → REJECT_ALL  
        # "select document X" → SELECT:document-id
```

**Powered by Groq LLama** for sub-second response times:
- Separate from main conversation flow
- Processes both interim and final transcripts
- Maintains conversation context for better classification

---

## 4. Infrastructure & Integration Layer

### 4.1 LiveKit Integration (`src/infrastructure/livekit/`)

#### Session Factory & Configuration
```python
def create_session(agent) -> AgentSession:
    return AgentSession(
        vad=silero.VAD.load(activation_threshold=0.5),
        stt=deepgram.STT(model="nova-3", filler_words=True),
        llm=openai.LLM(model="gpt-4o-2024-11-20", temperature=0.5),
        tts=cartesia.TTS(model="sonic-2", voice=agent.current_voice_id),
        turn_detection=EnglishModel(),
        allow_interruptions=True
    )
```

#### LLM Provider Strategy
**Multi-Model Approach:**
- **Execution LLM**: OpenAI GPT-4o (temp=0.2) for precise function calls
- **Shortcut LLM**: Groq LLama (temp=0.1) for fast classification  
- **Memory Parsing LLM**: OpenAI GPT-4o (temp=0.2) for memory updates

#### Text Processing Pipeline
```python
async def clean_and_adjust_text(input_text: AsyncIterable[str]):
    # Pronunciation improvements for TTS:
    # "API" → "A P I"
    # "kubectl" → "kube control" 
    # Remove emojis, clean HTML entities
    # Normalize excessive whitespace
```

### 4.2 API Integration (`src/infrastructure/api_clients/`)

#### `ServiceAPIClient` - Backend Communication
```python
class ServiceAPIClient:
    def __init__(self, base_url: str, service_token: str):
        # JWT-based authentication
        # Automatic token expiry checking
        # 30-second timeout configuration
        
    async def search_document_templates(self, query: str, limit: int, threshold: float):
        # Vector similarity search for document templates
```

#### `FirecrawlAPIClient` - Web Search Integration
```python
class FirecrawlAPIClient:
    async def search_and_scrape(self, query: str, max_results: int, include_domains: list):
        # Advanced web scraping with content extraction
        # Used automatically by plan execution for richer content
        # LLM-powered content extraction and markdown formatting
        # Available through Firecrawl MCP server integration
```

---

## 5. Error Handling & Resilience

### 5.1 Centralized Error Management (`src/common/error_handler.py`)

#### Error Hierarchy
```python
class AgentError(Exception):
    def __init__(self, message: str, user_message: str | None = None):
        # Technical message for logs
        # User-friendly message for TTS

class DocumentError(AgentError): pass
class PlanError(AgentError): pass  
class LLMError(AgentError): pass
class APIError(AgentError): pass
```

#### Error Handling Patterns
```python
@handle_errors("document creation", "Failed to create the document")
async def create_new_document(self, title: str, description: str):
    # Decorator automatically:
    # 1. Logs full error details
    # 2. Hides loading states 
    # 3. Notifies user via TTS
    # 4. Re-raises for upstream handling
```

**Context Manager Pattern:**
```python
async with error_context("document editing", agent):
    # Risky operations here
    # Automatic error handling and user notification
```

### 5.2 Graceful Degradation
- **Template Search Failure**: Falls back to LLM generation
- **Web Search Unavailable**: Continues without external data
- **Memory Update Failure**: Logs error but doesn't block conversation
- **RPC Communication Issues**: Async tasks continue independently

---

## 6. Prompt Engineering & LLM Orchestration

### 6.1 Prompt Template System (`prompts/`)

#### Modular Prompt Architecture
```
prompts/
├── main.txt                      # Master agent instructions (197 lines)
├── main_instructions_content.txt # Core behavioral guidelines (338 lines)  
├── generate_document.txt         # Document creation prompt (128 lines)
├── generate_semantic_diff.txt    # Edit operation prompt (256 lines)
├── analyze_document.txt          # Question generation prompt (18 lines)
├── filter_content.txt           # Content filtering prompt (27 lines)
├── summarize_content.txt        # Summarization prompt (18 lines)
├── shortcut_classifier.txt      # Voice shortcut recognition (27 lines)
└── update_agent_memory.txt      # Memory management prompt (72 lines)
```

#### Dynamic Prompt Composition
```python
main_prompt_instructions = load_prompt_from_file(MAIN_PROMPT_PATH).format(
    main_instructions=main_instructions_content,
    user_instructions=user_instructions_content, 
    user_profile=user_profile_content,
    project_profile=project_profile_content,
)
```

### 6.2 Context Management Strategy

#### Automatic Context Injection
Every LLM turn receives:
1. **State Context**: Document list, active selections, UI state
2. **Plan Context**: Current step, completed/pending steps with progress indicators
3. **Timestamp**: Current date/time for temporal awareness
4. **Hover History**: Recent UI interactions for context-aware responses

#### Memory Optimization  
- **Conversation Summaries**: Only summaries loaded, not full transcripts
- **Hover History**: Limited to 5 most recent events
- **Plan Context**: Includes recent completions + upcoming steps
- **Document References**: IDs only, full content fetched on demand

---

## 7. Advanced Workflows & Scenarios

### 7.1 Complex Planning Scenario

**User Request**: *"Research competitors, create a pricing strategy document, and generate 3 follow-up questions"*

**Generated Plan**:
```json
{
  "user_request": "Research competitors, create pricing strategy, generate questions",
  "steps": [
    {
      "type": "WEB_SEARCH",
      "description": "Search for competitor pricing strategies", 
      "details": {"query": "SaaS pricing strategies competitors", "max_results": 8}
    },
    {
      "type": "DOCUMENT_CREATE", 
      "description": "Create pricing strategy document",
      "details": {"title": "Pricing Strategy", "use_research": true}
    },
    {
      "type": "CREATE_QUESTION",
      "description": "Generate market positioning question",
      "details": {"type": "strategic_question"}
    },
    {
      "type": "CREATE_QUESTION", 
      "description": "Generate pricing tier question",
      "details": {"type": "clarifying_question"}
    },
    {
      "type": "CREATE_QUESTION",
      "description": "Generate competitive analysis question", 
      "details": {"type": "strategic_question"}
    }
  ]
}
```

**Execution Flow**:
1. **Web Search**: Firecrawl API call, advanced content extraction and stored in plan context
2. **Document Creation**: Template search (finds "Strategy Template"), adapts with research data
3. **Question Generation**: LLM analyzes document and generates contextual questions
4. **Frontend Updates**: Each step completion triggers RPC updates to UI
5. **Progress Tracking**: Plan visualization updates in real-time

### 7.2 Context-Aware Editing Workflow

**Scenario**: User hovers over a specific paragraph and says *"make this section more technical"*

**System Response**:
1. **Hover Capture**: Frontend sends hover event with element metadata
2. **Context Building**: 
   ```python
   context = {
       "document_content": full_document_text,
       "hover_state": [latest_hover_event],
       "text_selection": selected_text_if_any,
       "use_hover_context": True
   }
   ```
3. **Semantic Diff Generation**: LLM receives context and generates specific changes
4. **Change Application**: Structured diff applied to document via RPC
5. **Plan Integration**: If part of active plan, step marked as completed

### 7.3 Memory Evolution Scenario

**Scenario**: User corrects a previous statement about project requirements

**User**: *"Actually, we're targeting enterprise customers, not SMBs"*

**System Response**:
1. **Memory Analysis**: LLM processes correction against existing memory
2. **Memory Update**: 
   ```python
   await update_agent_memory_tool(
       agent,
       memory_type="project_profile",
       memory_update="Target market changed from SMB to enterprise customers"
   )
   ```
3. **Context Propagation**: Updated memory available in next conversation turn
4. **Plan Adaptation**: If active plan exists, considers if steps need modification

---

## 8. Performance & Scalability Considerations

### 8.1 Real-Time Performance
- **STT Latency**: Deepgram Nova-3 for fast transcription
- **LLM Response**: Multiple specialized models for different use cases
- **TTS Generation**: Cartesia Sonic-2 for low-latency speech synthesis
- **Shortcut Processing**: Groq LLama for sub-second classification

### 8.2 Memory Efficiency
- **Conversation Summarization**: Reduces context size by ~80%
- **State Snapshots**: Only active state kept in memory
- **Template Caching**: Document templates cached for reuse
- **LLM Instance Pooling**: Reuse configured LLM instances

### 8.3 Error Recovery Patterns
- **Async Task Isolation**: Background tasks don't block main conversation
- **Graceful Degradation**: Core functionality continues if optional features fail
- **Automatic Retry**: Network requests with exponential backoff
- **State Consistency**: Atomic state updates with rollback capability

---

## 9. Development & Extension Patterns

### 9.1 Adding New Function Tools

**Step 1**: Create tool implementation
```python
# src/core/tools/my_new_tool.py
async def my_new_tool(agent, param1: str, param2: int) -> str:
    """Tool description for LLM context."""
    # Implementation here
    return "Success message"
```

**Step 2**: Add to FunctionToolsMixin
```python  
# src/agent/function_tools.py
@function_tool
async def my_new_tool(self, param1: str, param2: int):
    """Exposed function tool with parameter validation."""
    return await my_new_tool_impl(self, param1, param2)
```

**Step 3**: Update plan step types (if needed)
```python
# src/core/plan/type.py  
class StepType(str, Enum):
    MY_NEW_TOOL = "my_new_tool"
```

### 9.2 Custom Prompt Development

**Best Practices**:
- Use placeholder variables for dynamic content injection
- Include examples for few-shot learning
- Specify output format clearly (JSON, plain text, etc.)
- Test with edge cases and varied inputs

**Example Template**:
```
You are an expert assistant specialized in {domain}.

Your task: {task_description}

Context:
{dynamic_context}

Output Format:
{format_specification}

Examples:
{few_shot_examples}
```

### 9.3 State Management Extensions

**Adding New State Fields**:
```python
# src/core/state/type.py
class AppState(BaseModel):
    # Existing fields...
    new_feature_data: MyCustomType | None = None
    
# src/core/state/state_manager.py  
async def get_new_feature_context(self) -> str:
    """Format new feature data for LLM context."""
    # Implementation here
```

---

## 10. Configuration & Deployment

### 10.1 Environment Configuration (`constants.py`)

**Required Environment Variables**:
```bash
# LiveKit Configuration
LIVEKIT_URL=wss://your-livekit-server.com
LIVEKIT_API_KEY=your-api-key
LIVEKIT_API_SECRET=your-api-secret

# LLM & Speech Configuration  
OPENAI_API_KEY=your-openai-key
DEEPGRAM_API_KEY=your-deepgram-key
CARTESIA_API_KEY=your-cartesia-key
GROQ_API_KEY=your-groq-key

# Backend Integration
BACKEND_API_BASE_URL=https://api.yourbackend.com
SERVICE_API_TOKEN=your-jwt-token

# External APIs
FIRECRAWL_API_KEY=your-firecrawl-api-key
TAVILY_API_KEY=your-tavily-key

# Voice Configuration
MALE_VOICE_ID=your-male-voice-id
FEMALE_VOICE_ID=your-female-voice-id
```

### 10.2 Model Configuration
**Current Defaults** (adjustable via constants):
- **STT**: Deepgram Nova-3 (fast, accurate)
- **Main LLM**: GPT-4o-2024-11-20 (temperature=0.5)
- **Execution LLM**: GPT-4o-2024-11-20 (temperature=0.2) 
- **Shortcut LLM**: Meta-Llama/Llama-4-Maverick-17b (temperature=0.1)
- **TTS**: Cartesia Sonic-2 (low latency)

### 10.3 Asset Management
- **Thinking Sounds**: Place audio files in `/assets` directory
- **Supported Formats**: .mp3, .wav, .ogg, .m4a, .flac
- **Random Selection**: System randomly selects thinking sounds during processing

---

This detailed guide provides a comprehensive understanding of the codebase architecture, data flows, and implementation patterns. Each component is designed for modularity, testability, and extensibility while maintaining high performance for real-time voice interactions. 