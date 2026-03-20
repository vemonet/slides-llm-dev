## LLM-assisted development

*aka. AI coding or agentic engineering*

---

## Outline

1. What are LLM agents?
1. Use examples
1. Various UI approaches
1. Available coding agent products
1. Standards
1. Recommendations
1. Demo

---

## What are LLM agents?

> An LLM agent runs tools in a loop to achieve a goal

In practice: 

- Generic tools like run a given bash command, and additional ones
- Coding agents often also have access to diagnostics from a language client or IDE
- Context management: summarize when conversation gets too long
- Planning: think step by step, todo lists

---

## AI is just a tool

> There is *zero* point in talking about AI slop. That's just plain stupid. [...]
>
> As I said in private elsewhere, I do *not* want any kernel  development documentation to be some AI statement. We have enough people on both sides of the "sky is falling" and "it's going to revolutionize  software engineering", I don't want some kernel development docs to take either stance.
>
> It's why I strongly want this to be that "just a tool" statement.

\- [Linus Torvalds](https://lore.kernel.org/lkml/CAHk-=wg0sdh_OF8zgFD-f6o9yFRK=tDOXhB1JAxfs11W9bX--Q@mail.gmail.com/)

[github.com/torvalds/AudioNoise](https://github.com/torvalds/AudioNoise?tab=readme-ov-file#another-silly-guitar-pedal-related-repo)

---

## Human-directed agentic engineering

[Ladybird browser adopts Rust, with help from AI](https://ladybird.org/posts/adopting-rust)

> I used Claude Code and Codex for the translation. **This was human-directed, not autonomous code generation**. I decided what to port, in what order, and what the Rust code should look like. It was **hundreds of small prompts**, steering the agents where things needed to go.
>
> After the initial translation, I ran multiple passes of adversarial review, asking different models to analyze the code for mistakes and bad patterns. [...]

---

## Blind AI trust is a risk

> I didn’t write a single line of code for @moltbook. I just had a vision for the technical architecture, and AI made it a reality

\- Moltbook creator

The [Supabase URL and API key were found](https://www.wiz.io/blog/exposed-moltbook-database-reveals-millions-of-api-keys) in clear in the JavaScript code sent to users browsers

---

## Various UI approaches

#### CLI

- ✅ Can easily be installed and run on remote machine
- ✅ For people coding with neovim
- 2 apps instead of 1
- ⚠️ TUI are not made for complex reactive UIs

---

## Various UI approaches

#### Desktop UI

- ✅ Freedom to build something more optimized for other tasks than code
- ✅ Some people prefer having their agents running separately
- 2 apps instead of 1
- ⚠️ Just reinventing a lightweight IDE?

---

## Various UI approaches

#### Integrated in IDE

- ✅ Better integrate with existing development setup
- ✅ Can more easily iterate and edit generated code
- ✅ Get access to the same diagnostics as you (e.g. language server)

---

## Coding agents products

| Agent                 | CLI  | Desktop<br />app | IDE  | Open<br />source | Free<br />tier | Tech        |
| --------------------- | ---- | ---------------- | ---- | ---------------- | -------------- | ----------- |
| Cursor                | ✅    |                  | ✅    |                  | ✅              | VSCode fork |
| VSCode GitHub Copilot | ✅    |                  | ✅    | ☑️                | ✅              | VSCode      |
| Claude Code           | ✅    | ✅                | ✅    |                  |                | TS          |
| OpenAI Codex CLI      | ✅    | ✅                | ✅    | ☑️                |                | Rust        |
| Antigravity           | ✅    |                  | ✅    |                  | ✅              | VSCode fork |
| OpenCode              | ✅    |                  | ☑️    | ✅                |                | TS          |
| Goose                 |      | ✅                |      | ✅                |                | Rust/TS     |

---

## Standards

- **MCP** [modelcontextprotocol.io](https://modelcontextprotocol.io) for exposing tools to LLM in a standard manner
- **Skills** [agentskills.io](https://agentskills.io) for providing instructions and scripts
- [AGENTS.md](https://agents.md/) for general project-level instructions
- [Spec-based development](https://github.com/github/spec-kit)

Linux Agentic AI foundation: [aaif.io](https://aaif.io)

---

## MCP server

Expose:

- **`tools`**: take input arguments, do something, return results
- `resources`: resources available to the agent
- `prompts`: list of prompts

Multiple transports:

- `stdio`: when running the server locally
- `streamable-http`: when deploying it as a remote HTTP endpoint

> Find servers in various MCP registries
>
> Official registry: [registry.modelcontextprotocol.io](https://registry.modelcontextprotocol.io)
>
> GitHub registry for curated servers: [github.com/mcp](https://github.com/mcp)

----

## MCP server · Tools

```python
from mcp.server.fastmcp import FastMCP
from pydantic import BaseModel

mcp = FastMCP(
    name="SIB MCP",
    dependencies=["mcp", "httpx", "pydantic"],
    instructions="Provide tools to access data from various SIB projects.",
    json_response=True, stateless_http=True,
)

class SearchResult(BaseModel):
    id: str
    description: str
    score: float

@mcp.tool()
async def search_datasets(search_input: str, update_date: str | None = None) -> list[SearchResult]:
    """Search for datasets relevant to the user question.

    Args:
        search_input: Natural language search input
        start_date: Optional last update date in yyyy-MM-dd

    Returns:
        Relevant datasets
    """
    return get_relevant_datasets(search_input, update_date)

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="MCP server for the SIB.")
    parser.add_argument("--http", action="store_true", help="Use Streamable HTTP transport")
    args = parser.parse_args()
    if args.http:
        mcp.run(transport="streamable-http")
    else:
        mcp.run()
```

----

## MCP server · Resources

```python
@mcp.resource("sib://databases")
async def list_databases() -> str:
    """List all available SIB databases with their descriptions."""
    return json.dumps(get_all_databases(), indent=2)

@mcp.resource("sib://dataset/{dataset_id}/schema")
async def get_dataset_schema(dataset_id: str) -> str:
    """Get the JSON schema for a specific dataset.

    Args:
        dataset_id: ID of the dataset

    Returns:
        JSON schema describing the dataset fields and types
    """
    return json.dumps(fetch_dataset_schema(dataset_id), indent=2)
```

----

## MCP server · Prompts

```python
@mcp.prompt()
def explore_dataset(dataset_id: str, goal: str) -> str:
    """Generate a prompt to guide the agent in exploring a dataset.

    Args:
        dataset_id: ID of the dataset to explore
        goal: What the user wants to achieve with the dataset
    """
    return f"""You are helping a researcher explore the '{dataset_id}' dataset from SIB.

Their goal: {goal}

Start by retrieving the dataset schema using the resource sib://dataset/{dataset_id}/schema,
then search for relevant entries using search_datasets, and summarize your findings clearly."""
```

---

## Skills

A collection of markdown files with YAML frontmatter, and optionally additional scripts.

Each skill has a short description, the agent will use it to decide if it needs to call a skill or not.

Skills can just be documentation on how to do something, or can contain additional scripts to make it easier for the agent to do certains tasks.

Browse skills at [skills.sh](https://skills.sh)

> Skills requires to **run on your machine**, whereas MCP can be deployed as a remote server, with the execution of tools happening on the server.

----

## Skills

````markdown
/---
name: roll-dice
description: Roll dice with true randomness. Use when asked to roll a die (d6, d20, etc.), generate a random dice roll.
/---

To roll a die, use the following command that generates a random number from 1
to the given number of `$SIDES`:

```bash
shuf -i 1-$SIDES -n 1
```
````

---

## Different usage profile

- Use a LLM occasionally for help
- Use LLM to write most of the code, but still read it and understand its structure
- Delegate everything to the agent, does not even read the code anymore

Even occasional users should consider using a coding agent, as it gives access to much more context when performing the task.

---

## Take into account the LLM limitations

Use the LLM more as a human computer interface than a trustworthy knowledge base

✅ Good with old living standards

⚠️ Bad out of the box with libs that change regularly

Make sure it gets access to the up-to-date documentation through tools, or just copy relevant context in your project subfolder.

---

## Recommendations

- Clearly define the problem to solve, provide tests to pass
- Use explicit types and linters (e.g. eslint)
- Careful with custom zsh config (e.g. oh-my-zsh)
- Explore available tools for your use-cases, e.g. [chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp)
- Commit before starting a major change

---

## Recommendations

Different coding agents count usage differently

- With GitHub Copilot it is per request (~0.3%), so it is economical to ask for multiple tasks and stuff the context
- With Claude Code it is per token consumed

Pick a LLM depending on task

- Small cheap LLM for simple task, e.g. generate a schema for this dictionary
- Large expensive LLM for more complex task, e.g. implement this feature, improve performance

---

## Recommended Patterns

1. Use Test‑Driven Development (TDD) to guide agents
2. Provide clear, visible context
3. Introduce new context when blocking on a problem
4. Treat code generation as “cheap” but verification as essential
5. Work in small, iterative feedback loops
6. Maintain human ownership of design decisions

---

## Anti-patterns

1. One‑shot “do everything” prompts
2. Context starvation
3. Over‑delegation

> [Writing about Agentic Engineering Pattern](https://simonwillison.net/2026/Feb/23/agentic-engineering-patterns/)
>
> [Augmented Coding Patterns](https://lexler.github.io/augmented-coding-patterns/)

---

## Demo

Enable/disable tools

Make changes to a project in VSCode using GitHub copilot

> Fix the errors in this page
>
> Rename the `db_names` param to `dbs` across the codebase and update all related docs
>
> Add molecular descriptor filtering

<!-- https://github.com/sib-swiss/mol-search-sparql-service -->

---

## Thank you
