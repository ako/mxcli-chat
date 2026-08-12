# FINDINGS

Durable notes for the next session. Anything surprising, broken, or worked around
while provisioning and developing MxcliChat.

**Versions in play**

| | |
|---|---|
| Mendix | 11.13.0 |
| mxcli | built from source, `ako/mxcli` `main` @ `d762d2e` (`mxcli --version` → `mxcli version d762d2e`). Provisioning was done on the earlier `d53691b`; findings 8/10 date from that build and are **superseded** — see the 15:50 section. |
| Go | 1.24.7 linux/amd64 |
| ANTLR | 4.13.2 generator (pinned by `mdl/grammar/Makefile`, matched to the 4.13.1 runtime in `go.mod`) |
| PostgreSQL | 16, local (`/usr/lib/postgresql/16/bin`) |
| Host | Claude Code on the web, linux/amd64, 15 GB RAM |

---

## 2026-08-12 — Provisioning

### 1. Building mxcli from source needs an `antlr4` launcher, not just Go

The generated ANTLR parser is not committed, so `go install …@latest` and a bare
`go build` both fail. `make build` depends on `make grammar`, which shells out to
whatever `antlr4` is on PATH and errors out if there is none.

`mdl/grammar/Makefile` documents `pip install antlr4-tools`. Fewer moving parts is
to take the pinned jar from Maven Central and put a two-line launcher on PATH:

```sh
curl -fsSL -o antlr.jar \
  https://repo1.maven.org/maven2/org/antlr/antlr4/4.13.2/antlr4-4.13.2-complete.jar
printf '#!/bin/sh\nexec java -jar "$PWD/antlr.jar" "$@"\n' > bin/antlr4 && chmod +x bin/antlr4
PATH="$PWD/bin:$PATH" make build
```

**Verified:** `make build` completed in 2m37s cold (Go module downloads dominate)
and produced an 86 MB `bin/mxcli` that runs. The version pin is load-bearing — the
Makefile warns a generator/runtime mismatch produces a parser that will not compile.

`make build` also runs `sync-vsix`, which prints `Warning: No .vsix found. Creating
empty placeholder.` when the VS Code extension has not been built. Harmless — it
only affects `mxcli` shipping the extension for install.

### 2. `mxcli new` will not write into the repo root, and the binary it leaves behind is a hardlink

`mxcli new` refuses a non-empty directory, and a git repo always has `.git`. Create
in a subfolder and move up — but delete `<AppName>/mxcli` first: `mxcli new` step 6
*links* the binary you invoked (shared inode), so `mv` refuses it as "the same file".

```sh
./mxcli new MxcliChat --version 11.13.0 --theme ledger
rm -f MxcliChat/mxcli
bash -c 'shopt -s dotglob && mv MxcliChat/* .' && rmdir MxcliChat
```

**Verified:** the moved tree boots; `.claude/bootstrap-mxcli.sh` already named
`MxcliChat.mpr` correctly after the move, so no fixup was needed there.

Note the repo already had `LICENSE` and a stub `README.md`; the generated project
carries no `README.md`, so nothing was clobbered by the move. A repo with more
starter files would need checking.

### 3. The generated bootstrap hook downloads the *mendixlabs* nightly binary

`mxcli init` writes `.claude/bootstrap-mxcli.sh` with a hardcoded
`https://github.com/mendixlabs/mxcli/releases/download/nightly/mxcli-<os>-<arch>`.
This project deliberately tracks `ako/mxcli` `main`, so a fresh clone after an idle
reap would have silently bootstrapped a *different* binary than the one that built
the model.

**Worked around:** the script was rewritten to clone and build from `ako/mxcli`
(overridable with `MXCLI_REPO` / `MXCLI_REF`), caching the checkout in
`~/.cache/mxcli-src` and the ANTLR jar beside it. Cost: ~3 min cold on session
start, ~15 s warm once the Go build cache is populated.

**Watch for:** re-running `mxcli init` regenerates config files and may overwrite
this script back to the nightly download. Re-check it after any `mxcli init`.

### 4. No Docker in this environment — `--ensure-db` still works

`docker ps` fails (`no such file or directory` on `/var/run/docker.sock`). This
rules out `./mxcli docker check` and `mxcli test --local`'s Docker paths.

`./mxcli run --local --setup --ensure-db` does **not** need Docker: it starts the
locally installed PostgreSQL 16, creates the `mendix` role and the `mxclichat`
database, and reuses the already-cached MxBuild.

**Verified:** completed in 4.3 s (warm), reporting
`Database ready: mxclichat (user "mendix") at 127.0.0.1:5432`.

For project validation, use the `mx` binary directly rather than `mxcli docker check`:
`~/.mxcli/mxbuild/11.13.0/modeler/mx check MxcliChat.mpr`.

### 5. `mxcli version` is not a subcommand

It is a flag: `./mxcli --version`. `./mxcli version` exits non-zero with
`unknown command "version"`.

### 6. Hub preview rejected the environment's `MXCLI_HUB_KEY` (HTTP 401)

`MXCLI_HUB_KEY` is set on this environment, but
`./mxcli run --hub https://hub.mxcli.org -p MxcliChat.mpr` reported:

```
Warning: hub registration failed (hub registration failed (HTTP 401):
missing or invalid X-Hub-Key (run 'mxcli auth hub login') or X-Hub-Secret);
continuing local-only — the app runs on localhost but has no public preview URL.
```

Degrades gracefully — the app still boots locally on :8080 — but there is no
public preview URL from this cloud session. Either the key is stale or
`mxcli auth hub login` has to be run for this account. Not chased further.

**Verified:** `curl http://localhost:8080/` → `HTTP 200` under both `--local`
and `--hub`, so only the tunnel is affected.

### 7. `--hub` boots its own runtime — stop the local run first

`--hub` implies `--local`, so it starts a *second* runtime rather than tunnelling
the one already up. Two runs both want :8080. Stop the first, or pass `--app-port`.

---

## 2026-08-12 — Upgrading marketplace content

Goal: bring all bundled marketplace content to the latest 11.13.0-compatible
version. Outcome *at the time*: 20 of 21 widgets done, 0 modules.

> **Read the next section before trusting this one.** An hour later, on a newer
> `main`, the modules and the last widget all upgraded too.

### 8. Marketplace **modules** cannot be updated from the CLI — confirmed twice

> **SUPERSEDED (same day, one hour later).** True of `d53691b`, false of `d762d2e`:
> `mxcli marketplace update` landed in between. Kept for the record — the *reasons*
> below are still why the command has to transplant GUIDs. See finding 12.

`mxcli marketplace install <id>` detects the installed module and stops:

```
$ ./mxcli marketplace install 114337 -p MxcliChat.mpr
Module "WebActions" is already installed (version 2.11.0).
Target version: 2.11.2.
In-place module updates are not applied automatically (they can discard local
edits and change persistent-entity IDs, which loses data). Update via Studio Pro.
```

The underlying tool refuses too — `mx module-import` documents exit detail `3`:
*"Project already contains a module with the name of an importing module."*
So there is no force flag hiding anywhere; Studio Pro's ID-preserving merge is
the only route.

**Do not "work around" it by dropping and re-importing.** Mendix references are
by element ID, so a re-import mints new IDs: every page that references an
Atlas_Core layout would break permanently, and persistent-entity IDs changing
orphans the runtime's data. This holds even for a brand-new app with an empty
database — the cross-module references break regardless of whether there is data.

Left at these versions (latest 11.13.0-compatible in brackets):

| Module | Installed | Latest | Content id |
|---|---|---|---|
| Atlas_Core | 4.1.3 | **4.3.8** | 117187 |
| Atlas_Web_Content | 4.1.0 | **4.3.0** | 117183 |
| Administration | 4.3.2 | **4.5.0** | 23513 |
| DataWidgets | 3.5.0 | **3.11.3** | 116540 |
| FeedbackModule | 4.0.2 | **5.0.0** | 205506 |
| NanoflowCommons | 6.0.0 | **7.2.1** | 109515 |
| WebActions | 2.11.0 | **2.11.2** | 114337 |

Note this also pins the Data Widgets family of `.mpk`s (Datagrid, Gallery,
Combobox, DropdownSort, SelectionHelper, TreeNode) — those widgets ship inside
the module, not as standalone marketplace items, so they are stuck at the
module's version too.

### 9. Standalone widgets DO upgrade cleanly — 20 of them

`marketplace install` overwrites the `.mpk` for `Widget`-type content. Upgraded,
then verified with a real `mx check` (0 errors) and a boot (`HTTP 200`):

Badge 3.2.2→3.2.3 · BadgeButton 3.2.1→3.3.0 · Charts 6.2.1→6.3.2 ·
Maps 4.0.0→4.1.0 · ProgressBar 3.2.2→3.2.3 · ProgressCircle 3.3.2→3.3.3 ·
RangeSlider 2.1.4→3.0.3 · Slider 2.1.4→3.0.4 · StarRating 3.2.1→3.2.2 ·
Switch 4.2.2→4.3.0 · AccessibilityHelper 2.2.1→2.2.2 · Accordion 2.3.4→2.3.5 ·
BarcodeScanner 2.5.0→2.5.1 · Fieldset 3.2.1→3.2.2 · HTMLElement 1.2.2→1.2.12 ·
LanguageSelector 1.1.3→1.1.4 · PopupMenu 4.0.2→4.3.1 · Timeline 3.2.2→3.2.3 ·
Tooltip 1.4.2→1.5.1 · VideoPlayer 3.2.3→3.2.4

Two notes. **StarRating** is published as *"Rating"* (id 54611) — the search term
"Star Rating" returns nothing. **LanguageSelector** (202738) is marked
*Deprecated* on the marketplace; 1.1.4 installs fine but expect no more releases.

One install failed transiently on `TLS handshake timeout` fetching the CDN
(Accordion); a plain retry succeeded. Worth a retry loop when doing a batch.

### 10. The Image widget upgrade is blocked — `widget sync` does not clear CE0463

> **SUPERSEDED by `mxcli fix widgets`** (finding 13). Image is now on 1.6.0 with
> `mx check` clean. Everything below about `widget sync` and about `mx update-widgets`
> collapsing MPR v2 still holds — `fix widgets` is what wraps the latter safely.

`Image` 1.5.0 → 1.6.0 (id 118579) adds four properties
(`maxHeight`/`maxHeightUnit`/`minHeight`/`minHeightUnit`). Installing it turns a
clean project into **64 × CE0463** *"The definition of this widget has changed"*,
across Atlas_Web_Content and FeedbackModule pages.

`mxcli widget sync` is documented as PARTIAL, and here it helped not at all:

- `--dry-run` found **4** widget instances / 16 property changes in 3 units — it
  does not see the other ~60 affected instances.
- Applying it wrote those 16 changes, and `mx check` still reported **64**
  errors. Not one was cleared.

`mx update-widgets` *does* fix all of them — and **destroys MPR v2**. Verified in
a throwaway copy: after `mx update-widgets`, `mprcontents/` was gone and the
`.mpr` had gone from 68 KB (+17 MB of `mprcontents/`) to a single **14 MB**
v1 blob. `mx check` then passes with 0 errors, but the project is no longer
git-diffable and no longer what mxcli's unit-level tooling works on.

`mx convert` is no escape hatch: it converts to a Mendix *version*, not a storage
*format* — there is no v1→v2 direction.

**Decision: `Image` stays at 1.5.0.** It is the one piece of marketplace content
in this app that needs Studio Pro. Everything else is current.

**Verified:** reverting only `widgets/com.mendix.widget.web.Image.mpk` (and the
3 `mprcontents/` units `widget sync` had touched) returns `mx check` to 0 errors
with the other 20 upgrades in place.

### 11. Verify widget upgrades with a real `mx check`, not with `mxcli check`

The 64 CE0463 errors are invisible to MDL-level validation — nothing in the
scripts changed, only the packages under `widgets/`. A stash-and-compare gave
the clean baseline:

```sh
git stash push -- widgets/ && mx check MxcliChat.mpr   # 0 errors
git stash pop            && mx check MxcliChat.mpr     # 64 errors
```

After upgrading widgets, run `./mxcli widget init -p <app>.mpr` so mxcli's stored
widget definitions and the generated `.ai-context/skills/widgets/` docs match the
new packages — otherwise `CREATE PAGE` authors against yesterday's schema. Here
it reported `0 new, 1 refreshed, 32 up to date`: only PopupMenu's schema had
actually changed, which is why the other 19 upgrades caused no errors.

---

## 2026-08-12, later — `marketplace update` lands; all modules upgraded

The headline correction: **findings 8 and 10 were obsolete within the hour.**
`main` moved from `d53691b` to `d762d2e` while this session was running, adding
`mxcli marketplace update`, `mxcli marketplace diff` and `mxcli fix`. Everything
that was reported as impossible above is now done.

### 12. `mxcli marketplace update` updates an installed module in place

```
mxcli marketplace diff   <content-id> -p <app>.mpr --to <version>   # preview
mxcli marketplace update <content-id> -p <app>.mpr --to <version>
```

It transplants element GUIDs (so the runtime does not treat the module's tables
as new and drop them) and restores the user-role → module-role grants that live
in the project's security document rather than in the module. Both showed up in
the output — Administration reported `9 element identities preserved, 2 role
grant(s) restored`, FeedbackModule `19 … 1`.

All six upgradeable modules are now current:

| Module | From | To |
|---|---|---|
| Atlas_Core | 4.1.3 | **4.3.8** |
| Atlas_Web_Content | 4.1.0 | **4.3.0** |
| Administration | 4.3.2 | **4.5.0** |
| DataWidgets | 3.5.0 | **3.11.3** |
| FeedbackModule | 4.0.2 | **5.0.0** |
| WebActions | 2.11.0 | **2.11.2** |

**It does not roll back.** If a step fails partway the module is already gone, so
commit first — which is the only reason this was safe to run six times in a row.

### 13. `mxcli fix widgets` / `fix design-properties` — the missing repair step

A headless module update leaves CE0463 (widget definitions) and CE6087 (renamed
design properties) behind; the update output says so and names the fix. These
wrap `mx update-widgets` / `mx rename-design-properties` — the tools that
otherwise collapse MPR v2 — by running them, reading the result back, and
rewriting it into v2 storage with mxcli's own writer.

**Verified:** `395 .mxunit file(s), unchanged from 395 before (MPR v2 preserved)`
and the `.mpr` stayed at 76 KB, against the 14 MB v1 blob bare `mx update-widgets`
produced in finding 10's sandbox. Run both after every module update; `mx check`
went 16 errors → 0.

This also unblocks the **Image widget**: install 1.6.0, run `mxcli fix widgets`,
0 errors. Finding 10's 64 CE0463s are gone.

### 14. A later module update silently downgrades widgets an earlier one installed

Module packages bundle their own copies of shared widgets. Updating
Atlas_Web_Content and FeedbackModule *after* DataWidgets overwrote five of the
nine Data Widgets packages back to **3.4.0**, and put **Charts 6.3.0** over the
6.3.2 installed earlier. Nothing warned; `mx check` stayed clean, because an
older widget is not an error.

**Order matters: do every module update first, then re-apply standalone widgets,
then verify versions.** Checking the `.mpk` versions is the only way to see it:

```sh
for f in widgets/*.mpk; do
  printf "%-50s %s\n" "$(basename $f)" \
    "$(unzip -p "$f" package.xml | grep -oP 'version="\K[0-9]+\.[0-9]+\.[0-9]+' | head -1)"
done
```

Repaired by copying the nine widgets out of the DataWidgets 3.11.3 `.mpk`
(`marketplace download 116540 --version 3.11.3`) and reinstalling Charts.

### 15. NanoflowCommons cannot be updated — its installed version was unpublished

The blank 11.13.0 app ships **NanoflowCommons 6.0.0**, and 6.0.0 is no longer on
the marketplace (the 6.x line now starts at 6.1.1). Both `diff` and `update` fail
with `version "6.0.0" not found`, because both download the *installed* version's
package to establish the local-edit baseline. `--force` does not bypass it.

So the one module still stale is NanoflowCommons 6.0.0 → 7.2.1. A `--no-baseline`
escape hatch (accept "we could not tell" and update anyway) would close this.

### 16. `marketplace diff` reports local edits on a project nobody has edited

On this untouched blank app, `diff` flagged `SNIPPET FeedbackWidget` (Atlas_Core),
`BUILDING_BLOCK Master_Detail` (Atlas_Web_Content) and four FeedbackModule
elements as locally modified. Nobody edited them — the comparison is on `DESCRIBE`
output, and these are elements `DESCRIBE` renders imperfectly. The saved
`--save-edits` MDL gives it away: the Atlas_Core snippet comes out as an empty
`create or modify snippet Atlas_Core.FeedbackWidget (Folder: 'Web') { }`, and the
building block as `DataSource: database from ,` — under a header that says
*"Building blocks are read-only; they cannot be created via MDL."*

**Do not replay saved edits blind.** Replaying that snippet would have emptied it.
Read the file, decide, then `--force`. `diff` is still worth running: it separates
`changed` from `unknown` honestly, and the `unknown` rows (every `PAGE_TEMPLATE`
in Atlas_Web_Content — "no DESCRIBE support") say plainly what it could not check.

### 17. `SHOW MODULES` page counts changed between the two mxcli builds

`d53691b` reported Atlas_Web_Content as having **46 pages**; `d762d2e` reports
**0** — for the *same* project. It reads as catastrophic content loss after an
update and is not: `SHOW PAGES IN Atlas_Web_Content` returns 0 on both builds, so
the older column was counting page templates as pages.

**Verified** by checking the pre-update commit out into a `git worktree` and
running the *new* binary against it: 0 pages there too. Compare like with like —
when the tool changed under you, re-measure the baseline with the new tool before
believing a diff.

### 18. The FeedbackModule update leaves an unpacked duplicate widget behind

After updating to 5.0.0, `widgets/` held both `SprintrFeedbackWidget.mpk` and an
unpacked `widgets/SprintrFeedbackWidget/` directory — same widget, same version
12.0.4. `mx check` tolerates it (0 errors either way). Removed as unintended;
every other widget in the project ships as a `.mpk` only.

### Final state

Every module and widget is on its latest 11.13.0-compatible version except
**NanoflowCommons 6.0.0** (finding 15). `mx check`: **0 errors**.
`./mxcli run --local`: **HTTP 200**.

---

## 2026-08-12, later still — installing the agent-editor stack

`.ai-context/skills/agents.md` lists seven modules as the prerequisite for the
`create agent` / `create model` / `create knowledge base` /
`create consumed mcp service` document types. Installed, plus three things the
skill does not mention. `mx check`: **0 errors**. App boots: **HTTP 200**.

### 19. The stack is seven modules **plus CommunityCommons plus two widgets**

| Content | id | Version |
|---|---|---|
| Encryption | 1011 | 11.1.1 |
| GenAI Commons | 239448 | 7.2.0 |
| Mendix Cloud GenAI Connector | 239449 | 7.2.0 |
| MCP Client | 244893 | 4.1.1 |
| Conversational UI | 239450 | 7.2.0 |
| Agent Commons | 240371 | 4.2.0 |
| Agent Editor | 257918 | 2.2.1 |
| **Community Commons** | 170 | 11.5.1 |
| **Markdown viewer** (widget) | 230248 | 1.0.3 |
| **Events** (widget) | 224259 | 1.3.1 |

The last three are transitive dependencies that `marketplace install` does not
resolve — it imports exactly the module you name. They surface only as `mx check`
errors afterwards, and the error text does not say which module wanted them:

- Without CommunityCommons: **48 × CE1613**, e.g. *"The selected Java action
  'CommunityCommons.RandomHash' no longer exists"* — mostly from Encryption.
- Without the two widgets: **22 × CE0462** *"Could not find widget 'Markdown
  viewer' in the 'widgets' directory"* / `'Events'` — from Conversational UI.

Install order that worked (dependency-first): Encryption → GenAICommons →
MxGenAIConnector → MCPClient → ConversationalUI → AgentCommons →
AgentEditorCommons → CommunityCommons → the two widgets.

**Read the error code, not the count.** CE1613 is "referenced element gone"
(missing module) and CE0462 is "widget package absent" — both mean *a dependency
is missing*, not *the install failed*. Run `mxcli fix widgets` and
`mxcli fix design-properties` after each batch before believing a count: the
first pass here renamed 154 design properties across 42 documents.

### 20. `sync-java-deps` reports dependencies as missing when they are not

`./mxcli sync-java-deps -p MxcliChat.mpr` ends with:

```
sync finished but 4 dependency/dependencies are still missing from vendorlib/:
  [commons-io:commons-io:2.17.0
   com.fasterxml.jackson.core:jackson-databind:[2.21.2,3) ×3]
```

Both are false. Gradle resolved `commons-io` 2.17.0 and 2.21.0 to the higher
**2.21.0**, and the `[2.21.2,3)` range to **2.22.1** — both jars are in
`vendorlib/`. The post-check compares declared coordinates to filenames by exact
version string, so a version *range* can never match and a conflict-resolved
version looks absent.

The same output also shows a Gradle-snippet parsing bug — the exclusions inside
`io.modelcontextprotocol.sdk:mcp:2.0.0` are being read as dependency coordinates:

```
  io.modelcontextprotocol.sdk:mcp:2.0.0'){
  exclude group: 'com.networknt', module: 'json-schema-validator'
```

Harmless (Gradle gets the real file), but the report is wrong. **Verified** the
sync actually worked: `mcp-2.0.0.jar`, `mcp-core-2.0.0.jar`,
`mcp-json-jackson2-2.0.0.jar`, `okhttp-sse-4.12.0.jar` and 38 others are in
`vendorlib/`, and the app boots.

### 21. Wiring `ASU_AgentEditor` — and the one after-startup slot

Mendix allows exactly **one** after-startup microflow, and two installed modules
ship one:

- `AgentEditorCommons.ASU_AgentEditor` — **required**; materialises the model's
  agent-editor documents into `AgentCommons.*` rows at boot.
- `AgentCommons.ASU_AgentTemplates_Create` — optional template seeder, **not
  wired**. When this app gets its own module, both need chaining from a single
  project-level startup microflow.

Set via `mdlsource/agent-stack-config.mdl`. **Verified in the runtime log** —
it runs, and registers two dev servlets:

```
Core: Running after-startup-action...
Agent Editor Commons: AgentEditor_ImportFromStudioPro: Starting ASU_AgentEditor.
M2EE: Added servlet at '/dev/preview_agent_test'
M2EE: Added servlet at '/dev/preview_agent_sync'
Agent Editor Commons: AgentEditor_ImportFromStudioPro: Finished ASU_AgentEditor.
Core: Successfully ran after-startup-action.
```

**Open question for the next session.** The same log says `0 agents document(s)
found in the Mendix Model`, while `mxcli -c "LIST AGENTS"` reports **4** in
AgentEditorCommons (TranslationAgent, ProductDescription, SummarizationAgent,
InformationExtractorAgent) and `AgentCommons.Agent` has 0 rows. Those four are
the shipped templates, so the ASU may be skipping its own module deliberately —
but it means **the import path is unproven**. Author one `create agent` in the
app's own module and re-check the log line and the row count before building on
it.

### 22. Encryption needs a 32-character key or the stack fails at runtime

`Encryption.EncryptionKey` ships empty, and models/knowledge bases reference it to
store provider credentials. Set as a `Default`-configuration override, not a model
default. Exactly 32 characters — `openssl rand -hex 16` gives that; piping
`openssl rand -base64 24` through `tr -dc 'A-Za-z0-9'` gave 31 and would have
failed at runtime, not at check time.

**The committed key is a development value.** It is in git, so treat it as public
and override it per environment before this app is deployed anywhere real.

### 23. `AgentCommons.Agent` has no `QualifiedName` attribute

The skill's suggested retrieve —
`where AgentCommons.Agent/QualifiedName = 'Module.MyAgent'` — does not work on
AgentCommons 4.2.0. The entity has `Title`, `UUID`, `AgentOwner`, `UsageType`,
`Entity` and `ModelDocumentID`. `ModelDocumentID` is the field that carries the
Studio-Pro document identity; match on that or on `Title`.

---

## 2026-08-12, evening — OpenRouter as the LLM backend

Decision: talk to OpenRouter through Mendix's **OpenAI Connector** (content id
220472, **9.1.0**) rather than MxGenAIConnector, which is Mendix Cloud only.
OpenRouter speaks the OpenAI chat-completions API, and the connector is built for
that — `OpenAIConnector.Configuration` carries an `Endpoint` (*"for example,
https://api.openai.com/v1"*) and an `IsNativeOpenAI` boolean documented as
*"For similar APIs such as Mistral, it should be set to false."*

### 24. Installing the OpenAI Connector breaks the build with CE0066

Straight after `marketplace install 220472`, before any `fix` command:

```
[error] [CE0066] "Entity access is out of date. Please update security by clicking
the 'Update security' button in the domain model editor." at Domain model of
module 'OpenAIConnector'
```

This is not cosmetic — `mxbuild --serve` refuses: *"The project cannot be
deployed, because it contains errors."*

**Verified it is the package, not mxcli's repair steps:** installed into a clean
`git worktree` of the previous commit and checked before running `fix widgets` or
`fix design-properties` — CE0066 was already there. Version **9.0.0** does it too.

### 25. Neither re-granting nor revoke-then-grant fixes it — the rules have to go

Three attempts, in order:

1. **Re-issue the grants in place** (dump them with `describe`, feed them back
   through `exec`). No effect. mxcli's `GRANT` adds and modifies member entries
   but never removes one, so whatever is stale survives.
2. **`REVOKE` then `GRANT` fresh.** Also no effect — and this is the informative
   one. Revoking the six `Administrator` rules left the error standing against
   mxcli-authored `User` rules; revoking those four cleared it. So the rules
   *mxcli builds* trigger CE0066 by themselves. Every entity involved specializes
   a GenAICommons entity, and mxcli's rule builder lists inherited members.
3. **`DROP MODULE ROLE`** on both roles clears it, but takes the roles' page and
   microflow access with them.

Settled on removing the ten entity access rules and keeping both module roles —
`mdlsource/openai-connector-security-fix.mdl`. Costs nothing at security level
**OFF**, which is this app's setting, because entity access rules are not
enforced there. **Raising security to Prototype or Production means restoring
OpenAIConnector's entity access in Studio Pro first.**

**Verified:** `mx check` 0 errors, and the app boots to HTTP 200 with the
connector installed.

### 26. mxcli's GRANT validator does not see inherited associations

```
Error: entity OpenAIConnector.OpenAIDeployedModel has no member(s)
DeployedModel_InputModality; grant only names members of the entity or of an
entity it inherits from
```

It does inherit it. `OpenAIConnector.OpenAIDeployedModel extends
GenAICommons.DeployedModel`, and `GenAICommons.DeployedModel_InputModality` is a
ReferenceSet whose FROM side is `GenAICommons.DeployedModel`. Inherited
*attributes* resolve; this inherited *association* does not. It made the module's
own shipped rule impossible to re-express in MDL.

### 27. `mx check` and the build disagree about how much is wrong

`mx check` reported 1 error. The build's JSON reported eight more diagnostics —
all **warnings**, and worth reading once:

- `CE4271` — *"Version '6.0.0' of the module 'NanoflowCommons' is not compatible
  with this Mendix version"*. That is finding 15's stuck module, and Mendix
  agrees it is stale.
- `CE0582` — a static image widget "not supported in React client".
- `CE0635` ×6 — building blocks and page templates referencing other documents.

Note the codes: `CE0582` and `CE0635` carry the `CE` (error) prefix while being
reported at **Warning** severity. Count severities, not prefixes.

### 28. OpenRouter's free tier: 16 models, 15 with tool calling

From `https://openrouter.ai/api/v1/models` (no auth needed), filtering
`id.endsWith(':free')` and checking `supported_parameters` for `tools` — which
matters here, because MCP tool use needs it:

| Model | Tools | Context |
|---|---|---|
| `nvidia/nemotron-3.5-lightning:free` | yes | 1,000,000 |
| `nvidia/nemotron-3-ultra-550b-a55b:free` | yes | 1,000,000 |
| `nvidia/nemotron-3-super-120b-a12b:free` | yes | 262,144 |
| `google/gemma-4-31b-it:free` | yes | 262,144 (text+image+video) |
| `openai/gpt-oss-20b:free` | yes | 131,072 |
| `nvidia/nemotron-3.5-content-safety:free` | **no** | 128,000 |

Free-tier membership churns, so re-run the query rather than trusting this table.

**The API key does not belong in this repo.** The connector stores configurations
as runtime data — enter the OpenRouter key on the `Configuration_Overview` page
(module folder `USE_ME/Configuration`, or embed the `Snippet_Configurations`
snippet) with `Endpoint = https://openrouter.ai/api/v1`, `ApiType = OpenAI` and
`IsNativeOpenAI = false`.

---

## 2026-08-12, night — the agent-document spike

**Question:** does a `create agent` document authored headlessly with MDL get
materialised into `AgentCommons` rows by `ASU_AgentEditor` at boot? Finding 21
left it open.

**Answer: no — and the failure takes the whole app down.** Version A of the
comparison has to be built on AgentCommons *runtime data*, not on agent-editor
documents.

### 29. ASU_AgentEditor does see documents in ordinary modules

Finding 21's `0 agents document(s) found in the Mendix Model` was not a bug: the
four agents in AgentEditorCommons are its own shipped templates and are excluded.
The moment one agent document existed in a normal module, the log read
`1 agents document(s) found`. That part of the pipeline works.

### 30. A bad agent document stops the runtime from starting

Not a warning, not a skipped import — the after-startup action throws and the
runtime refuses to come up:

```
Agent Editor Commons: AgentEditor_ImportFromStudioPro: Creating/Updating agent
  documents failed due to bad input.
Core: After-startup action failed.
M2EE: Starting Mendix Runtime failed.
  Caused by: The after-startup-action failed with an exception or returned false.
```

`mx check` reports **0 errors** on the same project. So an agent document that is
structurally fine and passes every static check can still make the app unbootable,
and the only way to find out is to boot it. Keep the project committed before
adding one.

### 31. Every agent document needs a Model document, and MxCloudGenAI is the only provider

Three variants, each a full boot cycle:

| Agent document | Result |
|---|---|
| `model: SpikeModel` (`Provider: MxCloudGenAI`, key constant empty) | `No value could be found for the local configuration for the constant` → model not imported → `Model with name  is referenced but could not be found for the Agent` |
| No `model:` at all | `No Model document is referenced for the Agent.` — same fatal error |
| `model: SpikeModel` with `Provider: OpenAI` | `ImportDeployedModel: Model provider is not one of the allowed types for Model document` (a `NullPointerException`) |

The skill warns that `Provider` is free-form and nothing validates it; this is
what that costs. `Provider: OpenAI` writes, round-trips through `describe model`,
and passes `mx check` — then the runtime rejects it at boot.

With a *non-empty* dummy key the MxCloudGenAI import got one step further and
failed at `Key could not be imported` — i.e. it validates against Mendix Cloud.
**A model document needs a real Mendix Cloud GenAI key**, which is exactly the
backend this app is not using.

`Model with name  is referenced` (note the blank) also suggests the agent's model
link is resolved by a name field the portal populates, not by the qualified name
mxcli writes. Not chased further, since the provider restriction already settles it.

### 32. Consequence for the build: Version A uses AgentCommons data, not documents

`AgentCommons.Agent`'s own documentation says agents "can be created by Agent
Admins in the UI" — the entity is ordinary data. So the marketplace runtime
(agents, tools, knowledge bases, MCP services, ConversationalUI, Call Agent
actions) is fully usable headlessly; only the Studio-Pro-extension *document*
layer is not. Version A creates its agent as data in a startup microflow.

Spike cleaned up: module, model, agent and constant dropped, `mx check` 0 errors,
app boots to HTTP 200 with `Successfully ran after-startup-action`.

### 33. CORRECTION to finding 22 — a configuration override never reached the runtime

`alter settings constant 'Encryption.EncryptionKey' value '…' in configuration
'Default'` **did not work**. mxbuild writes `deployment/model/config.json` with
each constant's *default* value, and that map is what `mxcli run --local` hands
the runtime as `MicroflowConstants`. Verified directly:

```
$ python3 -c "…json.load(open('deployment/model/config.json'))…"
EncryptionKey = ''          # after the configuration override
EncryptionKey = '95d6…3bb0' # after setting the constant's DEFAULT
```

So the app had been running with an **empty encryption key** since finding 22.
Fixed in `mdlsource/agent-stack-config.mdl` by setting the model-level default.

To be clear about the deployment story: a default is a dev convenience, not the
per-environment mechanism. Constant values are set on the running app **over the
M2EE admin port** — `update_configuration`'s `MicroflowConstants` — which is how
Mendix Cloud injects them per environment and how mxcli passes them at boot.

Caveat if you try that through mxcli: `--runtime-setting` folds entries into the
boot payload with `params[k] = v`, so
`--runtime-setting 'MicroflowConstants={…}'` would **replace** the whole map
rather than merge into it, dropping every other constant. Merging map-valued
settings would be a useful mxcli improvement.

### 34. mxcli refuses to boot onto an occupied port — and it is right to

Mid-spike a `curl` returned HTTP 200 from a runtime I thought I had just started.
I hadn't: an earlier `run --local` still held 8080, and the new one had exited with

```
A stale process is silently adopted otherwise, so edits appear to do nothing
(looks like a stale cache — it isn't).
Held by pid 10741: … -Dmendix.running.locally.by.studiopro=true …
That is not a process mxcli started, so it is not a leftover run — pick another
port rather than killing it.
```

Good diagnostics, and the reason the stale `config.json` reading above looked
contradictory at first. **After a boot, check the log says it started — an HTTP
200 on 8080 only proves *something* is listening.**

---

## 2026-08-12, night — MxcliChatCore and the memory MCP server

The shared half of the app is built and working: a `Memory` entity, four tool
microflows, and an MCP server publishing them at `/memory/mcp`. Verified with a
real MCP handshake over curl — `initialize`, `tools/list`, then all four tools
called and the results checked against the database.

### 35. `autocreateddate` is renamed to the system member `CreatedDate`

`"CapturedAt": autocreateddate` is refused at exec time with a good error:

```
attribute 'CapturedAt: AutoCreatedDate' is renamed to the fixed system member
'CreatedDate' on write — the declared name is discarded … This is a Mendix system
member and cannot be bound in a widget; to store a value a widget can show, use a
plain attribute (e.g. 'CapturedAt: DateTime') and set it yourself.
```

Used a plain `datetime` set to `[%CurrentDateTime%]` on create. Worth knowing
before designing a domain model around named creation timestamps.

There is no `create module if not exists`, so a domain script that creates its own
module aborts on re-run. `create or modify` covers the entities and enumerations
inside it; the `create module` line has to be dropped when replaying.

### 36. mxcli cannot call a Java action that has a microflow-typed parameter

This blocked the MCP server outright and is the most useful finding of the session.

`MCPServer.CreateMCPServer` and `MCPServer.AddTool` each take a **microflow-typed**
parameter — `AuthenticationMicroflow`, `ExecutingMicroflow`. Calling them from MDL
produces, for every call:

```
[CE0115] "The arguments that are passed to Java action 'MCPServer.AddTool' do not
match the expected parameters and need to be refreshed."
```

`mx check` reports it, and the project does not build.

**Diagnosis** — dump the BSON of MCPServer's own example microflow, which makes the
same calls, and compare argument value types:

| Parameter | Studio Pro wrote | mxcli writes |
|---|---|---|
| `AddTool.Name` | `Microflows$BasicCodeActionParameterValue` | same ✓ |
| `AddTool.ExecutingMicroflow` | **`Microflows$MicroflowParameterValue`** | `Microflows$BasicCodeActionParameterValue` ✗ |
| `CreateMCPServer.AuthenticationMicroflow` | **`Microflows$MicroflowParameterValue`** | `Microflows$BasicCodeActionParameterValue` ✗ |

`MicroflowParameterValue` exists in `generated/metamodel` and `modelsdk/gen`, but
nothing under `mdl/` references it — there is no MDL syntax that emits one. A
missing argument is a separate, easier trap: leaving out `AddTool.Schema` gives the
same CE0115, and supplying `Schema = ''` fixes *that* half.

**Workaround** — `mdlsource/core-mcp-java-bridge.mdl`. On the Java side those
parameters are plain `java.lang.String`; the microflow typing is model-level only.
So two thin Java actions take strings and call the originals directly:

```java
return new mcpserver.actions.CreateMCPServer(
        getContext(), Path, ServerName, Version, ProtocolVersion, AuthenticationMicroflow
).executeAction();
```

The registration microflow then stays in MDL, where the tool list belongs. Delete
the bridge if mxcli learns to emit `MicroflowParameterValue`.

### 37. `mxcli check --references` does not skip forward-referenced Java actions

A script that creates a Java action and then calls it in the same file reports
`java action not found: MxcliChatCore.JA_McpServer_AddTool (referenced by call java
action)` — even though the header says references created within the script are
skipped. `exec` runs it fine and `mx check` is clean afterwards, so it is a
false negative in the checker, not a real problem. Entities and microflows created
in the same script *are* skipped correctly.

### 38. The MCP server works, and its tool schemas come from the microflow signatures

`AddTool` with an empty `Schema` derives the JSON schema from the executing
microflow's parameters. **The microflow parameter names are the tool's contract**
— renaming one renames what the model sees:

```json
"memory_search": {"type":"object",
  "properties":{"Query":{"type":"string"},
                "MaxResults":{"anyOf":[{"type":"number"},{"type":"string"}]}},
  "required":["Query","MaxResults"]}
```

Note **every** parameter comes out `required` — there is no optional-parameter
concept, so tools need to tolerate zero/empty values. Both search and list take a
`MaxResults` that falls back to a default when it is 0 or less, for that reason.

Verified end to end over curl against `POST /memory/mcp` (streamable HTTP; the
response is `text/event-stream`, and the `Mcp-Session-Id` header from `initialize`
must be echoed on later calls):

```
add     -> Stored memory #1.  /  Stored memory #2.
search  -> #1 Andrej prefers the ledger theme. |
list    -> #2 The app talks to OpenRouter… | #1 Andrej prefers the ledger theme. |
forget  -> Forgot memory #1.
list    -> #2 The app talks to OpenRouter… |
```

and confirmed in the database with `mxcli oql` — including that the enum mapped to
`Preference` and that forget is a soft delete (`IsActive = false`, row retained).
Those two rows are left in place as seed data.

### 39. The advertised MCP endpoint is `nullmemory/mcp` on a local run

`CreateMCPServer` builds the server's `Endpoint` as
`Core.getConfiguration().getApplicationRootUrl() + Path + "/mcp"`, and on a plain
`run --local` the root URL is null:

```
MCPServer: CreateMCPServer: MCP Server created with name: 'mxclichat-memory',
endpoint: 'nullmemory/mcp', and version: 'v2025_03_26'.
```

Cosmetic here — the request handler is mounted at `memory/` and serving — but a
client that discovers the server by that attribute would get a broken URL.

Setting `ApplicationRootUrl` in the `Default` configuration did **not** help:
mxcli only passes it when the app is served behind an external URL (`--hub`), and
on a local run leaves the runtime to default it, which it does not do. Reverted to
the stock value. Untested alternative:
`run --local --runtime-setting ApplicationRootUrl=http://localhost:8080/`.

### 40. Seeding an AgentCommons agent as data: what has to be linked

Version A's agent is created in a startup microflow rather than as an
agent-editor document. The non-obvious parts:

- `AgentCommons.Agent` is versioned. `Agent_Version_InUse` is what
  `Agent_Call_WithHistory` resolves — the Java action's own documentation says
  *"There must be an 'In Use' version in order to not fail."* Both directions
  need setting: `Version_Agent` on the version, `Agent_Version_InUse` on the agent.
- A model is attached to the **version**, not the agent: `Version_DeployedModel`.
- `AgentCommons.MCP` extends `AgentCommons.Tool` and binds a whole MCP server to
  a version by **name** (`_MCPServerName`), linked through `Tool_Version`.
  `AgentCommons.SingleMCPTool` is the one-tool-at-a-time variant, which binds by
  association to `MCPClient.ConsumedMCPService` instead.
- `GenAICommons.DeployedModel.Microflow` names the microflow the runtime executes
  for that model — for the OpenAI Connector,
  `OpenAIConnector.ChatCompletions_WithHistory_Execute`.
- Enum spellings that will not be guessed right:
  `ENUM_ModelSupport._True` (leading underscore), `ENUM_ToolChoice.auto`
  (lower case), `ENUM_Agent_UsageType.Conversational` (captioned "Chat").

**Verified** after boot with `mxcli oql`: agent, version (`IsDraftVersion=false`),
MCP tool (`IsEnabled=true`), deployed model and consumed MCP service all present,
and `Successfully ran after-startup-action`.

### 41. Killing a stale runtime needs care — `pkill -f` matches its own shell

Finding 34's port-in-use trap cost several boots this session. Two things make it
worse than it sounds:

- `pkill -f "mxcli run --local"` **kills the shell running it**, because the
  shell's own command line contains the pattern. Exit code 144. Kill by PID from
  `pgrep` instead, and use a pattern that cannot match the invocation.
- A `kill -9` on the runtime leaves `mxbuild --serve` holding port 6543, and the
  next run fails on *that* port instead. mxcli's message is exact about it:
  *"That is a leftover from an earlier run that did not shut down cleanly (a
  kill -9 or a reaped container skips mxcli's own teardown)."*

Clean sequence: `for p in $(pgrep -f "mendix.running.locally"); do kill $p; done`,
then the same for `modeler/mxbuild`, then confirm port 8080 answers `000`.

---

## 2026-08-12, late — version A's chat page

The page is built and the whole chain runs. It stops at the last hop, for an
environment reason rather than a modelling one.

### 42. `ChatContext_Create_ForAgent` hits the microflow-typed-parameter wall too

Second independent module, same CE0115 as finding 36:
`AgentCommons.ChatContext_Create_ForAgent.ActionMicroflow` is microflow-typed, so
calling it from MDL fails to build. Same fix — `JA_ChatContext_CreateForAgent`
takes a string and calls the Java constructor, which declares it as
`java.lang.String`.

Two modules in one session means this is not an MCP Server quirk. Teaching mxcli
to emit `Microflows$MicroflowParameterValue` would remove both bridges.

### 43. `..._ActionMicroflow_AgentBuilder` is the Agent Builder's *test* chat

The obvious-looking choice by name is wrong.
`AgentCommons.ChatContext_ChatWithHistory_ActionMicroflow_AgentBuilder` starts
from `$ChatContext/AgentCommons.PageHelper_ChatContext`, which only exists for a
chat opened inside the Agent Builder UI. From anywhere else the PageHelper is
empty, so it finds no deployed model and falls into a branch that shows
validation feedback on an object that is not on the page:

```
ConversationalUI: ProviderConfig_ExecuteAction: Cannot invoke
"IMendixIdentifier.toLong()" because "validationObjectId" is null
  at AgentCommons.ChatContext_ChatWithHistory_ActionMicroflow_AgentBuilder
    (AddValidationFeedback : 'Show validation message on member
     Version_DeployedModel of Version')
```

**A validation-feedback activity on a null object surfaces as a generic runtime
error, not as a validation message** — the user sees an assistant bubble reading
just "Error". The database was fine: `Version_DeployedModel` was correctly linked,
verified by OQL, which is what made the message so misleading.

The right one is `ConversationalUI.ChatContext_ChatWithHistory_ActionMicroflow_Agent`,
whose own annotation says it: *"If the ChatContext was created via the 'New Chat
for Agent (Runtime)' action, the agent can be retrieved and used."* With that
name passed in, `ChatContext_Create_ForAgent` creates the ProviderConfig itself —
nothing else needs seeding.

**Read the action microflow before wiring it.** The name is not the contract; the
first activity is.

### 44. The runtime cannot reach OpenRouter from this container — DNS, not credentials

With the chain correct, sending a message now fails at the last hop:

```
OpenAI connector: An error occurred in the REST call for operation Chat Completions.
Microflow: Request_POST
HttpStatus : 503
Content: DNS resolution failure
```

That is the *whole* chain working — page → ChatContext → provider config → action
microflow → agent → GenAI Commons → OpenAI Connector → REST. It dies on the
outbound call.

`curl https://openrouter.ai/api/v1/models` from the shell works, because the shell
honours this environment's egress proxy. The Mendix runtime does not: its REST
call uses Apache HttpClient, which ignores the JVM's `http.proxyHost` properties
unless explicitly configured, so the hostname never resolves.

**A real API key will not fix this on its own** in a proxied cloud container. Two
things are needed to see the agent actually answer: the key on
`OpenAIConnector.Configuration_Overview`, and outbound access for the runtime
JVM — a local run on an unproxied machine, or proxy configuration passed to the
runtime.

### 45. Browser verification without playwright-cli

`mxcli playwright verify` needs a `playwright-cli` binary that is not installed
here, but Playwright's browsers are: `/opt/pw-browsers/chromium_headless_shell-1194/chrome-linux/headless_shell`
(the path in `.playwright/cli.config.json`, `/usr/local/bin/mx-headless-shell`,
does not exist in this image). `npm i playwright-core` plus a 20-line script
drives the app fine.

One Atlas detail: at 1280×900 the navigation sidebar is collapsed, so menu items
are in the DOM but not visible and `click()` times out. `dispatchEvent('click')`
on `a[title="…"]` gets through.

---

## 2026-08-12, late — version B begins

### 46. `create entity Module.X (NON_PERSISTENT)` is not valid syntax

`rest-call-from-json.md` documents

```sql
create entity Module.MyRootObject (NON_PERSISTENT)
  stringField : string
  intField    : integer;
```

which does not parse — `extraneous input ':' expecting the start of a statement`.
The working form is the one in `mdl-entities.md`:

```sql
create non-persistent entity Module."MyRootObject" (
  "StringField": string,
  "IntField": integer
);
```

Also caught by `mxcli check`: **`Id` is a reserved attribute name** (MDL021 →
CE7247), with a suggested rename. Worth quoting every identifier, as CLAUDE.md
says, but a reserved *attribute* name is refused regardless of quoting.

### 47. An export mapping over an array needs `owner both` on the association

Modelling `{"messages":[…]}` as a parent entity with child rows fails to build:

```
[CE0295] "Association 'MxcliChatRest.RequestMessage_RequestRoot' is not allowed."
  at Object mapping element 'Messages'
```

Three shapes tried, in order:

| Association | Result |
|---|---|
| `RequestMessage -> RequestRoot`, `type reference` | CE0295 |
| `RequestRoot -> RequestMessage`, `type reference_set` | CE0295 |
| `RequestMessage -> RequestRoot`, `type reference owner both` | **builds** |

So the direction is not the problem — ownership is. An export mapping walks
parent → children, and that navigation is only allowed when both ends own the
association. The error names the association rather than the ownership, which is
what makes it a twenty-minute problem instead of a one-minute one.

The import mapping over the same array shape needed no such thing: `create
Module.Child_Parent/Module.Child = jsonKey` works with a plain reference.
