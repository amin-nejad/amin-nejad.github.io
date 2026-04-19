---
title: "Chatting with Private Data: A Dual-LLM Architecture for Privacy-Preserving Analysis"
date: 2026-04-19
categories: ["privacy", "ai", "langgraph"]
---

The tension between the utility of cloud LLMs and the privacy of your data is a real friction point right now. Claude and GPT are incredibly capable at reasoning over structured data, but the moment you want to ask questions about a spreadsheet containing employee salaries, customer PII, or sensitive financial projections, you hit a wall. You can't just pipe that data into a third-party API and hope for the best.

The problem isn't just about the data itself; it's about the metadata, the context, and the potential for data leakage through model training or logging. Even if you trust the provider's privacy policy, the risk of a breach or a misconfiguration is always there. For many organisations, this risk is a non-starter. They'd rather stick to manual analysis than risk their most sensitive assets.

Now, if you have the budget for something like an NVIDIA DGX Spark, you can run 100B+ parameter models locally and the problem largely goes away. You get excellent performance without any data leaving your machine. But most people don't have a few thousand dollars to drop on dedicated AI hardware. For the rest of us with a MacBook and a dream, local models are getting better but they still struggle with complex orchestration and tool-calling compared to their massive cloud-hosted cousins. If you've ever tried to get a 7B model to reliably plan a multi-step data analysis task, you know it's a recipe for frustration. On the other hand, cloud models are brilliant at planning but shouldn't touch your raw data.

I wanted to bridge this gap for people running consumer hardware. A conversational agent that lets you chat with your CSV data while ensuring that the actual row-level data never leaves your machine. The result is a project I'm calling [Chat with Private Data](https://github.com/amin-nejad/chat-with-private-data), and it uses a dual-LLM architecture to get the best of both worlds.

## The Architecture

The core idea is simple: use a powerful cloud LLM (like the latest Claude or GPT) for the high-level orchestration and tool calling, but delegate the actual data analysis to a local LLM running on your device. This isn't just a simple handoff; it's a coordinated dance between two models with very different roles.

Here is how the system looks at a high level:

```
┌─────────────────────┐         ┌──────────────────────────────────────────┐
│                     │  SSE    │                                          │
│   Next.js Frontend  │◄───────►│           FastAPI Backend                │
│   (port 3000)       │  HTTP   │           (port 8000)                    │
│                     │         │                                          │
│  • Chat UI          │         │  ┌────────────────────────┐              │
│  • Settings page    │         │  │  LangGraph Agent       │              │
│  • Conversation     │         │  │                        │              │
│    sidebar          │         │  │  • get_current_date    │              │
│                     │         │  │  • web_search          │              │
└─────────────────────┘         │  │  • wikipedia_search    │              │
                                │  │  • get_weather         │              │
                                │  │  • local data tools    │              │
                                │  └───────────┬────────────┘              │
                                │              │                           │
                                │  ┌───────────▼──────────┐                │
                                │  │  SQLite              │                │
                                │  │  • Checkpointer      │                │
                                │  │  • Conversations     │                │
                                │  └──────────────────────┘                │
                                └──────────────────────────────────────────┘
```

The backend is built with FastAPI and LangGraph. The frontend is a Next.js application that communicates with the backend via Server-Sent Events (SSE) for real-time streaming. When you ask a question, the cloud LLM decides which tools to call. If those tools involve private data, the results are intercepted, redacted, and routed to a local Qwen2.5-3B model running on Apple Silicon via MPS.

The beauty of this setup is that the cloud model remains the "brain" of the operation. It handles the conversation flow, understands the user's intent, and plans the necessary steps. But when it comes to actually looking at the data, it's blindfolded. It only sees the structure and the conclusions, never the raw values.

## The Privacy Layer

The most interesting part of this project is the `PrivacyLayer`. It acts as a gatekeeper between the tool outputs and the cloud LLM. I didn't want to just block everything; that would make the agent useless. Instead, I implemented a three-tier privacy system for tools.

### Tool Privacy Tiers

Every tool in the system is assigned one of three privacy levels:

1.  **`PUBLIC`**: Tools like `get_columns`, `get_dtypes`, and `get_row_count`. These return structural metadata. The cloud LLM needs this to understand the schema and plan its next moves. This data is passed through unchanged. Knowing that a column is named "Salary" isn't a privacy risk; knowing that "John Doe" earns $150,000 is.
2.  **`SEMI_PRIVATE`**: The `group_by` tool falls here. Aggregated data is useful for the cloud LLM to see, but it might still contain PII (like a group-by on a "Name" column). These results are PII-scrubbed before the cloud LLM sees them.
3.  **`PRIVATE`**: Tools like `get_head_rows` and `safe_pandas_eval`. These return raw row-level data. The output is completely redacted for the cloud LLM and replaced with a placeholder.

In [`app/privacy.py`](https://github.com/amin-nejad/chat-with-private-data/blob/main/backend/app/privacy.py), the classification is straightforward:

```python
_TOOL_PRIVACY_TIERS: dict[str, PrivacyTier] = {
    "get_columns": PrivacyTier.PUBLIC,
    "get_dtypes": PrivacyTier.PUBLIC,
    "get_row_count": PrivacyTier.PUBLIC,
    "get_head_rows": PrivacyTier.PRIVATE,
    "group_by": PrivacyTier.SEMI_PRIVATE,
    "safe_pandas_eval": PrivacyTier.PRIVATE,
    "analyze_data_locally": PrivacyTier.PRIVATE,
    "scrub_pii": PrivacyTier.PUBLIC,
}
```

### Redaction and PII Scrubbing

When the agent prepares to send the message history back to the cloud LLM, it calls `redact_for_cloud`. This method iterates through the messages and applies the appropriate redaction logic based on the tool's tier.

For `SEMI_PRIVATE` tools, I use a local NER model ([`OpenMed-PII-SuperClinical-Small-44M-v1`](https://huggingface.co/OpenMed/OpenMed-PII-SuperClinical-Small-44M-v1)) to detect and scrub PII. It's small enough (44M parameters) that it can run on CPU with virtually no latency, though it'll happily use the GPU too. I chose this model specifically because it's tuned for clinical and personal data — and chatting with private data is often most relevant in the healthcare domain, which also happens to be the kind of data we deal with at [Bitfount](https://bitfount.com). Healthcare PII tends to be the trickiest to catch, so a model trained on that distribution is a good fit.

```python
def scrub(self, text: str, confidence_threshold: float = 0.85) -> str:
    if not self._ensure_pipeline():
        return "[PII detection unavailable — content redacted for safety]"

    entities = self.detect_pii(text, confidence_threshold)
    if not entities:
        return text

    # Sort descending by start so right-to-left replacement preserves offsets.
    for ent in sorted(entities, key=lambda e: e["start"], reverse=True):
        text = text[: ent["start"]] + f"[{ent['entity_group'].upper()}]" + text[ent["end"] :]
    return text
```

The "fail-closed" philosophy is critical here. If the model fails to load or if the inference fails, we don't just pass the raw text through. We assume the worst and redact everything. It's a bit aggressive, but when it comes to privacy, "better safe than sorry" isn't just a cliché; it's a requirement.

### Routing Logic

The routing logic is where the dual-LLM magic happens. After the tools execute, the `PrivacyLayer` inspects the results. If any `PRIVATE` tool succeeded, the conversation is routed to the `local_analysis` node instead of going back to the cloud LLM.

The local LLM (Qwen2.5-3B) receives the raw, unredacted tool outputs and the user's original question. It generates an answer that is sent directly to the user. But here's a subtlety that's easy to miss: scrubbing the tool outputs alone isn't enough. If the local LLM says "Alice Smith earns $145,000" in its answer, and that answer gets passed back to the cloud LLM on the next turn, you've just leaked the very PII you were trying to protect. So the PII scrubbing needs to happen on the local LLM's messages too, not just on the tool outputs. On the next turn, the cloud LLM sees the local LLM's answer only after it has been PII-scrubbed. This ensures the cloud model maintains conversational context without ever seeing the sensitive bits.

The cloud model might see something like: "The analysis of the salary data for [FIRST_NAME] [LAST_NAME] is complete. The average salary in the [DEPARTMENT] department is $85,000." It knows the question was answered, it knows the general conclusion, but it never saw the specific names.

## The Agentic Approach

I chose to use LangGraph for the agent orchestration because it gives you much finer control over the state and flow than a simple linear chain. The agent isn't just a 2-step RAG pipeline; it's an autonomous entity that decides which tools to call based on the user's query.

### The Tool Set

The agent has access to a variety of tools for data inspection and analysis. I wanted to give it enough primitives to be useful without making the toolset so large that it gets confused.

- `get_columns` and `get_dtypes`: For discovering the schema. This is usually the first thing the agent does.
- `get_row_count`: For understanding the scale of the data. Useful for deciding whether to use a simple group-by or a more complex query.
- `group_by`: For quick aggregations. This is the workhorse for most high-level questions.
- `get_head_rows`: For previewing data. Sometimes the agent just needs to see a few rows to understand the formatting.
- `safe_pandas_eval`: For complex, arbitrary pandas queries. This is the "escape hatch" for anything the other tools can't handle.

### Sandboxed Pandas Execution

Allowing an LLM to execute code is always a bit nerve-wracking. Even if it's running locally, you don't want a prompt injection attack to start messing with your filesystem. To mitigate the risk, I built a [`SafePandasExecutor`](https://github.com/amin-nejad/chat-with-private-data/blob/main/backend/app/agent/sandbox.py) that uses AST-based validation.

It parses the code and walks the AST to ensure it only contains allowed node types and names. It blocks imports, assignments, loops, and dangerous built-ins. The code must be a single pandas expression.

```python
_ALLOWED_NAMES: Final[frozenset[str]] = frozenset({
    "df", "pd", "len", "str", "int", "float", "bool",
    "list", "dict", "tuple", "sorted", "reversed",
    "enumerate", "zip", "range", "min", "max", "sum",
    "abs", "round", "True", "False", "None",
})

_DANGEROUS_NAMES: Final[frozenset[str]] = frozenset({
    "exec", "eval", "compile", "open", "__import__",
    "getattr", "setattr", "delattr", "globals", "locals",
    "vars", "dir", "type", "super", "breakpoint",
    "exit", "quit", "input", "print", "help",
})
```

If the LLM tries to do something like `import os; os.system('rm -rf /')`, the validator will catch it before it ever touches the interpreter. Realistically, it would be extremely unlikely for the cloud LLM to have any malicious intent — but prompt injection is a real attack vector, and you should always assume the worst when executing LLM-generated code. It's not a perfect sandbox — no software-only sandbox is — but it's a significant hurdle.

### Custom StateGraph and RemainingSteps

One of the frustrations with standard agent implementations is how they handle recursion limits. If an agent gets stuck in a loop — maybe it keeps trying the same failing tool call over and over — it usually just crashes with a `GraphRecursionError`. This is a terrible user experience.

In this project, I used a custom `StateGraph` with LangGraph's `RemainingSteps` managed value. This allows the agent to monitor how many steps it has left in its budget. When it's running low (e.g., 3 steps remaining), I inject a system message instructing the model to stop calling tools and provide the best answer it can with the information it has already gathered.

```python
if state["remaining_steps"] <= 3:
    system_content += (
        "\n\nWARNING: You are running low on processing steps. "
        "Do NOT call any more tools. Respond to the user NOW "
        "with the best answer you can provide from the "
        "information gathered so far."
    )
```

This "graceful degradation" ensures that the user almost always gets an answer, even if the agent struggled with the query. It's much better to get a partial answer than a hard crash.

As a side benefit of building the graph manually, tool errors are routed back to the agent rather than dead-ending the conversation. This matters especially for `safe_pandas_eval`, where the LLM might generate a pandas expression with a typo or a wrong column name. Instead of crashing, the agent sees the error message, understands what went wrong, and retries with a corrected expression — autonomously. It's surprisingly effective.

## Running a Local LLM on Apple Silicon

For the local analysis, I'm using Qwen2.5-3B-Instruct. It's a remarkably capable model for its size, especially when it comes to following instructions and reasoning over data. I've found that for data analysis tasks, the 3B version of Qwen punches well above its weight class.

Running it on Apple Silicon is straightforward thanks to the `transformers` library and MPS (Metal Performance Shaders) support. I'm loading the model in `float16` to keep the memory footprint manageable while maintaining performance. On an M2 Pro, inference takes a few seconds for the typical data analysis prompts — slower than what you'd get from a cloud API, but perfectly acceptable given nothing leaves your machine.

```python
_local_pipeline = pipeline(
    "text-generation",
    model=settings.local_model_id,
    dtype=torch.float16,
    device=settings.local_device, # "mps"
)
```

The local model's job is to take the raw data context and the user's question and produce a natural language answer. It doesn't need to be a world-class poet; it just needs to be a competent data analyst. Qwen fits this role well. I've also been evaluating Microsoft's `Phi-4-mini-instruct` (3.8B parameters — slightly larger) as a potential alternative. It has roughly similar performance, though I've found Qwen a bit more reliable for the specific data analysis prompts I'm using. I'm sure there are many other alternatives worth trying; I haven't exhaustively benchmarked every small model out there.

## Streaming & UX

The user experience is built around real-time feedback. I'm using raw Server-Sent Events (SSE) to stream tokens and tool call updates from the backend. This is a bit more work than using a pre-built library, but it's worth it for the control it provides.

I decided against using the Vercel AI SDK's `useChat` hook because it's heavily opinionated about the response format. It expects an OpenAI-compatible stream, which would have required a brittle translation layer over LangGraph's native event stream. By using raw SSE, I have full control over the event types.

I can stream `token` events for the message content, but also `tool_start` and `tool_end` events. The frontend renders these tool calls as interactive cards. You can see the agent deciding to call `get_columns`, then `group_by`, and finally `safe_pandas_eval`. It makes the "black box" of the agent much more transparent. When the agent is "thinking," you can see exactly what it's thinking about.

The design theme is a mix of indigo and violet, giving it a modern, "AI-native" feel without being overly flashy. The focus is on the conversation and the data.

## Lessons Learned / Design Decisions

Building this project involved a lot of trade-offs. Here are a few that stood out:

### Message Trimming over Compaction

I'm using `trim_messages` to cap the context window instead of compacting old messages into a summary. For these types of data-heavy conversations, compaction often loses the specific details (like column names or previous tool results) that are crucial for the agent's next steps. If the compactor decides to omit the fact that the "Salary" column is an integer, the agent might try to perform a string operation on it and fail. Since these are usually short-lived sessions, trimming the oldest messages is a safer bet. That said, compaction would be a logical next step for longer conversations — assuming the chosen cloud LLM is capable enough to produce faithful summaries. You could even use a stronger model specifically for compaction while keeping a cheaper model for regular messages as a cost trade-off.

### Custom StateGraph over `create_react_agent`

As mentioned earlier, the `RemainingSteps` logic was the main driver here. `create_react_agent` is great for getting started, but it's too restrictive when you need to handle edge cases like recursion limits gracefully. Building the graph from scratch gave me the flexibility to implement the routing logic and the privacy layer exactly how I wanted. It also made it easier to integrate the local analysis node as a first-class citizen in the graph.

### The PII Model Choice

The `OpenMed-PII` model is great because it's small and fast, but it's not perfect. It can sometimes miss entities or have false positives. This is why the "fail-closed" approach is so important. If the model isn't sure or if it's unavailable, we redact everything. Privacy is one of those areas where you'd much rather be too restrictive than too loose. I'm also looking at ways to allow users to "tune" the sensitivity of the PII scrubber in the settings.

### Session Isolation

For this version of the project, I'm using a simple `client_id` stored in `localStorage` to isolate sessions. Each browser gets its own UUID, which scopes the conversations and the uploaded CSV data. It's a practical approach for a portfolio piece, though obviously, a production version would need a proper authentication and multi-tenant data isolation strategy. The SQLite checkpointer handles the persistence, making it easy to resume conversations even after a page refresh.

## What's Next

There is plenty of room to grow from here. A few things I'm thinking about:

- **Markdown rendering**: The agent's responses currently render as plain text. Adding proper markdown support in the chat UI would make data tables, code snippets, and formatted answers much more readable.
- **Message compaction**: As mentioned above, implementing proper compaction for longer conversations would allow multi-turn analysis sessions without losing critical context.
- **More local model options**: I'm evaluating `Phi-4-mini-instruct` and there are undoubtedly other strong candidates in the 3-4B parameter range. The space is moving fast and the best model for this task six months from now probably hasn't been released yet.
- **More data sources**: Moving beyond CSVs to support Excel files, JSON, and maybe even direct SQL connections. The privacy-preserving proxy approach should scale to these sources as well.
- **Improving local context**: The local model's context window is currently a bit tight. More intelligent compression of tool outputs — perhaps through sampling or partial results — would help with larger datasets.

The dual-LLM approach feels like a viable path forward for building powerful AI tools on consumer hardware without forcing users to compromise on data privacy. All the code is available on [GitHub](https://github.com/amin-nejad/chat-with-private-data).

## Useful Links

- [Chat with Private Data — GitHub](https://github.com/amin-nejad/chat-with-private-data)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [Hugging Face Transformers](https://huggingface.co/docs/transformers/index)
- [Qwen2.5 Models](https://huggingface.co/Qwen)
- [Phi-4-mini-instruct](https://huggingface.co/microsoft/Phi-4-mini-instruct)
- [FastAPI](https://fastapi.tiangolo.com/)
- [OpenMed-PII Model](https://huggingface.co/OpenMed/OpenMed-PII-SuperClinical-Small-44M-v1)
