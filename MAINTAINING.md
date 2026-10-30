# Maintaining the registry

## Requirements for a plugin

- A public GitHub repo, made from the
  [plugin template](https://github.com/Termix-SSH/Termix-Plugin-Template) or with the same
  `.github/workflows/release.yml` (it records the build provenance the checks require).
- `manifest.json`: `repository` set to that repo, `engine.api` set, an id that is not an official
  Termix plugin's.
- `npm test`, `npm run typecheck`, `npm run validate`, `npm run build` pass.

The plugin repo needs **no secrets**: this registry signs the reviewed file.

## Release a version (in the plugin repo)

1. Work on a `dev-X.Y.Z` branch; `manifest.json`, `package.json` and the top of `CHANGELOG.md` say `X.Y.Z`.
2. The workflow files must already be on `main` (a `workflow_dispatch` workflow only runs from there,
   and a release that changes `.github/workflows` needs a `TERMIX_PAT` secret).
3. Actions → **Release** → branch `dev-X.Y.Z`, channel `stable` (or `beta`), mode `dry-run` first,
   then `release`. It publishes `v X.Y.Z` with `<id>-X.Y.Z.tmxplug`, merges the branch into `main` and
   deletes it.
4. Copy the `sha256` from the run summary (or the release asset's digest).

## Add a new plugin

Open a PR here that adds `snytexx/plugins/<id>.json`:

```json
{
  "$schema": "../../community-submission-schema.json",
  "id": "<id from manifest.json>",
  "repository": "https://github.com/sNyteXx/termix-plugin-<name>",
  "maintainers": ["sNyteXx"],
  "versions": [{ "version": "1.0.0", "sha256": "<from the release>" }]
}
```

Add the plugin to the table in `README.md`.

## Add a version

Open a PR that adds the new version at the **top** of `versions`. Only a listed maintainer can do
that. A version already listed can never change its `sha256`; release a new version instead.

## Review and merge

The **Check plugin entry** job checks the sha256, the manifest (id, version, repository), the build
provenance from the Termix release workflow and the maintainer, and writes a summary with links to the
source at the tag, the changes since the last listed version and the capabilities (new ones marked).

Before merging, read the source at the tag, as for the community registry:

- it does what its name and description say, and nothing else
- no obfuscated or minified source, no code downloaded and run at runtime
- no telemetry or network calls the user did not ask for
- the fewest capabilities it needs; new ones are justified
- install and build scripts fetch or run nothing unexpected

Merging starts **Sync**, which signs the version and updates `snytexx/index.json`. Termix sees it within
15 minutes. Sync also runs every 6 hours and on demand (Actions → Sync → Run workflow).

## Remove a plugin or a version

Remove the version from `versions` (or delete the file) and merge. Sync drops it from the index.
Installed copies keep working; Termix just stops offering them.

## The signing key

- Public: `keys/snytexx.pub`, key id `dbd4cfde7466862c` (first 16 hex of the sha256 of the raw key).
- Private: only the Actions secret `TERMIX_SNYTEXX_SIGNING_KEY`. It cannot be read back from GitHub.

To replace it (lost or leaked):

```bash
npx --package @termix-ssh/plugin-sdk termix-plugin keygen --out /tmp/newkey
gh secret set TERMIX_SNYTEXX_SIGNING_KEY --repo sNyteXx/termix-registry < /tmp/newkey/termix-plugin-signing.key
shred -u /tmp/newkey/termix-plugin-signing.key
```

Put the new public key in `keys/snytexx.pub` and the README, then run Sync with `--full` (edit the
workflow's command once, or run `npm run sync -- --full` locally with the secret in the environment)
so every version is signed again. Termix never follows a key change on its own: admins remove the
registry and add it again, and plugins installed from it are uninstalled and reinstalled first.

## Keeping the scripts current

`scripts/` follows [Termix-Registry](https://github.com/Termix-SSH/Termix-Registry) (copied from
commit `68ee278`). When Termix changes its index format, compare `scripts/lib.mjs`, `sync.mjs` and
`validate.mjs` and the two schema files with upstream and take the changes over. The local changes
are only in `check-submission.mjs` (registry folder, official ids read from upstream) and the
workflows (runner, secret name).
