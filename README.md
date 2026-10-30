# lightsystem-plugins

A personal Claude Code marketplace. Adding it makes its plugins installable; it installs nothing
on its own.

```sh
claude plugin marketplace add LightSystem/lightsystem-plugins
claude plugin install reviewkit@lightsystem-plugins
claude plugin install sddkit@lightsystem-plugins
```

Add `--scope project` to the marketplace add **and** each install to declare them in the
repository's `.claude/settings.json`. One without the other half-configures the project: the
marketplace add writes `extraKnownMarketplaces`, the install writes `enabledPlugins`, and a fresh
clone needs both. Declaring is not installing: Claude Code loads a declared plugin only in a
checkout it was installed in (`claude plugin list --json` shows each install's `projectPath`), so
each clone still needs the project-scope install run in it once per machine (a git worktree shares
its main checkout's). Until then the plugins appear only after `/reload-plugins`, and only for
that session.

Neither command turns on updates. A third-party marketplace does not auto-update by default, so an
installed plugin stays on the copy it was installed at until
`claude plugin update <name>@lightsystem-plugins`. Add `"autoUpdate": true` beside `source` in the
`extraKnownMarketplaces` entry, or turn on **Enable auto-update** for the marketplace in
`/plugin`'s **Marketplaces** tab.

## Plugins

| plugin                           | what it is                                                                                                                                                    |
|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [`reviewkit`](plugins/reviewkit) | A review round over three detectors — `/simplify`, the CodeRabbit CLI, IntelliJ's inspections — merged, deduplicated and verified before anything is applied. |
| [`sddkit`](plugins/sddkit)       | Spec Kit upgrades for spec-driven projects: the CLI, then the integration files and extensions, with template edits treated as drift to reset.                |
