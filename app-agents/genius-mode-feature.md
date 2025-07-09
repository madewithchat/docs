## 🔍 Feature Explanation: Genius Mode

### What is Genius Mode?

**Genius Mode** is an advanced voice-activated research and ideation engine that enables our users to conduct **deep research** through **natural conversation**. It uses **multiple AI agents working in parallel** to investigate topics, validate assumptions, generate ideas, and compile actionable insights — all in real-time and tailored to user-selected subject areas. In **Genius Mode**, users can generate comprehensive **Genius Reports** from selected content within a project.

---

## 🧠 How It Works: Functional Overview

### 1. **User Interaction Flow**

- **Step 1**: The user activates “Genius Mode” via voice or UI command.
- **Step 2**: The system presents customization options:
  - **Focus Mode Selector**:
    - Magic Mode (AI chooses dynamically)
    - Research (factual retrieval)
    - Validate (assumption/claim validation)
    - Ideate (creative idea generation)
  - **Subject Area Toggles (Documents)**:
    - Overview
    - Legal
    - Marketing
    - Viral Discussion
    - Reporting
- **Step 3**: User chooses genius mode (research, ideation etc.) and subjects they want to include. Additionally, user can refine output with their own prompt.
- **Step 4**: The results are aggregated and we get the Genius Report — a large document consisting of dozens of pages which the user can refine later.

---

## 🧩 Key Functionalities

### 1. **Modes = Agents**

These are **behavioral templates** or **prompt presets** that define *how* the assistant should think:

- 🧙 **Magic Mode**: Creative synthesis across broad topics
- 🧪 **Research Mode**: Evidence-based, analytical tone. Agent uses external web retrieval.
- 🧠 **Validate Mode**: Critical review, refinement, or summarization. Internal logic to assess source credibility or contradiction.
- 💡 **Ideate Mode**: Brainstorming and idea generation. Prompts tuned for creativity and blue-sky thinking, possibly chain-of-thought prompting.

**→ These should be implemented as agent configurations.** Each "mode" is essentially a **LangGraph node or subgraph** with a unique prompt scaffold and optional tool access.

### 2. **Subjects = Documents / Scopes**

Subjects define *what* the AI should focus on (these can be different for your project):

- Overview (high-level business model articulation)
- Legal (uses legal templates)
- Marketing (positioning, messaging, audience insights)
- Viral Discussion (social and competitor trends)
- Reporting

Each subject acts as a **data filter**—grouping relevant content from the document tree (e.g., “Investor Report”, “Founders election...”) before feeding it into the LangGraph context.

**→ These are not agents**—they’re input segmentation mechanisms. You can think of these as toggles controlling the subset of memory passed to the agent.

### 3. **User Prompt = Additional Modifier**

The user can add a **custom prompt** to further guide the tone, scope, or emphasis of the Genius output.

This is concatenated with the mode’s default prompt and passed to the language model during execution.

---

## 📚 More Resources

- [Video Walkthrough (Fathom)](https://fathom.video/share/HVcmL4hPBUgZUFi2WBicvHapNG57Mjzi)
- ![1.jpg](attachment:177ab170-aadd-4eef-8cf4-2fa86e0d4e3f:1.jpg)
- ![2.jpg](attachment:fc3eb1fb-6487-43eb-bfcc-c29da4b39c97:2.jpg)
- ![image.png](attachment:c6d97e68-9f04-414d-89a7-424ae52979e2:image.png)

### Additional Sources

- [Deep Research AI Workflow using LangGraph + Tavily](https://medium.com/@gaurav219688/deep-research-ai-workflow-using-langgraph-tavily-search-any-llm-provider-373ae5aa2cfd)
- [LangGraph on Datahub](https://datahub.io/@donbr/langgraph-unleashed/langgraph_deep_research)
- [LangGraph Studio Concepts](https://langchain-ai.github.io/langgraph/concepts/langgraph_studio/)
- [Build Deep Research Agent with Qwen3 (Dev.to)](https://dev.to/composiodev/a-comprehensive-guide-to-building-a-deep-research-agent-with-qwen3-locally-1jgm)
- [LangGraph 101 (Medium)](https://shuaiguo.medium.com/langgraph-101-lets-build-a-deep-research-agent-with-gemini-228b50076269)
- [Analytics Vidhya Guide](https://www.analyticsvidhya.com/blog/2025/02/build-your-own-deep-research-agent/)
- [Composio Blog](https://composio.dev/blog/building-a-deep-research-agent-using-composio-and-langgraph)
- [Reddit Post on Qwen3](https://www.reddit.com/r/LangChain/comments/1kmfcrp/built_a_local_deep_research_agent_using_qwen3/)
- [LangChain Blog on Exa](https://blog.langchain.com/exa/)
- [GoPubby: Multi-Agent RAG](https://ai.gopubby.com/building-rag-research-multi-agent-with-langgraph-1bd47acac69f)
