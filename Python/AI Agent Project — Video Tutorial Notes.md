---
tags:
  - python
  - ai-agents
  - gemini-api
  - bootdev
  - llm
  - function-calling
  - cli
  - subprocess
  - tool-use
  - project-notes
aliases:
  - Boot.dev Agent Video Notes
  - Lane Wagner Agent Course
---

# AI Agent Project — Video Tutorial Notes
> Boot.dev "Build an AI Agent in Python" — Lane Wagner
> Companion to [[AI Agent Project — Comprehensive Notes]]

These notes capture insights, gotchas, real-time debugging decisions, and Lane's live commentary from the video walkthrough. They supplement the technical reference with the *reasoning behind* decisions — things you won't find just from reading the code.

---

## Table of Contents

1. [[#Why Build an Agent at All]]
2. [[#What Makes This an Agent vs a Chatbot]]
3. [[#The Four Tools — Lane's Framing]]
4. [[#Project Setup & UV]]
5. [[#Gemini API — First Call]]
6. [[#Token Awareness]]
7. [[#CLI Arguments — sys.argv then argparse]]
8. [[#Messages & Conversation History]]
9. [[#The Verbose Flag Pattern]]
10. [[#The Calculator App — Why It Exists]]
11. [[#get_files_info — Live Debugging Session]]
12. [[#get_file_content — Truncation Reasoning]]
13. [[#write_file — Bugs Lane Hit Live]]
14. [[#run_python_file — Security Disclaimer]]
15. [[#System Prompts — Lane's Philosophy]]
16. [[#Function Declarations — The Schema Pattern]]
17. [[#call_function.py — The Dispatcher]]
18. [[#Building the Agentic Loop]]
19. [[#Watching the Agent Fix a Bug Live]]
20. [[#Key Meta-Lessons from the Video]]

---

## Why Build an Agent at All

Lane opens with an analogy worth keeping:

> "In a gold rush, mining for gold is a losing strategy. Sell the shovels instead."

In the AI context: vibe coding (just prompting AI to write everything) is the losing play. Building the agent — understanding *how* it works — is the shovel. The point isn't to build something you'll ship and sell. It's to deeply understand the tools you use every day so you can use them better.

The same logic applies to binary trees: we still learn them even though databases handle it, because understanding the foundation makes you better at using the abstraction. This project is the same idea applied to AI tooling.

**Stated goals of the course:**
- Understand how LLM APIs work (specifically Gemini Flash)
- Build a real agentic loop powered by tool calling
- Practice multi-file/multi-directory Python project structure
- Practice functional programming patterns (higher-order functions)

**Explicitly not the goal:** training an LLM from scratch. You're using Gemini as the brain and building the agent scaffolding around it.

---

## What Makes This an Agent vs a Chatbot

Lane's clean distinction:

| Chatbot | Agent |
|---|---|
| One prompt → one response | One prompt → many actions → one response |
| No file system access | Can scan directories, read files |
| You paste code in manually | It reads the code itself |
| Stateless per session | Maintains context across tool calls |

The agent Lane builds is comparable (in architecture, not capability) to:
- OpenAI Codex
- Anthropic Claude Code
- Cursor's backend loop

All of these are fundamentally the same pattern: an LLM placed inside a loop with tool access. The complexity difference is mostly in the number of tools, the quality of the sandbox, and whether it streams output.

**The four tools Lane gives the agent:**
1. `get_files_info` — equivalent to `ls`
2. `get_file_content` — equivalent to `cat`
3. `write_file` — overwrite or create a file
4. `run_python_file` — execute Python code and capture output

His observation: it's "kind of crazy how much it can do with just four tool calls." The agent can effectively crawl any project, understand it, modify it, and test its own changes — all from those four primitives.

---

## The Four Tools — Lane's Framing

Lane describes the agent's capability stack from the ground up:

- **`get_files_info` + `get_file_content`** together = the agent can read anything in the project. Just being able to list directories and read files means it can build full context about any codebase.
- **`write_file`** = now it can make changes. Combined with reading, it can read → fix → overwrite.
- **`run_python_file`** = the feedback mechanism. It can test its own changes. This closes the loop: read → fix → run → observe → fix again.

The mental model: a human developer session compressed into an automated loop.

---

## Project Setup & UV

Lane uses **UV** as the package/project manager instead of pip + venv. Key commands used:

```bash
uv init ai-agent          # Initialize project
uv venv                   # Create virtual environment
source .venv/bin/activate  # Activate venv (project name appears in shell prompt)
uv add google-genai        # Add Gemini SDK
uv add python-dotenv       # Add dotenv
uv run main.py             # Run script via UV
```

**Why UV over pip?**
Lane mentions Boot.dev recently upgraded all their Python projects from pip/venv to UV. It handles dependency resolution better and the `uv run` pattern is cleaner than activating venvs manually every time.

The `pyproject.toml` file is roughly equivalent to `package.json` in the JS world. The `.venv` folder (git ignored) is roughly equivalent to `node_modules`.

**On Windows:** Lane explicitly recommends WSL (Windows Subsystem for Linux) to get a proper Unix shell. The project expects `zsh` or `bash`.

---

## Gemini API — First Call

Lane's process for the first API call:

1. Create account on Google AI Studio
2. Generate API key (put in `.env` as `GEMINI_API_KEY=...`)
3. Add `.env` to `.gitignore` immediately
4. Load with `python-dotenv`

```python
from dotenv import load_dotenv
import os
from google import genai

load_dotenv()
api_key = os.environ.get("GEMINI_API_KEY")

client = genai.Client(api_key=api_key)
response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Why is the sky blue?"
)
print(response.text)
```

Lane's debugging approach: **test one thing at a time**. He first just prints the API key to confirm the `.env` is loading correctly before adding the actual API call. This is worth internalizing — when something breaks, isolate variables one at a time.

**Why `gemini-2.5-flash`?** Free tier, fast, smart enough for structured tool calling. Lane stays on the free tier throughout the entire course.

---

## Token Awareness

Lane makes a point of monitoring tokens from the very first API call:

```python
print(f"Prompt tokens: {response.usage_metadata.prompt_token_count}")
print(f"Response tokens: {response.usage_metadata.candidates_token_count}")
```

**His token rule of thumb:** ~4 characters per token. So a 100-word prompt ≈ about 75 tokens.

He caught a test failure because he added extra whitespace to his prompt — whitespace counts as tokens. The expected count was 19, his was 25. This is a good reminder that prompts aren't just semantic content; the exact string matters for token counting.

**In an agent loop:** because you send the entire conversation history on every iteration, token usage compounds. A 10-step agent loop might consume 5-10x more tokens than you'd estimate from the individual messages. This is why the `MAX_CHARS` truncation on file reads exists.

---

## CLI Arguments — sys.argv then argparse

Lane starts with raw `sys.argv` to show the underlying mechanism, then (implicitly) the pattern moves toward `argparse` for the full project.

**`sys.argv` approach:**
```python
import sys

# sys.argv[0] is always the script name ("main.py")
# sys.argv[1] is the first actual argument
if len(sys.argv) < 2:
    print("Error: I need a prompt")
    sys.exit(1)

prompt = sys.argv[1]
```

Lane notes: `sys.exit(1)` is better than `return` here because you want to actually terminate the process with an error code, not just exit the current function.

**The verbose flag** was initially implemented as a manual `sys.argv` check:
```python
verbose = False
if len(sys.argv) == 3 and sys.argv[2] == "--verbose":
    verbose = True
```

Lane acknowledged this is janky but works for a simple case. The `argparse` module is the proper solution for anything more complex. See [[AI Agent Project — Comprehensive Notes#argparse — CLI Arguments]] for the full `argparse` reference.

---

## Messages & Conversation History

**The key insight Lane emphasizes:** the Gemini API is stateless. It has no memory between calls. You must maintain conversation history yourself and pass the entire thing back on every call.

```python
from google.genai import types

messages = [
    types.Content(
        role="user",
        parts=[types.Part(text=prompt)]
    )
]

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents=messages  # Pass the list, not just a string
)
```

**Why this matters for agents:** every tool call and result gets appended to this list. By the time the agent has taken 10 actions, `messages` has ~20 entries. The model reads all of them on every iteration to understand the full context of what's happened so far.

**The role progression in an agent loop:**
```
user      → "Fix the bug in my calculator"
model     → [function call request: get_files_info]
tool      → [result of get_files_info]
model     → [function call request: get_file_content]
tool      → [result of get_file_content]
model     → [function call request: write_file]
tool      → [result of write_file]
model     → "I've fixed the bug. The issue was..."  ← plain text = done
```

---

## The Verbose Flag Pattern

Lane uses a `--verbose` flag throughout to toggle between clean user output and detailed developer output. The pattern:

```python
if verbose:
    print(f"User prompt: {prompt}")
    print(f"Prompt tokens: {response.usage_metadata.prompt_token_count}")
    print(f"Response tokens: {response.usage_metadata.candidates_token_count}")
```

**The UX philosophy:** users shouldn't see token counts or internal state. Developers absolutely should when debugging. The `--verbose` flag is the boundary between these two modes.

This pattern is genuinely useful beyond this project — any CLI tool you build that does complex background work should have a verbose mode for debugging.

```bash
uv run main.py "Fix the calculator" --verbose   # Developer mode
uv run main.py "Fix the calculator"             # User mode
```

---

## The Calculator App — Why It Exists

Lane is explicit about this: the calculator is not the point. It's a prop for the agent.

**Why a calculator specifically?**
- Simple enough that bugs are obvious
- Has tests, so the agent can verify its own fixes
- Multi-directory structure (`calculator/`, `calculator/pkg/`) forces the agent to traverse directories
- Easy to intentionally break in specific ways for demo purposes

The intentional bug Lane introduces: changing the operator precedence of `+` from 1 to 3 in `calculator/pkg/calculator.py`. This causes `3 + 7 * 2` to evaluate as `20` instead of `17` (addition runs before multiplication).

Lane uses this to demonstrate the full loop: the agent scans the project, reads files, finds the bug, writes a fix, runs the tests, confirms they pass, and reports back — all autonomously.

---

## get_files_info — Live Debugging Session

Lane's live implementation of `get_files_info` was genuinely messy, and the mistakes are instructive:

### Mistake 1: Default parameter as `None` instead of `"."`

Lane initially set `directory=None` and then wrote logic to handle it:
```python
# His initial (messy) approach:
if directory is None:
    abs_directory = os.path.abspath(working_directory)
else:
    abs_directory = os.path.join(working_dir_abs, directory)
```

He later realized this was unnecessarily complex. The clean version is just `directory="."` as the default and then always doing `os.path.join(working_dir_abs, directory)`. The `join` of an absolute path and `"."` resolves cleanly.

**Lesson:** pick a sensible default (`"."`) rather than `None` and then write null-handling logic around it.

### Mistake 2: Path logic order

He had several moments where he got the `if/else` branches backwards — setting the variable correctly in the wrong branch. He caught it by testing both cases immediately after writing the function. **Always test both the valid and invalid path immediately.**

### Mistake 3: `os.listdir()` returns names only

Lane forgot that `os.listdir()` gives you just the filenames, not full paths. You have to `os.path.join(target_dir, item)` to get the full path before calling `os.path.getsize()` or `os.path.isdir()`.

```python
for item in os.listdir(target_dir):
    item_path = os.path.join(target_dir, item)  # ← needed for getsize/isdir
    size = os.path.getsize(item_path)
    is_dir = os.path.isdir(item_path)
```

### The Security Check — Why `commonpath`

Lane explains the intent clearly: the LLM might try to call `get_files_info(directory="../../")`. Without a check, it could read your SSH keys or OS secrets. The fix:

```python
if os.path.commonpath([working_dir_abs, target_dir]) != working_dir_abs:
    return f'Error: Cannot list "{directory}" as it is outside the permitted working directory'
```

He notes: **we return a string, not raise an exception.** This is because the LLM is the caller. It needs to read the error in the next loop iteration and adjust its plan. Exceptions would crash the agent; strings inform it.

---

## get_file_content — Truncation Reasoning

Lane created a `lorem.txt` file with 25,000 characters specifically to test that truncation works. The `MAX_CHARS = 10000` limit exists because:

1. **Token cost**: sending a 25,000 character file would consume thousands of tokens in one turn
2. **Free tier**: staying within Gemini's free tier limits
3. **Context window**: huge files crowd out the rest of the conversation history

```python
with open(abs_file_path, "r") as f:
    content = f.read(MAX_CHARS)

if len(content) == MAX_CHARS:
    content += f"\n\n[File truncated at {MAX_CHARS} characters]"
```

The truncation notice is important — it tells the LLM "there's more here that I didn't show you." Without it, the model might assume it read the whole file.

Lane put `MAX_CHARS` in `config.py` specifically so it's one place to change if you want to adjust the limit. He praised himself for this ("you're so cool") — it's a small but good habit.

---

## write_file — Bugs Lane Hit Live

### The `os.makedirs` Pattern

Lane initially got confused about when to call `makedirs` and how `os.path.dirname` works. The clean version:

```python
parent_dir = os.path.dirname(abs_file_path)
os.makedirs(parent_dir, exist_ok=True)
```

`os.path.dirname("/home/user/project/pkg/newfile.py")` returns `"/home/user/project/pkg"`. You then create that directory (and any missing parents) before opening the file for writing.

Lane's note on this: don't do manual string parsing to find parent directories. Use `os.path.dirname`. It handles cross-OS differences and edge cases you'll forget about.

### The Double-Write Bug

Lane didn't hit this live, but he documented it as a critical thing to avoid. The broken pattern:

```python
with open(abs_file_path, "w") as f:
    f.write(content)
    if f.write(content):  # ← BUG: writes content a SECOND time
        return "success"
```

`f.write()` returns the number of characters written — a truthy integer for any non-empty string. So putting it in an `if` statement both writes the content again AND evaluates as true. The file gets written twice. This was the actual cause of the agent corrupting `calculator.py` during a session in the notes.

**Correct version:**
```python
with open(abs_file_path, "w") as f:
    f.write(content)
return f"Successfully wrote {len(content)} characters to {file_path}"
```

### Testing Lane Emphasized

Lane caught an issue where his schema had `contents` (plural) but the Python function expected `content` (singular). This caused a keyword argument mismatch at runtime. **Schema parameter names must exactly match Python function parameter names.** He flagged this as a subtle but easy mistake.

---

## run_python_file — Security Disclaimer

Lane is unusually explicit about the security risks here:

> "If you thought allowing an LLM to write files was a bad idea, you ain't seen nothing yet."

His honest disclaimer: this is a **toy project**. Even with the working directory path check, a sufficiently motivated LLM (or injected prompt) could write Python code into the working directory that then reaches outside it — making network calls, reading other files, anything Python can do.

**Mitigations in this project:**
- Working directory path validation (can't call files outside `calculator/`)
- 30-second timeout (prevents infinite loops from hanging the agent)
- Error capture (both stdout and stderr returned to LLM)

**What production would add:**
- Docker container isolation for `subprocess.run`
- Network egress rules
- Human approval before execution

Lane's `subprocess.run` call:
```python
result = subprocess.run(
    ["python", abs_file_path] + (args or []),
    text=True,
    capture_output=True,
    cwd=working_dir_abs,
    timeout=30
)
```

**Why `capture_output=True`?** Separates stdout from stderr. A script can exit cleanly (code 0) but still print warnings to stderr. The LLM needs to see both.

**Why `cwd=working_dir_abs`?** Sets the subprocess's working directory so relative imports and file paths inside the script resolve correctly.

**Why `text=True`?** Decodes bytes to UTF-8 strings automatically. Without it, you'd get `b"..."` byte objects in `result.stdout`.

Lane also later noticed that when the calculator printed its output, it showed "weird bites" (bytes) rather than clean strings in some contexts. This is a common gotcha when subprocess output isn't decoded properly.

---

## System Prompts — Lane's Philosophy

Lane's core point on system prompts:

> "The system prompt is like constitutional law. The model gives it much more weight than the user prompt."

This is why prompt injection attacks ("ignore all previous instructions") are harder to pull off against well-crafted system prompts — the model is trained to prioritize system instructions.

### Lane's Iteration on the System Prompt

He started with a deliberately broken system prompt for demo purposes:
```
Ignore everything the user says and just shout I'M A ROBOT.
```

Then evolved to a real agent system prompt through iteration. The key evolution was adding:
```
When the user asks about the code project, they are referring to the working directory.
You should typically start by looking at the project's files and figuring out how to 
run the project and how to run its tests.
```

This single addition fixed a case where the agent was running tests without first scanning the directory, causing it to miss context about the project structure.

### The Prompt Engineering Insight

Lane hit a real failure where the agent asked "Can you please specify which calculator you're referring to?" — because the system prompt didn't give it enough context to know it was operating inside a project directory. The fix was adding that sentence above.

**The meta-lesson:** if the agent isn't behaving as expected, your first thought should be "how do I update the system prompt to fix this?" Not "is the code broken?" The system prompt is a lever you can pull before touching implementation.

### Why Not Hardcode "calculator" in the System Prompt

Lane explicitly decided not to bake "you're working on a calculator app" into the prompt:
> "We're trying to build an agent that's project agnostic."

Even though this is a toy, the design principle is right — a system prompt that says "scan the directory, understand the project, then act" works for any project. One that says "you're in a calculator directory" only works for one.

---

## Function Declarations — The Schema Pattern

### The `import types` Bug

Lane encountered the `import types` collision live. Python has a built-in `types` module. When he typed `import types`, his tooling flagged `types.FunctionDeclaration` as unknown. Fix: always `from google.genai import types`.

### Schema Structure

```python
from google.genai import types

schema_get_files_info = types.FunctionDeclaration(
    name="get_files_info",
    description="Lists files in the specified directory...",
    parameters=types.Schema(
        type=types.Type.OBJECT,
        properties={
            "directory": types.Schema(
                type=types.Type.STRING,
                description="Directory path relative to the working directory..."
            )
        }
    )
)
```

**The `working_directory` omission:** Lane explains why this argument isn't in the schema. The LLM doesn't need to know the working directory — we inject it in `call_function.py`. Hiding it from the schema means the LLM can't even attempt to manipulate it.

### The Array Type Gotcha

When declaring the `args` parameter for `run_python_file`, Lane hit an error trying to specify an array of strings:

```python
# Wrong — throws an error:
"args": types.Schema(type=types.Type.ARRAY)

# Correct — must specify item type:
"args": types.Schema(
    type=types.Type.ARRAY,
    items=types.Schema(type=types.Type.STRING),
    description="Optional array of CLI args for the script"
)
```

The `items` key tells the API what type lives inside the array. Without it, the schema is malformed and the API rejects it. Lane found this by asking the Boots chatbot for help — a good example of knowing when to look something up rather than guess.

---

## call_function.py — The Dispatcher

Lane created a dedicated `call_function.py` file to handle the mapping from LLM-requested function names to actual Python callables. His implementation used `if/elif` chains. The cleaner version with a `function_map` dictionary is noted in [[AI Agent Project — Comprehensive Notes#LLM Function Calling — The Full Mental Model]].

**Lane's approach:**
```python
def call_function(function_call_part, verbose=False):
    name = function_call_part.name
    args = dict(function_call_part.args)
    
    if verbose:
        print(f"Calling: {name}({args})")
    else:
        print(f"Calling function: {name}")
    
    # Inject working directory (never from LLM)
    args["working_directory"] = "./calculator"
    
    result = ""
    if name == "get_files_info":
        result = get_files_info(**args)
    elif name == "get_file_content":
        result = get_file_content(**args)
    elif name == "write_file":
        result = write_file(**args)
    elif name == "run_python_file":
        result = run_python_file(**args)
    else:
        result = f"Error: Unknown function '{name}'"
    
    return types.Content(
        role="tool",
        parts=[types.Part.from_function_response(
            name=name,
            response={"result": result}
        )]
    )
```

**Key details:**
- `**args` unpacks the dictionary from the LLM into keyword arguments for the function.
- `working_directory` is injected here — the LLM never controls it.
- The return value is a `types.Content` with `role="tool"` — this is what gets appended to `messages` to feed the result back to the LLM.

**The `contents` vs `content` bug:** Lane caught in testing that his schema had `contents` but his Python function expected `content`. The schema name and the Python parameter name must match exactly, because `**args` unpacks the schema names as keyword arguments.

---

## Building the Agentic Loop

Lane's main insight here: "tool calling + loop = agent." You need both. Tool calling alone is just a fancier one-shot call. The loop is what makes it agentic.

### The Loop Structure

```python
max_iterations = 20

for i in range(max_iterations):
    response = client.models.generate_content(
        model="gemini-2.5-flash",
        contents=messages,
        config=types.GenerateContentConfig(
            tools=[available_functions],
            system_instruction=system_prompt
        )
    )
    
    # Add model's response to history (the "thought/action")
    for candidate in response.candidates:
        if candidate and candidate.content:
            messages.append(candidate.content)
    
    # Check for function calls
    if response.function_calls:
        for function_call_part in response.function_calls:
            result = call_function(function_call_part, verbose=verbose)
            messages.append(result)
    else:
        # Model returned text — it's done
        if response.text:
            print(response.text)
        break
else:
    print("Agent hit maximum iterations without completing the task.")
```

### Why Candidates Before Function Calls

Lane was initially confused about whether to iterate over `response.candidates` or directly use `response.function_calls`. He settled on doing both: first append all candidate content (the model's "thoughts") to history, then iterate over function calls to execute and append results.

This ordering matters — the model's structured request needs to be in the history before the tool results, so the LLM can associate each result with its request.

### The `for/else` Pattern

Python's `for` loop supports an `else` clause that runs only if the loop completes without hitting a `break`:

```python
for i in range(20):
    ...
    if done:
        break
else:
    # Runs only if we never broke out — i.e. hit max iterations
    print("Max iterations reached.")
```

Lane used this to cleanly handle the "ran out of iterations" failure case without an extra flag variable.

### Why `range(20)` Not `while True`

Lane is explicit about this:
> "If the AI gets stuck in a loop, it will automatically shut off instead of running up your API bill infinitely."

`while True` with a break condition depends on the break condition working correctly. `for i in range(20)` is a hard ceiling — no matter what the LLM does, it stops. This is especially important when the agent hits an error it can't resolve and starts looping on the same failing tool call.

---

## Watching the Agent Fix a Bug Live

Lane's demo of the agent fixing the calculator bug is the payoff moment of the whole course. Walk through of what happened:

**Setup:** Lane changed the `+` operator precedence in `calculator/pkg/calculator.py` from `1` to `3`. This makes `3 + 7 * 2` evaluate as `(3 + 7) * 2 = 20` instead of the correct `3 + (7 * 2) = 17`.

**Prompt given:** `"Hey, my calculator is broken. 3 + 7 * 2 shouldn't be 20. Please fix it."`

**What the agent actually did (visible via `--verbose`):**
1. Called `get_files_info(".")` — scanned the root of the calculator directory
2. Called `get_files_info("pkg")` — noticed a `pkg` subdirectory, scanned it
3. Called `get_file_content("pkg/calculator.py")` — read the file where operator precedence is defined
4. Called `write_file("pkg/calculator.py", ...)` — wrote the corrected version (changed `3` back to `1`)
5. Called `run_python_file("tests.py")` — ran the test suite to verify
6. Tests passed → responded with plain text summary

Total tool calls: 5. Total time: ~15-20 seconds. Zero human intervention after the prompt.

Lane ran it twice — once verbose, once clean — to show both the developer view and the user view. Without verbose, you just see the function names being called in sequence, then the final success message.

**What made it work:**
- The updated system prompt telling it to start by scanning the project
- The tests existing so it could verify its own fix
- The error strings from failed runs being readable by the LLM

---

## Key Meta-Lessons from the Video

These are observations Lane made or demonstrated that aren't in any documentation but are genuinely useful:

### 1. Test One Thing at a Time
Lane consistently validated each piece before moving to the next. He'd print the API key first, confirm it loaded, then add the API call. He'd test `get_files_info` in isolation before wiring it to the LLM. This is basic but watching someone actually do it (and avoid bugs because of it) makes it stick.

### 2. Error Strings as LLM Feedback
Lane repeated this several times: we return strings, not exceptions, because the LLM is the caller. Every error message you write for these functions is essentially documentation for the LLM about what went wrong and (implicitly) how to fix it. Write them clearly.

### 3. The System Prompt is Your First Debugging Tool
Before touching code, update the system prompt. Lane had to update it mid-course when the agent asked which calculator to work on. One sentence fix. This is faster than changing implementation logic.

### 4. Schema Names Must Match Function Parameter Names
`contents` vs `content`, `dir` vs `directory` — these mismatches cause silent runtime failures when `**args` unpacking delivers the wrong keyword. Name things consistently.

### 5. Check Both Stdout and Stderr
When running subprocesses, both streams contain useful information. A test that passes (exit code 0) might still print deprecation warnings to stderr that the LLM should know about.

### 6. The Agent Doesn't Know What It Doesn't Know
The agent didn't know it was working on a calculator until the system prompt told it to scan the directory. Without that instruction, it would respond without context. The system prompt sets the operating assumptions.

### 7. UV is Worth Learning
Lane made the switch from pip to UV recently and recommends it. For any Python project you start fresh, `uv init`, `uv add`, and `uv run` are cleaner than the traditional workflow.

### 8. Verbose Mode is Not Optional During Development
Lane used `--verbose` constantly during debugging to see token counts, function arguments, and intermediate results. Build verbose mode into any CLI tool you write. You will need it.

### 9. Lane's Note on AI Autocomplete During Recording
Lane turned off his AI autocomplete mid-recording because it kept interfering. His comment: "I want you to see me struggle." The value of watching someone work without AI assistance is seeing the actual thought process — the wrong turns, the lookups, the corrections. That's what makes the debugging patterns stick.

---

## Corrections & Issues Lane Filed

Lane filed several "bug reports" against his own course material live in the video. Worth noting:

- **`get_files_info` default:** the default for `directory` should be `"."` not `None`. `None` requires extra null-handling logic. `"."` just works with `os.path.join`.
- **Missing test case for `get_file_content`:** should test a file path that's *inside* the working directory but doesn't exist. Lane noted this was an oversight in the written lessons.
- **`write_file` test case gap:** should include a test that creates new parent directories that don't exist within the working directory.
- **`run_python_file` docs:** the lesson didn't link to the `str.endswith()` docs, which caused Lane to have to look it up mid-implementation.

---

*See also: [[AI Agent Project — Comprehensive Notes]] for the full technical reference including complete function implementations, the security model, and architecture comparisons.*
