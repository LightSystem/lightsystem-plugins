# Repository notes

Facts a reviewer cannot derive from the source. `review-round` and `coderabbit-triage` both read
this file; the headings below are its index.

## Validation

There is no build system and nothing is generated, so there is no ordering to respect and no
step that regenerates an input on your behalf. Run what the change touches:

```sh
claude plugin validate . --strict                    # the marketplace manifest
claude plugin validate ./plugins/<name> --strict     # a plugin manifest, and every skill in it; one per plugin touched
cr config validate                                   # .coderabbit.yaml
jq empty <file>                                      # any JSON edited
```

`claude plugin validate` covers the manifest **and** the skills, agents and commands in the
directory — a `SKILL.md` with missing or malformed frontmatter is reported (verified by probe:
stripping the frontmatter produced `frontmatter: No frontmatter block found`). Prose changes are
still validated by reading: grep for the phrasing the edit should have superseded, and open both
ends of every cross-reference you touched. A claim about what a consumer gets is validated by probe
where its precondition holds (for a fresh clone, a directory with no install record), never in this
checkout, which is already installed and passes whatever the claim says.

When anything under `plugins/` changed, check that it did not outrun its plugin's `version` (see
[Install scope](#install-scope) for why that matters):

```sh
for p in plugins/*/; do
  m="$p.claude-plugin/plugin.json"
  git diff --quiet HEAD -G'"version"' -- "$m" || continue   # bump already pending
  v=$(git log -1 --format=%h -G'"version"' -- "$m")        # commit that last changed version
  git diff --quiet "$v" -- "$p" ":!$m" ":!${p}README.md" ||
    echo "$p changed since its last version bump ($v)"
done
```

It compares against the commit that last moved `version`, not against `main`, so it also catches a
bump missed in a commit that has already landed — which a diff-scoped review cannot see. It does
not see untracked files; `git add -N` them first. A plugin's `README.md` is excluded because Claude
Code never loads it — a bump there would change nothing a consumer runs. Drop the exclusion if a
component ever starts reading it.

**Stale green.** None of these validators is incremental — each re-reads its file every run and
there is no cache, no daemon and no up-to-date marker to be fooled by. A green here is a real
green. Two things can still mislead:

- Without `--strict`, `claude plugin validate` prints `✔ Validation passed with warnings` and
  exits 0 on a genuinely broken skill. Pass `--strict` — that is why it is in the commands above.
- `cr config validate` rejects unknown **top-level** keys but accepts a misspelled key nested
  inside a recognised section, so re-read an entry you added rather than trusting the tick.

## Known noise

Nothing is suppressed, and there is no committed inspection profile. That is deliberate rather
than pending.

`MarkdownIncorrectTableFormatting` is kept **enabled**. Every table in this repository is padded
to equal column width by hand, so the inspection is load-bearing: when it fires, a table edit
left the columns ragged. Re-pad the whole table — do not dismiss it, and do not disable it. The
cost is that a one-word change to a long cell re-pads every row of that table; the widest are 196
characters (`README.md`) and 235 (`plugins/reviewkit/README.md`).

If a genuinely useless inspection ever starts recurring, it goes in a committed
`.idea/inspectionProfiles/` profile, not in this file — see the routing rule in `review-round`
step 4. That needs a two-step `.gitignore` negation, because a negation cannot reach into an
ignored directory.

## Versioned IDE config

`.idea/*` is denied wholesale; a file is negated only once proven machine-independent and not
rewritten by the IDE import.

- **Tracked:** `modules.xml`, `lightsystem-plugins.iml` — both `$PROJECT_DIR$`-relative with no
  absolute paths and no toolchain versions.
- **Deliberately not tracked:** `misc.xml` pins whatever JDK this machine has (`JDK_25`);
  `vcs.xml` is regenerated on open and carries nothing a clone needs; `.idea/.gitignore` is
  redundant with the deny-all above it.

No path is excluded from the IDE project, so Lane C reviews the resolved file list unfiltered.
The IDE project root **is** the repository root — the `.idea/` directory sits here, so
project-relative and repo-relative paths are interchangeable.

## Configuration that enumerates paths

Both go stale the moment a plugin is renamed, and neither failure is loud — the next
review simply runs under a false premise. Check them against any rename:

- `.coderabbit.yaml` — one `path_instructions` entry is scoped to `plugins/reviewkit/skills/**`.
  The entry globbing `plugins/*/skills/**` needs no upkeep.
- `.claude-plugin/marketplace.json` — each entry carries a `source` path and a `homepage` URL that
  restates it. The `name` shared with `plugin.json` is required on both sides, not drift: the
  entry's is the install and enable key, the manifest's namespaces the plugin's components.

## Install scope

Plugins listed in `enabledPlugins` in `.claude/settings.json` are declared at **project** scope;
a clone still needs its own install before they load (see the README); verified by probe in a
directory holding these settings and no install record. The marketplace source is
`github:LightSystem/lightsystem-plugins` — the published repository, not this working copy. A
session in this repo therefore runs an installed copy of a *released* version, never uncommitted or
unpushed skill edits — and not necessarily the latest release either. The entry sets
`"autoUpdate": true`; keep it, since this repository is a consumer too and a third-party
marketplace does not auto-update by default (see the README). Even so, that refresh runs in the
background up to ten minutes after a session's first message, and a running session keeps the
version it loaded until `/reload-plugins` or a restart. To test a skill change, start a session
with `claude --plugin-dir ./plugins/<name>`: the local copy replaces the installed one for that
session (verified by probe — one listing, the local description). Bump and push only to ship.

**A skill change ships only with a version bump.** Claude Code decides a consumer's copy is out of
date by comparing `version` in `plugin.json`, so a commit that leaves it unchanged never reaches a
consumer's cached copy. Bump it in the same change as the skill edit; the check under
[Validation](#validation) catches a missed one, including one already committed. Leaving `version`
out would make each commit a release instead, and was declined: `claude plugin validate --strict`
warns on a missing version, so the strict run the commands above depend on would fail permanently.

Not enabled, deliberately:

- `sddkit` — this repository does not use Spec Kit, so it has nothing to upgrade.
