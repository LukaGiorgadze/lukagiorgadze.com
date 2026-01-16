---
title: "Claude"
date: 2025-12-15
draft: false
tags: ["claude", "coding agent", "reverse engineering"]
categories: [""]
summary: "Claude code under the hood"
---

When Claude Code starts, it reads your local config, checks lockfiles (yarn.lock, npm.lock, bun.lock) to understand the project setup, and looks for tools like npm and git. It also scans the repo structure and git metadata without opening file contents. That early setup is why its suggestions usually match the project context so well. Claude uses various built in system utilites to do all that operations to collect all important information about the currenct project. It's then used later for wamrup system call and during work we will explore it later.

I used `❯ sudo fs_usage -w -f filesystem | grep claude | grep -E "open|read"` to monitor Claude in real-time.

### Claude Code startup pipeline:

| Action | `fs_usage` logs |
|----------------------------------|---------------------------|
| Load user config (global + project) | `openat ... /Users/luka/.claude.json`, `read ... B=0xe94f`, `openat ... /Users/luka/.claude/settings.json`, `openat ... .claude/settings.local.json` |
| Detect language ecosystem via lockfiles | `openat ... yarn.lock`, `openat ... pnpm-lock.yaml`, `openat ... bun.lock` |
| Verify tooling availability (git, npm, node) | `open ... /opt/homebrew/bin/git`, `open ... /usr/bin/git`, `open ... npm`, `open ... node` |
| Build ignore rules from git config | `open ... .gitignore`, `open ... .git/info/exclude`, `open ... .gitconfig`, `.gitignore_global` |
| Traverse Git repo structure (no file reads) | `open (R_____N____X___) /Users/luka/projects/lomsa/.git`, `open ... .git/refs`, `open ... .git/refs/remotes/origin`, `open ... .git/refs/heads`, `open ... .git/objects` |
| Index Git graph (refs + objects) | `open ... .git/refs/...`, large burst of `open ... .git/objects/*` |


---

For observing Claude Code network requests, I used Claude gatewya with litellm 

The moment I run cli command `claude` in my repository directory, Claude sent 5 requests imeddietely:
1. Repository warm-up and context gathering (git + filesystem snapshot) (@prompts/warmup/1):
    Request: Claude Code is being initialized with a full “agent toolbelt” (Bash/Glob/Grep/Read/etc.) plus strict read-only exploration rules in the system prompt. The user’s “Warmup” message is essentially a trigger for the agent to quickly orient itself: confirm where it’s running, whether it’s a git repo, what files exist at the root, and what the current working state/history looks like. The request also includes a large, cached system context (environment + initial git status + recent commits), which reduces cost for subsequent turns.
    Response: AI responds with a short intent statement (“quick warmup…”) and immediately schedules three Bash tool calls: list the repo directory, show recent commits, and show git status. This is the standard “situational awareness” phase Claude Code often performs before doing any real work—so later actions (searching files, proposing changes, explaining behavior) are grounded in the repo structure and current branch state. The response also shows heavy prompt caching (most tokens read from cache), which is typical when Claude Code repeatedly sends a big tool/system scaffold on every turn.

2. Minimal warm-up: verify repo state and execution context (@prompts/warmup/2):
    Request: This is another “Warmup” turn, but with a different system role: instead of “file search specialist,” Claude is instructed as a “command execution specialist.” The tool list is also slimmer (only Bash). So the point of the request is not codebase exploration via Glob/Grep/Read, but simply to confirm the current shell/repo context and gather the few git facts needed before running any further terminal-driven workflow.
    Response: Claude responds by scheduling three quick Bash calls: git status, git log --oneline -5, and pwd. Compared to the previous warmup, it drops the explicit absolute -C /path usage and assumes it’s already running in the repo working directory; it also adds pwd to confirm that assumption. The result is a lightweight “are we in the right place and what’s the repo state?” snapshot used to safely decide what to do next.

3. Planning-mode warm-up: map project structure before designing changes: (@prompts/warmup/3):
    Request: This “Warmup” run is framed as a software architect / planning specialist session, explicitly read-only, with the goal of exploring the codebase enough to produce an implementation plan later. The toolset includes repo navigation (Bash), file discovery (Glob/Grep), and file inspection (Read), which signals: “gather architecture context, find patterns, then propose a step-by-step plan and list critical files.”
    Response: Claude starts an exploration pass: it queues (1) a git log to understand recent changes and project direction, and (2) a directory listing to see the repo layout. It also attempts a Read on /Users/luka/projects/lomsa—which is a directory path, not a file—so that call would likely error; conceptually, it’s trying to “read the repo root” to identify key entrypoints/docs, but it’s using the wrong tool invocation (it should have used ls for directories, or Read on specific files like README). The intended outcome is a quick mental model of the repo so the next assistant turn can focus on relevant files and produce a coherent implementation plan.

4. Command Execution Warmup (Strict Bash-Only Mode) (@prompts/warmup/4):
    Request: Configures Claude Code as a strict command execution agent, focused exclusively on safe bash usage. It defines detailed rules for running terminal commands, especially git operations, forbids any file reading or modification, and enforces guardrails around commits, pull requests, and repository safety. The goal is to prepare the agent to interact with the repository only through controlled, non-destructive shell
    Response: The assistant performs a repository warmup by inspecting the current git state and environment. It summarizes the active branch, working directory, modified and untracked files, and recent commits, confirming that the workspace is understood and ready before executing any further user-directed actions.

5. Read-Only File Search Specialist Warmup (Repo State Summary) (@prompts/warmup/5):
    Request: Sets up Claude Code as a read-only file search / code exploration specialist for the lomsa repository. The prompt emphasizes strict non-modification rules, directs the agent to use Glob/Grep/Read for investigation (and Bash only for read-only commands), and provides fixed environment context (working directory, OS, branch, git status snapshot, and recent commits) to guide an initial “Warmup” exploration.
    Response: Instead of executing inspection tool calls, the assistant provides a warmup readiness + repository state summary using the provided context: it confirms read-only mode, identifies the repo path, current branch (feat/edit-workflow-ai-agent), main branch (main), and lists modified + untracked files (including internal/activities/build_plan.go as untracked). It ends by asking what the user wants to search for next.

6. Read-Only Architecture & Planning Specialist Warmup (Awaiting Requirements) (@prompts/warmup/6):
    Request: Configures Claude Code as a software architect / planning agent operating in strict read-only mode (no file creation/modification, no redirects/heredocs that write, no state-changing commands). The prompt defines a planning workflow (understand requirements → explore codebase via Glob/Grep/Read + limited Bash → design solution → produce step-by-step plan) and requires the final output to include a short “Critical Files for Implementation” list. It also injects environment context (repo path, OS, date, branch, git status snapshot, recent commits) and sends a single user message: “Warmup.”
    Response: The assistant treats “Warmup” as a readiness check and responds with a planning-oriented acknowledgment rather than repository inspection. It confirms read-only constraints and explains what inputs it needs next (requirements, context/files to focus on, optional design perspective), then outlines what it can deliver (architecture exploration, pattern analysis, implementation plan, dependencies, critical files list) once those requirements are provided.

After the scanning, collecting project information and sendind those requests, claude *⏳ waits for user prompt*
Well, as you see, when the `claude` CLI starts, it immediately sends multiple warmup prompts before the user issues any real command. Each prompt defines a different role, such as file exploration, command execution, or architectural planning, showing that Claude Code is role-switched rather than a single general-purpose agent. Read-only mode is enforced by default, which strongly limits file modifications and reduces the risk of unintended changes. The full environment and git state are injected upfront, giving Claude immediate context about the repository, branch, and recent changes. This design makes Claude Code context-aware from the very first token and highlights that its real power comes from a structured tool system rather than the language model alone.

Here's the list of all available tools:

```
- **Task** — Spawn a specialized sub-agent  
- **TaskOutput** — Retrieve task or agent results  
- **EnterPlanMode** — Enter planning/architecture mode  
- **ExitPlanMode** — Finish planning, request approval  
- **Bash** — Run shell commands  
- **KillShell** — Stop a background shell  
- **Read** — Read a file  
- **Edit** — Modify file contents  
- **Write** — Write or overwrite a file  
- **Glob** — Find files by pattern  
- **Grep** — Search text in files  
- **NotebookEdit** — Edit Jupyter notebook cells  
- **WebFetch** — Fetch and analyze a webpage  
- **WebSearch** — Search the web  
- **TodoWrite** — Manage a structured todo list  
- **AskUserQuestion** — Ask structured clarification questions  
- **Skill** — Invoke a specialized skill  
- **SlashCommand** — Run a custom slash command  
- **ListMcpResourcesTool** — List MCP resources  
- **ReadMcpResourceTool** — Read MCP resource  
```

The interesting fact is that Claude Code dynamically changes the available tool set based on the **role** assigned to the prompt. For example, a file search specialist prompt exposes tools like `Glob`, `Grep`, and `Read`, while a command execution specialist primarily exposes `Bash` and related shell-control tools. A planning or architecture prompt includes exploration tools but explicitly forbids any write or edit capabilities, even if those tools exist elsewhere in the system.

This means tool availability is **contextual and intentional**: Claude only sees the tools that make sense for the current phase (exploration, execution, or planning). This design acts as a hard safety boundary and prevents the model from performing actions that don’t match the user’s intent or the current workflow stage.


---


## Use cases

Before we dive into how Claude is dealing with complex tasks using AI and tooling, let's see how it deals with simple tasks.
I was always curious, how different is the system and user calls between simple and complex tasks, or maybe they are the same and just a few ones?
I was also curious, how Claude makes decisions when the tasks is about creating a new things, how things are working here and how update operation is done e.g. working something on already existing like when the task is about just changing the existing codebade, while maintaining the other logics working and maintaining the stability of the whole codebase. We can dive into this details later.

So let's ask Claude a few simple question and break down each case:
1. Which programming language is being used for this project?
2. Add debug logs in the `EnrichPlan` function where appropriate `@internal/activities/enrich.go`

In the first case, Claude sent 4 requests:

- Topic classification (@prompts/case-1/1)
  Classifies the user’s question as a new topic and assigns a short internal title used for conversation management.

- Agent and tool setup  (@prompts/case-1/2)
  Loads the full Claude Code execution context and exposes the complete toolset, enabling Claude to decide whether to spawn agents or use filesystem tools, without performing any action yet.

- Codebase exploration (@prompts/case-1/3)
  Actively inspects the repository using file search tools to confirm the programming language by detecting `.go` files and the presence of `go.mod`.

- Final answer generation (@prompts/case-1/4)
  Responds to the user with a clear, evidence-based statement identifying Go as the project’s programming language.


When a simple question is asked, Claude Code does not answer it directly with a single model call. Instead, it orchestrates a small workflow made of multiple internal requests, each with a distinct role.

First, Claude runs a lightweight classification step to determine whether the user’s message introduces a new conversation topic. This happens before any code analysis and is handled by a minimal prompt with no tools enabled.
Next, the main reasoning request is issued. In this phase, Claude is given the full system context, tool definitions, environment details, and repository state. Even for a trivial question, it reasons explicitly, inspects the repository structure, and chooses to use tools rather than relying on assumptions.
Claude then performs concrete verification by exploring the filesystem. It searches for language-specific signals such as file extensions and build metadata (for example `.go` files and `go.mod`) using fast, read-only discovery tools. This confirms the answer empirically instead of guessing.
After forming the final user-facing answer, Claude triggers an additional meta-step that proposes a likely next prompt. This shows that Claude Code continuously tries to keep the interaction flowing by anticipating what the user might want to do next, even when the original task is already complete.
The last response is interesting. Claude is trying to predict the next steps it should take. Let’s look at what it did in that response:

```
The user asked "Which programming language is being used for this project?" and I answered that it's Go. 

Now, what would be a natural next step? Some possibilities:
1. Understanding the project structure better
2. Understanding what the project does
3. Getting more details about specific components
4. Setting up the project
5. Running tests
6. Understanding the recent changes (based on the git status showing modifications)

Looking at the git context:
- Current branch: feat/edit-workflow-ai-agent
- Modified files: internal/activities/node.go, internal/activities/planner.go, internal/workflows/workflow_chat.go, internal/workflows/workflow_planner.go, prompt-plan.md
- New file: internal/activities/build_plan.go
- Recent commits mention workflow loops, debug logs, timeouts

So the user seems to be actively working on a feature. Natural next steps could be:
1. "What does this project do?" - to understand the purpose
2. "Show me the project structure" - to understand how it's organized
3. "What are the recent changes?" - to understand what's being worked on
4. "How do I run this project?" - to understand how to use it
5. "What's in the modified files?" - to understand the current work
6. "Explain the workflow system" - to understand a key component

Given that this is a feature branch for "edit-workflow-ai-agent" and there are modifications to workflow-related files, a natural follow-up would be:
- "What's the current branch working on?" 
- "Explain the workflow system"
- "What are the recent changes?"
- "Show me the project structure"
- "What does this project do?"

I think "Show me the project structure" or "What does this project do?" would be most natural. Let me go with something about the project's purpose or structure.

Actually, looking at the modified files and the feature branch name, a very natural next step would be asking about what the project does or what the current feature branch is working on.
 
I'll propose a concise, direct query to understand the project's core purpose. The branch name suggests an AI agent workflow system, so I'll aim to clarify the specific focus and functionality.
```

Claude Code depends on a lot on tool usage and they are using it in a brilliant way.
Conversation history (or context window) includes user request, assistant request of tool usage, the result of tool usage and assistant verdict.
Each tool usage has it's uniq id to reference tool usage result in the context window later.
[case-1.2.png]

Now if we ask "what the project does?"

This time Claude sent 7 requests, context window contains messages from previous history even they might not be 100% relevant, it's still there.

Another interesting part in the prompt is Claude heavily using <system-reminder>
If todo list is empty, the system reminder tells Claude that there are currently no active tasks being tracked. Its purpose is to guide Claude’s internal behavior: if the current request looks like a multi-step or complex task, Claude should create and manage a todo list; if the request is simple, it should do nothing. In this case, the question is trivial, so the reminder has no effect. The important part is that Claude Code is constantly evaluating task complexity in the background and adjusting its workflow automatically.

I'll not write all requests here, but mostly it's doing the same thing: first it understand the user request, then it tries to explore codebase even further using tools,
then it tries to read contents of `README.md`, `go.mod`, `main.go`. In most of the times Claude for such kind of operations, Claude uses `You are an interactive CLI tool that helps users with software engineering tasks` system prompt. So, Claude reads some file contents requested by assistent and sends back as a context window, after that, Claude has better understand of project, but still asks for other files to get more context about the project. Now it asks more specific files such as `workflows.go` and `activities.go`.
Now after reading more files, Claude has better understand of what is project about and gives final answer to the user. It seems Claude always asking assistant as a last step to predict next possible scenarious.

Here's the last request from that session:

```
You are now a prompt suggestion generator. The conversation above is context - your job is to suggest what Claude could help with next.

Based on the conversation, suggest the user's next prompt. Short casual input, 3-8 words. Read the moment - what's the natural next step?

Be specific when you can. Even if the task seems done, think about natural follow-ups. Say "done" only if the work is truly complete.

Reply with ONLY the suggestion text, no quotes, no explanation, no markdown.
```

And the corresponding response:

```
The user just asked what the project does, and I provided a comprehensive overview of Lomsa - an AI-powered workflow orchestration platform. 

Now, what would be a natural next step? Given that we're in a code exploration phase, some natural follow-ups could be:

1. Understanding specific components in more detail
2. Looking at the current changes (since there are uncommitted changes visible in git status)
3. Getting setup instructions
4. Understanding the architecture more deeply
5. Looking at the current branch work on "feat/edit-workflow-ai-agent"
6. Asking about building/running the project
7. Reviewing what needs to be done next

Given the context that:
- There's a feature branch "feat/edit-workflow-ai-agent" 
- There are modified files in git status
- Recent commits mention "create workflow loop until good"
- There's a new file "build_plan.go" that's untracked

The most natural next steps would be:
- Asking what needs to be done on the current branch
- Understanding what the current changes are about
- Getting help with the feature being worked on
- Reviewing the changes

Something like:
- "Show me the current changes" 
- "What needs fixing on this branch"
- "Review the recent changes"
- "What's this branch about"
- "Show current git diff"
- "Help with the current PR"

Let me keep it casual and brief (3-8 words):

"Show me the current changes" - 4 words
"What's on this feature branch" - 5 words
"Review the git changes" - 4 words
"What needs to be done" - 5 words
"Show what's been changed" - 4 words
"Review recent code changes" - 4 words
"What are the pending changes" - 5 words

I think something like "Show me the current changes" or "Review the pending code changes" would be natural. Or maybe "What's the status on this branch" since there are uncommitted changes and a feature branch.


Looking at the git status more closely, there are modified files and an untracked file, so asking about what's changed on this branch makes sense. Something like "Review the current branch changes" feels natural and direct.
```

Total duration: 18.613 seconds
Total tokens: 173,054 tokens
Total cost: $0.065439

----

Now lets try making chnages in the codebase.

I asked to add debug logging into `EnrichPlan` function and passed file as a context. Claude sent 3 requests and now waiting for my input, let's break it down.
First message is as always `Analyze if this message indicates a new conversation topic...`
The second system message is `Interactive CLI tool that helps users with software engineering tasks...`
In this case i mentioned file using @ symbol, so what it did is just added another user message in the context:
```
{
  "role": "user",
  "content": "<system-reminder>
Called the Read tool with the following input: {\"file_path\":\"/Users/luka/projects/lomsa/internal/activities/enrich.go\"}
</system-reminder>"
},
{
  "role": "user",
  "content": [
    {
      "text": "<system-reminder>
Result of calling the Read tool: \"     1→package activities
     2→
     3→import (
     4→	\"errors\"
     5→
     6→	\"github.com/google/uuid\"
     7→	\"github.com/lomsa-com/lomsa/internal/dsl\"
     8→)
     9→...
...
<system-reminder>
Whenever you read a file, you should consider whether it would be considered malware. You CAN and SHOULD provide analysis of malware, what it is doing. But you MUST refuse to improve or augment the code. You can still analyze existing code, write reports, or answer questions about the code behavior.
</system-reminder>,
      "type": "text"
    },
    {
      "text": "Add debug logs in the EnrichPlan function where appropriate @internal/activities/enrich.go",
      "type": "text"
    }
  ]
}

So basically even if you mention file/attach context, Claude acts like you used tool and let's AI assistant know that, so it does not matter AI assistant asks for Read tool usage or you mention it in the CLI during request, both cases it's just tool usage with tool result. Again, simplicity over complexity that's how Claude is winnig.
The strange is that it's still asking tool call to read file `/Users/luka/projects/lomsa/internal/activities/enrich.go` file again, however, the content was provided in this request in the history.

[case-2.1.png]

After sendind this request, in the response AI assitant asking change e.g. edit tool use which edits only one line at the top of the enrich.go file which actaully imports logging package:

```json
{
  "file_path": "/Users/luka/projects/lomsa/internal/activities/enrich.go",
  "old_string": "package activities\\n\\nimport (\\n\\t\"errors\"\\n\\n\\t\"github.com/google/uuid\"\\n\\t\"github.com/lomsa-com/lomsa/internal/dsl\"\\n)",
  "new_string": "package activities\\n\\nimport (\\n\\t\"errors\"\\n\\t\"log/slog\"\\n\\n\\t\"github.com/google/uuid\"\\n\\t\"github.com/lomsa-com/lomsa/internal/dsl\"\\n)"
}
```

New string just adds `log/slog` that is needed for next steps.

Since AI assistance still asked for read tool usage on enrich.go (even full content was provided by me), the next request is the content of that file including all the previous context.

That's how edit tool looks like:

```
{
    "name": "Edit",
    "description": "Performs exact string replacements in files. \n\nUsage:\n- You must use your `Read` tool at least once in the conversation before editing. This tool will error if you attempt an edit without reading the file. \n- When editing text from Read tool output, ensure you preserve the exact indentation (tabs/spaces) as it appears AFTER the line number prefix. The line number prefix format is: spaces + line number + tab. Everything after that tab is the actual file content to match. Never include any part of the line number prefix in the old_string or new_string.\n- ALWAYS prefer editing existing files in the codebase. NEVER write new files unless explicitly required.\n- Only use emojis if the user explicitly requests it. Avoid adding emojis to files unless asked.\n- The edit will FAIL if `old_string` is not unique in the file. Either provide a larger string with more surrounding context to make it unique or use `replace_all` to change every instance of `old_string`. \n- Use `replace_all` for replacing and renaming strings across the file. This parameter is useful if you want to rename a variable for instance.",
    "input_schema": {
        "type": "object",
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "required": [
            "file_path",
            "old_string",
            "new_string"
        ],
        "properties": {
            "file_path": {
                "type": "string",
                "description": "The absolute path to the file to modify"
            },
            "new_string": {
                "type": "string",
                "description": "The text to replace it with (must be different from old_string)"
            },
            "old_string": {
                "type": "string",
                "description": "The text to replace"
            },
            "replace_all": {
                "type": "boolean",
                "default": false,
                "description": "Replace all occurences of old_string (default false)"
            }
        },
        "additionalProperties": false
    }
}
```

Now in the next steps, Claude edits the file and adds logs. It's not one shot editing file, but Claude edits files several times to add logs on different places in the same file, even the request was simple and it required only one function change, Claude took several request/responses to add logs. Actually it could in one request, but it's very interesting why Claude behaves like that. Most likely to maintain context and quality of the code.

Observing the context, Claude edited the code the same ordering that it has planned by AI in the first request, this is the assistent response to the third request,
most likely Claude is storing that as a plan and then following it strictly, becase it edited the code exactly like this ordering. So it seems like Claude is strictly following it's plan. Small chunk edits have better results then big ones.

```
Now I'll add debug logs to the EnrichPlan function. I need to:
1. Add "log/slog" import
2. Add debug logs at strategic points:
   - Entry to EnrichPlan
   - When plan is nil (error case)
   - After enriching nodes (count of nodes)
   - After enriching edges (count of edges)
   - Before returning
```

So Claude exactly followed this plan step by step and edited the file and updated code by following that plan.



