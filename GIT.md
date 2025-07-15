# 📘 Git Commands Cheat Sheet

A comprehensive reference guide for Git operations from basic to advanced.

## Table of Contents
[1. Repository Setup](#1-repository-setup)
[2. Basic Operations](#2-basic-operations)
[3. Branching & Switching](#3-branching--switching)
[4. History & Changes](#4-history--changes)
[5. Sync Operations](#5-sync-operations)
  - [Pull](#pull)
  - [Push](#push)
  - [Interactive Rebase](#interactive-rebase)
[6. Reset & Revert](#6-reset--revert)
[7. Merge & Rebase](#7-merge--rebase)
[8. Advanced Operations](#8-advanced-operations)
[9. Comparing Changes](#9-comparing-changes)
[10. Configuration](#10-configuration)
[11. Stashing Changes](#11-stashing-changes)
[12. Tagging](#12-tagging)
[13. Commit Message Tags](#13-commit-message-tags)


## SSH Key Setup
Generate an SSH key for secure authentication with Git servers:
```bash
# Generate SSH key
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
# Add SSH key to the agent
ssh-add ~/.ssh/id_rsa
# Start the SSH agent
eval "$(ssh-agent -s)"
```

## 1. Repository Setup

```bash
# Initialize a new Git repository (locally)
git init <repository-name>

# Rename the default branch to main (if not already set)
git branch -M main

# Add a remote repository
git remote add origin <repository-url>

# Push to the remote repository
git push -u origin main

# Clone an existing repository from a remote server
git clone <repository-url>

# Clone with a specific branch
git clone -b <branch-name> <repository-url>
```

## 2. Basic Operations

```bash
# Check the status of your working directory
git status

# Add files to staging area
git add <file-name>       # Add specific file
git add .                 # Add all files in current directory

# Commit staged changes
git commit -m "Descriptive commit message"
```

## 3. Branching & Switching

```bash
# List all branches
git branch                # Local branches
git branch -r             # Remote branches
git branch -a             # All branches (local + remote)

# Create a new branch
git branch <branch-name>

# Create and switch to a new branch
git checkout -b <branch-name>

# Switch to an existing branch
git checkout <branch-name>

# Switch to a specific commit (detached HEAD state)
git checkout <commit-hash>
```

## 4. History & Changes

```bash
# View commit history
git log                   # Full history with details
git log --oneline         # Compact history (one line per commit)
git log --graph --oneline # Visual branch history
git log -p                # Show patches (changes) in each commit
git log -n 5              # Show only the last 5 commits

# View changes
git diff                  # Between working directory and staging
git diff --staged         # Between staging and last commit
git diff <commit1>..<commit2>  # Between two commits
```

## 5. Sync Operations

### Pull

When you need to update your local branch with remote changes:

```bash
# Basic scenario: Remote has new commits
Remote (origin/main): A---B---C
Local (main):         A---B

# Standard pull (fetch + merge)
git pull origin main
# Local (main):   	 A---B---C
```

```bash
# Complex scenario: Remote has changes and local has diverged
Remote (origin/main): A---B---C
Local (main):         A---B---D---E

# Pull with rebase (avoid merge commit)
git pull --rebase origin main
# Local (main):       A---B---C---D'---E'

# Set tracking for easier future pulls
git branch --set-upstream-to=origin/main main

# After setting upstream, you can simply use:
git pull --rebase

# Handling rebase conflicts
git rebase --abort     # Cancel the rebase operation
git rebase --continue  # Continue after resolving conflicts
git rebase --skip      # Skip the current commit in the rebase
```

### Push

```bash
# Push your changes to remote
git push origin <branch-name>

# Set upstream and push (for future pushes use just 'git push')
git push -u origin <branch-name>

# Push all branches
git push --all origin

# Push all tags
git push --tags origin

# Force push (⚠️ CAUTION: overwrites remote history)
git push --force origin <branch-name>

# Safer force push (rejects if others updated the branch)
git push --force-with-lease origin <branch-name>
```

### Interactive Rebase

For cleaning up commits before pushing:

```bash
# Start an interactive rebase - opens editor
git fetch origin                # Update remote refs
git rebase -i origin/main       # Interactive rebase against main

# In the editor, you'll see something like:
# pick abc123 First commit
# pick def456 Second commit
# pick ghi789 Third commit

# Available commands:
# p, pick   = use commit
# r, reword = use commit, but edit the commit message
# e, edit   = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup  = like "squash", but discard this commit's message
# d, drop   = remove commit

# Conflict resolution during rebase
git checkout --ours <file>    # Keep changes from current branch
git checkout --theirs <file>  # Keep changes from branch being rebased onto
git rm <file>             	  # Remove conflicting file from index
git add .                     # Stage resolved conflicts
git rebase --continue         # Continue after resolving
git push -u origin main --force  # Push rebased branch (force required)
```

## 6. Reset & Revert

```bash
# Reset options (moving HEAD and branch pointer)
git reset --soft <commit>   # Reset to commit, keep changes staged
git reset --mixed <commit>  # Default: Reset to commit, keep changes unstaged
git reset --hard <commit>   # Reset to commit, DISCARD ALL CHANGES

# Example: Reset workflow
# Before (A → B → C → D → E, HEAD at E)
#     A---B---C---D---E (HEAD -> master)

git reset --hard B
# After:
#     A---B (HEAD -> master)
#         \
#          C---D---E (unreferenced, will be garbage collected)

# Revert (safer alternative to reset - creates new commit)
git revert <commit>       # Create a new commit undoing specified commit
git revert <commit>..<commit>  # Revert a range of commits
```

## 7. Merge & Rebase

### Merge

Combines two branches with a merge commit:

```bash
# Merge another branch into current branch
git checkout main
git merge feature-branch

# Before merge:
# main:          A---B---C
# feature:            \
#                      D---E

# After merge:
# main:          A---B---C---F
#                          /
# feature:            D---E

# F is a merge commit containing both histories
```

### Rebase

Re-applies your commits on top of another branch:

```bash
# Rebase current branch onto another branch
git checkout feature-branch
git rebase main

# Before rebase:
# main:          A---B---C
# feature:            \
#                      D---E

# After rebase:
# main:          A---B---C
# feature:                \
#                          D'---E'

# D' and E' are new commits with the same changes as D and E
```

## 8. Advanced Operations

### Making an Earlier Commit the New Master

```bash
# Initial scenario: We deployed commits D and E to production, but they introduced bugs
#
# main:      A---B---C---D---E  (HEAD -> main, origin/main)
#                    ^
#            Last known good state
#
# Goal: Return to the state at commit C, when everything worked correctly
```

#### Option 1: Revert (Non-destructive)

```bash
# Identify the problematic commit
git log --oneline

# Create a new commit that undoes the problematic commit
git revert <bad-commit-hash>

# Revert a range of commits
git revert <first-bad-commit>..<last-bad-commit>

# Push the reverted changes
git push origin main

# After revert operations:
#
# main:      A---B---C---D---E---F  (HEAD -> main, origin/main)
#                                ^   
#                     		Revert D & E
#
# F is a new commit that undoes the changes made by D and E
# Original commits remain in history, but their effects are undone
# No force push needed - safest option for shared repositories
```

#### Option 2: Branch from Earlier State (⚠️ Rewrites history)

```bash
# Find the last good commit
git log --oneline

# Create a new branch from that commit
git checkout <good-commit-hash>
git checkout -b fixed-branch

# After checkout and branch creation:
#
# main:      A---B---C---D---E  (origin/main)
#                    ^
# fixed-branch:      C  (HEAD -> fixed-branch)

# Make any additional changes if needed
git commit -am "Additional fixes"

# Replace the main branch
git branch -D main           # Delete local main
git branch -m fixed-branch main  # Rename the fixed branch to main

# After adding fixes and replacing main:
#
# Old main:  A---B---C---D---E  (unreachable)
#                      \
# New main:            C---F  (HEAD -> main)
#                          ^
#                     Additional fixes

# Update remote (⚠️ Destructive)
git push --force origin main     

# After force push:
#
# main:      A---B---C---F  (HEAD -> main, origin/main)
#
# Commits D and E are entirely removed from visible history
# ⚠️ WARNING: Never do this on shared branches without coordinating with your team.
```

## 9. Comparing Changes

```bash
# Compare two commits
git diff <commit1>..<commit2>

# Compare specific file between commits
git diff <commit1>..<commit2> -- <path/to/file>

# Compare with previous commit
git diff <commit>^..<commit>

# Show changes in a commit
git show <commit>

# Show changes with context
git log -p <commit1>..<commit2>

# See what changed in specific files
git log -p <commit1>..<commit2> -- <path/to/file>
```

## 10. Configuration

```bash
# Set your identity
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Set default editor
git config --global core.editor "code --wait"  # For VS Code

# Set default branch name for new repositories
git config --global init.defaultBranch main

# Configure line endings
git config --global core.autocrlf true  # For Windows
git config --global core.autocrlf input # For macOS/Linux

# List all settings
git config --list

# Enable colorful output
git config --global color.ui auto
```

## 11. Stashing Changes

```bash
# Save changes without committing
git stash save "Work in progress"

# List stashes
git stash list

# Apply most recent stash and keep it in stash list
git stash apply

# Apply specific stash
git stash apply stash@{2}

# Apply and remove most recent stash
git stash pop

# Remove specific stash
git stash drop stash@{0}

# Create a branch from a stash
git stash branch <branch-name> stash@{0}
```

## 12. Tagging

```bash
# Create a lightweight tag
git tag v1.0.0

# Create an annotated tag
git tag -a v1.0.0 -m "Version 1.0.0 release"

# List all tags
git tag

# Show tag details
git show v1.0.0

# Push specific tag to remote
git push origin v1.0.0

# Push all tags
git push origin --tags

# Delete a local tag
git tag -d v1.0.0

# Delete a remote tag
git push origin --delete v1.0.0
```

## 13. Commit Message Tags

| Type       | Description                               | Example                                                    |
| ---------- | ----------------------------------------- | ---------------------------------------------------------- |
| `feat`     | New feature or capability                 | `feat(api): add rate-limiting middleware`                  |
| `fix`      | Bug fix                                   | `fix(login): correct redirect on auth failure`             |
| `docs`     | Documentation only                        | `docs(readme): update installation guide`                  |
| `style`    | Code style (whitespace, formatting, etc.) | `style: fix indentation in script.js`                      |
| `refactor` | Code change **without behavior change**   | `refactor(service): simplify logic using ternary operator` |
| `perf`     | Performance improvements                  | `perf(db): optimize query with index`                      |
| `test`     | Adding or updating tests                  | `test(utils): add unit tests for formatDate`               |
| `build`    | Build system or dependency changes        | `build: upgrade webpack to v5`                             |
| `ci`       | CI/CD configuration changes               | `ci: add GitHub Actions workflow for testing`              |
| `chore`    | Other changes (tooling, maintenance)      | `chore: update .gitignore and bump pre-commit hooks`       |
| `revert`   | Revert a previous commit                  | `revert: feat(auth): add login throttling`                 |
