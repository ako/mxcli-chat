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
