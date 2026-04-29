# Releasing `near-slip10`

Releases are automated by [release-plz](https://release-plz.dev/) and published
to [crates.io](https://crates.io/crates/near-slip10) via **OIDC trusted
publishing** — there is no `CARGO_REGISTRY_TOKEN` secret stored in this repo.

## One-time setup (maintainer only)

Before the first automated release can succeed, a crates.io owner of
`near-slip10` must register this repository as a Trusted Publisher:

1. Go to <https://crates.io/crates/near-slip10/settings> (you must be an owner
   of the crate).
2. Open the **Trusted Publishing** section and click **Add**.
3. Fill in:
   - **Repository owner:** `near`
   - **Repository name:** `near-slip10`
   - **Workflow filename:** `release-plz.yml`
   - **Environment:** *(leave blank — this workflow does not use a GitHub
     Environment)*
4. Save.

Notes:

- crates.io trusted publishing **cannot publish a crate for the first time** —
  the very first version of any new crate must still be uploaded with a
  conventional API token. `near-slip10` is already published, so this does not
  apply to us, but it matters if we ever spin out a new sibling crate.
- The first GitHub release after this PR merges will also need the workflow
  file (`release-plz.yml`) to already exist on the default branch with the
  exact filename registered above. Renaming the workflow file later requires
  re-registering the trusted publisher.

## Normal release flow

1. Land conventional-commit PRs into `master` (`feat:`, `fix:`, `chore:`, …).
2. On every push to `master`, the **Release-plz PR** job opens or updates a
   pull request titled `chore: release` that bumps the version and updates
   `CHANGELOG.md` based on the commits since the last release.
3. Review and merge the release PR.
4. On the resulting push to `master`, the **Release-plz release** job:
   - Authenticates to crates.io via OIDC (no secret needed).
   - Runs `cargo publish` for the new version.
   - Creates the matching git tag and GitHub Release.
   - Re-packages the crate locally and attaches a sigstore build-provenance
     attestation (`actions/attest-build-provenance`) to the workflow run. The
     attestation is verifiable with `gh attestation verify`.

## Emergency / manual release

If the workflow is broken and a release must go out immediately:

1. Bump `version` in `Cargo.toml` and update `CHANGELOG.md` by hand on a
   branch, open a PR, merge it.
2. From a clean checkout of `master` at the merge commit, run:

   ```sh
   cargo publish
   ```

   This requires a crates.io API token in `~/.cargo/credentials.toml` (or via
   `CARGO_REGISTRY_TOKEN`) for an account that is an owner of `near-slip10`.
3. Tag and push:

   ```sh
   git tag -a v$(grep '^version' Cargo.toml | head -n1 | cut -d'"' -f2) -m "Release"
   git push origin --tags
   ```

4. Create the GitHub release in the UI (or `gh release create`) pointing at
   the new tag.

A manual release skips the sigstore attestation step. Prefer fixing the
automation.

## Follow-ups

- The `release-plz/action` does not currently expose the path to the `.crate`
  artifact it published, so `release-plz.yml` re-runs `cargo package` locally
  to produce the file we attest. Cargo's `.crate` packaging is reproducible
  from the same commit, so the attestation still binds the published artifact
  to this workflow run, but if release-plz adds an output for the published
  artifact path we should switch to attesting that file directly. Tracking
  upstream: <https://github.com/release-plz/release-plz/issues/1029>.
