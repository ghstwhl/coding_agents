# coding_skills

A repository for storing **skills** — reusable, self-contained definitions of *how* to carry out a
specific coding task, which an AI coding agent loads on demand.

Each skill is a folder under [`skills/`](skills/) containing a `SKILL.md`: the domain knowledge, the
workflow, and the traps an agent needs to do one kind of work well. The content is deliberately
**agent-agnostic** — it describes what to look at and do, not which product to do it in — so it works
under any agent runtime, or pasted into a chat window.

## Skills index

| Skill | Path | Purpose |
|---|---|---|
| Valheim Modder | [`skills/valheim-modder/SKILL.md`](skills/valheim-modder/SKILL.md) | Build, fix, debug, and ship Valheim (BepInEx/HarmonyX) mods. Covers game-API and IL investigation, cross-mod compatibility, and release packaging. |

<!-- New skills go in the table above, in alphabetical order by skill name. -->

## Repository rules

1. **Keep the index current.** Every time a skill is added, renamed, or removed, update the
   [Skills index](#skills-index) in the same change. The index is the entry point to this repository and
   must always match the files on disk.
2. **One skill per folder, named for the skill.** Each skill lives at `skills/<name>/SKILL.md`, and the
   `name` in its frontmatter must match the folder name (`<name>` is lowercase alphanumeric with
   hyphens).
3. **Keep skills self-contained and agent-agnostic.** A skill should be usable on its own, name no host
   or product specifics, and keep any helper tools it needs beside it under `skills/<name>/scripts/`.

## License

See [`LICENSE`](LICENSE).
