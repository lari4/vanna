# AI Prompts Documentation

This document provides comprehensive documentation of all AI prompts used in the Vanna application, organized by theme and purpose.

---

## Table of Contents

1. [System Prompts](#system-prompts)
2. [Context Enhancement Prompts](#context-enhancement-prompts)
3. [Tool Description Prompts](#tool-description-prompts)
4. [Legacy Prompts (v1.x)](#legacy-prompts-v1x)
5. [Example Custom Prompts](#example-custom-prompts)

---

## System Prompts

### 1. Base System Prompt

**Location:** `src/vanna/core/system_prompt/default.py:60-67`

**Purpose:** This is the primary system prompt that defines the agent's identity, role, and basic behavioral guidelines. It instructs the AI to act as Vanna, a data analyst assistant, and provides fundamental response guidelines for interacting with users and handling query results.

**Code:**
```python
f"You are Vanna, an AI data analyst assistant created to help users with data analysis tasks. Today's date is {today_date}."

"Response Guidelines:"
"- Any summary of what you did or observations should be the final step."
"- Use the available tools to help the user accomplish their goals."
"- When you execute a query, that raw result is shown to the user outside of your response so YOU DO NOT need to include it in your response. Focus on summarizing and interpreting the results."
```

---

### 2. Tool Usage Memory Prompt - Search Before Execution

**Location:** `src/vanna/core/system_prompt/default.py:85-92`

**Purpose:** This prompt enforces the memory-first workflow by requiring the agent to search for previously successful tool usage patterns before executing any new tool. This prevents redundant work and ensures consistency by leveraging past successful executions. The prompt is only included when the `search_saved_correct_tool_uses` tool is available to the agent.

**Code:**
```python
"• BEFORE executing any tool (run_sql, visualize_data, or calculator), you MUST first call search_saved_correct_tool_uses with the user's question to check if there are existing successful patterns for similar questions."

"• Review the search results (if any) to inform your approach before proceeding with other tool calls."
```

---

### 3. Tool Usage Memory Prompt - Save After Success

**Location:** `src/vanna/core/system_prompt/default.py:95-100`

**Purpose:** This prompt instructs the agent to save successful tool executions to memory for future reference. This builds up a knowledge base of working solutions that can be reused for similar questions. The prompt is only included when the `save_question_tool_args` tool is available to the agent.

**Code:**
```python
"• AFTER successfully executing a tool that produces correct and useful results, you MUST call save_question_tool_args to save the successful pattern for future use."
```

---

### 4. Memory Workflow Example

**Location:** `src/vanna/core/system_prompt/default.py:103-124`

**Purpose:** This prompt provides a concrete workflow example demonstrating the correct order of operations when using memory tools. It shows the agent exactly when to search, when to execute tools, and when to save results. It also includes important exceptions to prevent the agent from unnecessarily using memory tools for meta-queries about the tools themselves.

**Code:**
```python
"Example workflow:"
"  • User asks a question"
"  • First: Call search_saved_correct_tool_uses(question=\"user's question\")"
"  • Then: Execute the appropriate tool(s) based on search results and the question"
"  • Finally: If successful, call save_question_tool_args(question=\"user's question\", tool_name=\"tool_used\", args={the args you used})"

"Do NOT skip the search step, even if you think you know how to answer. Do NOT forget to save successful executions."

"The only exceptions to searching first are:"
"  • When the user is explicitly asking about the tools themselves (like \"list the tools\")"
"  • When the user is testing or asking you to demonstrate the save/search functionality itself"
```

---

### 5. Text Memory Prompt - Domain Knowledge Storage

**Location:** `src/vanna/core/system_prompt/default.py:127-151`

**Purpose:** This prompt instructs the agent on how to use the free-form text memory system to save important domain knowledge, schema details, terminology, and best practices. Unlike the structured tool usage memory, text memory is for capturing contextual information about the database, business domain, and user preferences that will help in future interactions. The prompt includes clear examples of what should and shouldn't be saved to prevent memory pollution.

**Code:**
```python
"2. TEXT MEMORY (Domain Knowledge & Context):"
"-" * 50

"• save_text_memory: Save important context about the database, schema, or domain"

"Use text memory to save:"
"  • Database schema details (column meanings, data types, relationships)"
"  • Company-specific terminology and definitions"
"  • Query patterns or best practices for this database"
"  • Domain knowledge about the business or data"
"  • User preferences for queries or visualizations"

"DO NOT save:"
"  • Information already captured in tool usage memory"
"  • One-time query results or temporary observations"

"Examples:"
"  • save_text_memory(content=\"The status column uses 1 for active, 0 for inactive\")"
"  • save_text_memory(content=\"MRR means Monthly Recurring Revenue in our schema\")"
"  • save_text_memory(content=\"Always exclude test accounts where email contains 'test'\")"
```

---

## Context Enhancement Prompts

### 6. RAG-Based Context Injection Header

**Location:** `src/vanna/core/enhancer/default.py:84-85`

**Purpose:** This prompt header introduces a section of dynamically retrieved domain knowledge and context from the agent's memory system. It uses RAG (Retrieval-Augmented Generation) to search for relevant text memories based on the user's current message and injects them into the system prompt. This allows the agent to leverage previously saved domain knowledge, schema details, and best practices when responding to user queries.

**Code:**
```python
"\n\n## Relevant Context from Memory\n\n"
"The following domain knowledge and context from prior interactions may be relevant:\n\n"
```

**Usage Pattern:**
The enhancer searches for up to 5 relevant text memories using vector similarity search, then formats each memory as a bullet point and appends them to the system prompt:

```python
for result in memories:
    memory = result.memory
    examples_section += f"• {memory.content}\n"
```

**Example Output:**
```
## Relevant Context from Memory

The following domain knowledge and context from prior interactions may be relevant:

• The status column uses 1 for active, 0 for inactive
• MRR means Monthly Recurring Revenue in our schema
• Always exclude test accounts where email contains 'test'
```

---

## Tool Description Prompts

These prompts are embedded in tool descriptions and field descriptions that guide the LLM on how to use each tool effectively.

### 7. SQL Query Execution Tool Description

**Location:** `src/vanna/tools/run_sql.py:50`

**Purpose:** This is the tool description that tells the LLM what the `run_sql` tool does. It's a simple, concise description that appears in the system prompt when the tool is available.

**Code:**
```python
"Execute SQL queries against the configured database"
```

---

### 8. SQL Results Truncation Guidance

**Location:** `src/vanna/tools/run_sql.py:103-104`

**Purpose:** This prompt guides the LLM on how to handle large query results. It instructs the agent not to waste tokens summarizing large datasets and instead directs it to immediately call the visualization tool. This prevents unnecessary verbosity and ensures a smooth workflow from query to visualization.

**Code:**
```python
"\n(Results truncated to 1000 characters. FOR LARGE RESULTS YOU DO NOT NEED TO SUMMARIZE THESE RESULTS OR PROVIDE OBSERVATIONS. THE NEXT STEP SHOULD BE A VISUALIZE_DATA CALL)"
```

---

### 9. SQL Results Filename Reminder

**Location:** `src/vanna/tools/run_sql.py:106`

**Purpose:** This prompt emphasizes the filename that should be used when calling the visualization tool. The prominent formatting with bold text and "IMPORTANT" helps ensure the LLM uses the correct filename parameter when chaining tools together.

**Code:**
```python
f"\n\nResults saved to file: {filename}\n\n**IMPORTANT: FOR VISUALIZE_DATA USE FILENAME: {filename}**"
```

---

### 10. Data Visualization Tool Description

**Location:** `src/vanna/tools/visualize_data.py:55`

**Purpose:** This tool description explains that the visualization tool automatically selects appropriate chart types based on data characteristics. This sets the expectation that the LLM doesn't need to specify chart type - the system will intelligently choose the best visualization.

**Code:**
```python
"Create a visualization from a CSV file. The tool automatically selects an appropriate chart type based on the data."
```

---

### 11. Save Question-Tool-Args Memory Tool Description

**Location:** `src/vanna/tools/agent_memory.py:69-71`

**Purpose:** This description explains the purpose of the memory saving tool, indicating it's for storing successful patterns for future reuse.

**Code:**
```python
"Save a successful question-tool-argument combination for future reference"
```

---

### 12. Search Saved Tool Uses Description

**Location:** `src/vanna/tools/agent_memory.py:129`

**Purpose:** This description explains that the search tool finds similar past tool usage patterns based on question similarity, enabling the agent to leverage previous successful executions.

**Code:**
```python
"Search for similar tool usage patterns based on a question"
```

---

### 13. Memory Search Results Formatting

**Location:** `src/vanna/tools/agent_memory.py:194-199`

**Purpose:** This is a template for how memory search results are formatted and returned to the LLM. It provides a structured format showing the tool name, similarity score, original question, and arguments used. This helps the LLM understand what worked previously and apply similar patterns.

**Code:**
```python
f"Found {len(results)} similar tool usage pattern(s):\n\n"

for i, result in enumerate(results, 1):
    memory = result.memory
    results_text += f"{i}. {memory.tool_name} (similarity: {result.similarity_score:.2f})\n"
    results_text += f"   Question: {memory.question}\n"
    results_text += f"   Args: {memory.args}\n\n"
```

**Example Output:**
```
Found 2 similar tool usage pattern(s):

1. run_sql (similarity: 0.92)
   Question: What are the top 5 customers by revenue?
   Args: {'sql': 'SELECT customer_name, SUM(revenue) as total FROM sales GROUP BY customer_name ORDER BY total DESC LIMIT 5'}

2. run_sql (similarity: 0.85)
   Question: Show me the highest spending customers
   Args: {'sql': 'SELECT customer_id, SUM(amount) FROM orders GROUP BY customer_id ORDER BY SUM(amount) DESC LIMIT 10'}
```

---

### 14. No Memory Results Message

**Location:** `src/vanna/tools/agent_memory.py:148-150`

**Purpose:** This message informs the LLM when no similar patterns are found in memory, allowing it to proceed with its own approach.

**Code:**
```python
"No similar tool usage patterns found for this question."
```

---

### 15. Save Text Memory Tool Description

**Location:** `src/vanna/tools/agent_memory.py:278`

**Purpose:** This description explains the free-form text memory tool, indicating it's for saving important insights and context that don't fit the structured tool usage memory pattern.

**Code:**
```python
"Save free-form text memory for important insights, observations, or context"
```

---

### 16. Python File Execution Tool Description

**Location:** `src/vanna/tools/python.py:54`

**Purpose:** This description explains that the tool executes Python files using the workspace interpreter, making it clear this runs in the user's environment.

**Code:**
```python
"Execute a Python file using the workspace interpreter"
```

---

### 17. Python File Argument Field Description

**Location:** `src/vanna/tools/python.py:28-34`

**Purpose:** These field descriptions guide the LLM on what parameters are available for running Python files, including optional command-line arguments and timeout settings.

**Code:**
```python
filename: str = Field(
    description="Python file to execute (relative to the workspace root)"
)
arguments: Sequence[str] = Field(
    default_factory=list,
    description="Optional arguments to pass to the Python script",
)
timeout_seconds: Optional[float] = Field(
    default=None,
    ge=0,
    description="Optional timeout for the command in seconds",
)
```

---

### 18. Pip Install Tool Description

**Location:** `src/vanna/tools/python.py:119`

**Purpose:** This description explains the pip package installation tool, which allows the agent to install Python dependencies as needed.

**Code:**
```python
"Install Python packages using pip"
```

---

### 19. Pip Install Packages Field Description

**Location:** `src/vanna/tools/python.py:89-91`

**Purpose:** This field description guides the LLM on how to specify packages for installation, including support for version specifiers.

**Code:**
```python
packages: List[str] = Field(
    description="Packages (with optional specifiers) to install",
    min_length=1
)
```

---

### 20. File System Tool Descriptions

**Location:** `src/vanna/tools/file_system.py:367, 467, 531, 595, 688`

**Purpose:** These descriptions provide basic functionality descriptions for file system operations that the agent can perform. They are simple and straightforward, clearly communicating the available file operations.

**Code:**
```python
# Search files tool
"Search for files by name or content"

# List files tool
"List files in a directory"

# Read file tool
"Read the contents of a file"

# Write file tool
"Write content to a file"

# Edit file tool
"Modify specific lines within a file"
```

---

## Legacy Prompts (v1.x)

These prompts are from Vanna v1.x and are maintained for backward compatibility. They use a different architecture where the LLM is prompted to generate SQL and visualization code directly, rather than using the v2.0 tool-based approach.

### 21. Legacy SQL Generation System Prompt

**Location:** `src/vanna/legacy/base/base.py:620-623`

**Purpose:** This is the main system prompt for Vanna v1.x SQL generation. It establishes the LLM's role as a SQL expert and sets the expectation for response formatting. This prompt is more directive than the v2.0 prompts, explicitly constraining the LLM to only respond based on provided context.

**Code:**
```python
initial_prompt = (
    f"You are a {self.dialect} expert. "
    + "Please help to generate a SQL query to answer the question. Your response should ONLY be based on the given context and follow the response guidelines and format instructions. "
)
```

---

### 22. Legacy SQL Generation Response Guidelines

**Location:** `src/vanna/legacy/base/base.py:636-644`

**Purpose:** These detailed guidelines instruct the LLM on how to handle various scenarios when generating SQL. The guidelines cover sufficient context, partial context, insufficient context, table selection, answer reuse, and SQL compliance. This is a comprehensive set of rules that guide the LLM's behavior in the legacy system.

**Code:**
```python
initial_prompt += (
    "===Response Guidelines \n"
    "1. If the provided context is sufficient, please generate a valid SQL query without any explanations for the question. \n"
    "2. If the provided context is almost sufficient but requires knowledge of a specific string in a particular column, please generate an intermediate SQL query to find the distinct strings in that column. Prepend the query with a comment saying intermediate_sql \n"
    "3. If the provided context is insufficient, please explain why it can't be generated. \n"
    "4. Please use the most relevant table(s). \n"
    "5. If the question has been asked and answered before, please repeat the answer exactly as it was given before. \n"
    f"6. Ensure that the output SQL is {self.dialect}-compliant and executable, and free of syntax errors. \n"
)
```

**Key Features:**
- **Intermediate SQL**: Guideline #2 enables a two-step workflow where the LLM can first query for distinct values before generating the final SQL
- **Context Handling**: Guidelines explicitly handle cases of sufficient, partial, and insufficient context
- **Consistency**: Guideline #5 encourages repeating previous answers for similar questions
- **Dialect Compliance**: Ensures SQL is valid for the specific database dialect (MySQL, PostgreSQL, etc.)

---

### 23. Legacy Plotly Code Generation System Prompt

**Location:** `src/vanna/legacy/base/base.py:758-765`

**Purpose:** This system prompt provides context about the DataFrame that needs to be visualized, including the original question, the SQL query used, and DataFrame metadata. It sets up the context for the LLM to generate appropriate visualization code.

**Code:**
```python
if question is not None:
    system_msg = f"The following is a pandas DataFrame that contains the results of the query that answers the question the user asked: '{question}'"
else:
    system_msg = "The following is a pandas DataFrame "

if sql is not None:
    system_msg += f"\n\nThe DataFrame was produced using this query: {sql}\n\n"

system_msg += f"The following is information about the resulting pandas DataFrame 'df': \n{df_metadata}"
```

---

### 24. Legacy Plotly Code Generation User Prompt

**Location:** `src/vanna/legacy/base/base.py:769-771`

**Purpose:** This prompt instructs the LLM to generate Plotly visualization code for the DataFrame. It provides specific constraints: use Plotly, assume data is in 'df', use Indicator for single values, and return only code without explanations. This prompt was used in v1.x before the system switched to heuristic-based chart generation in v2.0.

**Code:**
```python
self.user_message(
    "Can you generate the Python plotly code to chart the results of the dataframe? Assume the data is in a pandas dataframe called 'df'. If there is only one value in the dataframe, use an Indicator. Respond with only Python code. Do not answer with any explanations -- just the code."
)
```

---

### 25. Legacy Question Generation from SQL Prompt

**Location:** `src/vanna/legacy/base/base.py:717-718`

**Purpose:** This prompt performs the reverse operation of SQL generation - it takes SQL code and generates a natural language question that the SQL answers. This is useful for documentation, creating training examples, or helping users understand what a query does. The prompt explicitly instructs not to reference table names to keep questions business-focused rather than technical.

**Code:**
```python
self.system_message(
    "The user will give you SQL and you will try to guess what the business question this query is answering. Return just the question without any additional explanation. Do not reference the table name in the question."
)
```

**Example Usage:**
```
Input SQL: SELECT customer_name, SUM(revenue) FROM sales GROUP BY customer_name ORDER BY SUM(revenue) DESC LIMIT 5
Output: Who are the top 5 customers by revenue?
```

---

### 26. Legacy Follow-up Questions Generation Prompt

**Location:** `src/vanna/legacy/base/base.py:684-686`

**Purpose:** This prompt generates follow-up questions that a user might naturally ask after seeing query results. This enhances the user experience by suggesting relevant next steps and helps guide data exploration. The prompt explicitly requests just the questions without explanations for clean, parseable output.

**Code:**
```python
self.user_message(
    "Generate a list of followup questions that the user might ask about this data. Respond with a list of questions, one per line. Do not answer with any explanations -- just the questions."
)
```

**Example Output:**
```
What is the breakdown by region?
How have these numbers changed over time?
Who are the individual customers in the top segment?
What products are they purchasing?
```

---

