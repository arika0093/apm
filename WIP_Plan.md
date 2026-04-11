# Issue 636 plan

## Problem

APM already parses full `http://` Git URLs, and the manifest schema documentation
already lists them as valid inputs. However, the install and validation flows
rebuild remote URLs as `https://`, so `apm install http://git.server.internal.com/user/repo`
fails against HTTP-only Git servers.

## Current state

- `DependencyReference.parse()` accepts `http://` inputs.
- `DependencyReference.to_canonical()` strips transport details, so explicit
  `http://` intent is not preserved after canonicalization.
- `GitHubPackageDownloader._build_repo_url()` and `build_https_clone_url()`
  always generate `https://` for non-SSH clone paths.
- `_validate_package_exists()` uses `git ls-remote` against the rebuilt URL for
  generic hosts, so the failure happens before the actual install.
- `resolve_git_reference()`, `list_remote_refs()`, and clone/download paths all
  depend on the same URL-building flow, so the fix must be end-to-end.
- Virtual package validation and download still use HTTPS raw/API fetches in
  `download_raw_file()` and `validate_virtual_package_exists()`.
- Issue #636 is now scoped to whole-repository installs for generic HTTP hosts;
  virtual packages are explicitly out of scope.
- Docs currently drift from runtime behavior:
  - `docs/src/content/docs/reference/manifest-schema.md` already documents
    `http://...git` as valid.
  - `packages/apm-guide/.apm/skills/apm-usage/dependencies.md` does not explain
    HTTP-only host behavior or any security limits.

## Proposed approach

1. Security and transport contract:
   - Decision confirmed: if the user explicitly provides `http://...`, honor it
     for generic Git hosts without requiring a separate manifest flag.
   - Keep shorthand and FQDN forms on their current secure defaults.
   - Do not send APM-managed GitHub or ADO tokens over HTTP.
2. Preserve explicit transport metadata in dependency parsing for remote URL
   inputs so downstream validation and clone flows can distinguish `http` from
   `https` and `ssh`.
3. Update URL builders and repository access flows to honor the stored scheme
   for generic hosts:
   - `src/apm_cli/models/dependency/reference.py`
   - `src/apm_cli/utils/github_host.py`
   - `src/apm_cli/deps/github_downloader.py`
   - `src/apm_cli/commands/install.py`
4. Keep auth boundaries intact:
   - GitHub, GHE, and ADO remain HTTPS/token-managed through `AuthResolver`.
   - Generic HTTP hosts continue to rely on git-native auth and credential
     helpers rather than APM embedding tokens.
5. Leave virtual packages out of scope for this issue:
   - Do not add a generic-host fallback for `download_raw_file()` or virtual
     package validation.
   - Keep existing virtual package behavior unchanged on hosts that already
     expose supported raw/API endpoints.
6. Add regression coverage for:
   - parsing and canonical behavior for `http://` URLs
   - repo URL construction in downloader helpers
   - `git ls-remote` validation for generic HTTP hosts
   - clone/reference resolution paths that currently rewrite to HTTPS
   - no new virtual package behavior for generic HTTP hosts
   - no regression for GitHub and GHE HTTPS raw/API behavior
7. Update docs and release notes to match the final behavior:
   - `docs/src/content/docs/reference/manifest-schema.md`
   - `packages/apm-guide/.apm/skills/apm-usage/dependencies.md`
   - `CHANGELOG.md`

## Notes

- No README change is expected unless the final behavior diverges from the
  current public positioning.
- Scope confirmed: Issue #636 should cover whole-repository installs over
  HTTP-only generic hosts only; virtual packages are excluded.
