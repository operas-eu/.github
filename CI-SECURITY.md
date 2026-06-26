# CI/CD Security Standards

How we secure GitHub Actions and GitLab CI across the EDCH and OPERAS repositories, what the automated checks enforce, and what you need to do when you write or change a workflow.

This is the canonical reference, published in `operas-eu/.github` (OPERAS, the parent organisation) and mirrored to `EUDCH/.github`. A shorter context brief for the wider OPERAS team lives in Confluence (ITSUPPORT) and links back here.

## Why

CI/CD pipelines have become a primary target for supply-chain attacks. A workflow runs with credentials and often with write access to the repository, so a single compromised third-party action or a leaked token can be enough to push malicious code or exfiltrate secrets. Two recent incidents shaped this work:

- **tj-actions/changed-files (2025):** a widely used action was compromised and made to print CI secrets into build logs. Repositories that pinned the action by tag (`@v44`) silently pulled the malicious version; repositories that pinned by commit SHA did not.
- **Shai-Hulud (2025):** a self-propagating npm worm that harvested tokens from CI environments. Fresh package and action releases are the highest-risk window.

In parallel, vulnerability coordination is shifting in Europe: ENISA now operates the EU Vulnerability Database (EUVD) and acts as a CVE root for EU CNAs, alongside the historically US-run CVE program. Tracking dependency vulnerabilities against both is becoming the baseline expectation.

The goal of these standards is to make the common failure modes structurally impossible or, where that is not possible, to make them fail the build so they get caught in review rather than in production.

## The checks we run

The tooling is chosen per CI platform.

### GitHub Actions

- **zizmor** — static analysis for GitHub Actions workflows (unpinned actions, credential persistence, template injection, dangerous triggers, over-broad permissions). Run through a single org-shared reusable workflow (see "Adopting zizmor" below) so the configuration is defined once.
- **actionlint** — workflow syntax and expression checking. Runs inside MegaLinter where present.
- **MegaLinter** — multi-linter that also bundles dependency-vulnerability scanners (osv-scanner, grype, trivy) and secret scanners (gitleaks, trufflehog). Used on repositories that already run it.
- **OSV-Scanner** — standalone dependency-vulnerability scanner for repositories that do not run MegaLinter. Reads the lockfile directly and needs no GitHub Advanced Security.
- **Dependabot** with a release-age cooldown — automated dependency and action updates, delayed by a few days so a compromised fresh release is less likely to be applied automatically.

### GitLab CI

- **poutine** — static analysis for GitLab CI pipelines (planned; the GitLab repositories are the next track). zizmor is GitHub-only, so poutine plus a manual checklist is the floor there.

## Workflow standards

These apply to every GitHub Actions workflow in scope. zizmor enforces most of them.

1. **Pin actions by commit SHA, with an exact-version comment.**
   ```yaml
   - uses: actions/checkout@9c091bb21b7c1c1d1991bb908d89e4e9dddfe3e0 # v7.0.0
   ```
   The SHA is what actually runs (a tag can be moved to malicious code; a SHA cannot). The `# vX.Y.Z` comment is read by Dependabot, which keeps the pin updated.

2. **Set least-privilege permissions.** Declare `permissions:` explicitly rather than relying on the default token scope. Grant only what the job needs (often `contents: read`).

3. **Disable credential persistence on checkout** unless the job genuinely needs to push:
   ```yaml
   - uses: actions/checkout@<sha> # vX.Y.Z
     with:
       persist-credentials: false
   ```
   The exception is a job that commits back (for example MegaLinter auto-fix in commit mode), which needs the persisted credentials. Mark that case explicitly, with a justification on the ignore:
   ```yaml
   - uses: actions/checkout@<sha> # vX.Y.Z
     with:
       # commit-mode auto-fix pushes its changes back using these credentials
       persist-credentials: true # zizmor: ignore[artipacked]
   ```

4. **Use `pull_request`, not `pull_request_target`.** `pull_request_target` runs with the base repository's token and is safe only if it never checks out the pull request's code. Plain `pull_request` gets a non-privileged token on forks and checks out the proposed change, which is what you usually want. Only reach for `pull_request_target` with a clear, commented reason.

5. **Use `GITHUB_TOKEN`, not a personal access token.** A long-lived PAT is a standing liability if leaked. `GITHUB_TOKEN` is scoped to the run and expires with it. (One consequence: `GITHUB_TOKEN` cannot push changes to files under `.github/workflows/`. See Operational notes.)

6. **Never interpolate context into a shell `run:` block.** Pass values through `env:` and reference them as shell variables, so an attacker-controlled value cannot break out into the command:
   ```yaml
   - name: Print PR URL
     env:
       PR_URL: ${{ steps.cpr.outputs.pull-request-url }}
     run: echo "URL - $PR_URL"
   ```

7. **Enable a Dependabot cooldown** so a yanked or compromised release is not applied automatically:
   ```yaml
   cooldown:
     default-days: 7
   ```
   zizmor's `dependabot-cooldown` audit requires at least 7 days, so Dependabot uses 7. The cooldown governs only routine version bumps: security fixes arrive from GitHub's advisory feed as separate Dependabot security updates and are not held by it. (The self-hosted Renovate on GitLab uses a 1-day release age instead, paired with OSV fast-tracking security fixes past the gate. The two tools encode different judgments on the cooldown length, so the value differs by platform; the principle, a cooldown for routine bumps plus separate fast-tracking of security fixes, is the same.)

## Adopting zizmor

zizmor is added to a repository by calling the reusable workflow published by that repository's own organisation. Each organisation keeps its own copy of the reusable workflow in its `.github` repository:

- OPERAS repositories call `operas-eu/.github`.
- EDCH repositories call `EUDCH/.github`.

OPERAS is the parent of the EDCH, so OPERAS repositories must not depend on the EDCH org for their tooling. The two copies start identical and can diverge over time if an org's needs differ.

The caller is a small file at `.github/workflows/zizmor.yml` (substitute your own org):

```yaml
name: zizmor
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
permissions:
  contents: read
  security-events: write
jobs:
  zizmor:
    uses: operas-eu/.github/.github/workflows/zizmor.yml@<sha> # operas-eu/.github#<n>
    with:
      advanced-security: false
```

The reusable workflow is pinned by commit SHA. The org `.github` repository is not release-tagged, so the pin carries a `# operas-eu/.github#<n>` reference to the pull request that set it, rather than the `# vX.Y.Z` comment used for released actions.

The caller grants `security-events: write` because the reusable workflow declares that permission, and a caller that grants less than a reusable workflow declares fails the call. The permission is exercised only on the `advanced-security: true` path (the SARIF upload to the Security tab); on `advanced-security: false` it is granted but unused. Tightening this further would be a change to the reusable workflow, not the caller.

Two inputs control behaviour:

- **`advanced-security`** — set `true` on public repositories (and any repository with GitHub Advanced Security): findings are uploaded as SARIF to the Security tab, which is report-only by default. Set `false` on private repositories without Advanced Security: findings appear as inline annotations and the check fails on any finding, because there is no Security-tab sink, so this path blocks by nature.
- **`enforce`** — relevant on the `advanced-security: true` path, where findings are otherwise report-only. Setting `enforce: true` makes any finding fail the check, taking that repository from "report" to "blocking". Genuine exceptions are marked inline with `# zizmor: ignore[<rule>]` and a justifying comment.

The recommended path depends on the mode:

- **Public (`advanced-security: true`):** add the caller report-only, clear the baseline findings shown in the Security tab, then set `enforce: true`.
- **Private (`advanced-security: false`):** the check blocks on any finding from the start, so clear the baseline findings in the adopting pull request itself.

### If the repository also runs MegaLinter

MegaLinter (9.5.0 and later) bundles its own copy of zizmor. Left enabled, it scans the workflows a second time under a different configuration, which gives inconsistent results and a duplicate failure. Disable the bundled copy so the dedicated caller is the single source of zizmor coverage:

```yaml
# .mega-linter.yml
DISABLE_LINTERS:
  - ACTION_ZIZMOR
```

Also remove any `ACTION_ZIZMOR_UNSECURED_ENV_VARIABLES` entry: it exists only to let the bundled copy reach the GitHub API, and it is not needed once the bundled copy is off. MegaLinter's other security scanners (osv-scanner, grype, trivy) and actionlint keep running normally; only the redundant bundled zizmor is removed.

## What this means when you contribute

- **A failing `zizmor` (or MegaLinter) check is blocking** on repositories that have enforcement on. The annotation or Security-tab entry names the rule and the line; fix it or, if it is a justified exception, add an inline `# zizmor: ignore[<rule>]` with a reason.
- **New actions must be SHA-pinned** with a version comment. Dependabot handles updates afterwards.
- **Keep workflow YAML formatted.** Where MegaLinter auto-fixes formatting in commit mode, it cannot push changes to workflow files (the token has no `workflows` scope), so an unformatted workflow file will block the run rather than be auto-fixed. In practice: inline comments take a single space before `#`, and the formatter is prettier (the version MegaLinter ships).

## Operational notes

- **`GITHUB_TOKEN` cannot modify `.github/workflows/`.** This is a GitHub restriction (only a PAT with the `workflow` scope can). Because we deliberately do not use PATs, any automation that would commit a change to a workflow file is blocked. Keep workflow files clean by hand.
- **Pull-request checks scan only the diff; the push to `main` scans everything.** A finding in an unchanged file can therefore surface only after merge, on the full-codebase run. When hardening a repository, expect the first push to `main` to reveal pre-existing findings the PR checks did not see.
- **MegaLinter's bundled scanners.** Where a repository runs MegaLinter, its bundled dependency-vulnerability scanners (osv-scanner, grype, trivy) are used as-is. zizmor is the exception: it runs through the dedicated reusable workflow, and the bundled copy is disabled (see "If the repository also runs MegaLinter" above), so the configuration lives in one place for the whole org.

## Rollout

Adoption is staged. The reusable zizmor workflow is live and several GitHub Actions repositories are already on blocking enforcement; the remaining GitHub repositories, then the GitLab CI repositories (poutine) and the Forgejo repositories, follow. The detailed per-repository state is tracked internally.

New repositories adopt the standard in three steps: add the zizmor caller in report mode, clear the baseline findings, then flip to blocking.

## References

- zizmor documentation: https://docs.zizmor.sh
- OSV-Scanner: https://google.github.io/osv-scanner/
- EU Vulnerability Database (EUVD): https://euvd.enisa.europa.eu
- GitHub: security hardening for GitHub Actions: https://docs.github.com/actions/security-guides/security-hardening-for-github-actions
