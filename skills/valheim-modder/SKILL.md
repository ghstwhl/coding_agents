---
name: valheim-modder
description: Build, fix, debug, and ship Valheim (BepInEx/HarmonyX) mods. Handles game-API and IL investigation, cross-mod compatibility, and release packaging. Use for any Valheim modding or Unity game-assembly investigation task.
---

# Valheim Modder

A working definition, packaged as a skill, for an AI coding agent asked to build, fix, or ship a
**Valheim** mod.

Written from a complete project cycle: a BepInEx/HarmonyX mod built from scratch, taken through
~15 implementation tasks, twelve in-game test builds, a multiplayer release, and a fresh-eyes review.

*Provenance: 2026-10, established against BepInEx 5.4, HarmonyX 2.x, Jötunn 2.30 on Windows. Treat every
game fact below as a hypothesis to re-verify (§18), not as a constant.*

*Second cycle (also 2026-10): the same doctrine applied to a mod that was **forked, re-identified and
publicly released** — where the hard parts were not game APIs but dependency provenance (§13), a
generated documentation pipeline (§14), and a build that had to enforce its own invariants (§15).
Lessons marked "(fork cycle)" come from there; everything else held across both projects.*

**Prime directive: never guess about game behaviour.** The game assembly, every installed mod's DLL and
source, the player's log, and the config files are all readable. Read them. A statement about the game
that came from memory or a wiki rather than from an artifact is a liability.

**Second directive: prefer the method to the fact.** Version-specific facts rot (a hotkey, a ZDO key, a
method signature). Every fact in this document therefore comes with *how it was established*, so it can
be re-established on a new game version in minutes. Teach the method, keep the fact as a hypothesis.

---

## 0. How to use this skill

This skill is plain Markdown **on purpose**. It names no tools, no slash commands, and no host features,
and assumes no capabilities — it is **agent-agnostic** by design, describing *what to look at and do*
rather than *which product does it*. Load it under whatever agent runtime you use; the `name` and
`description` frontmatter is all a skill loader needs to decide when to surface it, and you can strip
that frontmatter if you are pasting the body into a chat window.

The scaffolding sections are Valheim-specific (§1, §2, §5, §10), while the method sections generalise:
§3–§4 to any HarmonyX/Unity mod, §7–§12 to almost any AI-assisted coding task on Windows, and §13–§18
(forking/provenance, the documentation pipeline, reproducible builds, re-verification) to any mod you
did not write from scratch.

Helper tooling that assists this skill in its work — the IL/decompile probe of §9, log-grep helpers — is
something you build on demand. If you bundle such tools with this skill, keep them beside this file under
`./scripts/`.

**Working rules that make it portable**

- **Everything here is an operation, not a tool call.** "Read the log", "dump the IL", "run the tests" —
  achieve it with whatever mechanism the host provides. The doctrine is about *what to look at*, never
  about which button does it.
- **Degrade gracefully, and say what you lost.** No shell? Ask the user to run the command and paste the
  output. No file access? Ask for the artifact. No web access? §16 is ranked so that the offline sources
  come first for exactly this reason. Never silently substitute a guess for a capability you lack.
- **Durable notes live wherever the host keeps them** — a memory file, a project notes document, a
  scratch file, session memory. What matters is that decisions and their reasons are written down where
  the *next* session will read them, not which file holds them.
- **If the host offers subagents, delegate investigation to one** (decompiling, grepping a dependency's
  source) so exploration does not consume the main context. If it does not, do it inline and summarise.
- **If the host offers a plan/todo mechanism, use it** and keep it updated as work lands. If it does
  not, maintain the plan as a short file and say where it is.
- **Stop-and-ask actions are host-independent**: publishing, pushing, force-pushing, deleting remote
  refs, and any claim that in-game behaviour has been verified. See §10 and §11.
- **This skill is self-contained.** It requires no other files; the tooling described in §9 is
  something you build on demand.

**The doctrine in twelve lines** (the rest of the file is the justification)

1. Read the artifact before forming a theory — IL, log, config, DLL.
2. A plugin on disk is not a plugin loaded; a config on disk is not the value in force.
3. `[HarmonyPatch]` goes on the class, or nothing happens and nothing complains.
4. Only `public`-in-the-shipping-assembly members may be called directly; the publicized reference lies.
5. Prefer the game's own method as the shared notion, so other mods' extensions come along by
   construction.
6. Keep decisions pure and dependency-free so they can be tested; verify game-facing seams on the
   built DLL.
7. A diagnostic that cannot report the excluded case cannot explain an absence.
8. Compare builds by IL per method, never by file bytes.
9. Bump code, manifest, and changelog version together; a published version is immutable.
10. Their game, their log, their call — hand over a checklist, not a claim.
11. Know which artifacts are generated and which are hand-maintained; never let one overwrite the other.
12. A fork owes its origin: credit it, track what upstream has that you lack, and keep the old identity alive for migration.

---

## 1. First hour on a new project

Do these before writing any code. Each one has saved days on real projects.

1. **Find the game install and the *active* profile.**
   - Game: `<steam>\steamapps\common\Valheim\` — the managed assemblies are in `valheim_Data\Managed\`.
   - Mod managers keep *profiles*, and the profile in use is not always the obvious one. On Windows,
     look under `%APPDATA%\com.kesomannen.gale\valheim\profiles\<profile>\`, plus
     `%APPDATA%\r2modmanPlus-local\Valheim\profiles\<profile>\` and
     `%APPDATA%\Thunderstore Mod Manager\DataFolder\Valheim\profiles\<profile>\`.
   - Never assume; ask which profile the player is testing in, then verify by timestamps.

2. **Read the log before touching code.** `<profile>\BepInEx\LogOutput.log` is the cheapest ground truth
   in the ecosystem: it lists every plugin that loaded, every patch count, every exception, and the
   player's own messages. Concretely:
   - `Loading [Name Version] (guid)` — what actually loaded. **A plugin folder on disk is not a loaded
     plugin**, and an incompatibility can remove one from the chainloader *without* logging a skip.
   - A mod that logs its patched-method count at startup is worth more than one that does not.
   - Failures during plugin start-up can be *silent*: if one plugin's `Awake` throws, its patches never
     apply while the log still reports "N loaded, 0 skipped, 0 failed".

3. **Inventory the mod set and its versions.** Cross-mod behaviour is where most "impossible" bugs live.
   Record, per mod: version, its config file, and its hotkeys. Two mods patching the same method is
   normal; two mods claiming the same hotkey is a support burden you inherit.

4. **Establish the build/test/package loop and prove it works** before changing anything: build the
   existing source, run its tests, produce a package. A project you cannot build is a project you
   cannot verify.

5. **Build the evidence tooling first** (§9). A decompile/IL probe and a log-grep habit cost an hour and
   pay for themselves on the first bug.

6. **Read the owner's live config**, not the defaults: `BepInEx\config\<guid>.cfg`. Their values decide
   what they are actually testing, and they often differ from what the code ships.
7. **Place the project: fork or fresh, and what is generated.** Decide the mod's *identity* (whose GUID,
   what the package is called) and whether an existing mod is its origin (§13). Find the version's single
   source of truth, the one build command that enforces it (§15), and which docs are generated versus
   hand-maintained (§14). These four facts shape every later change.

---

## 2. Environment map

| What | Where |
|---|---|
| Game logic / entities | `valheim_Data\Managed\assembly_valheim.dll` |
| Unity modules (incl. `KeyCode`, `Input`) | `valheim_Data\Managed\UnityEngine.*Module.dll` |
| Mod loader | BepInEx 5.4 (`BepInExPack_Valheim`), `BepInEx\core\` |
| Patching library | HarmonyX (`0Harmony.dll`) — BepInEx's own copy is the runtime authority |
| Mod framework | Jötunn (`ValheimModding-Jotunn`) — config sync, networking helpers, prefab/item managers |
| Installed mods | `<profile>\BepInEx\plugins\<Team>-<Name>\` |
| Live config | `<profile>\BepInEx\config\<plugin-guid>.cfg` |
| Log | `<profile>\BepInEx\LogOutput.log` |
| Publicized reference assemblies | the project's `.references\` (gitignored, never redistributed) |
| NuGet cache (docs XML, pinned libs) | `%USERPROFILE%\.nuget\packages\` |
| Mod-manager downloads, per version | `<manager>\cache\<Team>-<Package>\<version>\` |
| World saves (ZDO data) | the Valheim save folder — rarely worth parsing by hand |
| Build entry point | the project's *one* build/package script (e.g. `tools\Build.ps1`) — the only sanctioned way to compile or package |
| Version single source | a code constant (e.g. `ModInfo.cs`) that every manifest and the changelog must match |
| Generated docs | templates (T4 or otherwise) + their *tracked* outputs — never hand-edit the output, never regenerate over hand-written sections |

Useful: `BepInEx` output is per-profile, so two profiles can disagree about what is installed. Always
identify the profile first.

---

## 3. Harmony: the rules that decide whether anything happens at all

1. **A patch class must carry `[HarmonyPatch]` on the *type*.** HarmonyX scans for annotated types; a
   class whose `[HarmonyPatch]` attributes sit only on its methods is **skipped silently** — no patches,
   no warning, no exception, and the loader still reports success. This is the single most expensive
   failure mode in this ecosystem: the mod loads, logs nothing wrong, and does nothing. Fix: a bare
   `[HarmonyPatch]` (no arguments) on the class, with the specific targets on the methods.
   *Prove it in the test project rather than trusting it* (§7).

2. **Targeting.** `[HarmonyPatch(typeof(X), "Method")]` for a known overload;
   `[HarmonyPatch(typeof(X), nameof(X.Method))]` when compiling against references; a `Type[]` of
   argument types for overloads. For a target in an optional dependency, resolve it at runtime with
   `[HarmonyPrepare]` + `[HarmonyTargetMethod]` and let `Prepare` return false when the dependency is
   absent — HarmonyX then skips the class cleanly.

3. **Prefixes have no guaranteed order between classes.** Two mods prefixing the same method may run in
   either order, and a prefix that depends on another mod's state being set up is therefore
   non-deterministic. If ordering matters, make it explicit with `[HarmonyPriority]` or
   `[HarmonyAfter]`, or — better — remove the dependency entirely (§6).

4. **Finalizers run after postfixes**, so a mod that scopes state in a prefix and releases it in a
   finalizer has that state visible to your postfix on the same method. That is the hook to rely on.

5. **Log what you patched, at startup**: iterate `Harmony.GetPatchedMethods()` and log the count and
   names. A silent zero is the difference between "the mod is broken" and "the mod has no patches".

6. **Never let start-up code throw before patching.** Wrap config initialisation, resolve optional
   dependencies defensively, and set an "enabled" flag rather than aborting `Awake`.

---

## 4. The publicized-assembly trap

Modding projects compile against **publicized** copies of the game assemblies so private members
compile. The shipped game has those members `private` or `internal`.

- **Rule: you may only call a member directly if it is `public` in the *unpublicized*
  `assembly_valheim.dll`.** Everything else must go through reflection (resolved once, cached, failing
  closed) or be replaced by an equivalent public call.
- Why it matters: a publicized reference makes the wrong call *compile and pass every test* while
  being unresolvable at runtime — at best dead code, at worst throwing out of a patch on a vanilla
  method and breaking that interaction for the player.
- How to check, per member: decompile the shipping assembly and read the accessibility
  (`cecil.ps1 -Type X -Method Y -Definitions`-style probe). Fields too — a probe that omits theirs is
  an incomplete check.
- Prefer **removing** a non-public dependency over reflecting it: a public equivalent is usually
  available (e.g. an inventory lookup instead of a private search helper). Reflect only when there is
  no public path, and always inside a `try`, cached in a `static readonly`, never throwing from a patch.

---

## 5. Game-API facts worth carrying between projects

Each of these cost real time to establish. Re-verify with IL on a new game version; the *method* of
verification is the durable part.

- **`Inventory.CountItems(string name, int quality, bool onlyWorldLevel)`** iterates `m_inventory`,
  matches `item.m_shared.m_name == name`, applies the `m_quality` compare **only when `quality >= 0`**
  and the world-level compare **only when the third argument is true**, and sums `m_stack`. So
  `CountItems(sharedName, -1, false)` means "every stack of that shared name, any quality, any world
  level" — the superset of what a craft consumes. *Established from IL, not docs.*
- **Three identities, easily confused:**
  - `item.m_dropPrefab.name` — the **prefab** name (`"Wood"`). The game itself hashes this as a prefab
    name (see `Humanoid.SetupVisEquipment`), which is good evidence it is always the prefab, not a
    cloned instance name. Key storage and cross-mod comparisons by this.
  - `item.m_shared.m_name` — the **localized shared name** (`"$item_wood"`). This is what inventory
    counting and consumption match on.
  - `requirement.m_resItem.gameObject.name` — the recipe/piece requirement's prefab name; the natural
    key for mapping a requirement onto storage.
- **Requirements**: `Piece.Requirement.GetAmount(quality)` is the per-craft amount;
  `Player.HaveRequirementItems(Recipe, bool discover, int quality, int amount)` answers the crafting
  panel, `Player.HaveRequirements(Piece, RequirementMode)` the build panel, and
  `Player.ConsumeResources(...)` is the single consume path for both. Quality is *ignored* by
  consumption identity; world level and quality can still block a removal, so anything you count as
  "held" must be a superset of what will be consumed.
- **Input**: `Input.GetKey` matches key **names case-sensitively** (`"Left Alt"` != `"left alt"`), and
  throws on an unrecognised name — guard it. `KeyCode` numeric values come from Unity's own enum, and
  reading them from `UnityEngine.CoreModule.dll` is a five-line Cecil job rather than a guess. Verified
  there: `LeftShift = 304`, `RightShift = 303`, `LeftControl = 306`, `LeftAlt = 308`, `A..Z = 97..122`,
  `Tab = 9`, `Space = 32`.
- **Stations have no inventory.** A fireplace/smelter/cooking station exposes no `Inventory`; the
  game feeds them through their own RPCs and consumes from the *player's* inventory. Design any
  "feed from storage" feature around that, not around a hoped-for container.
- **Drawer-style storage mods** (KG Item Drawers) keep one prefab and an amount on the object's ZDO
  (`"Prefab"`, `"Amount"`, `"Quality"`) and mutate only through their own RPCs. Read the ZDO for
  counting; use their removal RPC for taking; never write their ZDO.
- **Custom UI panels need a priority and an escape hatch (fork cycle).** A panel added with a bare
  `AddComponent<UIGroupHandler>()` defaults `m_groupPriority` to 0, and `UIGroupHandler.Update` makes the
  panel non-interactive whenever *any other* active handler has a strictly higher priority — so a
  background HUD silently swallows every click while the panel still looks open. Add the `CanvasGroup`
  **before** the `UIGroupHandler` (its `Awake` does `GetComponent<CanvasGroup>()`) and set
  `m_groupPriority` above the ambient panels. If you also hold `GUIManager.BlockInput(true)` while the
  panel is up, you **must** handle Escape yourself, or the game menu is unreachable and the player has to
  force-quit. `GUIManager.BlockInput` is ref-counted and a no-op outside the `main` scene.
- **Server-authoritative config is not something to hand-roll (fork cycle).** Mark the entries the server
  owns and let the framework's sync push them (Jötunn: `ConfigurationManagerAttributes { IsAdminOnly =
  true }` on a plugin with `[SynchronizationMode(AdminOnlyStrictness.Always)]`). Only those entries sync,
  and the ConfigurationManager window shows **no** lock, because it resolves the attributes class by
  *type name* and copies only same-named fields — its own class has no admin concept. Enforcement is the
  sync library's, not the UI's. *Established by decompiling the framework and reading its shipped XML
  docs, not from a wiki.*

---

## 6. Cross-mod interoperability

Most "my mod is broken" reports are really "two mods disagree". The durable principles:

- **Adopt the game's own method as the shared notion wherever possible.** If another mod extends
  `Inventory.CountItems` to include its own storage (backpacks do), then *using*
  `Inventory.CountItems` inherits that mod's behaviour — and its caveats — by construction, instead of
  hand-rolling a parallel count that silently disagrees with the mod that will do the paying. This
  turned a whole class of bug into a tautology on a real project.
- **Respect a dependency's scope/context gates.** Mods commonly count their extra storage only inside a
  semantically-scoped window (a depth-counted "crafting context" entered by prefixes on the game's craft
  methods). If you read a count outside that window you get a different answer than the mod will act
  on. Check where the dependency enters its scope; if your read site is not covered, do not assume.
- **Never write another mod's data structures**, and never call its "withdraw/consume" path that
  bypasses its concurrency handling. Use its public API or its RPCs.
- **Prefer a dependency's own published API client over reflection into its internals.** Some mods ship
  a zero-dependency client wrapper or a separate API assembly for exactly this. If a dependency's
  documentation says "do not reflect into internals", take it literally.
- **Inventory the hotkeys of every installed mod before choosing yours** — and never bind a
  repeating/ambient action to a key the player presses while moving. A hotkey that applies to
  "whatever is under the crosshair" becomes a *silent* state change; if it shares a key with movement,
  players disable their own storage by walking. Read the other mods' hotkeys out of their DLLs (string
  literals next to `Input.GetKey`) or their configs, and prefer a single, semantically isolated key.
  Verified the hard way: another mod advertised one modifier in its hover text while its code read a
  *different* one — believe the code, never the label.
- **Declare what you replace.** If your mod supersedes another, declare it as an incompatibility so the
  loader refuses one of them, and document the uninstall step prominently.
- **Version enforcement is a feature and a constraint**: with strict version matching, every build you
  hand over must be deployed on *both* sides (server and every client) together, or joins fail. Say so
  when you hand over a build.
- **Do not re-implement a dependency's protocol (fork cycle).** If the framework already syncs config or
  state, adopting its mechanism also adopts its constraints — the strictest being that all peers must run
  the *same* build. Say so at hand-over.
- **A dependency's gate applies to more than storage.** If a mod only exposes extra capability inside a
  semantic window (a crafting context, a permission scope), reading the underlying state outside that
  window gives a different answer than the mod will act on — and the same is true of its config values.

---

## 7. Testing strategy

- **Put pure decision logic in a dependency-free layer** (no `UnityEngine`, no game types): range
  checks, planning, config parsing, key-name normalisation, capacity maths. That layer is what you can
  actually unit-test, and it is where the subtle invariants live.
- Link those sources into the test project (`<Compile Include="..\<product>\Logic\**\*.cs" />`) rather
  than referencing the product assembly, so tests need no game DLLs. **Consequence to remember: such
  tests cannot see whether your Harmony patches apply at all.** Add a separate test that constructs a
  patched dummy target and asserts the patch fired, and that a class whose `[HarmonyPatch]` sits only on
  its methods is *not* applied. That test is your guard on §3.1.
- **Write the failing test first for pure logic**; for game-facing seams, the "test" is artifact
  verification (§9) plus the owner's in-game pass.
- **Keep one command that runs everything** (`dotnet test …`) and run it before every hand-over.
- **Pin the decisions that came from a ruling**, not just the mechanics: a test asserting a default
  value, a key's shape, or a documented invariant turns "we decided this once" into "the build tells
  you if it changes".
- **Extract the *predicate*, not just the "logic" (fork cycle).** The most valuable testable seam is
  `bool Decide(state)` — access rules, "may this piece be removed", "may this be consumed", allow-list
  matching. On a real fork exactly this predicate lived inside a Harmony patch with no test, and an `OR`
  that should have been an `AND` shipped: enabling one rule silently defeated the other, and it was
  caught in-game, not in CI. The patch should marshal inputs and call the predicate; the predicate gets a
  table-driven test of `(state → expected)` rows. *When a bug is found in a predicate, add the row that
  would have caught it, in the same change.*
- **A test you cannot write is a design smell.** If a decision needs a running game to exercise, it is
  probably hard-coded in a patch; move it to the dependency-free layer first, then test it.
- **Editor/linter diagnostics are not a build.** "No errors found" has been reported for files the
  compiler then rejected. Run the project's real build (§15) before believing a green editor.

---

## 8. Diagnostics: build the instrument before you need it

- **Log the decision, not just the outcome**, behind a debug switch: what was counted, from where, and
  why something was rejected. `"held 0 + storage 0 of 1 required | nothing holds it: 290 source(s)"`
  answered a bug report in one line after a day of speculation had failed.
- **Include the excluded.** A diagnostic that only reports what was *used* cannot explain an absence.
  Enumerate what was skipped and why (out of range, switched off, inaccessible, wrong type). Real
  lesson: a diagnostic that could not see switched-off storage sent a whole investigation down the
  wrong path, because the one thing it could not mention was the answer.
- **Deduplicate and throttle.** Log once per distinct answer, not once per frame; include a
  coarse-grained state in the key so it does not re-log as the player walks. Respect log-shy players:
  ship the switch **off** by default, and gate any per-frame work behind it.
- **Breadcrumbs at start-up**: version, plugin guid, patch count, and one line proving each
  risky subsystem initialised (e.g. the first tracked object). These make a support log self-explaining.
- **Read your own log before theorising.** Most "it does nothing" reports are answered by the load
  lines.

---

## 9. Verification without running the game

Most of the confidence on a project can be bought without launching it.

- **Decompile and read, don't recall.** A Cecil-based probe that, given an assembly, prints a type's
  methods, fields with access modifiers, a method's IL, and the call sites of a method is the single
  most useful tool. Extend it as needed: search for string literals, find literal constants near a
  call, dump enum values, compare two builds.
- **Verify on the *built* artifact**, not the source: confirm the method exists in the DLL, that its IL
  calls what you think, and that removed members are gone. Source that compiles is not source that
  behaves.
- **Comparing two builds ("nothing changed except X")**: compare **IL**, not bytes. A byte diff is
  dominated by the COFF timestamp and by metadata heap offsets shifting when any string changes length
  (a real project showed 14,607 differing bytes for a build whose only change was a version string).
  Per-method IL comparison showed the truth: one method differed, the one embedding the version.
- **Check constants against their owning enum** rather than memorising them; check hotkeys against the
  game *and* the installed mod set.
- **Log evidence beats reasoning for runtime questions.** "Does the client see the station update
  immediately after an RPC?" is not answerable from source alone; it is answerable from the owner's log
  and one test.
- **Be explicit about what still needs a human.** Keep a checklist of in-game checks the owner must run,
  and never report an in-game behaviour as verified on their behalf.

---

## 10. Releasing

- **Version discipline**: the version lives in code (a constant), in the package manifest, and in the
  changelog. Bump them together, and make the build fail if they disagree. Bump **every** build handed
  over for testing, so the tester can prove which one is loaded (the log prints it at load).
- **Package layout** (Thunderstore format): `plugins/<Name>.dll`, `manifest.json`, `README.md`,
  `CHANGELOG.md`, `icon.png`, `LICENSE`. The README *is* the mod page and the CHANGELOG *is* the
  changelog tab — so they are product documentation, not repo notes.
- **A published version is immutable.** A README or doc fix cannot be retro-fitted to a released
  version; it rides the next release. Do not edit a shipped artifact.
- **`manifest.json`**: `name`, `version_number`, `website_url` (the project's home page — pointing it
  at the Thunderstore page itself is circular; the repo-side link belongs in the repository's own
  website setting), `description`, `dependencies` as `<Team>-<Package>-<Version>` strings.
- **Thunderstore URL shape**: `https://thunderstore.io/c/valheim/p/<Team>/<Package>/` — check it
  matches the team/package the project declares, rather than typing what you remember.
- **GitHub release**: tag then release with the built zip attached. Verify what actually landed from
  the *public API* (`https://api.github.com/repos/<owner>/<repo>/releases`) rather than the local
  state — a public listing also proves the release is published and not still a draft, and comparing
  the asset's byte size against the local build proves the right file went up.
- **Before publishing**: the owner's in-game pass. Publishing, force-pushing, and any remote write are
  **stop-and-ask** actions; a draft release is invisible until published, so prepare drafts freely and
  let the human publish.
- **Declare dependencies in the manifest** — required and optional — and say in the README that every
  player and the server need the mod at matching versions when enforcement is on.
- **Make the build enforce the release invariants (fork cycle).** The version constant and every manifest
  must be checked by the build itself (it should *fail* on mismatch), alongside malformed data files
  (config/translation JSON) and missing references. Do not rely on memory to keep three files agreeing.
- **Know your doc surfaces** (§14): the mod page README and the changelog are *product* documentation
  generated from templates; a fix to them is not retro-fittable to a published version.

---

## 11. Working with the owner

- **In-game validation is the bottleneck, not your typing.** Ship one correct build rather than three
  half-fixes; each round trip costs the owner a session.
- **Give them a checklist, not a vibe.** For every behaviour that needs a human, write the check
  (including the interesting edges: empty inventory, no free slot, a full station, a second player, a
  non-host client). Mark what has actually been through a session in a dated log, so "reviewed clean"
  is never read as "works".
- **Record rulings with their reasons.** When a decision is made (a default, a divergence from the mod
  you ported, an accepted limitation), write the decision, the evidence and the cost of being wrong.
  Durable, dated notes are how a project survives a context loss; keep them where the repo's history
  keeps them, not only in a scratch directory.
- **Prefer primary artifacts over summaries** — the DLL, the IL, the log, the config. Say which source
  an answer came from, and say plainly when something could not be checked.
- **Distinguish "correct behaviour I could not foresee" from "bug".** A user-report triage that ends in
  "the mod was right, here is the flag that did it" is a success, and the finding usually says what the
  documentation should have said.
- **Offer the fresh-eyes review.** The author of a change is the worst reviewer of it. An independent
  pass over the diff, handed the invariants and asked to *test* the claims rather than accept them,
  found four real defects on a project that had already been play-tested.

---

## 12. Windows/PowerShell traps (the environment most Valheim modding happens in)

- **A CLI installed after a terminal started is not on that process's PATH.** Installers update the
  *registry*; running processes keep the environment they were born with, and new terminals inherit it
  from the editor's process, not from the registry. Fix in-session with:
  ```powershell
  $env:Path = [Environment]::GetEnvironmentVariable('Path','Machine') + ';' + [Environment]::GetEnvironmentVariable('Path','User')
  ```
  and permanently by fully restarting the editor (a window reload is not always enough).
- **`-File script.ps1 -List a,b,c` binds as ONE string**, silently. Pass a single comma-separated
  value and split it inside the script, or use `-Command` with proper arrays. This has produced a
  false "no results" from a probe more than once.
- **`-eq` / `-like` are case-insensitive.** Comparing identifiers that way silently matches
  `ManagedPins` to `managedPins`. Use `-ceq`/`-clike` when case matters.
- **`<` input redirection does not exist**; pipe instead. `2>$null` sometimes does not suppress a
  native command's stderr noise either — wrap stderr in `ForEach-Object { $_.ToString() }` when the
  output matters.
- **`${env:ProgramFiles(x86)}`** — the braces are required; `$env:ProgramFiles(x86)` parses as
  `$env:ProgramFiles` followed by a literal.
- **Decompiler libraries lock the files they read.** Read assemblies in a child process
  (`powershell -NoProfile -File …`) so the handle dies with it; otherwise the next build of that DLL
  fails with "being used by another process".
- **Probe scripts that prompt for a missing parameter will silently swallow the next command** you
  send to a persistent shell. Always pass every required argument.
- **A shared terminal interleaves input.** If the human is typing in the same terminal, your command
  line can arrive with stray control characters, quotes stripped, or prefixed garbage — which surfaces
  as a baffling "not recognized as the name of a cmdlet". If it happens, send a sacrificial first line
  (e.g. a comment) and keep the real work on the next line; and prefer short, single-purpose commands.
- **Long-lived shells remember types**: assigning `[xml]$x` once constrains `$x` for the session.
  Use fresh variable names for new types.
- **A type failure inside a per-item loop does not stop PowerShell.** It emits one error record *per
  item* — on a large assembly, hundreds of thousands of lines — and looks exactly like a hang. Set
  `$ErrorActionPreference='Stop'`, resolve reflected types *before* the loop, and buffer output into a
  bounded summary. If it already happened, treat it as a runaway loop, not a cosmetic error.
- **A wrapper that strips a leading `cd X;`** leaves the shell in the wrong directory, so a relative path
  fails with a misleading "does not exist". Use an absolute path, or verify the working directory first.
- **Stage commits by explicit path.** `git add -A` sweeps up build output and scratch files; review
  `git status --short` and add exactly the paths you mean.

---

## 13. Forking, porting, and provenance

A large share of real-world Valheim mod work is *not* greenfield: you adopt, fork, re-identify, or port
an existing mod. That changes what "correct" means.

- **Identity is a decision with migrations attached.** A plugin's GUID names its config file
  (`BepInEx\config\<guid>.cfg`), and its `Name` can be embedded in ZDO/RPC keys, asset paths and store
  listings. If you take your own identity (a fork commonly must), the change is **MAJOR**: every existing
  install's config, and any data keyed by the old name, must be migrated. Keep the old values as explicit
  constants (e.g. `LegacyGUID`, `LegacyName`) and write the migration deliberately — do not leave the old
  values "live" by accident, and do not treat a future identity change as forbidden.
- **Keep a divergence register.** A fork branches at a commit; upstream keeps moving and you silently
  miss its fixes. Record, per cycle, what upstream has that you do not (and why you did not take it). A
  real fork branched before upstream's "UI interaction hardening" release and shipped without two fixes
  it never knew existed. Check upstream's history before you call a release done.
- **Prove dependency provenance; the pack name is not the assembly version.** The Thunderstore pack
  (`denikson-BepInExPack_Valheim-5.4.2350`) ships assembly `5.4.23.5`, built from a **fork**
  (`AzumattDev/BepInEx`, branch `v5-lts`) — *not* the EOL upstream. Detect this by reading the assembly's
  file/product version and resolving the embedded commit hash; cite the exact source in the repo's
  references file. Writing "BepInEx 5.4" and moving on is how you end up unable to reproduce a build.
- **Licence hygiene, even when there is no licence.** Credit the origin of anything you port, in the repo
  *and* in the mod's own credits. If a source declares no LICENSE, do not copy its code at all — treat it
  as a **behavioural reference only** (read its IL/config to learn *what* it does), credit it explicitly,
  and say so in every mention. "Inspired by" and "based on" are different claims.
- **A peer mod is a reference implementation of behaviour, not of code.** Verify what it does from its
  DLL/log/config, not from its description or hover text (labels have been wrong). Specifying your feature
  as "behave like mod X" then means "match the behaviour you verified", never "paste X".
- **Re-identifying a fork is also a documentation act** (§14): the human name, the package name, the
  store links and the repository all move together, and the outputs that embed them must be regenerated or
  hand-mirrored.

---

## 14. Documentation and generated artifacts

Half of a shipped mod is text, and most of it is generated. Getting this wrong produces stale docs that
contradict the code — a defect players see.

- **Classify every doc file: generated or hand-maintained — and never cross the line.** Generated
  outputs (from T4 or any templating step) must not be hand-edited: the next regeneration discards the
  edit. Conversely, a file that contains hand-written sections must not be regenerated: on a real project
  a template had to be *deleted* because regenerating it would have deleted hand-written configuration and
  installation sections. List which is which, in the repo, once.
- **Templates read identity from the built assembly, never from literals.** The template source of truth
  is the code constant (name, GUID, version, URLs). Hard-coded names/URLs in a template are how a rename
  leaves stale branding frozen in every output — exactly what a fork inherits.
- **Generators that only run inside an IDE do not run for a CLI or an agent.** Design-time T4
  (`TextTemplatingFileGenerator`) regenerates only in Visual Studio; in any other environment the tracked
  outputs simply freeze, with no error. When you cannot run the generator, **hand-mirror the tracked
  outputs to match** the template change and *say that you did* — do not assume a regeneration happened.
  (A project may even ship a template the command-line host cannot compile; verify before trusting it.)
- **Enumerate every doc surface a release must touch.** For this ecosystem that is typically the
  Thunderstore README (often in more than one format), a second hand-maintained README, the changelog, and
  the issue templates. Walk the list each release; the build only checks what you taught it to.
- **Write the changelog for the reader, one behaviour at a time.** Give each setting its own section
  describing how it behaves *on its own*; only then add a section on how settings interact. Never explain
  a pre-existing setting solely through a new one. Historical entries keep their release's wording.
- **Two names, used deliberately.** Keep a *technical* identity (package/repo/GUID/ZDO prefixes — the
  searchable, precise string) and a *human* name (prose, headings). Use each in its place; never rename
  existing artifacts or identifiers to match a prose name.

---

## 15. Build reproducibility and one source of truth for the version

"I built it" must mean the same thing in every environment and for every agent.

- **One command, whole pipeline, non-zero on failure.** Restore → validate → build → package belongs in a
  single script (e.g. `tools\Build.ps1`). It runs from any directory, prints a terminal `BUILD OK` /
  `BUILD FAILED`, and exits non-zero on any problem. Do **not** assemble an ad-hoc sequence of
  `msbuild`/`nuget`/`Compress-Archive` invocations: they drift, they are not the same twice, and an agent
  cannot reliably tell success from failure. See §1.4.
- **The version has exactly one home.** One code constant; every manifest mirrors it and is *validated*
  against it. The build must **fail** when the constant and the manifests disagree — not warn.
- **Validate more than compilation.** Fold the cheap, mechanical checks into the same command:
  well-formed data files (config/translation JSON), presence of the reference assemblies, matching
  manifests. Every invariant you can express as a check should run before a human ever sees the build.
- **Build, then verify the artifact, then decide** (§9). The one command makes the artifact; a separate
  probe reads the IL to confirm what landed.
- **One bump per change set, in the same change set.** Version and changelog ride with the code (§10);
  version every hand-over so the tester can prove from the log which build is loaded.

---

## 16. References, best first

1. **The shipping game assembly's IL** — `valheim_Data\Managed\assembly_valheim.dll` via Mono.Cecil.
   The only authority on what the game does.
2. **The installed mods' DLLs and their sources** — the DLL for what they do, the upstream repo for why.
   `raw.githubusercontent.com/<owner>/<repo>/<branch>/<path>` is readable with plain HTTP requests when
   a browser-style fetch is unavailable, and `api.github.com/repos/<o>/<r>/git/trees/<branch>?recursive=1`
   lists the files. Check the repo's default branch *and* its directory layout first — a missing path is
   usually a wrong branch or a subfolder, not an absent file.
3. **The owner's `LogOutput.log` and `.cfg` files** — what is actually loaded and configured.
4. **XML API documentation shipped inside NuGet packages** (e.g. `Jotunn.xml` under
   `~/.nuget/packages/`) — authoritative for the framework's public surface.
5. **The project's own ruling/decision record** — the fastest route back into a codebase's intent.
6. **Public mod lists and manifests** (Thunderstore) — for dependency IDs, versions and what a mod
   claims to do.
7. **Community wikis and guides** — useful for recipes, item names and player-facing behaviour; never
   for API detail. Treat as a hypothesis generator.

---

## 17. Anti-patterns

- Shipping a behaviour change and a config default change in the same release without saying so.
- Deriving a prefab/localized identity from a display name, or keying storage by a localized name.
- Writing another mod's inventory, ZDO fields, or container inventory "just this once".
- Trusting a publicized reference's accessibility.
- Adding a diagnostic that cannot report the negative case.
- Changing a default and expecting existing installs to change (they will not — configs are sticky).
- Treating a wiki/README/hover-text label as evidence about code.
- Reading a "no results" from an untrusted probe as evidence of absence.
- Publishing, force-pushing, or deleting a remote ref without an explicit instruction.
- Declaring work verified on the owner's behalf when only they can run the game.
- Changing a mod's GUID/Name without migrating the old config/data (or leaving the old identity live).
- Copying a peer mod's source when its licence does not permit it — "inspiration only" is binding.
- Hand-editing generated output, or regenerating a file that is hand-maintained.
- Re-implementing a framework's config-sync or state protocol instead of adopting it and its constraints.
- Shipping more than one version reference (code vs manifests vs changelog) that disagree.
- Adding an ungated admin/server short-circuit that bypasses a rule you just added (this exact bug shipped
  once: the host could bypass a restriction it was meant to be subject to).

---

## 18. Unknowns to re-establish per project (do not carry them as facts)

- Which Valheim/BepInEx/Jötunn versions the target uses, and whether the mod manager's profile or the
  game folder is authoritative.
- The current `KeyCode` values, the current hotkey set of the installed mods, and which of the game's
  own bindings are hard-coded versus rebindable.
- Which members are `public` in the current shipping assembly.
- Each dependency's exact scope/context gating, and whether your read sites fall inside it.
- The ZDO keys and RPC names of every storage/container mod in play.
- Where the game currently prints its own user-facing messages, so your prefixes can suppress or
  complement them correctly.
- Whether this project is a fork, where it branched, and what upstream has gained since (§13).
- Which doc files in *this* repo are generated and which are hand-maintained (§14).
- The project's single build command and its version single source of truth (§15).
- Whether a given setting is server-authoritative, and through which sync mechanism.
