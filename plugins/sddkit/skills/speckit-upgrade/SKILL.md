---
name: "speckit-upgrade"
description: "Upgrade the Spec Kit CLI, then refresh the project's integration files and extensions"
compatibility: "Requires the `specify` CLI on PATH, and `gh` (authenticated) with `jq` for Step 2's release notes."
metadata:
  author: "João Horta Alves"
user-invocable: true
disable-model-invocation: false
---

# Upgrade Spec Kit

The procedure is Spec Kit's own manifest-aware upgrade path, documented at
<https://github.github.com/spec-kit/upgrade.html>. Read that for what each
command does, what it protects, and how `--force` behaves — Spec Kit moves fast,
the guide is the authority, and this file deliberately does not restate it.

What is here is the command sequence, plus the things the guide does not cover.
Those are the reason this skill exists.

**This skill takes a position: `.specify/templates/` stays byte-identical to
upstream.** What a project needs to say belongs in
`.specify/memory/constitution.md`, which `plan`, `tasks` and `analyze` read at
runtime; a template edit is a second copy with no reader, and it drifts in
silence. So a modified template is drift to reset, never a customization to
weigh — which is why nothing below is a restore procedure. A project that
instead treats materialized templates as reviewed, committed artifacts wants
`specify preset add constitution-sync` and a different skill.

Two placeholders appear throughout. `<key>` is an installed integration key — the
agent the project was initialized for (`claude`, `copilot`, `cursor`, …); read
the real ones out of Step 2 rather than assuming. `<agent-dir>` is where that
integration renders its command or skill files (`.claude/skills/` for Claude
Code, and the equivalent elsewhere).

## Execution

**Step 1 — clean working tree.**

```bash
git status --porcelain
```

Any output: stop and ask the user to **commit**, not stash. A stash restores
managed files to their committed content, and a managed file that then matches
the manifest is one Step 4 refreshes instead of preserving.

**Step 2 — check what an upgrade would do.** Read-only.

```bash
specify self check
specify integration status
```

Report both, and **write down which files are reported modified** — Step 6
compares against that list. The status output predicts Step 4: files reported
modified under `speckit` (shared infrastructure — `.specify/scripts/`,
`.specify/templates/`) are preserved; files reported under an integration key
block the run. If the CLI is current and nothing is modified, say so and ask
whether to continue anyway — extension versions are tracked separately from the
CLI's.

Then read the release notes of every release after the older of two versions —
the installed CLI's, and `speckit_version` in `.specify/init-options.json`, which
another machine's upgrade can leave ahead of it — up to the latest. Not just the
latest, because a note is not repeated in later ones. This prints them newest
first:

```bash
# Every page: there are more releases than one page holds, and jq slices a whole
# page for a tag it cannot find (`.[:null]`) rather than failing.
gh api --paginate 'repos/github/spec-kit/releases?per_page=100' | jq -rs \
  'add | (map(.tag_name) | index("<older-tag>")) as $i
   | if $i == null then error("<older-tag> not found") else .[:$i][] end
   | "## \(.tag_name)\n\(.body)"'
```

The Step 4–5 diff shows what changed; only a note announces what is *going to*
change, since an upcoming deprecation ships beside a command that still works.

Report what bears on this project, checked against its own usage — the `speckit-*`
skills, `.specify/`, and anything that calls `.specify/scripts/`: deprecations,
announced or done; removed or renamed commands, scripts and flags; changes to the
upgrade path itself; changes to an installed integration or extension. Catalog
entries for extensions not installed, and the repository's own chores, do not
bear. A breaking change makes going on to Step 3 the user's call. An upcoming
deprecation is recorded for a later decision, not acted on now.

**Step 3 — upgrade the CLI.**

```bash
specify self upgrade
```

This reinstalls through whichever installer the CLI detects it was installed
with. If it reports an unsupported install kind it exits 0 with guidance —
surface that and stop. Reinstalling by a different method than the original can
leave `self upgrade` unable to detect the install at all, so switching is the
user's decision, not a workaround to apply silently.

It installs the latest release, not a pinned one. `--tag` with Step 2's newest tag
would guarantee the notes read cover what installs, but it departs from the guide
and adds a wrong-tag failure to close a window of minutes, which Step 6's version
report exposes anyway.

**Step 4 — refresh the integration files.**

```bash
specify integration upgrade <key>
```

Run once per key reported in Step 2; do not assume there is only one.

Its warning that shared infrastructure paths "already exist and were not updated"
lists every existing one, including those it refreshed. The manifest
diff says which were; ignore the warning and its `--force` suggestion.

Being preserved or blocked on — Step 2 predicts which — is the check working,
not an error. Name each file the run reports, say what the upgrade would have
changed in it, and ask — never decide on the user's behalf that a customization
is expendable.

**A preserved template is drift.** Reset it. If it carries something the
constitution does not state — a required section, a fixed table layout, an agreed
vocabulary — record that in the constitution first, through the project's own
amendment procedure, so nothing is lost; then reset. To reset, delete the file
and re-run this step's command: it rewrites any managed file that is absent and
leaves every other one alone. `git checkout` is the wrong instinct — Step 1
mandates a clean tree, so drift that survives to here is committed drift, and
restoring HEAD restores it.

Do not diff a template against the CLI's bundled copy to judge whether it is
customized. The bundle stores `__SPECKIT_COMMAND_<NAME>__` placeholders, so a raw
diff reports a difference in **every** template. The manifest hash check behind
`integration status` is the authority.

After this step `integration status` should report no modified template. Whatever
else it reports is the project's declared steady state — record it, because a
future run seeing a different one needs to know whether that is drift or a
decision.

**Never pass `--force` on your own initiative.**

**Step 5 — update the extensions.**

```bash
specify extension update
```

It asks for confirmation and has no `--yes` flag, so under an agent's stdin the
first run lists what it would update and then aborts on EOF. Treat that listing
as the ask; once the user approves, re-run it as
`printf 'y\n' | specify extension update`. Reporting the abort as a failed run
is the mistake to avoid — nothing was attempted.

The rendered command or skill files for installed extensions live under
`<agent-dir>`, so they change here rather than in Step 4.

**"Up to date" is not evidence the payload matches.** That check compares the
declared `version` in `extension.yml`, and upstream ships content changes without
bumping it — observed on a core extension whose installed and bundled copies both
declared `1.0.0` while their command files, scripts and README differed
substantially. Confirm it against a reference project instead: rendered by `init`
in an empty scratch directory, never the project, once per key, with the script
from `.specify/init-options.json` and every extension installed here. Keep it for
Step 6 by its printed path — a shell variable does not survive to a later command.

```bash
REF=$(mktemp -d); echo "$REF"
(cd "$REF" && specify init --here --force --integration <key> --script <script> \
    --ignore-agent-tools --extension <id> --extension <id> < /dev/null)
for d in .specify/templates .specify/scripts .specify/extensions <agent-dir>; do
    diff -r -q "$REF/$d" "$d"
done
```

The reference resolves placeholders, so unlike the bundled copy Step 4 warns
against it diffs byte for byte, and it renders each extension's skills as well as
its payload. `init` installs an extension's bundled copy where the release has
one; an extension it reports it cannot install has nothing to compare against and
is not stale. Diff those directories, not all of `.specify/`: the reference also
carries a placeholder constitution and a workflow (below).

Expected differences, to leave alone:

- `*-config.yml` — project settings;
- `.specify/extensions/.registry` — install timestamps, and the reference's own
  empty `registered_skills` (below);
- `.cache/` and `.backup/` under `.specify/extensions/` — local CLI state;
- `Only in <agent-dir>` for a skill this project authored.

Anything else is stale despite the version check. Report it; for an extension,
ask before the fix, since the fix is a forced overwrite:

```bash
specify extension add <id> --force
```

It preserves existing `*-config.yml`, so project settings survive, but leaves
them without newly added keys, which then fall back to their defaults. Two more
consequences to check afterward:

- **`.specify/extensions/.registry` loses `registered_skills`** — the forced
  install writes the skill files but records an empty list. Where the
  integration's commands *are* its skills (Claude), `specify extension remove`
  still deletes them through `registered_commands`, so nothing is lost; for any
  other integration it reads that list to know what to delete, so an empty one
  silently orphans them. Restore the names there.
- **Scripts can be renamed** — v0.16.2 renamed the git extension's
  `create-new-feature.sh` to `create-new-feature-branch.sh`. Grep for the old
  name before assuming nothing referenced it.

**Workflows.** `init` adds a `speckit` workflow that nothing refreshes —
`specify workflow update` skips what `init` installed, and `init` never replaces
it — so a project that does not run it is better without `.specify/workflows/`;
the CLI reads a missing registry as empty. Re-adding it from the catalog would let
`update` reach it, but the catalog serves upstream's `main`, not the release. A
workflow installed on purpose gets `specify workflow update` as its step here,
with the same confirmation as above.

**Step 6 — verify.**

```bash
specify version
specify integration status
```

`specify version` reads the installed CLI locally. Deliberately not `self check`
here: Steps 2 and 3 have already reported the version change between them, and a
third release-API call to re-answer it fails confusingly offline — an upgrade
that worked reports "Could not check latest release" at the verify step.

Status should report the modified-file set the user deliberately kept in Step 4,
and nothing else. Compare against Step 2's list and explain any difference.

Status only checks files against their own manifests, so it cannot see a layer
left behind. Re-run Step 5's diff loop against the same reference: only its
expected differences should remain.

## Output

1. The old and new CLI versions, or that it was already current.
2. `git diff --stat` **and** `git status --short` — both, because an upgrade adds
   files as well as changing them and `git diff` shows none of them until they
   are staged. Group by source: `<agent-dir>` (integration and extension
   renders), `.specify/scripts/` and `.specify/templates/` (shared
   infrastructure), `.specify/extensions/` (payloads), and everything else — the
   `.specify/integrations/` manifests Step 4 reads on the next run, plus
   top-level `.specify` state.
3. Which managed files were **preserved** as locally modified, which were reset
   to upstream by decision, and which were force-refreshed and re-applied by
   hand. Leaving that out reads as "fully upgraded" when it is not — the thing
   most worth stating.
4. Anything in the diff that changes behaviour rather than wording — a new or
   renamed script, a changed hook contract, different arguments — since those are
   what break the other `speckit-*` commands. Add the release-note items Step 2
   found to apply.
5. State that nothing is committed. If the git extension is installed, suggest
   `/speckit-git-commit` once the user has reviewed the result.

## Graceful Degradation

- **`specify self upgrade` fails** → surface the error and stop; do not run the
  project-file steps. Exit code 124 is ambiguous: either the
  `SPECIFY_UPGRADE_TIMEOUT_SECS` cap fired or the installer itself exited 124.
  Read the message before reporting which.
- **`specify integration upgrade` fails for any reason other than modified
  files** → surface it and stop before Step 5, so the integration and extension
  layers are not left refreshed against different CLI versions.
- **`specify extension update` fails** → surface it and report the run as
  partial: Step 4's changes are already on disk and the extensions are not.

## Fallback: re-running init

```bash
specify init --here --force --integration <key>
```

An escape hatch, not the upgrade path, justified only when the manifest-aware
path cannot run — the guide's *Fallback: re-run init* names the conditions.
**Never run it without explicit user approval**: say which of those conditions
applies, say what will be overwritten, and wait for a yes. Naming the reason is
not the same as being allowed to proceed.

It has effects the guide does not mention, each confirmed in the `init.py` of the
version it names:

- `.specify/templates/` is **not** preserved (v0.16.2) — and under this skill's
  policy that costs nothing, because those files are already pristine. Confirm
  rather than assume: `integration status` must report no modified template
  afterwards.
- With `--force` it re-registers installed extensions and presets
  (`_register_extensions_for_agent(..., force=True)`, v0.16.2), overwriting the
  rendered extension files under `<agent-dir>` from the **installed** payload
  without updating that payload. Include those in the post-init diff, and still
  run Step 5 afterward.
- It reinstalls the `speckit` workflow (v1.0.7). A project that kept none deletes
  `.specify/workflows/` to have none again; `specify workflow remove` leaves
  an empty registry file behind.

`.specify/memory/constitution.md` and `specs/` are preserved. The guide says so,
and a v0.15.0 → v0.16.2 run that did use `init --here --force` left the
constitution unchanged. Older revisions of that doc said the opposite, so a
search result claiming otherwise is likely quoting a stale copy.
