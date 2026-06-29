# Claude Certified Architect Foundations (CCA-F) Cheat Sheet
This is my personal cheat sheet I created while preparing for the CCA-F exam. <br> 
See my blog post [here](https://strunje.com/posts/2026/06/cca-f-preparation/) for more information.

## Core Architecture
### Agentic Loops

The agentic loop is structured as follows:

1. Prompt
2. Agentic Loop (repeated)
    1. Gather Context
    2. Take Action (call tools)
    3. Verify Results
3. Return Result

Every roundtrip inside the loop is a turn. They continue until Claude generates output without tool calls. One can configure the maximum turns or budget allowed in the loop.

A number of different messages are yielded by the system, depending on the stage of the loop:

- `SystemMessage` : Session Lifecycle Events (First message or after compaction)
- `AssistantMessage` : Every response from Claude (including the final one). Can be used to display progress
- `UserMessage` : Emitted after user input or every tool call with result.
- `ResultMessage` : Emitted at the end of the agent loop, contains final text result, token usage and cost. Can be used to display the final result only.
- `StreamEvent` : Emitted only if partial messages are enabled

### Tools

Claude has a number of built-in tools that the agents can use, here is an overview of the most important ones:

- **Agent**: Spawn a subagent with its own context window (allowed agents can be limited)
- **Bash**: Execute shell commands (allowed commands can be limited)
- **Glob**: Search for files based on pattern matching
- **Grep**: Search for file content based on pattern matching
- **Read**: Read the whole content of a file (can be limited)
- **Edit**: Perform exact string replacement on a file
- **Write**: Create or fully overwrite a file

## Prompting

### Prompt Design
One should try to make the prompts as precise as possible and provide helpful content, so that Claude does not have to guess the intent.

2-4 few-shot examples help the model to understand the intent and mitigate problems like inconsistency.
Few-shot examples often help more than a more detailed description.

When making non-straightforward changes, one should use plan mode first to separate exploration from execution.

When working on features one hasn't fully thought through yet or are outside of one's expertise, ask Claude to do an interview first.

### Prompt Caching

Caching makes responses faster and cheaper by reusing what has already been send and only processing what has been appended, instead of reprocessing the full history on every request.

The server caches by matching the start of each request (called prefix), so on a normal turn the prefix is the entire previous request and only the latest messages are uncached. Once something in the chain is changed, everything after it has to recomputed.

At the beginning of each prefix is the system prompt (core instructions, tool definitions) as well as the project context (`CLAUDE.md`, auto memory). As such there are some typical actions that invalidate (parts of) the cache:

- switching the model
- changing the effort level
- toggling a plugin
- compacting the conversation

### Structured Output

One can define a JSON schema for the exact shape of data one wants to get back from the agent. It will then try to use any of the tools it has to match the schema. The SDK will validate the output and re-prompt the agent on mismatch until the maximum number of retries is reached.

When using the Agent SDK, the JSON Schema has to be passed into `query()` using the `output_format` option and the result message will include a `structured_output` field.

One can also use Zod (TypeScript) or Pydantic (Python) to define the schema, which is easier to use and also allows to automatically parse the response into a fully-typed object.

When designing the JSON schema, fields that are not absolutely required should be marked as optional.
Also, when asking for an enum, one should include “other” as an option with an additional string field for a description.

Structured Output helps with Schema syntax errors, but not with semantic validation errors (e.g. items that do not sum up). These require additional validation logic.
When asking the model for correction, always include the original document, the wrong extraction as well as validation error.

## Multi-Agent Systems
### Orchestration

Multiple agents rely on the hub-and-spoke concept:

- **Coordinator Agent:** Receives tasks, decomposes them into subtasks and decides which subagent to call with which prompt, aggregates results from subagents
- **Subagents:** Execute only for a specific task, return result to coordinator agent

All communication has to go through coordinator agent, NEVER between subagents.

In order to allow agents to spawn other agents, one has to include `Agent` in the `tools` frontmatter of the Agent. One can limit which Agents can be spawned by specifying the Agent name (e.g. `Agents(worker, ...)` ). To speed things up, multiple agents should be spawned in parallel where possible.

### Prompting Subagents

Subagents do not inherit the context of the main conversation, only what is provided by coordinator agent.
To give a subagent all the required information, the prompt should also include additional metadata needed alongside the content. The prompt should be about the specific goal, not step-by-step instructions.

If the final response is missing something (e.g. not all aspects of a topic researched), the coordinator agent is to blame as it did not task a subagent to execute on it.

### Citation Handling

For every claim an agent makes, it should provide a structured citation covering the claim, the source, the relevant excerpt as well as the publication date.

Subsequent agents working with the results have to ensure the citations do not get lost, even when compacting the conversation.

### Error Handling

The subagent should return a structured summary of the error it encountered, including the failure type, what it attempted, potential partial results as well as potential alternative approaches.

A failing subagent should not cause the whole pipeline to fail, as other subagents might have been successful. Instead, the failure should be noted in the final response of the Coordinator Agent.

The subagent should never return an empty result and mark the request as a success, as no retry will be attempted.

### Agent Teams

While subagents run inside the same session and rely on a hub-and-spoke architecture, Agent Teams are independent sessions that can communicate with each other.

As such, it's best suited for complex work that can be broken up into individual tasks, that still need discussion and collaboration.

## Sessions
### Session Management

Sessions are stored locally as JSONL. To continue with a previous session, one can use `--continue` (automatically loads the most recent one), `--resume` or `/resume` .

One can use `--fork-session` or `/branch` to branch off a session. It starts with a copy of the original history, but subsequent messages don't modify the original one or other forks. Only the final result is added to the main session.

Claude keeps a checkpoint of all the file changes it made. To load a previous state, use `/rewind`. When going back within a session, Claude will also restore the state at that checkpoint.

Sessions are not tied to git branches. To fully isolate file changes and modify the same files in parallel sessions, one can use git worktrees.

### Permission Modes

Whenever Claude wants to perform actions, it pauses and asks the human for permissions. The permission mode controls how often this happens by allowing certain actions without asking:

- `default`, `plan`: Reads only
- `acceptEdits`: Reads, Edits and common filesystem operation
- `auto`: Uses classifier model to only block what escalates beyond request
- `dontAsk`: Only explicitly pre-approved tools
- `bypassPermissions`: Everything

### Context Window

Some things are loaded at the beginning of every session: `CLAUDE.md`, Auto Memory, MCP Tool Names as well as Skill descriptions.

To prevent reaching the context limit, Claude automatically compacts the message history. While it tries to keep the most relevant information, it might lose critical information.
A potential solution is to summarize the most important facts into a structured file that lives outside of the conversation and is included in every prompt.
When manually triggering a compact, it can also be guided by giving instructions (`/compact <instructions>`  or placed in `CLAUDE.md`).

Not everything gets compacted, what survives are system prompts, CLAUDE.md instructions, auto memory, rules with `path:` frontmatter and invoked skill bodies.

It can be observed that models give information that is in the middle of a long context less weight than information at the beginning or end. Thus, key findings should be placed in these positions.

Tools often return a lot of information that is not needed and just fill up the context. One should either use post-processing to trim the results or design more focused tools.

## MCP

### Tool Design

A good description is the most important mechanism for robust tool selection and should contain the following elements: what it does, expected input, expected output, when to not use.

A good description is better than adding few-shot examples to the system prompt.

### Error Handling

There are two types of failures when using MCP tools: failures with the tool execution or failures somewhere within the surrounding system.

The MCP protocol provides the `isError` flag to communicate tool failures back to the agent.

The error should also contain a description as well as information on whether the error might resolve when trying again.

### Resources

Resources that are exposed can be referenced just like files using `@` 

### Prompts

Exposed prompts become available as commands. They appear in the format `/mcp__servername__promptname` .

## Configuration and Setup

### Layout

Settings and configurations can be stored in multiple locations

| Feature | Local (`.claude/_.local._` | Project (`.claude/...` ) | User (`~/.claude/...` ) |
| --- | --- | --- | --- |
| Settings, Plugins | `.claude/settings.local.json` | `.claude/settings.json` | `~/.claude/settings.json` |
| Skills |  | `.claude/skills/`  | `~/.claude/skills/`  |
| Subagents |  | `.claude/agents/`  | `~/.claude/agents/`  |
| MCP Server | `~/.claude.json` (per project entry) | `.mcp.json` | `~/.claude.json` (top-level `mcpServers` key) |
| CLAUDE.md | `CLAUDE.local.md` | `CLAUDE.md` or `.claude/CLAUDE.md`  | `~/.claude/CLAUDE.md`  |
| Memory |  |  | `~/.claude/projects/<project>/memory/MEMORY.md` |
| Agent Memory |  | `.claude/agent-memory/<agent-name>/MEMORY.md` | `~/.claude/agent-memory/<agent-name>/MEMORY.md` |

They are applied in the following order:

1. Managed
2. CLI arguments
3. Local
4. Project
5. User

### CLAUDE.md

`CLAUDE.md` files are loaded into every session and thus should be as short and precise as possible. Put specific information, that are not always needed, into other markdown files and reference them inside the `CLAUDE.md` .

`CLAUDE.md` files can also be put inside subdirectories of a project to provide specific context for that part (e.g. monorepo setups). Claude walks up the current directory tree to the repo root and concatenates all `CLAUDE.md` files it can find, with the one closest to where Claude is launched read last and appears at the end of the context.

### Auto Memory

Claude automatically saves notes for itself about things it learns and saves them separated by project in the user's directory.

The first 200 lines of the projects `MEMORY.md` are loaded into the start of every conversation. Claude keeps it concise by moving detailed notes into separate files and references them to load when needed.

### Rules

Inside the `.claude/rules` or `~/.claude/rules/` folder, one can organize instructions as markdown files that are focused on one specific topic (e.g. testing, api design, …).

Rules can be scoped to specific file paths using the frontmatter of the rule, so they are only loaded into the context when working with matching files. This can for example be an advantage when there are conventions for file types spread across many directories and putting the same `CLAUDE.md` inside each does not make sense.

For task-specific instructions that don't need to be in the context all the time or when a matching file is opened, use a skill instead.

### Hooks

Prompt-based instructions are not a safe way to enforce alignment to processes or the usage of specific tools. If a failure would cause financial loss, security breach or compliance violation, one should use programmatic enforcement.

Hooks are shell commands stored in the `settings.json` . The most useful ones for controlling Claude’s behaviour are:

`PreToolUse` run before the agent can use the tool and can be used to check for permissions or potential dangerous usage.

`PostToolUse` run after tool execution and before the result is returned to the agent. They can be used to transform the results or handle failures.

### Skills

Skills are reusable knowledge that can be invoked by users or Claude using a slash command (e.g. `/my-skill` ). They can contain specific information or repeatable workflows. Unlike `CLAUDE.md` files, they are only added to the context when used.

The frontmatter of the `SKILL.md` can be used to specify additional settings like arguments, allowed tools or hooks.

One can add further files inside the skill folder (e.g. specific templates, detailed documentation or scripts) and reference them in the main `SKILL.md` to keep it focused and have them only loaded when needed.

### Plugins

Plugins bundle skills, hooks, subagents and MCP servers into a single installable unit.

A plugin can be shared across projects and teams by directly installing or including it in a marketplace.

## Programmatic & Scaled Use

### Programmatic Use

Claude Code can be invoked programmatically (e.g. as part of a CI/CD pipeline) and run in the non-interactive mode by passing the `-p` (or `--print`) flag.

By default it loads the same context at startup as the interactive mode (Hooks, Skills, CLAUDE.md, MCP, …), to prevent this and reduce startup time one can add `--bare` . Additional context has to be provided explicitly, for example by using `--append-system-prompt` or `--agents` .

To get structured output, that is easier to work with programmatically, one can add `--output-format json` and optionally `--json-schema` followed by a schema definition to enforce it.

To receive tokens as they are generated, one has to use a combination of `--output-format stream-json` , `--verbose` and `--include-partial-messages` .

`--allowed-tools` let Claude use tools without asking for confirmation, alternatively `--permission-mode` can be used to set a baseline.

### Batch Processing

Instead of using the synchronous API, in some cases it might also make sense to use the Batch API instead. While it offers 50% cost savings, it might at the same time take up to 24 hours to get the results back.

Each request can consist of up to 100.000 messages, where each message has a unique ID. The results can be accessed, once all messages finished processing. There is a polling endpoint to find out the current progress of the request.

### Human Escalation

There are three reasons to escalate a conversation to a human:

1. The customer explicitly asks for it
2. Policy exceptions (customer asks for undocumented behavior, e.g. competitor price matching)
3. Stuck without progress

These techniques shall not be used as triggers:

1. Sentiment-based 
2. Self-reported scores

When handing off issues to humans, it is critical to keep in mind that the human should NOT need to have access to the entire message history.
Instead, the agent should provide a summary with key details.

### Evaluation

The results of the AI should be monitored and evaluated continuously. One should not just trust aggregated metrics, but use stratified random sampling to evaluate samples across each dimension (e.g. document type, confidence).

Self-reported confidence from AI models are not calibrated and might be off. In order to still be able to employ them, one should compare them against validation sets and extract a calibrated confidence curve per model.

To best use human review capacity, one should prioritize low confidence scores or known weaknesses and not spread equally among all results.
