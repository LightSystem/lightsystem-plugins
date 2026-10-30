# sddkit

Skills for projects that follow spec-driven development with [Spec Kit](https://github.com/github/spec-kit).

| skill                     | what it does                                                                                                                                         |
|---------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| `/sddkit:speckit-upgrade` | Upgrades the `specify` CLI, then refreshes the project's integration files and extensions, and verifies the result against a freshly rendered copy.  |

## Requirements

- The `specify` CLI on `PATH`.
- `gh`, authenticated, and `jq` — for reading every release note between the installed and latest
  versions.

## What a project commits to

`speckit-upgrade` keeps `.specify/templates/` byte-identical to upstream and treats any local edit
there as drift to reset, unless the user chooses to keep it. What a project needs to say belongs in
`.specify/memory/constitution.md`. A project that maintains customized templates wants a different
upgrade procedure.
