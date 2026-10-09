---
name: "review-round"
description: "Run a full review round — the bundled /simplify angles, a local CodeRabbit CLI review, and IntelliJ's inspection engine — concurrently over the same scope, then merge, dedup and verify every finding before acting on it."
compatibility: "The companion `coderabbit-triage` skill is **required as a document** — this skill defers to it for the operating constraints, the verification altitudes, the FIX/SKIP/FLAG classification and the report piles, none of which depend on its lane running. The three lanes are each optional: that skill running the CodeRabbit CLI (on PATH and authenticated), Claude Code's bundled `/simplify`, and IntelliJ IDEA with its MCP server connected (`mcp__idea__*`). If a lane is unavailable, say so in the report and run the rest — never silently drop one."
metadata:
  author: "João Horta Alves"
user-invocable: true
disable-model-invocation: false
---

## User Input

```text
$ARGUMENTS
```

Scope only, passed through to every lane. Step 1 maps each accepted value to a file list;
anything not in that table has no defined effect, so say so rather than improvising one.

## The companion skill

`coderabbit-triage` is required as a **document**, whether or not Lane A runs: read
`${CLAUDE_PLUGIN_ROOT}/skills/coderabbit-triage/SKILL.md` before step 1. Every reference to it
below points at that file. Where a step delegates to it rather than quoting it, its Skill-tool
name is `reviewkit:coderabbit-triage` — plugin skills register under their plugin's prefix.

## Repository notes

If `REPO-NOTES.md` sits at the repository root, **read it first** — Lane C needs it before it
runs, and steps 3 to 5 all depend on it. Its headings are its own index; this skill deliberately
does not restate them, because a copy of that list here would be one more thing to keep current.
It owns the repository's own facts, and `coderabbit-triage` reads the same file, which is why
neither skill states them itself.

If it is absent, recover what you can and say what you could not: the validation commands come
off the build files (step 5), and the generated-input ordering off the build file's generator
task and the package manifest's pre-hooks. What the inspection profile deliberately leaves
enabled cannot be recovered that way — it is empirical, built by running rounds. Treat every
Lane C finding as new, and offer to write the file at the end of the round.

## Goal

One review round, three detectors, one merged set of piles.

Running them separately means reconciling three reports by hand, and they have genuinely
different reach: `/simplify` reads intent, CodeRabbit sees cross-file consistency, IntelliJ resolves
types.

## Operating constraints

`coderabbit-triage` "Operating Constraints" is the canonical list and applies here unchanged.
Two additions this skill needs:

- **Three detectors, not one.** Each produces confident, well-formatted findings that are
  routinely wrong, and the value is in verifying them, not applying them. Two lanes agreeing is
  signal, not proof — verification is per finding, never per lane, and never inherited from a
  second reporter.
- **Fix early, wherever the finding is.** A verified finding gets the same FIX/SKIP/FLAG
  treatment whether or not its file is in the scope — correcting it now is cheaper than carrying
  it, and a deferred cleanup rarely gets done. Don't defer one to a separate `chore` commit or
  recommend splitting commits; the commit shape is the user's call. The minimal-changes bar still
  applies: fix the named defect, just not only in scoped files.

## Execution

### 1. Resolve the scope once, for all three lanes

Settle the index first, then resolve the scope — `coderabbit-triage` step 1 owns the
staged-and-modified artifact, why it matters, and the rule that the index is the user's to
change. It matters here because clearing it by committing moves those files out of the
working-tree scopes entirely, so the file list below must be resolved **after** that decision,
not before.

The base branch is `main`, assumed rather than resolved — `coderabbit-triage`'s **User Input**
states that assumption and is where a repository with a different name changes it.

| scope       | file list                                                                                                                               |
|-------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| *(nothing)* | `git diff --cached --name-only`, `git diff --name-only`, `git ls-files --others --exclude-standard`                                     |
| `branch`    | `git diff main...HEAD --name-only`, `git diff --cached --name-only`, `git diff --name-only`, `git ls-files --others --exclude-standard` |
| `committed` | `git diff main...HEAD --name-only`                                                                                                      |
| a path      | the `branch` row's commands, each with `-- <path>`                                                                                      |

A path does not combine with `branch` or `committed`; pass one scope.

**These lists must match what each lane actually reviews.** They are written to agree with the
scope coverage `coderabbit-triage` documents for the same four names — that skill owns the
mapping from a scope to its flags, what each one covers, and which scopes reach untracked files
at all. If the two ever disagree, it is
correct and this table is the one to fix, because a mismatch shows up as Lane A's cross-check
reporting absences that are not real.

Deduplicate the list: a file that is both staged and modified must appear once. Record the file
count and the insertion/deletion stat.

### 2. Launch all three lanes concurrently

The two slow lanes start first and the IntelliJ sweep fills the wait.

**Lane A — CodeRabbit.** Hand the scope from step 1 to `coderabbit-triage` and let it run its
step 1. That skill owns everything about the tool — how it is invoked, which flags each scope
maps to, and how to tell a completed review from a skipped or failed one. Do not restate any of
it here. Two things this lane adds, because the sibling is written to run the review in the
foreground as a whole loop:

**Run it in the background with its output captured to files**, so Lanes B and C are not
blocked, and apply the sibling's completion check to that captured output. If the review did not
complete, re-run **once**, then report Lane A as failed per the compatibility header rather than
looping.

**Cross-check the files the review reports against step 1's file list.** Only this skill can do
that, because only this skill resolves one scope for three lanes. The sibling explains why a
file's presence in that list proves nothing; a *missing* expected file is the signal, and a
skipped run carries no such list at all. A disagreement is a scope defect —
resolve it before merging findings, rather than triaging a lane that looked at the wrong thing.

**Lane B — `/simplify`.** Invoke it through the Skill tool as `simplify`, with the resolved scope
from step 1. It ships compiled into the `claude` executable, so it has no filesystem presence and is
often missing from the session's available-skills listing even while it works. Both come back empty
whether or not it is available, so neither emptiness is evidence and invoking it is the only test —
see the not-run rule below. Reference it by contract, never by internals, and do not fork it into
the repository — its agent count and angle names are upstream's to change.

**Lane C — IntelliJ.** Run in the foreground while A and B work. Batch the changed files:

```text
mcp__idea__lint_files
  files: <the resolved list from step 1, project-relative>
  projectPath: <repo root>        # always pass it; it avoids ambiguous-project calls
  min_severity: "warning"
```

Omit any path `REPO-NOTES.md` records as excluded from the IDE project; it says why.

Paths are relative to the IDE project root, which is the repo root only if the project was
opened there — check before assuming the two lists are interchangeable. Batch in groups rather
than one call per file. Three re-check triggers in the response: a `notAnalyzedReason` means the
file was not analysable, a top-level `more: true` means the batch was truncated, and
**`timedOut: true` on a file entry means it was not analysed at all** — that one is dangerous,
because the file comes back with an empty `problems` list and reads as clean.
Re-call a timed-out file on its own, raising `timeout`.

All three read a **file entry**, and `items` carries one only for a file that produced problems —
a clean file is simply absent, so absence fires none of them and means nothing on its own.
`get_file_problems` answers per call and names the file either way: run it over the silent ones to
turn their absence into a result, **passing `errorsOnly: false`** — it reports errors alone by
default, so a plain call answers `errors: []` for a file carrying warnings and quietly grants the
sweep less than it asked for. Even flagged it settles reach, not analysis: an IDE-excluded file
answers `errors: []` exactly as a clean one does.

This lane earns its slot because it **resolves types**, which the other two do not. Its
characteristic catch is a declaration that is dead only under resolution — an import whose
symbol appears in the file but resolves to something else, which no text-level linter can
distinguish from a live one.

**Lane B has no "reviewed nothing" tell of its own** — an empty return can mean it found
nothing, that the scope reached it wrong, or that a sub-agent died, so confirm what it actually
examined before recording it as clean. A and C have theirs above. A lane that cannot be shown to
have run is reported as not run.

**Known noise.** The committed inspection profile suppresses it, so this lane reports what
survives it; `REPO-NOTES.md` records what the profile deliberately leaves enabled, and why. It
also records why this lane is not replaceable by a linter here.

**A claim in that file contradicted by what you actually see gets corrected in place**, before
the round continues rather than noted in the report afterwards, because a stale notes file
silently degrades every future round.

### 3. Merge, dedup, then verify

Collect all three lanes before acting on any of them, so a finding is fixed once and in the
right place.

**Dedup** on the mechanism, not the wording. Two lanes describing the same line, or the same
cause at two call sites, is one finding. Note when more than one detector independently found
it; that is signal, though not proof. Where two lanes propose **incompatible** corrections to
the same line and both survive verification, that is FLAG by definition — it is exactly
`coderabbit-triage` step 3's "two or more legitimate approaches with different trade-offs", and
not a call to make silently. That step owns the whole FIX / SKIP / FLAG classification.

**Then verify each survivor against the current code.** `coderabbit-triage` step 2 owns the
altitude list — which claims to chase to a dependency's own artifact, which to a config file,
which to a regenerated one — and it applies unchanged, as does its rule that severity labels do
no triage work in either direction — including its generated-types clause, which is the one that
fails silently. `REPO-NOTES.md` names this repository's commands and their ordering.

Its same-class sweep applies here too, scoped to the round's file list — label anything it
turns up as reviewer-missed, and classify it like any other finding.

### 4. Apply, propagate, encode

Apply per `coderabbit-triage` step 4 — the smallest edit that corrects the defect, propagate the
fact everywhere it is stated, and leave a short why-note where the corrected thing is subtle or
was wrong once already.

Then feed the durable ones back, per `coderabbit-triage` step 5 — it owns the earning criteria,
the "write the fact, not an instruction to be quiet" rule, the guard against encoding away a
*valid* finding, and the validation command. Two routing rules are this skill's own:

- A recurring IntelliJ false positive goes to the committed inspection profile, not to
  `REPO-NOTES.md` and not to `.coderabbit.yaml`. Per-developer state is versionable too, so reach
  for configuration before prose: `.idea/*` plus a negation carries machine-independent IDE
  config, and `externalDependencies.xml` carries a required plugin. Documenting noise that
  committed configuration could suppress is the failure mode, not the fallback.
- **Check whether the change invalidates the review config itself.** An instruction that
  enumerates paths — which files are hand-authored, which are generated, which are excluded —
  goes stale the moment a file is added or renamed, and the next review then runs under a false
  premise. `REPO-NOTES.md` names this repository's instances.

### 5. Validate

Proportionate to what changed, using the repo's own checks. `REPO-NOTES.md` names them; if it is
absent, read them off the build file, package manifest, or CI config rather than guessing.

Two traps that are not specific to any stack:

- **Confirm the checks actually ran.** An incremental build system will report success without
  re-running anything when it believes its inputs are unchanged. A cached green is not a green
  — check for an up-to-date/skipped marker, or compare a result artifact's timestamp against
  your edit, and force a re-run if in doubt.
- **Generated inputs must be regenerated first**, and note which commands do this for you.
  A check that silently reads a stale generated artifact will pass while proving nothing.

Finish with `git diff HEAD --stat` **and** `git ls-files --others --exclude-standard`. Not a
bare `git diff --stat` — it shows unstaged changes only, so it reports nothing on a staged tree,
which step 1 can legitimately produce.

Report what you ran and what it said. If something fails, say so with the output.

### 6. Report

`coderabbit-triage` step 7 defines the piles — Fixed, Skipped, Flagged, Encoded — and how much
evidence each carries. Report **one merged set** of them, not three reports stapled together.
Three additions:

- **Outside the scope** — mark which Fixed findings landed in files the scope did not cover.
- **Attribution** — note which lane found a finding where that is interesting, and say when
  more than one found it independently.
- **Coverage** — state which lanes ran and which did not; step 7 already requires the scope. A
  clean round means nothing without both.
