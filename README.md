# coding_agents

A repository for storing **customized agent definitions** — reusable instructions, personas, and
working doctrine that help an AI coding agent perform specific coding tasks well.

Each document here is a self-contained definition you can install into an agent runtime (as a custom
agent, subagent, always-on instruction set, or a pasted system prompt) so the agent arrives already
briefed on the domain, the workflow, and the hazards of a particular kind of work.

## Agent definitions index

| Agent | File | Purpose |
|---|---|---|
| Valheim Modder | [`AGENT_VALHEIM_MODDER.md`](AGENT_VALHEIM_MODDER.md) | Build, fix, debug, and ship Valheim (BepInEx/HarmonyX) mods. Covers game-API and IL investigation, cross-mod compatibility, and release packaging. |

<!-- New agent definitions go in the table above, in alphabetical order by agent name. -->

## Repository rules

1. **Keep the index current.** Every time a new agent definition document is added, renamed, or
   removed, update the [Agent definitions index](#agent-definitions-index) in the same change. The
   index is the entry point to this repository and must always match the files on disk.
2. **One agent, one file.** Each agent definition lives in its own top-level Markdown document with a
   descriptive, `UPPER_SNAKE_CASE` filename (for example `AGENT_VALHEIM_MODDER.md`).
3. **Keep definitions self-contained.** An agent document should be usable on its own and should not
   assume another file in this repository is present.

## License

See [`LICENSE`](LICENSE).
