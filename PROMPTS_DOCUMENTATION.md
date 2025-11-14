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

