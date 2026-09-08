# lightsystem-plugins

A personal Claude Code marketplace. Adding it makes its plugins installable; it installs nothing
on its own.

```sh
claude plugin marketplace add LightSystem/lightsystem-plugins
claude plugin install reviewkit@lightsystem-plugins
```

Add `--scope project` to either command to record it in the repository's `.claude/settings.json`,
so a fresh clone picks it up without per-developer setup.

## Plugins

| plugin | what it is |
|---|---|
| [`reviewkit`](plugins/reviewkit) | A review round over three detectors — `/simplify`, the CodeRabbit CLI, IntelliJ's inspections — merged, deduplicated and verified before anything is applied. |
