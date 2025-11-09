# Stacked Diff Workflow with Jujutsu

This guide explains how to craft and publish stacked diffs in this repo using [Jujutsu (jj)](https://github.com/martinvonz/jj) so that each diff can become its own pull request.

## 1. Prerequisites

1. Make sure `jj` is installed (`jj --version`).
2. Configure your identity once so commits can be pushed:
   ```bash
   jj config set --repo user.name "Your Name"
   jj config set --repo user.email "you@example.com"
   ```
3. Import the latest Git state into `jj` when you start a session:
   ```bash
   git fetch origin
   jj git import
   ```

## 2. Create the first change

1. Start a new change on top of `main` (or whatever base you need):
   ```bash
   jj new main -m "feat: something"
   ```
2. Edit files, then snapshot the change:
   ```bash
   jj status       # inspect
   jj commit -m "feat: something"  # or jj commit -i for partials
   ```
3. Give it a bookmark that will become the PR branch name:
   ```bash
   jj bookmark create pr/my-feature -r @
   ```

## 3. Stack a second change on top

1. Create another change using the first one as the parent (this is automatic when you are on the latest change):
   ```bash
   jj new -m "feat: follow-up"
   # edit files
   jj commit -m "feat: follow-up"
   jj bookmark create pr/my-follow-up -r @
   ```
2. Check the stack:
   ```bash
   jj log -r ::@    # shows base commit plus both stacked commits
   jj diff -r pr/my-feature-
   jj diff -r pr/my-follow-up-
   ```

## 4. Push stacked PR heads

1. Export jj commits to the Git store:
   ```bash
   jj git export
   ```
2. Push both bookmarks so that GitHub sees them as branches:
   ```bash
   jj git push --remote origin --bookmark pr/my-feature --bookmark pr/my-follow-up --allow-new
   ```
3. Verify the remote branches if desired:
   ```bash
   git ls-remote origin "refs/heads/pr/*"
   ```

## 5. Open the pull requests

1. **PR 1:** base `main`, compare `pr/my-feature`. This shows only the first diff.
2. **PR 2:** base `pr/my-feature`, compare `pr/my-follow-up`. GitHub now displays only the incremental diff because the base already includes PR 1.

## 6. Iterate on review

- Apply review feedback to the relevant commit by editing it directly:
  ```bash
  jj edit pr/my-feature
  # make fixes
  jj commit -m "feat: something"   # rewrites the same change-id
  jj git export
  jj git push --remote origin --bookmark pr/my-feature --allow-new
  ```
- Use `jj rebase -s 'descendants(pr/my-feature)' -d main` if `main` moves forward while PRs are open.

## 7. Land the stack

1. Merge PR 1 on GitHub (squash or rebase + fast-forward works best).
2. After PR 1 merges, retarget PR 2 to `main` or rebase it:
   ```bash
   jj rebase -s pr/my-follow-up -d main
   jj git export
   jj git push --remote origin --bookmark pr/my-follow-up
   ```
3. Once all PRs land, you can delete the bookmarks:
   ```bash
   jj bookmark delete pr/my-feature pr/my-follow-up
   git push origin --delete pr/my-feature pr/my-follow-up
   ```

Following these steps produces two clean, dependent pull requests derived from a JJ stacked diff.
