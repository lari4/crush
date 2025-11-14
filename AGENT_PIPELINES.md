# Crush Agent Pipelines Documentation

This document describes all agent execution pipelines in the Crush application, including data flow, prompt handling, and agent orchestration.

## Table of Contents

1. [Main Coder Agent Pipeline](#1-main-coder-agent-pipeline)
2. [Sub-Agent Pipeline (Agent Tool)](#2-sub-agent-pipeline-agent-tool)
3. [Agentic Fetch Pipeline](#3-agentic-fetch-pipeline)
4. [Session Management Pipelines](#4-session-management-pipelines)
5. [Initialization Pipeline](#5-initialization-pipeline)
6. [Overall System Architecture](#overall-system-architecture)

---

## 1. Main Coder Agent Pipeline

**Purpose**: Primary pipeline for interactive coding sessions where user sends prompts and receives AI-powered assistance.

### Entry Points
- **Interactive TUI**: `internal/cmd/root.go:77` - `rootCmd.RunE()`
- **CLI Mode**: `internal/cmd/run.go` - `crush run "prompt"`

### Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                        USER INPUT                                │
│                  (via TUI or CLI command)                        │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│               Application Initialization                         │
│         app.InitCoderAgent(ctx) - app.go:108                     │
│                                                                  │
│  • Creates Coordinator with config                              │
│  • Sets up services (Sessions, Messages, Permissions)           │
│  • Initializes LSP clients                                      │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│              Coordinator Setup - coordinator.go:71               │
│                  NewCoordinator()                                │
│                                                                  │
│  1. Load AgentCoder config from cfg.Agents:90-92                │
│  2. Build system prompt via coderPrompt():96                    │
│  3. Create main agent via buildAgent():101-106                  │
│  4. Build tools asynchronously:300-307                          │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│           Prompt Assembly - prompts.go:20-26                     │
│                  coderPrompt()                                   │
│                                                                  │
│  • Loads template: templates/coder.md.tpl:21                    │
│  • Renders with context (WorkingDir, IsGitRepo, Platform,       │
│    Date, GitStatus, ContextFiles):283                           │
│  • Adds provider-specific prefix:292                            │
│                                                                  │
│  Template Variables:                                            │
│    {{.WorkingDir}}     - Current directory                      │
│    {{.IsGitRepo}}      - Git repo detection                     │
│    {{.Platform}}       - Operating system                       │
│    {{.Date}}           - Current date                           │
│    {{.GitStatus}}      - Branch/status info                     │
│    {{.ContextFiles}}   - Memory files                           │
│    {{.Config.LSP}}     - LSP configuration                      │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│         Agent Creation - coordinator.go:277-310                  │
│                  buildAgent()                                    │
│                                                                  │
│  • Build models (large & small):278                             │
│  • Assemble final system prompt:283                             │
│  • Create SessionAgent with options:289-299                     │
│    - Large model for main processing                            │
│    - Small model for quick tasks                                │
│    - System prompt prefix                                       │
│    - Auto-summarize flag                                        │
│    - YOLO mode flag                                             │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│          Tool Building - coordinator.go:312-391                  │
│                  buildTools()                                    │
│                                                                  │
│  Core Tools (always available):                                 │
│    • Bash          - Command execution                          │
│    • Edit          - File modification                          │
│    • Write         - File creation                              │
│    • View          - File reading                               │
│    • Glob          - Pattern matching                           │
│    • Grep          - Content search                             │
│    • LS            - Directory listing                          │
│    • MultiEdit     - Multiple edits                             │
│                                                                  │
│  Conditional Tools:                                             │
│    • Agent         - Sub-agent launcher:314-320                 │
│    • AgenticFetch  - Web analysis:322-328                       │
│    • Diagnostics   - LSP errors:354-356                         │
│    • References    - LSP symbol search:354-356                  │
│    • Fetch         - Raw URL fetch:338-352                      │
│    • Download      - File download:338-352                      │
│                                                                  │
│  Filtering:                                                     │
│    • Based on AllowedTools config:358-391                       │
│    • MCP permissions check:358-391                              │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│         Execution Start - coordinator.go:111-145                 │
│                  coordinator.Run()                               │
│                                                                  │
│  • Validate model and check token limits:116-124                │
│  • Filter attachments by model capabilities:116-124             │
│  • Merge provider options (temperature, penalties):126-143      │
│  • Call currentAgent.Run(SessionAgentCall{...}):133-144         │
│                                                                  │
│  SessionAgentCall contains:                                     │
│    - SessionID                                                  │
│    - Prompt (user message)                                      │
│    - ProviderOptions                                            │
│    - Attachments                                                │
│    - MaxOutputTokens, Temperature, TopP, TopK                   │
│    - FrequencyPenalty, PresencePenalty                          │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│        Main Execution - agent.go:123-517                         │
│              sessionAgent.Run()                                  │
│                                                                  │
│  Phase 1: Setup (lines 132-178)                                 │
│    • Queue request if session busy:132-140                      │
│    • Create fantasy.Agent with system prompt and tools:147-151  │
│    • Fetch session messages from DB:159-162                     │
│    • Generate title async (first message only):166-172          │
│    • Create user message with attachments:175-178               │
│    • Add SessionID to context:181                               │
│                                                                  │
│  Phase 2: Streaming (lines 196-396)                             │
│    agent.Stream() with callbacks:                               │
│                                                                  │
│    • PrepareStep:                                               │
│      - Create assistant message in DB                           │
│      - Process queued messages                                  │
│      - Handle cache control                                     │
│                                                                  │
│    • OnTextDelta:                                               │
│      - Append response text to message                          │
│      - Update UI in real-time                                   │
│                                                                  │
│    • OnToolInputStart:                                          │
│      - Track tool call initiation                               │
│                                                                  │
│    • OnToolCall:                                                │
│      - Execute tool with parameters                             │
│      - Pass session context to tool                             │
│      - Handle permissions if required                           │
│                                                                  │
│    • OnToolResult:                                              │
│      - Record tool execution result                             │
│      - Add to conversation history                              │
│                                                                  │
│    • OnStepFinish:                                              │
│      - Calculate token usage                                    │
│      - Update session cost                                      │
│      - Save session to DB                                       │
│                                                                  │
│  Phase 3: Post-Processing (lines 400-517)                       │
│    • Error handling (cancellation, permissions):400-485         │
│    • Auto-summarize if context threshold reached:488-503        │
│    • Process queued messages recursively:505-516                │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                    Result & Persistence                          │
│                                                                  │
│  • Assistant message saved to DB with full content              │
│  • Session updated with token usage and cost                    │
│  • Tool calls and results recorded                              │
│  • fantasy.AgentResult returned to coordinator                  │
└──────────────────────────────────────────────────────────────────┘
```

### Data Flow Summary

**Input**: User prompt + attachments + session context
↓
**Processing**: System prompt + message history + tool execution
↓
**Output**: Assistant response + updated session + tool results

### Key Code References

- Entry: `internal/cmd/root.go:77`, `internal/cmd/run.go`
- Initialization: `internal/app/app.go:108`, `internal/app/app.go:318-326`
- Coordinator: `internal/agent/coordinator.go:71-108`, `coordinator.go:111-145`
- Prompt: `internal/agent/prompts.go:20-26`, `coordinator.go:283`
- Execution: `internal/agent/agent.go:123-517`
- Streaming: `agent.go:196-396`

---

## 2. Sub-Agent Pipeline (Agent Tool)

**Purpose**: Allows main agent to delegate complex, multi-step tasks to specialized sub-agents that can search, read, and analyze code independently.

### Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────┐
│              Main Agent Decides to Use Agent Tool                │
│             (During tool selection in main pipeline)             │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│          Tool Definition - agent_tool.go:27-105                  │
│              coordinator.agentTool()                             │
│                                                                  │
│  • Loads tool description: templates/agent_tool.md:17           │
│  • Retrieves AgentTask config from cfg.Agents:28                │
│  • Loads task prompt template via taskPrompt():32               │
│  • Builds task agent using buildAgent():37-40                   │
│    (Same buildAgent() as main coder agent)                      │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│         Tool Execution - agent_tool.go:44-104                    │
│              Tool Handler Function                               │
│                                                                  │
│  Step 1: Extract Context                                        │
│    • Get SessionID from context:49-52                           │
│    • Get AgentMessageID from context:49-52                      │
│                                                                  │
│  Step 2: Create Sub-Session                                     │
│    • Generate sub-session ID:59                                 │
│      CreateAgentToolSessionID(agentMessageID, call.ID)          │
│    • Create task session linked to parent:60                    │
│                                                                  │
│  Step 3: Build Call Parameters                                  │
│    • Construct SessionAgentCall:65-83                           │
│      - SessionID: sub-session ID                                │
│      - Prompt: user's task description                          │
│      - ProviderOptions: inherited from parent                   │
│      - Attachments: none (sub-agents don't have attachments)    │
│                                                                  │
│  Step 4: Execute Task Agent                                     │
│    • Run task agent in isolated session:74-84                   │
│      agent.Run(ctx, SessionAgentCall{...})                      │
│    • Task agent has limited tools:                              │
│      - GlobTool (file pattern matching)                         │
│      - GrepTool (content search)                                │
│      - LS (directory listing)                                   │
│      - View (file reading)                                      │
│    • NO editing tools (Edit, Write, Bash)                       │
│                                                                  │
│  Step 5: Cost Aggregation                                       │
│    • Update parent session with sub-agent costs:88-102          │
│    • Fetch task session from DB:88-90                           │
│    • Add costs to parent session:91-95                          │
│    • Save parent session:96-100                                 │
│                                                                  │
│  Step 6: Return Result                                          │
│    • Extract response text from task agent result:103           │
│    • Return as fantasy.NewTextResponse()                        │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                   Result Integration                             │
│                                                                  │
│  • Sub-agent response returned to main agent                    │
│  • Main agent receives text response as tool result             │
│  • Main agent continues processing with new information         │
│  • Sub-session visible in TUI as nested conversation            │
└──────────────────────────────────────────────────────────────────┘
```

### ASCII Diagram

```
Main Coder Agent
     │
     │ (needs to search/analyze code)
     │
     ├──> Tool Call: agent
     │    Params: {prompt: "Find all error handling patterns"}
     │
     └──> agent_tool.go:44
          │
          ├──> Create Sub-Session
          │    ID: agent_tool_<parent_msg_id>_<call_id>
          │
          ├──> Build Task Agent
          │    Prompt: templates/task.md.tpl
          │    Tools: [Glob, Grep, LS, View]
          │
          ├──> Execute: agent.Run(sub-session, prompt)
          │    │
          │    ├──> Task agent searches codebase
          │    ├──> Uses Glob to find files
          │    ├──> Uses Grep to search content
          │    ├──> Uses View to read files
          │    └──> Returns analysis result
          │
          ├──> Aggregate costs to parent session
          │
          └──> Return result text to main agent
               │
               └──> Main agent continues with findings
```

### Key Differences from Main Agent

| Aspect | Main Coder Agent | Task Sub-Agent |
|--------|-----------------|----------------|
| **Prompt Template** | `templates/coder.md.tpl` | `templates/task.md.tpl` |
| **Configuration** | `AgentCoder` config | `AgentTask` config |
| **Available Tools** | Full tool set (Edit, Bash, etc.) | Limited (Glob, Grep, LS, View) |
| **Session** | User's main session | Isolated sub-session |
| **Purpose** | Interactive coding assistance | Focused search/analysis task |
| **Editing Capability** | Yes (can modify files) | No (read-only) |
| **Response Style** | Conversational, detailed | Concise, direct answers |

### Code References

- Tool Definition: `internal/agent/agent_tool.go:27-105`
- Prompt: `internal/agent/prompts.go:28-34` - `taskPrompt()`
- Template: `internal/agent/templates/task.md.tpl`
- Tool Description: `internal/agent/templates/agent_tool.md`

---

## 3. Agentic Fetch Pipeline

**Purpose**: Fetches web content and uses a specialized AI agent to analyze, extract information, or answer questions about the content.

### Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────┐
│          Main Agent Calls agentic_fetch Tool                     │
│        Params: {url: "...", prompt: "extract..."}               │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│       Tool Entry - agentic_fetch_tool.go:57                      │
│          coordinator.agenticFetchTool()                          │
│                                                                  │
│  • Embeds tool description: templates/agentic_fetch.md:20       │
│  • Defines when to use vs fetch tool                            │
│    - Use agentic_fetch: extract, analyze, summarize             │
│    - Use fetch: raw content, API responses                      │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Validation & Permission - agentic_fetch_tool.go:72-92       │
│                                                                  │
│  • Validate URL and prompt parameters:73                        │
│  • Request user permission:78-88                                │
│    - Description: "Fetch and analyze URL"                       │
│    - Shows URL to user                                          │
│    - User approves/denies                                       │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Content Fetching - agentic_fetch_tool.go:94-124             │
│                                                                  │
│  Step 1: Fetch URL                                              │
│    • Call tools.FetchURLAndConvert(url):94                      │
│    • Converts HTML to markdown                                  │
│                                                                  │
│  Step 2: Handle Content Size                                    │
│    • Create temp directory:99-103                               │
│                                                                  │
│    If LARGE content (>threshold):105-124                        │
│      - Save to temp file                                        │
│      - Return path to agent                                     │
│      - Agent must use view/grep tools                           │
│                                                                  │
│    If SMALL content:                                            │
│      - Embed directly in prompt                                 │
│      - Agent analyzes immediately                               │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│    Fetch Agent Creation - agentic_fetch_tool.go:126-168          │
│                                                                  │
│  Step 1: Build Prompt                                           │
│    • Create from template: agentic_fetch_prompt.md.tpl:126-133  │
│    • Render with context:135-143                                │
│      - WorkingDir: temp directory                               │
│      - IsGitRepo: false                                         │
│      - Platform, Date                                           │
│                                                                  │
│  Step 2: Build Tool Set                                         │
│    Restricted tools for web analysis:150-156                    │
│      • WebFetchTool - Follow links on page                      │
│      • GlobTool     - Search temp directory                     │
│      • GrepTool     - Search fetched content                    │
│      • ViewTool     - Read fetched files                        │
│                                                                  │
│  Step 3: Create Session Agent                                   │
│    • Uses SMALL model (cost efficiency):158-168                 │
│    • Working dir: temp directory                                │
│    • Auto-summarize: disabled                                   │
│    • System prompt: task-style for web analysis                 │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Session Management - agentic_fetch_tool.go:170-177          │
│                                                                  │
│  • Create isolated fetch session:170                            │
│    ID: agentic_fetch_<parent_msg_id>_<call_id>                  │
│  • Auto-approve permissions for nested fetches:176              │
│    (Sub-agent can fetch linked pages without asking)            │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Execution & Result - agentic_fetch_tool.go:184-215          │
│                                                                  │
│  Step 1: Execute Fetch Agent                                    │
│    • Build analysis prompt:184-194                              │
│      - User's original question/task                            │
│      - Fetched content (or file path)                           │
│    • Run agent: agent.Run(ctx, SessionAgentCall{...}):184-194   │
│    • Agent processes content and returns analysis               │
│                                                                  │
│  Step 2: Cost Aggregation                                       │
│    • Update parent session with fetch costs:199-213             │
│    • Fetch sub-session from DB:199-201                          │
│    • Add costs to parent:203-207                                │
│    • Save parent session:208-211                                │
│                                                                  │
│  Step 3: Return Result                                          │
│    • Extract analysis text:215                                  │
│    • Return as fantasy.NewTextResponse()                        │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                   Result Integration                             │
│                                                                  │
│  • Analysis returned to main agent                              │
│  • Main agent receives extracted information                    │
│  • Temp directory cleaned up after session                      │
└──────────────────────────────────────────────────────────────────┘
```

### ASCII Diagram

```
Main Agent
     │
     │ (needs web content analysis)
     │
     ├──> Tool Call: agentic_fetch
     │    Params: {
     │      url: "https://example.com/docs",
     │      prompt: "Extract API endpoints and their descriptions"
     │    }
     │
     └──> agentic_fetch_tool.go:72
          │
          ├──> Request Permission
          │    User approves URL fetch
          │
          ├──> Fetch URL Content
          │    tools.FetchURLAndConvert()
          │    ├──> HTML fetched
          │    └──> Converted to markdown
          │
          ├──> Check Content Size
          │    │
          │    ├──> LARGE: Save to /tmp/fetch_xxx/content.md
          │    │
          │    └──> SMALL: Embed in prompt
          │
          ├──> Create Fetch Agent
          │    Prompt: templates/agentic_fetch_prompt.md.tpl
          │    Model: SMALL (for cost efficiency)
          │    Tools: [WebFetch, Glob, Grep, View]
          │    WorkingDir: /tmp/fetch_xxx/
          │
          ├──> Execute Fetch Agent
          │    agent.Run(fetch_session, analysis_prompt)
          │    │
          │    Agent can:
          │    ├──> View content with View tool
          │    ├──> Search with Grep tool
          │    ├──> Follow links with WebFetch tool
          │    └──> Return extracted information
          │
          ├──> Aggregate costs to parent
          │
          └──> Return analysis to main agent
               │
               └──> Main agent uses extracted info
```

### When to Use: agentic_fetch vs fetch

| Use Case | Tool | Reason |
|----------|------|--------|
| Extract specific data from webpage | `agentic_fetch` | Needs AI to parse and extract |
| Answer questions about content | `agentic_fetch` | Needs AI to understand and respond |
| Summarize article | `agentic_fetch` | Needs AI to summarize |
| Get raw HTML | `fetch` | No processing needed |
| Get API JSON response | `fetch` | No analysis needed |
| Direct content access | `fetch` | Faster, cheaper |

### Code References

- Tool Entry: `internal/agent/agentic_fetch_tool.go:57-215`
- Fetch Agent Prompt: `internal/agent/templates/agentic_fetch_prompt.md.tpl`
- Tool Description: `internal/agent/templates/agentic_fetch.md`
- URL Fetching: `internal/agent/tools/tools.go` - `FetchURLAndConvert()`

---

## 4. Session Management Pipelines

### 4.1 Summarization Pipeline

**Purpose**: Automatically condenses conversation history when approaching context window limits, preserving essential information while reducing token usage.

### Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────┐
│          Context Window Threshold Detection                      │
│              (during agent streaming)                            │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Threshold Check - agent.go:378-395                          │
│            StopWhen condition in Stream()                        │
│                                                                  │
│  Calculate remaining tokens:380-388                             │
│    remaining = max_output - (input + accumulated_output)        │
│                                                                  │
│  Trigger summarize if:389-392                                   │
│    • remaining <= threshold AND                                 │
│    • auto_summarize not disabled                                │
│                                                                  │
│  Thresholds:                                                    │
│    • Large models (>200k context): 20k tokens                   │
│    • Other models: 20% of max_output                            │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Summarization Start - agent.go:519-624                      │
│              Summarize() Method                                  │
│                                                                  │
│  Step 1: Validation                                             │
│    • Check session not busy:520-522                             │
│    • Retrieve session and messages:524-535                      │
│                                                                  │
│  Step 2: Prepare Message History                                │
│    • Call preparePrompt():537                                   │
│    • Converts all messages to prompt format                     │
│    • Includes user messages, assistant responses, tool calls    │
│                                                                  │
│  Step 3: Create Summary Agent                                   │
│    • Load summary prompt:544-546                                │
│      Template: templates/summary.md                             │
│    • Create fantasy.Agent with summary instructions             │
│    • Uses same model as main agent                              │
│                                                                  │
│  Step 4: Create Summary Message                                 │
│    • Create in database:547-555                                 │
│    • Type: AssistantMessage                                     │
│    • Role: Summary                                              │
│                                                                  │
│  Step 5: Stream Summary                                         │
│    • Execute agent.Stream():557-586                             │
│                                                                  │
│    Callbacks:                                                   │
│      • OnReasoningDelta:                                        │
│        Track reasoning content (thinking process)               │
│                                                                  │
│      • OnReasoningEnd:                                          │
│        Handle reasoning signatures                              │
│                                                                  │
│      • OnTextDelta:                                             │
│        Append summary text to message                           │
│                                                                  │
│  Step 6: Update Session                                         │
│    • Aggregate token usage:615                                  │
│    • Set SummaryMessageID:619-621                               │
│    • Reset input/output token counters:619-621                  │
│    • Save session:622                                           │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Message History Truncation - agent.go:683-703               │
│              getSessionMessages() Method                         │
│                                                                  │
│  If summary exists:689-701                                      │
│    • Find summary message index in history                      │
│    • Keep only messages from summary onwards                    │
│    • Convert summary to user message for context               │
│    • Discard older messages (before summary)                    │
│                                                                  │
│  Result:                                                        │
│    • Reduced message history                                    │
│    • Summary preserves context from discarded messages          │
│    • Next agent call uses truncated history                     │
└──────────────────────────────────────────────────────────────────┘
```

### Summary Prompt Structure

From `templates/summary.md`:

```
Required Sections:
  1. Current State
     - Exact user request
     - Progress completed
     - Current work in progress
     - Remaining tasks (specific)

  2. Files & Changes
     - Modified files with descriptions
     - Read/analyzed files and relevance
     - Files yet to be touched
     - File paths with line numbers

  3. Technical Context
     - Architecture decisions and rationale
     - Patterns followed with examples
     - Libraries/frameworks used
     - Commands (successful and failed)
     - Environment details

  4. Strategy & Approach
     - Overall approach
     - Why chosen over alternatives
     - Key insights or gotchas
     - Assumptions made
     - Blockers or risks

  5. Exact Next Steps
     - Specific actions (not vague)
     - File paths and line numbers
     - Commands to run
```

### ASCII Diagram

```
Agent Processing
     │
     │ (approaching context limit)
     │
     ├──> Check Threshold
     │    remaining_tokens <= 20k?
     │    │
     │    └──> YES → Trigger Summarize
     │
     └──> agent.Summarize()
          │
          ├──> Prepare Message History
          │    All messages since last summary
          │
          ├──> Create Summary Agent
          │    Prompt: templates/summary.md
          │    Instructions: comprehensive summary
          │
          ├──> Stream Summary Generation
          │    │
          │    ├──> Section 1: Current State
          │    ├──> Section 2: Files & Changes
          │    ├──> Section 3: Technical Context
          │    ├──> Section 4: Strategy & Approach
          │    └──> Section 5: Exact Next Steps
          │
          ├──> Save Summary Message
          │    Role: Summary
          │    Content: Full summary text
          │
          ├──> Update Session
          │    Set SummaryMessageID
          │    Reset token counters
          │
          └──> Next agent call
               │
               └──> Load messages from summary onwards
                    (older messages discarded)
```

### Code References

- Threshold Detection: `internal/agent/agent.go:378-395`
- Summarization: `internal/agent/agent.go:519-624`
- Message Truncation: `internal/agent/agent.go:683-703`
- Summary Prompt: `internal/agent/templates/summary.md`

---

### 4.2 Title Generation Pipeline

**Purpose**: Automatically generates concise titles for conversations based on the user's first message.

### Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────┐
│          First Message in Session Detected                       │
│                (in sessionAgent.Run())                           │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Async Title Generation - agent.go:166-172                   │
│              Spawned in goroutine                                │
│                                                                  │
│  • Check if first message (session.Title empty)                 │
│  • Launch wg.Go(generateTitle) asynchronously                   │
│  • Main execution continues without blocking                    │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Title Generation - agent.go:705-770                         │
│              generateTitle() Method                              │
│                                                                  │
│  Step 1: Validation                                             │
│    • Check prompt not empty:705-708                             │
│    • Skip if already has title                                  │
│                                                                  │
│  Step 2: Model Selection                                        │
│    • Use SMALL model for cost efficiency:710-713                │
│    • Faster generation                                          │
│    • Lower cost                                                 │
│                                                                  │
│  Step 3: Create Title Agent                                     │
│    • Load title prompt:715-718                                  │
│      Template: templates/title.md                               │
│      System prompt includes "/no_think" directive               │
│    • Set max output tokens:                                     │
│      - 40 tokens (or model default if reasoning enabled)        │
│                                                                  │
│  Step 4: Generate Title                                         │
│    • Build prompt:720-729                                       │
│      - First 100 characters of user message                     │
│      - Wrapped in thinking tags if needed                       │
│    • Execute agent.Stream() with OnTextDelta callback           │
│    • Accumulate title text                                      │
│                                                                  │
│  Step 5: Post-Process Title                                     │
│    • Remove newlines:737                                        │
│    • Strip thinking tags if present:740-742                     │
│    • Truncate to reasonable length                              │
│                                                                  │
│  Step 6: Update Session                                         │
│    • Set session.Title:750                                      │
│    • Calculate costs:752-764                                    │
│      - Token usage                                              │
│      - OpenRouter costs if applicable                           │
│    • Save session:765                                           │
└──────────────────────────────────────────────────────────────────┘
```

### Title Prompt Rules

From `templates/title.md`:

```
Rules:
  • Max 50 characters
  • Summary of user's message
  • One line only
  • No quotes or colons
  • Entire response becomes title
  • No multi-sentence responses
```

### ASCII Diagram

```
Session Start
     │
     │ (first message)
     │
     ├──> sessionAgent.Run()
     │    │
     │    ├──> Create user message
     │    │
     │    └──> Spawn async: generateTitle()
     │              │
     │              ├──> Use SMALL model
     │              │    (cost efficiency)
     │              │
     │              ├──> Load title.md prompt
     │              │    Rules: <50 chars, no quotes/colons
     │              │
     │              ├──> Stream title generation
     │              │    Input: first 100 chars of message
     │              │    Output: concise title
     │              │
     │              ├──> Post-process
     │              │    Remove newlines, strip tags
     │              │
     │              └──> Update session.Title
     │                   Save to DB
     │
     └──> Main execution continues
          (doesn't wait for title)
```

### Example

**User Input**: "Help me refactor the authentication module to use JWT tokens instead of sessions"

**Title Generation Process**:
1. Extract: "Help me refactor the authentication module to use JWT tokens instead..."
2. Generate: "Refactor auth to JWT tokens"
3. Save: session.Title = "Refactor auth to JWT tokens"

### Code References

- Async Launch: `internal/agent/agent.go:166-172`
- Title Generation: `internal/agent/agent.go:705-770`
- Title Prompt: `internal/agent/templates/title.md`

---

## 5. Initialization Pipeline

**Purpose**: Analyzes codebase on first use and generates initialization files (like `.cursorrules`) to help future agents understand project structure, commands, and conventions.

### Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────┐
│               Application Startup                                │
│         User launches Crush in new project                       │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Initialization Check - init.go:39-74                        │
│          ProjectNeedsInitialization()                            │
│                                                                  │
│  Step 1: Check Init Flag                                        │
│    • Look for .crush/initialized flag file:48-50                │
│    • If exists → Skip initialization                            │
│                                                                  │
│  Step 2: Check Context Files                                    │
│    • Search for existing documentation:56-62                    │
│      - README.md, Makefile, package.json, etc.                  │
│    • If context files exist → Skip (already configured)         │
│                                                                  │
│  Step 3: Check Directory                                        │
│    • Check if directory has visible files:65-71                 │
│    • If empty → Skip (nothing to analyze)                       │
│                                                                  │
│  Return: true if initialization needed                          │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      UI Integration - splash.go:107-109                          │
│          Chat Page Setup                                         │
│                                                                  │
│  • TUI detects initialization flag                              │
│  • Displays initialization dialog in splash screen              │
│  • SetProjectInit(needsInit bool)                               │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      User Prompt - splash.go:230-273                             │
│          Yes/No Dialog                                           │
│                                                                  │
│  Dialog: "Would you like Crush to analyze your codebase         │
│           and create initialization files?"                      │
│                                                                  │
│  [Yes] → Proceed to initialization                              │
│  [No]  → Skip, mark as initialized                              │
└──────────────────────────────────────────────────────────────────┘
                             ↓
                      ┌──────────┴──────────┐
                      │                     │
                   [NO]                  [YES]
                      │                     │
                      ↓                     ↓
            ┌─────────────────┐   ┌─────────────────────┐
            │ Mark Initialized│   │ Execute Init Process│
            │  (Skip Analysis)│   │                     │
            └─────────────────┘   └─────────────────────┘
                                           ↓
┌──────────────────────────────────────────────────────────────────┐
│      Initialization Execution - splash.go:324-346                │
│              initializeProject()                                 │
│                                                                  │
│  Step 1: Mark as Initialized                                    │
│    • Create flag file: .crush/initialized:327                   │
│    • Prevents future prompts                                    │
│                                                                  │
│  Step 2: Get Initialize Prompt                                  │
│    • Load via InitializePrompt(cfg):334                         │
│      File: internal/agent/prompts.go:36-42                      │
│      Template: templates/initialize.md.tpl                      │
│                                                                  │
│  Step 3: Build Initialization Message                           │
│    • Render template with config:340-342                        │
│    • Target file from config: cfg.Options.InitializeAs          │
│      (default: ".cursorrules")                                  │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Agent Execution - Regular Chat Pipeline                    │
│          (Main Coder Agent)                                      │
│                                                                  │
│  • Initialize prompt sent as normal user message:340-342        │
│  • Main coder agent processes request                           │
│  • Agent performs analysis:                                     │
│    1. Check if directory empty (stop if yes)                    │
│    2. Run ls to see directory structure                         │
│    3. Look for existing rule files:                             │
│       - .cursor/rules/*.md                                      │
│       - .cursorrules                                            │
│       - .github/copilot-instructions.md                         │
│       - claude.md, agents.md                                    │
│    4. Identify project type from configs                        │
│    5. Find build/test/lint commands                             │
│    6. Read representative source files                          │
│    7. Understand code patterns                                  │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│      Initialize File Creation                                    │
│                                                                  │
│  Agent creates file (default: .cursorrules) with:              │
│    • Essential commands (build, test, run, deploy)             │
│    • Code organization and structure                            │
│    • Naming conventions and style patterns                      │
│    • Testing approach and patterns                              │
│    • Important gotchas or non-obvious patterns                  │
│    • Project-specific context from existing rules               │
│                                                                  │
│  Critical: Only documents observed patterns                     │
│    • Never invents commands                                     │
│    • Never assumes conventions                                  │
│    • If can't find something, doesn't include it                │
└──────────────────────────────────────────────────────────────────┘
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                   Completion                                     │
│                                                                  │
│  • Initialization file created                                  │
│  • Project marked as initialized                                │
│  • Future agents can read initialization file                   │
│  • File included in context via {{.ContextFiles}}               │
└──────────────────────────────────────────────────────────────────┘
```

### Initialize Prompt Structure

From `templates/initialize.md.tpl`:

```
Goal: Document what agents need to know to work in this codebase

Discovery Process:
  1. Check directory contents (ls)
  2. Look for existing rule files (only if they exist)
  3. Identify project type from config files
  4. Find build/test/lint commands
  5. Read representative source files
  6. Understand code patterns
  7. If target file exists, read and improve it

Content to Include:
  • Essential commands (relevant for this project)
  • Code organization and structure
  • Naming conventions and style patterns
  • Testing approach and patterns
  • Important gotchas or non-obvious patterns
  • Project-specific context from existing rules

Format: Clear markdown sections, aim for completeness

Critical: Only document what you actually observe
```

### ASCII Diagram

```
Application Start
     │
     ├──> Check: ProjectNeedsInitialization()
     │    config/init.go:39
     │    │
     │    ├──> Flag file exists? → Skip
     │    ├──> Context files exist? → Skip
     │    └──> Directory empty? → Skip
     │
     └──> YES → Show Dialog
               │
               ├──[User: No]──> Mark initialized, Skip
               │
               └──[User: Yes]─> Execute Init
                    │
                    ├──> Mark as initialized
                    │    Create .crush/initialized
                    │
                    ├──> Get InitializePrompt()
                    │    Template: initialize.md.tpl
                    │    Target: {{.Config.Options.InitializeAs}}
                    │
                    ├──> Send as chat message
                    │    To main coder agent
                    │
                    └──> Agent analyzes project
                         │
                         ├──> ls (directory structure)
                         ├──> Check for existing rules
                         ├──> Identify project type
                         ├──> Find commands in configs
                         ├──> Read source files
                         ├──> Understand patterns
                         │
                         └──> Create .cursorrules
                              (or configured target)
                              │
                              └──> Future agents load this file
                                   via {{.ContextFiles}}
```

### Example Output

`.cursorrules` file created:

```markdown
# Project: MyApp (Node.js + TypeScript)

## Essential Commands
- Build: `npm run build`
- Test: `npm test`
- Lint: `npm run lint`
- Dev Server: `npm run dev`
- Deploy: `npm run deploy:prod`

## Code Organization
- `src/` - TypeScript source files
- `src/components/` - React components
- `src/api/` - API client code
- `src/utils/` - Utility functions
- `tests/` - Test files (*.test.ts)

## Naming Conventions
- Components: PascalCase (e.g., UserProfile.tsx)
- Utilities: camelCase (e.g., formatDate.ts)
- Constants: UPPER_SNAKE_CASE
- Test files: *.test.ts alongside source

## Testing
- Framework: Jest + React Testing Library
- Run single test: `npm test -- ComponentName`
- Coverage: `npm run test:coverage`

## Gotchas
- Always run `npm run lint` before committing
- API base URL configured in .env file
- Database migrations in src/db/migrations/
```

### Code References

- Initialization Check: `internal/config/init.go:39-74`
- UI Integration: `internal/tui/components/chat/splash/splash.go:107-109`
- User Dialog: `splash.go:230-273`
- Execution: `splash.go:324-346`
- Prompt: `internal/agent/prompts.go:36-42`
- Template: `internal/agent/templates/initialize.md.tpl`

---

## Overall System Architecture

### High-Level Agent Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER                                     │
│                    (TUI / CLI)                                  │
└─────────────────────────────────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                    COORDINATOR                                  │
│               (coordinator.go)                                  │
│                                                                 │
│  • Manages agent lifecycle                                      │
│  • Builds system prompts                                        │
│  • Configures tools                                             │
│  • Handles permissions                                          │
└─────────────────────────────────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                  MAIN CODER AGENT                               │
│               (SessionAgent - agent.go)                         │
│          Prompt: templates/coder.md.tpl                         │
│                                                                 │
│  Full Tool Access:                                              │
│    • File Operations: View, Edit, Write, MultiEdit             │
│    • Search: Glob, Grep                                         │
│    • Execution: Bash, Job Output/Kill                           │
│    • Network: Fetch, Download                                   │
│    • LSP: References, Diagnostics, Sourcegraph                  │
│    • Meta: Agent (sub-agents), AgenticFetch                     │
└─────────────────────────────────────────────────────────────────┘
       │                    │                    │
       │                    │                    │
       ↓                    ↓                    ↓
┌─────────────┐   ┌─────────────────┐   ┌──────────────────┐
│ SUB-AGENT   │   │ FETCH AGENT     │   │ SUMMARY AGENT    │
│ (Task)      │   │ (Agentic Fetch) │   │                  │
│             │   │                 │   │                  │
│ Limited     │   │ Web-Focused     │   │ Context          │
│ Tools:      │   │ Tools:          │   │ Compression      │
│ • Glob      │   │ • WebFetch      │   │                  │
│ • Grep      │   │ • Glob          │   │ Analyzes full    │
│ • LS        │   │ • Grep          │   │ conversation     │
│ • View      │   │ • View          │   │ history          │
│             │   │                 │   │                  │
│ Read-Only   │   │ Analysis-Only   │   │ Creates summary  │
└─────────────┘   └─────────────────┘   └──────────────────┘
```

### Data Flow Across Pipelines

```
┌──────────────────────────────────────────────────────────────────┐
│                        PERSISTENT STORAGE                        │
├──────────────────────────────────────────────────────────────────┤
│  • Sessions Table: session metadata, cost, tokens, summary_id   │
│  • Messages Table: all messages (user, assistant, system)       │
│  • Permissions Table: tool permission requests and approvals    │
└──────────────────────────────────────────────────────────────────┘
                             ↑
                             │ (read/write)
                             │
┌──────────────────────────────────────────────────────────────────┐
│                      SESSION SERVICES                            │
├──────────────────────────────────────────────────────────────────┤
│  • Session Service: CRUD for sessions                           │
│  • Message Service: CRUD for messages                           │
│  • Permission Service: Request/approve permissions              │
│  • History Service: Query conversation history                  │
└──────────────────────────────────────────────────────────────────┘
                             ↑
                             │ (used by)
                             │
┌──────────────────────────────────────────────────────────────────┐
│                    AGENT EXECUTION LAYER                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Main Agent (coder)                                             │
│    ├──> Tool: agent ──> Sub-Agent (task)                        │
│    │      • Creates isolated session                            │
│    │      • Aggregates costs to parent                          │
│    │      • Returns result                                      │
│    │                                                             │
│    ├──> Tool: agentic_fetch ──> Fetch Agent                     │
│    │      • Creates isolated session                            │
│    │      • Fetches URL content                                 │
│    │      • Analyzes with AI                                    │
│    │      • Aggregates costs                                    │
│    │                                                             │
│    ├──> Threshold reached ──> Summarize Agent                   │
│    │      • Analyzes full conversation                          │
│    │      • Creates summary message                             │
│    │      • Truncates message history                           │
│    │      • Resets token counters                               │
│    │                                                             │
│    └──> First message ──> Title Agent (async)                   │
│         • Generates concise title                               │
│         • Uses small model for efficiency                       │
│         • Updates session                                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                             ↑
                             │ (orchestrated by)
                             │
┌──────────────────────────────────────────────────────────────────┐
│                        COORDINATOR                               │
├──────────────────────────────────────────────────────────────────┤
│  • Prompt building (coderPrompt, taskPrompt, etc.)              │
│  • Agent configuration (models, tools, options)                  │
│  • Tool filtering (based on config and permissions)              │
│  • Cost tracking and aggregation                                │
└──────────────────────────────────────────────────────────────────┘
```

### Context Flow

```
Application Config
     ↓
Coordinator (builds context)
     ├──> WorkingDir (from os.Getwd)
     ├──> IsGitRepo (detect .git)
     ├──> Platform (runtime.GOOS)
     ├──> Date (time.Now)
     ├──> GitStatus (git commands)
     └──> ContextFiles (read from config paths)
          │
          └──> Template Variables: {{.WorkingDir}}, {{.Date}}, etc.
               │
               ↓
          System Prompt (rendered from template)
               │
               ↓
          Agent Created with System Prompt
               │
               ↓
          Tools Receive Context via Context Keys:
               • SessionIDContextKey
               • MessageIDContextKey
```

### Cost Tracking Flow

```
Main Agent Execution
     │
     ├──> Token Usage Tracked
     │    (input_tokens, output_tokens, cache tokens)
     │
     ├──> Cost Calculated
     │    (based on provider pricing)
     │
     ├──> Session Updated
     │    (cumulative cost and tokens)
     │
     └──> Sub-Agent Called
          │
          ├──> Sub-Agent Tracks Own Usage
          │
          ├──> Sub-Session Saved
          │
          └──> Parent Session Updated
               (add sub-agent costs)
               │
               └──> Recursive Cost Aggregation
                    (all nested agents contribute)
```

---

## Summary Table: All Pipelines

| Pipeline | Entry Point | Agent Type | Prompt Template | Tools | Purpose |
|----------|------------|------------|-----------------|-------|---------|
| **Main Coder** | User prompt | Coder Agent | `coder.md.tpl` | All tools | Interactive coding assistance |
| **Sub-Agent** | Tool: `agent` | Task Agent | `task.md.tpl` | Glob, Grep, LS, View | Focused search/analysis tasks |
| **Agentic Fetch** | Tool: `agentic_fetch` | Fetch Agent | `agentic_fetch_prompt.md.tpl` | WebFetch, Glob, Grep, View | Web content analysis |
| **Summarization** | Context threshold | Summary Agent | `summary.md` | None | Conversation compression |
| **Title Generation** | First message | Title Agent | `title.md` | None | Session title creation |
| **Initialization** | App startup | Coder Agent | `initialize.md.tpl` | All tools | Project analysis and setup |

---

## Key Design Patterns

### 1. Agent Isolation
- Each agent type has isolated session
- Sub-agents cannot affect parent state directly
- Cost aggregation flows upward
- Sessions linked via parent-child relationships

### 2. Prompt Assembly
- Templates use Go `text/template` syntax
- Context data injected via `PromptDat` struct
- Provider-specific prefixes supported
- Runtime rendering with full config access

### 3. Tool Access Control
- Tools filtered by agent config (`AllowedTools`)
- Permissions requested via UI
- MCP server permissions checked
- Context keys pass session/message IDs to tools

### 4. Cost Management
- Token usage tracked at each streaming step
- Costs calculated per provider pricing
- Sub-agent costs aggregated to parent
- OpenRouter credits separately tracked

### 5. Session Management
- Auto-summarization preserves context
- Title generation async (non-blocking)
- Message history truncation after summary
- Token counters reset post-summarization

---

## Configuration Reference

### Agent Configurations

From `internal/config/config.go`:

```go
type Agents struct {
    AgentCoder AgentConfig  // Main interactive agent
    AgentTask  AgentConfig  // Sub-agent for tasks
}

type AgentConfig struct {
    Model        string          // Model identifier
    AllowedTools []string        // Tool whitelist
    AutoSummarize *bool          // Enable auto-summarization
    // ... other options
}
```

### Context Files

Context files loaded into `{{.ContextFiles}}`:
- `.cursorrules` (default initialization file)
- `.crush/memory` (custom memory files)
- Any paths specified in config

### Template Variables

Available in all prompt templates:
- `{{.WorkingDir}}` - Current working directory
- `{{.IsGitRepo}}` - Boolean: git repository detected
- `{{.Platform}}` - OS: linux, darwin, windows
- `{{.Date}}` - Current date (formatted)
- `{{.GitStatus}}` - Git branch, status, recent commits
- `{{.Config}}` - Full application configuration
- `{{.ContextFiles}}` - Array of context file content
- `{{.Provider}}` - AI provider name
- `{{.Model}}` - Model identifier
- `{{.Config.Options.InitializeAs}}` - Target initialization file

---

## Conclusion

Crush implements a sophisticated multi-agent architecture with:
- **Clear separation of concerns** between agent types
- **Comprehensive context passing** via templates and context keys
- **Robust cost tracking** with recursive aggregation
- **Intelligent session management** with auto-summarization
- **Flexible tool access control** based on agent capabilities
- **Efficient resource usage** (small models for titles/fetch, background processing)

Each pipeline is designed for a specific purpose while sharing common infrastructure for consistency and maintainability.
