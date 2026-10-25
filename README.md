# renovate-config

Shared [Renovate](https://docs.renovatebot.com) presets for xiphux's repositories.

| Preset | Use |
| --- | --- |
| `default.json5` | Everything every repository shares: cooldowns, the auto-merge policy, the weekly timetable and its groups, lockfile maintenance, vulnerability alerts. |
| `python.json5` | Opt-in, for uv-managed Python (pyproject.toml + uv.lock): `update-lockfile` for pep621, and ruff's 0.x minors trusted to CI. |

The rationale for each setting is in the comments beside it.

## Using it

A repository's `.github/renovate.json5` extends the presets and keeps only what is its own:

```json5
{
  $schema: 'https://docs.renovatebot.com/renovate-schema.json',
  extends: ['github>xiphux/renovate-config:default.json5'],
  commitMessagePrefix: 'chore(deps):',
  prConcurrentLimit: 4,
  packageRules: [
    {
      matchManagers: ['npm'],
      matchDepTypes: ['devDependencies'],
      commitMessagePrefix: 'chore(deps-dev):',
    },
  ],
}
```

Not in the preset, because repositories differ: `commitMessagePrefix` (with its dev-dependency counterpart) and `prConcurrentLimit`.

## How a repository's config combines with the presets

- Presets apply in `extends` order, then the repository's own settings. A plain setting in the repository replaces the preset's.
- `packageRules` are appended rather than replaced: the presets' rules come first, the repository's after. Where two matching rules set the same option, the later one wins.

So a repository customizes by adding rules:

- **Add a package to a shared group:** a rule with the same `groupName` and `schedule`, naming the extra packages. Group lists in the preset name packages a given repository may not use; that is deliberate, so a package lands in the same group everywhere.
- **Let a 0.x package's minors auto-merge:** `matchUpdateTypes: ['minor']`, `matchCurrentVersion: '/^v?0\\.[^-]*$/'`, `automerge: true`. Use that pattern rather than `/^v?0\\./`: the repository's rule lands after the preset's "never auto-merge prereleases" rule (`matchCurrentVersion: '/-/'`), and the looser pattern would re-enable auto-merge for a 0.x prerelease.
- **Turn something off:** a later rule with `enabled: false`, or `automerge: false`.

Note that `/-/` also matches Docker tags such as `26-alpine`, so a repository auto-merging Dockerfile digest updates needs its rule to come after it, which a repository rule always does.

## Runner labels in a matrix

Renovate's github-actions manager updates `runs-on` labels but not runner labels listed in a matrix. Put `# renovate: github-runner` on the line above each one, written one per line (a block list, not `[a, b]`), and the preset's custom manager moves it in the same pull request as the `runs-on` labels.

## Things that stay in each repository

- `hostRules` for private registries, such as the `@xiphux` scope on GitHub Packages. The token comes from a per-repository Renovate secret (`{{ secrets.NPM_PKG_GITHUB_TOKEN }}`), so the rule belongs where that secret is.
- Each repository's package-manager cooldown (pnpm's `minimumReleaseAge`, uv's `exclude-newer`). Keep it at or below the shortest age here (3 days, for patches), or Renovate's own pull requests fail to install.

## Changing a preset

A change here reaches every repository on its next Renovate run, with no pull request in those repositories. Validate first:

```sh
npx --package renovate -- renovate-config-validator --strict default.json5 python.json5
```

To stage a risky change, publish it as a new named preset (or pin `extends` to a tag, `github>xiphux/renovate-config:default.json5#v2`) and move repositories over one at a time.
