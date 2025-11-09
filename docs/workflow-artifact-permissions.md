# GitHub Actions: Workflow Artifact Permissions

## Issue: Cross-Workflow Artifact Download Failures in Private Repositories

### Problem Description

When using the `workflow_run` trigger to download artifacts from another workflow, you may encounter the following error in private repositories:

```
Error: Unable to download artifact(s): Resource not accessible by integration
```

### Root Cause

The behavior differs between public and private repositories due to GitHub's security model:

#### Public Repositories
- The default `GITHUB_TOKEN` has sufficient permissions to access artifacts across workflow runs
- `actions/download-artifact@v4` can successfully download artifacts from the triggering workflow using the `run-id` parameter
- Cross-workflow artifact access works out of the box

#### Private Repositories
- The default `GITHUB_TOKEN` has more restricted permissions for security reasons
- Cross-workflow artifact access requires additional permissions that the default token doesn't have by default
- The `actions/download-artifact@v4` action cannot access artifacts from different workflow runs
- This restriction prevents unauthorized access to private repository artifacts

### Why "Resource not accessible by integration" Error Occurs

The `actions/download-artifact@v4` action uses the `run-id` parameter to download from another workflow run. In private repositories, the default `GITHUB_TOKEN` doesn't have the necessary `actions: read` permission scope to access artifacts from different workflow runs, resulting in the permission error.

## Solutions for Private Repositories

If you need to download artifacts across workflows in a private repository, use one of these solutions:

### Solution 1: Use Third-Party Action (Recommended)

Use `dawidd6/action-download-artifact@v6` instead of the official action:

```yaml
- name: Download version info
  uses: dawidd6/action-download-artifact@v6
  with:
    name: version-info
    run_id: ${{ github.event.workflow_run.id }}
    github_token: ${{ secrets.GITHUB_TOKEN }}
```

This action uses the GitHub API differently and can properly access artifacts from other workflow runs, even in private repositories.

### Solution 2: Use Personal Access Token (PAT)

Create a Personal Access Token with `repo` scope and add it as a repository secret:

1. Go to GitHub Settings → Developer settings → Personal access tokens
2. Generate a new token with `repo` scope
3. Add it as a repository secret (e.g., `PAT_TOKEN`)
4. Use it in the workflow:

```yaml
- name: Download version info
  uses: actions/download-artifact@v4
  with:
    name: version-info
    github-token: ${{ secrets.PAT_TOKEN }}
    run-id: ${{ github.event.workflow_run.id }}
```

### Solution 3: Use Workflow Outputs Instead of Artifacts

Instead of passing data via artifacts, use workflow outputs to pass data between workflows. This approach doesn't require artifact downloads but has limitations on data size.

### Solution 4: Make Repository Public

If the repository doesn't contain sensitive information, making it public will allow the default `GITHUB_TOKEN` to access artifacts across workflows.

## Our Current Setup

### Workflows

1. **build.yml** - Builds the application and uploads version info as an artifact
2. **deploy.yml** - Automatically deploys after successful build (downloads version artifact)
3. **deploy-manual.yml** - Manual deployment with user-specified version (no artifact needed)

### Current Configuration

Our deploy workflow currently uses:

```yaml
- name: Download version info
  uses: actions/download-artifact@v4
  with:
    name: version-info
    github-token: ${{ secrets.GITHUB_TOKEN }}
    run-id: ${{ github.event.workflow_run.id }}
```

**Status:** Works with public repositories only.

**For private repositories:** Update to use `dawidd6/action-download-artifact@v6` as shown in Solution 1.

## References

- [GitHub Actions: workflow_run event](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_run)
- [GitHub Actions: Permissions](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token)
- [dawidd6/action-download-artifact](https://github.com/dawidd6/action-download-artifact)
