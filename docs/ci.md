# CI and releases

Everything a new project needs for its builds, its variables and its releases.

## After creating a repository from the template

GitHub copies the files of a template, not its settings. **Environments, secrets
and branch rules do not come with it**, so do this once in every new repository.

**1. Protect the environments.** A workflow that names a missing environment makes
GitHub create it with no protection, and the job then runs without asking. Create
both with you as a required reviewer:

```sh
OWNER_REPO=$(gh repo view --json nameWithOwner --jq .nameWithOwner)
REVIEWER_ID=$(gh api user --jq .id)

for ENVIRONMENT in pr-review release; do
  echo "{\"reviewers\":[{\"type\":\"User\",\"id\":$REVIEWER_ID}]}" |
    gh api -X PUT "repos/$OWNER_REPO/environments/$ENVIRONMENT" --input -
done
```

The same page is Settings → Environments. On a private repository, required
reviewers depend on your GitHub plan; check it there.

**2. Add the secrets**, only if the app uses the updater: see
[Variables and secrets](#variables-and-secrets).

## Workflows

| Workflow                                        | Trigger          | Approval    | Does                                                           |
| ----------------------------------------------- | ---------------- | ----------- | -------------------------------------------------------------- |
| `pr-quality-check.yml` → `reusable-quality.yml` | PR to `main`     | none        | lint, format check, typecheck, Vitest, `cargo fmt/clippy/test` |
| `pr-build.yml`                                  | PR to `main`     | `pr-review` | builds the Windows and Linux bundles as artifacts              |
| `artifact-comment.yml`                          | after a PR build | none        | comments the download links on the PR                          |
| `release.yml`                                   | tag `v*`         | `release`   | builds every platform and drafts a GitHub release              |

Jobs are skipped when nothing relevant changed. The path filters live in
`.github/filters.yml`; `pr-build.yml` runs when `js` or `rust` matched, and always
when started by hand. Every new push to a PR cancels the running build and waits
for approval again.

To approve a run, open it in the Actions tab and press **Review deployments**.

### Platforms

The Windows and Linux runners are listed in `matrix.platform` in `pr-build.yml` and
`release.yml`, and their bundles in `bundle.targets` in `tauri.conf.json`
(`nsis`, `msi`, `deb`, `appimage`). Tauri builds only the targets of the runner it
is on. System packages for Linux are installed by
`.github/actions/setup-dependencies`.

macOS is not set up: add `macos-latest` to the matrices, `dmg` and `app` to the
targets, and the commented-out step in the action.

## Variables and secrets

| Name                 | Kind   | Where                   | Needed for                                    |
| -------------------- | ------ | ----------------------- | --------------------------------------------- |
| `GITHUB_TOKEN`       | secret | provided by GitHub      | every workflow; nothing to set                |
| `TAURI_PRIVATE_KEY`  | secret | environment `release`   | updater only: the key from `tauri signer`     |
| `TAURI_KEY_PASSWORD` | secret | environment `release`   | updater only: that key's password             |
| `VITE_*`             | env    | `.env`, or the workflow | frontend values, only if the app defines them |

`release.yml` passes the two secrets to Tauri as `TAURI_SIGNING_PRIVATE_KEY` and
`TAURI_SIGNING_PRIVATE_KEY_PASSWORD`. Without the updater they can stay unset.

A `VITE_*` value is built into the frontend, so **it must never hold a secret**:
anyone can read it in the bundle. Locally it goes in `.env`, copied from
`.env.example` and ignored by git. For a CI build, add it to the `env` of the build
step. A secret of the app belongs in the Rust side, and not in the repository.

## Making a release

1. Set the same version in `package.json`, `src-tauri/tauri.conf.json` and
   `src-tauri/Cargo.toml`, and merge that to `main`. `npm install` and a Cargo
   build refresh the lockfiles.
2. Tag `main` and push the tag:

   ```sh
   git checkout main && git pull
   git tag -s v1.0.0 -m "My App 1.0.0"
   git push origin v1.0.0
   ```

3. Approve the run on the `release` environment.
4. Download the bundles from the draft release, try them, and publish it.

`release.yml` stops at once if the tag is not `v` plus the version in
`tauri.conf.json`: otherwise the release would carry files of another version.

To replace a wrong tag before publishing, cancel its run, then:

```sh
git tag -d v1.0.0
git push origin :refs/tags/v1.0.0
```

### Tags

- `git tag v1.0.0` is a plain pointer. GitHub shows the status of the commit under
  it, so after a merge on GitHub it reads as verified, but it does not prove who
  placed the tag.
- `git tag -s v1.0.0 -m "..."` is signed with your GPG key and proves it.
- `git tag -a` has a message and **no signature**: GitHub shows it as unsigned, even
  on a signed commit. Avoid it.

## Two kinds of signing

They are separate, and the template sets up neither.

**The updater signature** proves an update came from you. It is free and needed
only for in-app updates.

1. `npm run tauri add updater`, and add `updater:default` to
   `src-tauri/capabilities/default.json`.
2. `npx tauri signer generate -w ~/.tauri/my-app.key`
3. In `tauri.conf.json`:

   ```json
   {
     "bundle": { "createUpdaterArtifacts": true },
     "plugins": {
       "updater": {
         "pubkey": "the contents of my-app.key.pub",
         "endpoints": [
           "https://github.com/OWNER/REPO/releases/latest/download/latest.json"
         ]
       }
     }
   }
   ```

4. Add `TAURI_PRIVATE_KEY` and `TAURI_KEY_PASSWORD` to the `release` environment.

`release.yml` already uploads `latest.json`. Keep the private key safe: **without it
the installed apps can never be updated again.** The dev config turns the updater
artifacts off, so local dev builds need no key.

**Windows code signing** removes the SmartScreen "unknown publisher" warning. It
needs a paid certificate or a signing service: see
[the Tauri guide](https://tauri.app/distribute/sign/windows/). Without it the
installer works, and users get the warning.
