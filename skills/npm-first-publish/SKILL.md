---
name: npm-first-publish
description: Publish a package to npm for the first time and set up tokenless releases from GitHub Actions with trusted publishing (OIDC). Use when a package has never been on npm, when moving a release workflow off NPM_TOKEN, or when a CI publish fails with EOTP, ENEEDAUTH, E401 or "OIDC token exchange error - package not found".
---

# First npm publish with trusted publishing

npm is phasing out tokens that bypass 2FA, so CI must not publish with one. Instead:

1. Stage the first version, so the package exists on npm.
2. Configure a trusted publisher, which lets the GitHub workflow publish through OIDC.
3. Let CI release from then on, with no npm token.

A trusted publisher can only be added to a package that already exists, so the stage step can't be skipped.

The package owner (a human) does every step that needs 2FA: `npm login`, `npm stage approve`, `npm trust`, and settings on npmjs.com. Write their commands out for them to run. A `!` prompt in Claude Code can't wait for a browser 2FA approval, so the commands need `--otp=<code>` from their authenticator app, or have to be run in a normal terminal. Never print, commit or paste a token.

## 1. Check the name and the scope

- Check the name is free with `npm view <name>`. An E404 means it's free. Unscoped names are often taken, so prefer `@<scope>/<name>`.
- Check who owns the scope with `npm org ls <scope>` while logged in. The owner's npm user must be an owner or member of that org. A personal scope (`@<username>`) works too.
- `package.json` needs:
  - the scoped `name`;
  - `"publishConfig": { "access": "public" }`, because scoped packages are private by default;
  - correct `main`/`module`/`types`/`exports` pointing at built files that exist;
  - `"repository": { "type": "git", "url": "git+https://github.com/<owner>/<repo>.git" }`. **Trusted publishing requires this to match the GitHub repo exactly**, and npm uses it for provenance and the npm page's link.
  - `homepage` and `bugs` are worth adding while you're there.
- Run `npm pack --dry-run` and check the file list and the version.

## 2. Stage the first version (owner)

```bash
npm login                      # once; opens the browser
npm run build                  # if the tarball ships built files
npm stage publish              # no 2FA needed; the version is hidden until approved
npm stage list <name>          # note the stage id
```

This should create the package on npm, and the package's **Settings → Trusted publishing** section then appears.

Then either:
- approve it, with `npm stage approve <stage-id> --otp=<code>`, to make the version public now; or
- reject it, with `npm stage reject <stage-id> --otp=<code>`, and let CI publish the first real version. A staged version takes its number, so reject it if CI is about to release that same version.

**Only the publish route below has been tested on a brand-new package.** npm's `npm stage` docs list "Package must exist" as a prerequisite. If staging refuses, publish the first version by hand from a clean checkout of the release tag:

```bash
npm publish --otp=<code>   # session login with 2FA, which is still allowed
```

Observed on a real first publish (November 2026): for a new name, the registry first adds a stub version `0.0.0-stage` ("Temporary package placeholder for staged publishing", by the publishing user). The real version appears a minute or two later, after "Your package is being processed". The stub is expected and harmless: it isn't on any dist-tag, and `latest` points to the real version. Settings → Trusted publishing is available once the package exists.

## 3. Configure the trusted publisher (owner)

On npmjs.com, go to the package → Settings → Trusted publishing → GitHub Actions. Or use the CLI, which avoids typos:

```bash
npm trust github <name> --repo <owner>/<repo> --file <workflow>.yml --allow-publish --otp=<code>
npm trust list <name> --otp=<code>     # check every field
```

- **Workflow filename** is the file name only, with the exact extension: `release.yml` and `release.yaml` are different. A mismatch shows up only at publish time, as `404 OIDC token exchange error - package not found`.
- **Environment:** leave it empty unless the job uses a GitHub environment.
- **Allow `npm publish`:** entries created after September 2026 allow only `npm stage publish` by default.
- **Fields are case-sensitive, and entries can't be edited.** To change one, delete it and add it again, or rename the workflow file to match it.
- **The first publish through a new entry must succeed within 2 days,** or the entry expires. Push a release soon after creating it.

## 4. Release workflow

```yaml
permissions:
  contents: write        # tags, release commit, GitHub release
  issues: write          # semantic-release comments and labels
  pull-requests: write
  id-token: write        # required for trusted publishing

jobs:
  release:
    runs-on: ubuntu-latest          # GitHub-hosted runners only
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
          token: ${{ secrets.RELEASE_TOKEN }}   # only if a protected branch needs a bypass
      - uses: actions/setup-node@v6
        with:
          node-version: 24          # needs Node >= 22.14 and npm >= 11.5.1
          # no registry-url (see below)
      - run: npx semantic-release   # or npm publish
        env:
          GITHUB_TOKEN: ${{ secrets.RELEASE_TOKEN }}
          # no NPM_TOKEN
```

- **Don't set `registry-url` on `setup-node` when semantic-release publishes.** It writes an `.npmrc` that reads `NODE_AUTH_TOKEN`, which hides the auth semantic-release sets up. The result is `401 Unauthorized - GET /-/whoami`.
- **`@semantic-release/npm` 13 and later handle OIDC themselves.** The log shows "OIDC token exchange with the npm registry succeeded". The repo must be public for npm to add provenance, which it then does automatically.
- **Remove `NPM_TOKEN`** from the workflow and the repo secrets, and revoke the token on npm.
- Once a CI release has gone through, the owner can set Publishing access to "Require two-factor authentication and disallow tokens".

## 5. `RELEASE_TOKEN` (semantic-release on a protected branch)

The default `GITHUB_TOKEN` can't push past a ruleset, so semantic-release needs the owner's token.

- **Fine-grained token:** scope it to the one repo, with Contents, Issues and Pull requests set to Read and write.
- **The token's owner must be on the ruleset's bypass list.**
- **semantic-release also pushes `refs/notes/semantic-release-v<version>`.** Some tokens are rejected there with only `(failed)`. Before a real release, test with a throwaway ref: push `refs/notes/token-check`, then delete it.

## When a release stops halfway

semantic-release pushes the release commit, then the tag, then the notes ref, and only then publishes to npm and creates the GitHub release. If it fails after the tag:

- **A rerun releases nothing.** It treats the tag as an already finished release.
- **If the notes ref is missing,** push it: `git notes --ref semantic-release-v<version> add -m '{"channels":[null]}' v<version>`, then `git push origin refs/notes/semantic-release-v<version>`.
- **Publish by hand from that tag.** Use a clean checkout (`git worktree add --detach <dir> v<version>`) and the `dist/` committed at the tag. Don't rebuild: a Windows build changes line endings. The owner runs `npm publish --otp=<code>` there.
- **Create the GitHub release** with the release commit's body as notes: `gh release create v<version> --notes-file <file>`.
- **Or leave that version unpublished.** The next release will contain the same changes.

## Errors

| Error | Cause |
|---|---|
| `E401` on `whoami` in CI | `setup-node` `registry-url` hiding auth, or a bad `NPM_TOKEN` |
| `EOTP` in CI | Publishing with a token that needs 2FA. Switch to trusted publishing. |
| `ENONPMTOKEN` after "OIDC token exchange … failed" | The trusted publisher doesn't match the run |
| `404 OIDC token exchange error - package not found` | Workflow filename, repo or owner differs from the `npm trust list` entry |
| `ENEEDAUTH` | Missing `id-token: write`, a self-hosted runner, or a filename mismatch |
| `Unable to get ACTIONS_ID_TOKEN_REQUEST_URL` | Missing `id-token: write`. Harmless while still using a token. |
| `remote rejected refs/notes/...` | `RELEASE_TOKEN` can't push notes refs |
