# FINDINGS

Durable notes for the next session. Anything surprising, broken, or worked around
while provisioning and developing MxcliChat.

**Versions in play**

| | |
|---|---|
| Mendix | 11.13.0 |
| mxcli | built from source, `ako/mxcli` `main` @ `d53691b8ce159c8d64b7645b9a95b028235ecb3e` (self-reported `mxcli --version` → `mxcli version d53691b`) |
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
version. Outcome: **20 of 21 upgradeable widgets done, 0 modules** — the module
half is not possible from the CLI at all, by design.

### 8. Marketplace **modules** cannot be updated from the CLI — confirmed twice

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
