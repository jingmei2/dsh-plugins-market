# dsh-plugins-market

A plugin market index for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) bundles, in
[format 1](https://www.npmjs.com/package/@deepseek-ai/dsh-plugin-market-index).

The file `index.json` is the whole market. There is no server: DSH fetches this file, validates it with
`parseMarketIndex`, and turns each entry into a `plugin_manager` install target.

## Add it to DSH

Index URL:

```
https://jingmei2.github.io/dsh-plugins-market/index.json
```

In DSH: **Settings → Plugins → Discover → add index source**, paste the URL, then search for the plugin and
install it. Or programmatically:

```js
await pluginMarket.setIndexes(['https://jingmei2.github.io/dsh-plugins-market/index.json'])
```

`setIndexes` replaces the whole `indexes` list, so include any other sources you already configured.

## What is listed

| Plugin | What it does |
| --- | --- |
| `@jingmei2/dsh-session-bin` | Session bin: move a session to the trash bin from its menu, restore it, or delete it permanently. |

## Publishing a new version

1. Read `index.json` and add one object to the plugin's `versions` array:
   - `version` — the exact SemVer of the tagged commit,
   - `publishTimestamp` — integer Unix **seconds**, e.g. `git log -1 --format=%ct`,
   - `source.type` — `git` with `url` + lowercase 40-hex `commit`, or `npm` with an optional `registry`.
2. Validate before committing. `parseMarketIndex` is the rule set DSH runs, so a document that passes it cannot
   be rejected by the Host:

   ```bash
   node --input-type=module -e "
   import { parseMarketIndex, marketInstallTarget } from '@deepseek-ai/dsh-plugin-market-index'
   import { readFileSync } from 'node:fs'
   const index = parseMarketIndex(JSON.parse(readFileSync('index.json', 'utf8')),
     'https://jingmei2.github.io/dsh-plugins-market/index.json')
   for (const plugin of index.plugins)
     for (const version of plugin.versions)
       console.log(plugin.name, version.version, JSON.stringify(marketInstallTarget(plugin.name, version)))
   "
   ```

3. Commit and push. GitHub Pages redeploys in about a minute; `cacheTtlMs` in DSH is 10 minutes, so use
   **Refresh** in the Discover tab to bypass the Host's cached copy.

## Rules the validator enforces (selected)

- `format` must be `1`; `title` needs an `en` member; language ids are **lowercase** (`zh-cn`, not `zh-CN`).
- A plugin name must match the npm name grammar; a version must be exact SemVer, unique per plugin.
- `publishTimestamp` must be an integer Unix second in `[0, 10^11)`.
- A `git` source needs a canonical `https` repository URL (at least two path segments, no tarball) and a
  lowercase 40-hex commit; the install spec becomes `git+<url>.git#<commit>`.
- A relative `icon` resolves against the index URL and must stay on `https` or the index origin.
- Unknown members are dropped, so a typo in a field name fails silently by omission — validate, then look at
  the parsed output.
