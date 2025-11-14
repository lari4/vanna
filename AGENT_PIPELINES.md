# Agent Pipelines Documentation

This document describes all the workflows and pipelines in the Vanna agent system, showing how different components interact, which prompts are used at each stage, and how data flows through the system.

---

## Table of Contents

1. [Standard Query-to-Visualization Pipeline](#standard-query-to-visualization-pipeline)
2. [Memory-Enhanced Query Pipeline](#memory-enhanced-query-pipeline)
3. [Workflow Handler Short-Circuit Pipeline](#workflow-handler-short-circuit-pipeline)
4. [Error Recovery Pipeline](#error-recovery-pipeline)
5. [Legacy v1.x SQL Generation Pipeline](#legacy-v1x-sql-generation-pipeline)

---

## Standard Query-to-Visualization Pipeline

This is the most common pipeline where a user asks a data question, the agent generates and executes SQL, and then creates a visualization.

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER SENDS MESSAGE                          │
│                     "Show top 5 customers"                          │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    1. USER RESOLUTION                               │
│                                                                     │
│  Component: UserResolver                                           │
│  Purpose: Identify user and load permissions                       │
│  Output: User object with permissions                              │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    2. LIFECYCLE HOOKS                               │
│                                                                     │
│  Phase: before_message                                             │
│  Purpose: Run pre-processing hooks (quota checks, etc.)            │
│  Output: Potentially modified message                              │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 3. CONVERSATION LOADING                             │
│                                                                     │
│  Component: ConversationStore                                      │
│  Purpose: Load existing conversation or create new one             │
│  Output: Conversation object with message history                  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 4. TOOL SCHEMA RETRIEVAL                            │
│                                                                     │
│  Component: ToolRegistry                                           │
│  Purpose: Get tools available to this user                         │
│  Output: List of ToolSchema objects                                │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 5. SYSTEM PROMPT BUILDING                           │
│                                                                     │
│  Component: SystemPromptBuilder (DefaultSystemPromptBuilder)       │
│  Prompts Used:                                                     │
│    - Base system prompt (Prompt #1)                                │
│    - Memory workflow instructions (Prompts #2-5) if tools avail.   │
│  Input: User, List[ToolSchema]                                     │
│  Output: System prompt string                                      │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 6. CONTEXT ENHANCEMENT (RAG)                        │
│                                                                     │
│  Component: LlmContextEnhancer (DefaultLlmContextEnhancer)         │
│  Process:                                                          │
│    1. Search text memories for user's message (vector search)      │
│    2. Retrieve top 5 most relevant memories                        │
│    3. Format as context injection (Prompt #6)                      │
│  Input: system_prompt, user_message                                │
│  Output: Enhanced system prompt with relevant memories             │
│                                                                     │
│  Example Enhanced Prompt:                                          │
│  ┌───────────────────────────────────────────────────────┐       │
│  │ [Original system prompt]                              │       │
│  │                                                        │       │
│  │ ## Relevant Context from Memory                       │       │
│  │                                                        │       │
│  │ • The status column uses 1 for active, 0 for inactive │       │
│  │ • MRR means Monthly Recurring Revenue                 │       │
│  └───────────────────────────────────────────────────────┘       │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 7. LLM REQUEST BUILDING                             │
│                                                                     │
│  Component: Agent._build_llm_request()                             │
│  Process:                                                          │
│    1. Filter conversation history (ConversationFilter)             │
│    2. Convert to LLM messages format                               │
│    3. Add system prompt                                            │
│    4. Add tool schemas                                             │
│  Output: LlmRequest object                                         │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 8. LLM MIDDLEWARE                                   │
│                                                                     │
│  Component: LlmMiddleware chain                                    │
│  Purpose: Intercept/transform request (caching, logging, etc.)     │
│  Output: Potentially modified LlmRequest                           │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 9. LLM SERVICE CALL                                 │
│                                                                     │
│  Component: LlmService (AnthropicLlmService, OpenAILlmService)     │
│  Process: Send request to LLM API                                  │
│  Output: LlmResponse with content and/or tool_calls                │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
                  ┌──────────┴──────────┐
                  │ Has Tool Calls?     │
                  └──────────┬──────────┘
                             │
                    ┌────────┴────────┐
                    │ YES             │ NO → Skip to final response
                    ▼                 │
┌─────────────────────────────────────────────────────────────────────┐
│                 10. TOOL EXECUTION LOOP                             │
│                                                                     │
│  For each tool call in response:                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ a) Lifecycle Hooks: before_tool                             │  │
│  │ b) Get Tool from ToolRegistry                               │  │
│  │ c) Parse arguments to tool's args schema                    │  │
│  │ d) Execute tool with ToolContext                            │  │
│  │ e) Lifecycle Hooks: after_tool                              │  │
│  │ f) Add tool result to conversation                          │  │
│  │ g) Yield UI components to user                              │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Example Tool Execution: run_sql                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Tool: run_sql                                               │  │
│  │ Input: {"sql": "SELECT * FROM customers LIMIT 5"}          │  │
│  │ Process:                                                    │  │
│  │   1. Execute SQL query via SqlRunner                        │  │
│  │   2. Convert results to DataFrame                           │  │
│  │   3. Save to CSV file (query_results_abc123.csv)            │  │
│  │   4. Return truncated preview with filename (Prompt #9)     │  │
│  │ Output:                                                     │  │
│  │   result_for_llm: "...preview... (truncated) Use file:     │  │
│  │                    query_results_abc123.csv" (Prompt #9)    │  │
│  │   ui_component: DataFrameComponent                          │  │
│  └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 11. BUILD NEXT LLM REQUEST                          │
│                                                                     │
│  Component: Agent._build_llm_request()                             │
│  Process:                                                          │
│    1. Add assistant message with tool_calls to conversation        │
│    2. Add tool result messages to conversation                     │
│    3. Rebuild LLM request with updated conversation                │
│  Output: Updated LlmRequest                                        │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼ (Loop back to step 9 until no tool calls)
┌─────────────────────────────────────────────────────────────────────┐
│                 12. FINAL LLM RESPONSE                              │
│                                                                     │
│  LLM receives tool results and generates final response            │
│  Typical flow for visualization:                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ LLM sees:                                                   │  │
│  │   - run_sql result: "Use file: query_results_abc123.csv"   │  │
│  │   - System prompt: Use visualize_data for charts           │  │
│  │                                                             │  │
│  │ LLM responds with tool_call:                                │  │
│  │   {                                                         │  │
│  │     "name": "visualize_data",                               │  │
│  │     "arguments": {                                          │  │
│  │       "filename": "query_results_abc123.csv",               │  │
│  │       "title": "Top 5 Customers"                            │  │
│  │     }                                                        │  │
│  │   }                                                         │  │
│  └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼ (Loop back to step 10)
┌─────────────────────────────────────────────────────────────────────┐
│                 13. VISUALIZE_DATA EXECUTION                        │
│                                                                     │
│  Tool: visualize_data                                              │
│  Input: {"filename": "query_results_abc123.csv"}                   │
│  Process:                                                          │
│    1. Read CSV file from FileSystem                                │
│    2. Parse into DataFrame                                         │
│    3. Generate chart using PlotlyChartGenerator (heuristic)        │
│    4. Return chart data                                            │
│  Output:                                                           │
│    result_for_llm: "Created visualization..."                      │
│    ui_component: ChartComponent with Plotly chart                  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 14. FINAL TEXT RESPONSE                             │
│                                                                     │
│  LLM receives visualization result and generates summary           │
│  No more tool calls - text response only                           │
│  Example: "I've analyzed your top 5 customers. The chart shows..." │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 15. CONVERSATION STORAGE                            │
│                                                                     │
│  Component: ConversationStore                                      │
│  Process: Save all messages (user, assistant, tool) to storage     │
│  Output: Updated conversation in persistent storage                │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 16. UI FINALIZATION                                 │
│                                                                     │
│  Yield final UI components:                                        │
│    - StatusBarUpdateComponent (status: "idle")                     │
│    - ChatInputUpdateComponent (disabled: false)                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Data Transformations

1. **User Message → LLM Request:**
   - User text → Message object → LlmMessage with role "user"
   - Added to conversation history

2. **LLM Response → Tool Execution:**
   - LlmResponse.tool_calls → ToolCall objects
   - ToolCall.arguments (JSON) → Parsed to tool's args schema (Pydantic)

3. **Tool Result → LLM Context:**
   - ToolResult.result_for_llm → Message with role "tool"
   - Added to conversation for next LLM call

4. **SQL Results → CSV File:**
   - DataFrame → CSV string
   - Written to file system
   - Filename communicated to LLM via prompt (Prompt #9)

5. **CSV File → Visualization:**
   - CSV content → DataFrame
   - DataFrame → Plotly chart dict (heuristic-based)
   - Chart dict → ChartComponent → User UI

### Prompts Used in Pipeline

| Stage | Prompt | Purpose |
|-------|--------|---------|
| System Prompt | Prompt #1 | Base agent identity and guidelines |
| System Prompt | Prompts #2-5 | Memory workflow instructions (if applicable) |
| Context Enhancement | Prompt #6 | RAG context injection header |
| Tool Result | Prompt #8 | Truncation guidance for large SQL results |
| Tool Result | Prompt #9 | Filename reminder for visualization |
| Tool Description | Prompt #7 | SQL execution tool description |
| Tool Description | Prompt #10 | Visualization tool description |

### Typical Message Count

For "Show top 5 customers":
- Initial: 1 user message
- After SQL execution: 3 messages (user, assistant with tool_call, tool result)
- After visualization: 5 messages (+ assistant with tool_call, tool result)
- Final response: 6 messages (+ assistant text response)

---

## Memory-Enhanced Query Pipeline

This pipeline shows how the agent uses memory tools to search for and reuse previous successful query patterns, then saves new successful patterns for future use.

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER SENDS MESSAGE                          │
│                "What are the top revenue customers?"                │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│            [Steps 1-6: Same as Standard Pipeline]                   │
│                                                                     │
│  System prompt NOW includes memory workflow (Prompts #2-5):        │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ • BEFORE executing any tool (run_sql, visualize_data),     │  │
│  │   you MUST first call search_saved_correct_tool_uses       │  │
│  │                                                             │  │
│  │ • AFTER successfully executing a tool, you MUST call       │  │
│  │   save_question_tool_args to save the pattern              │  │
│  └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 7. FIRST LLM RESPONSE                               │
│                                                                     │
│  LLM sees system prompt with memory instructions                   │
│  LLM follows workflow: search memory BEFORE running SQL            │
│                                                                     │
│  LLM response includes tool_call:                                  │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ {                                                           │  │
│  │   "name": "search_saved_correct_tool_uses",                 │  │
│  │   "arguments": {                                            │  │
│  │     "question": "What are the top revenue customers?",      │  │
│  │     "limit": 10                                             │  │
│  │   }                                                         │  │
│  │ }                                                           │  │
│  └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 8. SEARCH MEMORY TOOL EXECUTION                     │
│                                                                     │
│  Tool: search_saved_correct_tool_uses                              │
│  Input: {"question": "What are the top revenue customers?"}        │
│  Process:                                                          │
│    1. Generate embedding for question (via AgentMemory)            │
│    2. Vector similarity search in memory database                  │
│    3. Retrieve similar past tool usage patterns                    │
│    4. Format results (Prompt #13)                                  │
│                                                                     │
│  Output (example with matches):                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Found 2 similar tool usage pattern(s):                     │  │
│  │                                                             │  │
│  │ 1. run_sql (similarity: 0.92)                               │  │
│  │    Question: What are the top 5 customers by revenue?      │  │
│  │    Args: {                                                  │  │
│  │      "sql": "SELECT customer_name, SUM(revenue)            │  │
│  │               FROM sales GROUP BY customer_name             │  │
│  │               ORDER BY SUM(revenue) DESC LIMIT 5"           │  │
│  │    }                                                        │  │
│  │                                                             │  │
│  │ 2. run_sql (similarity: 0.85)                               │  │
│  │    Question: Show me highest spending customers            │  │
│  │    Args: {                                                  │  │
│  │      "sql": "SELECT customer_id, SUM(amount)               │  │
│  │               FROM orders GROUP BY customer_id              │  │
│  │               ORDER BY SUM(amount) DESC LIMIT 10"           │  │
│  │    }                                                        │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  UI Component: StatusBarUpdate or CardComponent (based on user)    │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 9. LLM ANALYZES MEMORY RESULTS                      │
│                                                                     │
│  LLM receives formatted memory results (Prompt #13)                │
│  LLM adapts one of the similar patterns to current question        │
│                                                                     │
│  LLM response includes tool_call:                                  │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ {                                                           │  │
│  │   "name": "run_sql",                                        │  │
│  │   "arguments": {                                            │  │
│  │     "sql": "SELECT customer_name, SUM(revenue) as total    │  │
│  │             FROM sales GROUP BY customer_name               │  │
│  │             ORDER BY total DESC LIMIT 10"                   │  │
│  │   }                                                         │  │
│  │ }                                                           │  │
│  │                                                             │  │
│  │ Note: LLM adapted the pattern from memory (changed LIMIT)  │  │
│  └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 10. SQL EXECUTION                                   │
│                                                                     │
│  [Same as Standard Pipeline step 10]                               │
│  SQL executes, results saved to CSV                                │
│  Result returned to LLM with filename                              │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 11. LLM DECIDES TO SAVE PATTERN                     │
│                                                                     │
│  LLM sees:                                                         │
│    - SQL execution was successful                                  │
│    - System prompt says: "AFTER successfully executing a tool,     │
│      you MUST call save_question_tool_args"                        │
│                                                                     │
│  LLM response includes tool_call:                                  │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ {                                                           │  │
│  │   "name": "save_question_tool_args",                        │  │
│  │   "arguments": {                                            │  │
│  │     "question": "What are the top revenue customers?",      │  │
│  │     "tool_name": "run_sql",                                 │  │
│  │     "args": {                                               │  │
│  │       "sql": "SELECT customer_name, SUM(revenue)...DESC"    │  │
│  │     }                                                        │  │
│  │   }                                                         │  │
│  │ }                                                           │  │
│  └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 12. SAVE MEMORY TOOL EXECUTION                      │
│                                                                     │
│  Tool: save_question_tool_args                                     │
│  Input: {question, tool_name, args} from above                     │
│  Process:                                                          │
│    1. Generate embedding for question                              │
│    2. Create ToolUsageMemory object                                │
│    3. Store in vector database with metadata                       │
│    4. Associate with user context                                  │
│                                                                     │
│  Output:                                                           │
│    result_for_llm: "Successfully saved usage pattern for           │
│                     'run_sql' tool"                                │
│    ui_component: StatusBarUpdateComponent (success)                │
│                                                                     │
│  Memory now contains: Question → Tool → Args mapping               │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 13. CONTINUE WITH VISUALIZATION                     │
│                                                                     │
│  [Same as Standard Pipeline]                                       │
│  - visualize_data called                                           │
│  - Chart generated                                                 │
│  - Final response from LLM                                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Differences from Standard Pipeline

1. **Additional Tool Calls**: Memory-enhanced pipeline adds 2 extra tool calls:
   - `search_saved_correct_tool_uses` at the beginning
   - `save_question_tool_args` after successful execution

2. **Conversation Length**: Typical message count is 8-10 messages instead of 6:
   - +2 for memory search (assistant tool_call + tool result)
   - +2 for memory save (assistant tool_call + tool result)

3. **Prompt Influence**: System prompt (Prompts #2-5) explicitly guides the agent to use memory workflow

4. **Performance Impact**:
   - **First Time**: Slightly slower (search returns no results, extra tool call overhead)
   - **Subsequent Times**: Much faster (LLM can adapt existing patterns instead of generating from scratch)

### Memory Search Result Variations

**Case 1: No Similar Patterns Found**
```
Tool result (Prompt #14):
"No similar tool usage patterns found for this question."

LLM behavior: Generates SQL from scratch using schema knowledge
```

**Case 2: Perfect Match Found**
```
Tool result (Prompt #13):
"Found 1 similar tool usage pattern(s):
1. run_sql (similarity: 0.98)
   Question: What are the top revenue customers?
   Args: {sql: '...exact same query...'}

LLM behavior: Reuses exact same SQL, minimal modification
```

**Case 3: Similar Patterns Found**
```
Tool result (Prompt #13):
"Found 2 similar tool usage pattern(s):
1. run_sql (similarity: 0.85)...
2. run_sql (similarity: 0.82)...

LLM behavior: Adapts pattern to current question, modifies SQL accordingly
```

### Data Flow for Memory

```
Question Text
     │
     ▼
[Embedding Model] ─────► Vector (e.g., 1536 dimensions)
     │
     ▼
[Vector DB Search] ─────► Similar vectors with cosine similarity
     │
     ▼
[Memory Retrieval] ─────► ToolUsageMemory objects
     │
     ▼
[Format Results] ───────► Prompt #13 formatted text
     │
     ▼
[LLM Processing] ───────► Adapted SQL query
```

### Prompts Used

| Stage | Prompt | Purpose |
|-------|--------|---------|
| System Prompt | Prompts #2-3 | Search before execution instruction |
| System Prompt | Prompt #4 | Example workflow |
| System Prompt | Prompt #5 | Save after success instruction |
| Tool Description | Prompt #11 | save_question_tool_args description |
| Tool Description | Prompt #12 | search_saved_correct_tool_uses description |
| Tool Result | Prompt #13 | Memory search results formatting |
| Tool Result | Prompt #14 | No results message |

---

