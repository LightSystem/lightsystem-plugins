# lightsystem-plugins

A personal Claude Code marketplace. Adding it makes its plugins installable; it installs nothing
on its own.

```sh
claude plugin marketplace add LightSystem/lightsystem-plugins
claude plugin install reviewkit@lightsystem-plugins
```

Add `--scope project` to **both** commands to record the marketplace and the plugin in the
repository's `.claude/settings.json`, so a fresh clone picks them up without per-developer setup.
One without the other half-configures the project: the marketplace add writes
`extraKnownMarketplaces`, the install writes `enabledPlugins`, and a fresh clone needs both.

## Plugins

| plugin                           | what it is                                                                                                                                                    |
|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [`reviewkit`](plugins/reviewkit) | A review round over three detectors — `/simplify`, the CodeRabbit CLI, IntelliJ's inspections — merged, deduplicated and verified before anything is applied. |
