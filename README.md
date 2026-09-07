# @iarroyoescobar/eslint-config-inspector

> ⚠️ **This is a custom fork of [eslint/config-inspector](https://github.com/eslint/config-inspector). It is not recommended for general use unless you know exactly what you are doing.**

This package includes personal modifications on top of the original ESLint Config Inspector. Use the official package instead:

```bash
npx @eslint/config-inspector
```

## Custom Features

### Export Rules (CSV)

On the **Rules** page, an **Export** button lets you download a CSV report of the currently filtered rules. The report includes:

- Rule name, plugin, and severity level
- **Applied By** — the config name, or `anonymous #N (globs)` if the config has no name
- Whether the rule is fixable, recommended, or deprecated
- Rule description and docs URL

### Export ESLint Config (JSON)

On the **Configs** page, an **Export Config** button downloads the full resolved ESLint configuration as a JSON file (`eslint-config.json`). Each entry includes a 1-based index and a human-readable name (e.g. `anonymous #6`) so you can identify configs that appear without a name in the inspector.

## Usage

If you still want to use this custom version:

```bash
npx @iarroyoescobar/eslint-config-inspector
```

## License

[Apache-2.0](./LICENSE)
