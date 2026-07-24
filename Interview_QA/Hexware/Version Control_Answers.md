# Version Control Interview Questions and Answers

## Git, GitHub, and Azure Repos Fundamentals

Below are detailed, interview-ready answers written from a **Senior DevOps Engineer’s perspective**. The explanations include practical commands, production considerations, and common interview follow-up points.

***

## 1. What is version control, and why is it important?

**Version control** is a system that records changes made to files over time. It enables developers and DevOps engineers to track modifications, collaborate safely, compare versions, identify who changed something, and restore an earlier version when required.

Version control is important because it provides:

* **Change history:** Every committed change is recorded with an author, timestamp, and message.
* **Collaboration:** Multiple developers can work on the same application without overwriting each other’s work.
* **Rollback capability:** Teams can restore a stable version if a deployment or code change fails.
* **Branching and isolation:** Features, fixes, and experiments can be developed independently.
* **Auditability:** Organizations can trace who changed what and why, which is especially important in regulated environments.
* **CI/CD integration:** Build and deployment pipelines are usually triggered by changes in version-control repositories.
* **Disaster recovery:** Repositories stored on remote platforms provide an additional copy of the source code and infrastructure definitions.

In DevOps, version control should not be limited to application code. It should also manage:

* Terraform and Bicep templates
* Kubernetes manifests
* Helm charts
* Ansible playbooks
* Pipeline definitions
* PowerShell and Bash scripts
* Configuration templates
* Operational documentation

> **Interview summary:** Version control is the system of record for software delivery. It provides traceability, collaboration, rollback, and automation capabilities across the development lifecycle.

***

## 2. What is Git, and how does it differ from older VCS tools such as CVS or SVN?

**Git** is a distributed version control system created to manage source-code changes efficiently. Each developer generally has a complete local copy of the repository, including its branches and commit history.

### Git versus CVS or SVN

| Area               | Git                                              | CVS/SVN                                        |
| ------------------ | ------------------------------------------------ | ---------------------------------------------- |
| Architecture       | Distributed                                      | Primarily centralized                          |
| Local repository   | Contains complete history                        | Usually contains only a working copy           |
| Offline operations | Most operations work offline                     | Many operations require server access          |
| Branching          | Lightweight and fast                             | Traditionally heavier                          |
| Commits            | Can be created locally                           | Usually sent directly to the central server    |
| Performance        | Most operations are local and fast               | Network dependency can affect performance      |
| Data tracking      | Snapshot-oriented                                | Primarily file and directory revision-oriented |
| Failure resilience | Every full clone can preserve repository history | Central server is more critical                |
| Integration model  | Supports merge requests and pull requests        | Traditionally uses centralized commits         |

Git stores project states as a series of snapshots. If a file has not changed, Git effectively references the previous stored version instead of creating an unnecessary duplicate.

A major advantage is that developers can perform operations such as the following without connecting to the remote server:

```bash
git status
git diff
git log
git branch
git commit
```

Remote access is required for operations such as:

```bash
git fetch
git pull
git push
git clone
```

> **Senior-level point:** Git is distributed, but most enterprises still adopt centralized governance through platforms such as GitHub, Azure Repos, GitLab, or Bitbucket. This combines Git’s distributed capabilities with access control, pull-request policies, auditing, and CI/CD integration.

***

## 3. What is the difference between a local repository and a remote repository?

A **local repository** exists on a developer’s workstation or build agent. It contains the working directory, staging area, local branches, commits, tags, and Git metadata.

A **remote repository** is hosted on a shared server or repository management platform, such as:

* GitHub
* Azure Repos
* GitLab
* Bitbucket
* An internally hosted Git server

### Local repository

A local repository allows you to:

* Modify files
* Stage changes
* Create commits
* Create and switch branches
* View history
* Merge or rebase branches
* Work without network connectivity

### Remote repository

A remote repository provides:

* Centralized team collaboration
* Backup and availability
* Pull-request or merge-request workflows
* Branch protection
* Access control
* Audit history
* CI/CD integration
* Security scanning and compliance policies

You can view configured remotes with:

```bash
git remote -v
```

A commonly used remote name is `origin`:

```bash
git remote add origin https://github.com/organization/project.git
```

To send local commits to the remote:

```bash
git push origin main
```

> **Important distinction:** A commit does not automatically reach GitHub or Azure Repos. `git commit` records the change locally, while `git push` transfers local commits to the remote repository.

***

## 4. What is a commit in Git?

A **commit** is an immutable Git object that records a snapshot of staged project changes at a particular point in time.

A commit contains:

* A unique SHA-based identifier
* Author information
* Committer information
* Date and time
* Commit message
* Reference to the project tree
* Reference to one or more parent commits

Example:

```bash
git add src/payment-service.cs
git commit -m "Add validation for payment request"
```

A commit ID may look like this:

```text
c3a7e30d9f6c4cde1692ca1c29b85cfbde234abc
```

The commit ID is calculated from commit-related content. Changing the commit changes its identifier.

### Characteristics of a good commit

A good commit should be:

* Small and logically focused
* Independently understandable
* Tested before being submitted
* Associated with a ticket or work item where required
* Described using a meaningful message
* Free from passwords, tokens, certificates, and secrets

Example of a meaningful commit message:

```bash
git commit -m "Fix timeout handling in payment API"
```

Less useful example:

```bash
git commit -m "changes"
```

> **Senior-level recommendation:** Use atomic commits. A single commit should represent one logical change so that it can be reviewed, reverted, or cherry-picked safely.

***

## 5. What is a branch in Git?

A **branch** is a movable reference to a commit. It provides an isolated line of development that allows engineers to work on features, bug fixes, releases, or experiments without directly changing the main branch.

Common branch names include:

```text
main
develop
feature/payment-validation
bugfix/login-timeout
release/2.5.0
hotfix/critical-auth-fix
```

Create a branch:

```bash
git branch feature/payment-validation
```

Create and switch to it:

```bash
git switch -c feature/payment-validation
```

A branch is lightweight because Git does not create a complete copy of the project. It creates a reference that points to a commit.

Branches support workflows such as:

* Feature branching
* Trunk-based development
* Git Flow
* Release branching
* Hotfix workflows

> **Senior-level recommendation:** Keep branches short-lived. Long-running branches increase merge conflicts, integration risk, and delayed feedback.

***

## 6. What is the staging area, or index, in Git?

The **staging area**, also known as the **index**, is an intermediate area between the working directory and the Git repository.

Git changes generally move through three principal states:

1. **Working directory:** Files are created or modified.
2. **Staging area:** Selected changes are prepared for the next commit.
3. **Repository:** Staged changes are saved as a commit.

Example:

```bash
git add app/config.yaml
git commit -m "Update application timeout configuration"
```

You can stage only part of a modified file:

```bash
git add -p app/config.yaml
```

This allows you to create focused commits even when a file contains multiple unrelated changes.

To remove a file from the staging area without deleting its modifications:

```bash
git restore --staged app/config.yaml
```

> **Interview summary:** The staging area allows engineers to control exactly which changes are included in the next commit.

***

## 7. What is the difference between `git add` and `git commit`?

`git add` copies selected changes from the working directory into the staging area.

`git commit` creates a repository snapshot from the changes currently in the staging area.

### Example

Modify two files:

```text
app.py
README.md
```

Stage only `app.py`:

```bash
git add app.py
```

Commit the staged change:

```bash
git commit -m "Add input validation"
```

In this example:

* The change in `app.py` is committed.
* The change in `README.md` remains uncommitted in the working directory.

Stage all changes under the current directory:

```bash
git add .
```

Stage tracked-file modifications and deletions:

```bash
git add -u
```

Review staged changes before committing:

```bash
git diff --staged
```

> `git add` prepares content for a commit. `git commit` records the prepared content in the local repository.

***

## 8. How do you initialize a new Git repository?

Use `git init` to initialize an existing directory as a Git repository.

```bash
mkdir payment-service
cd payment-service
git init
```

This creates a hidden `.git` directory containing Git metadata and object storage.

Check the repository:

```bash
git status
```

Add and commit initial files:

```bash
git add .
git commit -m "Initial project structure"
```

To initialize the repository with `main` as the initial branch:

```bash
git init -b main
```

Connect it to a remote repository:

```bash
git remote add origin https://github.com/organization/payment-service.git
git push -u origin main
```

The `-u` option establishes the upstream tracking relationship. Future pushes can normally use:

```bash
git push
```

> Do not modify or delete the `.git` directory manually. Removing it removes the repository’s local Git metadata and history references.

***

## 9. How do you clone a remote repository?

Use `git clone` to create a local copy of a remote Git repository.

### HTTPS example

```bash
git clone https://github.com/organization/payment-service.git
```

### SSH example

```bash
git clone git@github.com:organization/payment-service.git
```

### Azure Repos example

```bash
git clone https://dev.azure.com/myorganization/myproject/_git/payment-service
```

Clone into a specific local directory:

```bash
git clone https://github.com/organization/payment-service.git payments
```

Clone a specific branch:

```bash
git clone --branch release/2.5.0 --single-branch \
  https://github.com/organization/payment-service.git
```

Create a shallow clone containing limited history:

```bash
git clone --depth 1 https://github.com/organization/payment-service.git
```

Cloning generally:

* Downloads repository objects and history
* Creates a working directory
* Configures the source as the `origin` remote
* Creates remote-tracking references
* Checks out the default branch

> **DevOps consideration:** Shallow clones can improve CI pipeline performance, but they may not support operations requiring complete history, such as certain versioning, comparison, or security-analysis tasks.

***

## 10. What is `git status` used for?

`git status` displays the current state of the working directory and staging area.

```bash
git status
```

It shows:

* Current branch
* Upstream tracking status
* Untracked files
* Modified but unstaged files
* Staged changes
* Merge or rebase state
* Whether the local branch is ahead of or behind its upstream branch

A concise version is:

```bash
git status --short
```

Example short output:

```text
 M app.py
M  config.yaml
?? notes.txt
```

Interpretation:

* ` M app.py`: Modified in the working directory but not staged
* `M  config.yaml`: Modified and staged
* `?? notes.txt`: Untracked file

> A good operational habit is to run `git status` before staging, committing, switching branches, pulling, merging, or rebasing.

***

## 11. What does `git log` show?

`git log` displays the commit history reachable from the current branch or specified reference.

```bash
git log
```

It typically displays:

* Commit hash
* Author
* Date
* Commit message

A compact log:

```bash
git log --oneline
```

Display branch and tag relationships graphically:

```bash
git log --oneline --graph --decorate --all
```

Show changes made to a specific file:

```bash
git log -- app/config.yaml
```

Show commits created by a particular author:

```bash
git log --author="Rajesh Singh"
```

Show commits made in a date range:

```bash
git log --since="2026-07-01" --until="2026-07-24"
```

Show the patch introduced by every commit:

```bash
git log -p
```

Show a summary of changed files:

```bash
git log --stat
```

> `git log` is useful for troubleshooting regressions, understanding design decisions, reviewing release history, and identifying commits associated with incidents.

***

## 12. How do you view changes in files before committing?

Use `git diff` to compare file content across different Git states.

### View unstaged changes

```bash
git diff
```

This compares the working directory with the staging area.

### View staged changes

```bash
git diff --staged
```

This compares the staging area with the latest commit.

The following is equivalent:

```bash
git diff --cached
```

### Compare the working directory with the latest commit

```bash
git diff HEAD
```

### Compare two commits

```bash
git diff <commit1> <commit2>
```

Example:

```bash
git diff a7b31f2 c83d9a4
```

### Compare two branches

```bash
git diff main..feature/payment-validation
```

### Compare a particular file

```bash
git diff -- src/payment-service.cs
```

> Before committing, use both `git status` and `git diff --staged` to confirm that only the intended changes are included.

***

## 13. What is `git checkout` used for?

Historically, `git checkout` has been used for multiple purposes:

* Switching branches
* Creating and switching branches
* Checking out a particular commit
* Restoring files from another commit or branch

Switch to an existing branch:

```bash
git checkout develop
```

Create and switch to a new branch:

```bash
git checkout -b feature/payment-validation
```

Check out a specific commit:

```bash
git checkout a7b31f2
```

Checking out a commit directly creates a **detached HEAD** state. New commits made in this state may become difficult to locate unless a branch is created.

Create a branch from a detached HEAD:

```bash
git switch -c investigation/fix
```

Modern Git separates the overloaded `checkout` functionality into:

* `git switch` for branch operations
* `git restore` for restoring files

Examples:

```bash
git switch develop
git restore app/config.yaml
```

> **Recommendation:** Use `git switch` for branches and `git restore` for files. They are clearer and reduce the possibility of accidental data loss.

***

## 14. How do you create a new branch?

Create a branch without switching to it:

```bash
git branch feature/payment-validation
```

Create and switch to the branch:

```bash
git switch -c feature/payment-validation
```

Older syntax:

```bash
git checkout -b feature/payment-validation
```

Create a branch from a specific commit:

```bash
git switch -c hotfix/authentication a7b31f2
```

Create a branch from another branch:

```bash
git switch -c release/2.5.0 develop
```

Push the new branch and configure upstream tracking:

```bash
git push -u origin feature/payment-validation
```

List local branches:

```bash
git branch
```

List local and remote branches:

```bash
git branch -a
```

> Before creating a branch, update the intended base branch so that the new branch starts from the latest approved code.

Example:

```bash
git switch main
git pull --ff-only
git switch -c feature/payment-validation
```

***

## 15. How do you switch to a branch?

Using modern Git:

```bash
git switch feature/payment-validation
```

Using the older command:

```bash
git checkout feature/payment-validation
```

Switch to the previous branch:

```bash
git switch -
```

Create and switch to a branch:

```bash
git switch -c feature/payment-validation
```

To create a local branch that tracks a remote branch:

```bash
git switch --track origin/feature/payment-validation
```

Git may prevent a branch switch if uncommitted changes would be overwritten. In that situation, you can:

1. Commit the changes.
2. Stash them temporarily.
3. Discard them if they are not required.

Example using stash:

```bash
git stash push -m "Temporary changes before branch switch"
git switch main
```

Restore the stashed changes:

```bash
git stash pop
```

> Always run `git status` before switching branches to avoid losing or unintentionally carrying changes into another branch.

***

## 16. How do you merge one branch into another?

To merge one branch into another, first switch to the **target branch**, then merge the **source branch**.

Example: merge `feature/payment-validation` into `main`.

```bash
git switch main
git pull --ff-only
git merge feature/payment-validation
```

If the merge succeeds, push it:

```bash
git push origin main
```

If conflicts occur:

```bash
git status
```

Open the conflicting files and resolve markers such as:

```text
<<<<<<< HEAD
Code from the target branch
=======
Code from the source branch
>>>>>>> feature/payment-validation
```

After resolving the conflicts:

```bash
git add <resolved-file>
git commit
```

Abort an incomplete merge:

```bash
git merge --abort
```

Force the creation of a merge commit:

```bash
git merge --no-ff feature/payment-validation
```

> In GitHub and Azure Repos, protected branches should normally be updated through pull requests rather than direct local merges and pushes. Pull requests provide peer review, automated validation, policy enforcement, and audit history.

***

## 17. What is a fast-forward merge?

A **fast-forward merge** occurs when the target branch has not received any new commits since the source branch was created.

Before merging:

```text
A---B  main
     \
      C---D  feature
```

Since `main` still points to `B`, Git can move the `main` pointer directly to `D`:

```text
A---B---C---D  main, feature
```

Command:

```bash
git switch main
git merge feature
```

No separate merge commit is required.

To allow only fast-forward merges:

```bash
git merge --ff-only feature
```

If a fast-forward is not possible, the command fails rather than creating a merge commit.

To deliberately create a merge commit:

```bash
git merge --no-ff feature
```

### Advantages

* Produces a clean, linear history
* Avoids unnecessary merge commits
* Makes the history easier to follow

### Possible disadvantage

The feature’s branch boundary may not be obvious after the merge.

> Many teams address this through pull-request metadata, squash merging, ticket references, or explicit merge commits.

***

## 18. What is a three-way merge?

A **three-way merge** is used when the source and target branches have both progressed since their common ancestor.

Git uses three inputs:

1. The common ancestor, also called the merge base
2. The latest commit on the target branch
3. The latest commit on the source branch

Example:

```text
      C---D  feature
     /
A---B---E---F  main
```

`B` is the common ancestor. Git compares:

* `B` to `D`
* `B` to `F`

It then combines the changes and generally creates a merge commit:

```text
      C---D
     /     \
A---B---E---F---M
```

`M` is the merge commit and has two parent commits.

Command:

```bash
git switch main
git merge feature
```

If the branches modify different parts of the code, Git can usually merge them automatically. If they modify incompatible parts of the same content, Git reports a conflict requiring manual resolution.

> A three-way merge preserves the actual branch topology, which can be useful for auditability and understanding how parallel development was integrated.

***

## 19. What is `git pull`, and what does it do?

`git pull` retrieves changes from a remote repository and integrates them into the current local branch.

Conceptually, the default behavior is:

```text
git pull = git fetch + git merge
```

Example:

```bash
git pull origin main
```

This normally:

1. Contacts the remote repository.
2. Downloads new objects and references.
3. Updates the relevant remote-tracking branch.
4. Merges the fetched remote branch into the current local branch.

Pull using rebase instead of merge:

```bash
git pull --rebase origin main
```

This reapplies local commits on top of the updated remote branch and can help maintain a linear history.

Allow the pull only when it can be fast-forwarded:

```bash
git pull --ff-only
```

This is safer for engineers who want to avoid an unexpected merge commit.

Set a preferred pull strategy:

```bash
git config --global pull.ff only
```

Or use rebase by default:

```bash
git config --global pull.rebase true
```

> **Senior-level recommendation:** Do not run `git pull` blindly. Check the current branch and working tree first. For controlled integration, use `git fetch`, inspect the incoming changes, and then merge or rebase explicitly.

***

## 20. What is `git fetch`, and how is it different from `git pull`?

`git fetch` downloads new commits, branches, tags, and references from a remote repository without automatically modifying the current local branch or working directory.

```bash
git fetch origin
```

Fetch all configured remotes:

```bash
git fetch --all
```

Remove remote-tracking references that no longer exist on the remote:

```bash
git fetch --prune
```

After fetching, inspect incoming commits:

```bash
git log HEAD..origin/main --oneline
```

Compare the local branch with the remote-tracking branch:

```bash
git diff main..origin/main
```

Then integrate the changes explicitly:

```bash
git merge origin/main
```

Or:

```bash
git rebase origin/main
```

### `git fetch` versus `git pull`

| Operation                             | `git fetch` | `git pull`                        |
| ------------------------------------- | ----------- | --------------------------------- |
| Downloads remote changes              | Yes         | Yes                               |
| Updates remote-tracking references    | Yes         | Yes                               |
| Changes the current local branch      | No          | Usually yes                       |
| May create a merge commit             | No          | Yes, depending on strategy        |
| May cause merge conflicts immediately | No          | Yes                               |
| Best for reviewing before integration | Yes         | Less controlled                   |
| Typical risk level                    | Lower       | Higher if used without inspection |

### Controlled production workflow

```bash
git status
git fetch origin --prune
git log HEAD..origin/main --oneline
git diff HEAD..origin/main
git merge --ff-only origin/main
```

> **Interview summary:** `git fetch` downloads changes without integrating them. `git pull` downloads and immediately integrates changes using merge or rebase, depending on configuration.

***

# Practical End-to-End Git Workflow

The following example represents a common feature-development workflow:

```bash
# Clone the repository
git clone https://github.com/organization/payment-service.git

# Enter the repository
cd payment-service

# Update remote references
git fetch origin --prune

# Switch to the main branch
git switch main

# Update main safely
git pull --ff-only

# Create a feature branch
git switch -c feature/payment-validation

# Check modified files
git status

# Review changes
git diff

# Stage selected files
git add src/payment-service.cs tests/payment-service-tests.cs

# Review exactly what will be committed
git diff --staged

# Create a meaningful commit
git commit -m "Add validation for invalid payment requests"

# Push and establish upstream tracking
git push -u origin feature/payment-validation
```

After pushing, the engineer creates a **pull request** in GitHub or Azure Repos, where the following controls may be applied:

* Mandatory reviewers
* Successful CI build
* Unit and integration tests
* Code quality analysis
* Security and secret scanning
* Work-item association
* Comment resolution
* Restricted direct pushes
* Approved merge strategy

***

# Version Control Interview Questions and Answers

## Git, GitHub, and Azure Repos: Questions 21–50

These answers continue the earlier Git fundamentals and are written for **Senior DevOps Engineer and DevOps Architect interviews**. They include operational cautions, production use cases, and ready-to-use commands.

***

## 21. What is `git push` used for?

`git push` uploads local commits, branches, and tags to a remote repository such as GitHub, Azure Repos, GitLab, or Bitbucket.

Basic syntax:

```bash
git push <remote> <branch>
```

Example:

```bash
git push origin main
```

This sends commits from the local `main` branch to the `main` branch on the `origin` remote.

### Push a new branch and configure upstream tracking

```bash
git push -u origin feature/payment-validation
```

The `-u` option associates the local branch with the remote branch. Subsequent pushes can normally use:

```bash
git push
```

### Push all tags

```bash
git push origin --tags
```

### Push a specific tag

```bash
git push origin v2.1.0
```

### Push safely after rewriting branch history

```bash
git push --force-with-lease origin feature/payment-validation
```

`--force-with-lease` is safer than `--force` because it checks whether the remote branch has changed since the last fetch.

### Important production considerations

* `git push` does not automatically deploy an application, but it may trigger a CI/CD pipeline.
* Pushes to protected branches may be rejected.
* GitHub and Azure Repos can enforce pull requests, reviewers, successful builds, and security checks.
* Avoid force-pushing shared branches.
* Never force-push protected branches such as `main`, `release`, or production branches.

> **Interview summary:** `git push` publishes local Git objects and references to a remote repository. In enterprise environments, direct pushes to critical branches should be restricted through branch protection policies.

***

## 22. How do you delete a local branch?

Use the `-d` option to delete a local branch that has already been merged:

```bash
git branch -d feature/payment-validation
```

Git prevents deletion if the branch contains unmerged commits.

To force-delete an unmerged local branch:

```bash
git branch -D feature/payment-validation
```

The uppercase `-D` is effectively a force-delete operation.

You cannot delete the branch that is currently checked out. Switch to another branch first:

```bash
git switch main
git branch -d feature/payment-validation
```

Before deleting a branch, verify whether it has been merged:

```bash
git branch --merged
```

List branches that have not been merged:

```bash
git branch --no-merged
```

> Use `git branch -D` carefully. Unmerged commits may become difficult to locate after the branch reference is removed, although they may remain temporarily recoverable through the reflog.

***

## 23. How do you delete a remote branch?

Use the following command:

```bash
git push origin --delete feature/payment-validation
```

This asks the remote repository named `origin` to remove the specified branch reference.

After deleting the branch, clean stale remote-tracking references:

```bash
git fetch origin --prune
```

You can also prune directly:

```bash
git remote prune origin
```

Deleting a remote branch does not automatically delete another developer’s local copy of that branch.

To remove the corresponding local branch:

```bash
git switch main
git branch -d feature/payment-validation
```

Branch protection policies may prevent remote branch deletion. For GitHub or Azure Repos, the user must also have the necessary repository permissions.

> Delete remote branches after their pull requests are merged, unless retention is required for release management, compliance, or operational support.

***

## 24. What is a tag in Git, and how do you create one?

A **Git tag** is a named reference to a specific commit. Unlike a branch, a tag normally remains fixed and does not move when new commits are created.

Tags are commonly used for:

* Application releases
* Production deployment versions
* Stable baselines
* Rollback points
* Semantic versions
* Audit and compliance references

### Create a lightweight tag

```bash
git tag v2.1.0
```

This tags the current commit.

### Tag a specific commit

```bash
git tag v2.1.0 a7b31f2
```

### Create an annotated tag

```bash
git tag -a v2.1.0 -m "Production release version 2.1.0"
```

### List tags

```bash
git tag
```

Filter tags:

```bash
git tag -l "v2.*"
```

### Push a tag

```bash
git push origin v2.1.0
```

Push all local tags:

```bash
git push origin --tags
```

### Delete a local tag

```bash
git tag -d v2.1.0
```

### Delete a remote tag

```bash
git push origin --delete v2.1.0
```

> In CI/CD, use immutable tags for releases. Do not silently move an existing production tag to another commit because that destroys release traceability.

***

## 25. What is the difference between lightweight and annotated tags?

Git supports two primary types of tags.

### Lightweight tag

A lightweight tag is simply a named reference to a commit.

```bash
git tag v2.1.0
```

It does not store a separate tag message, tagger identity, or tagging date.

Lightweight tags are suitable for:

* Temporary references
* Local testing points
* Personal bookmarks
* Short-lived development markers

### Annotated tag

An annotated tag is stored as a separate Git object and includes:

* Tag name
* Tagger name
* Tagger email
* Tagging date
* Tag message
* Reference to the tagged object

Create one with:

```bash
git tag -a v2.1.0 -m "Production release version 2.1.0"
```

Display its details:

```bash
git show v2.1.0
```

Annotated tags can also be cryptographically signed if signing is configured:

```bash
git tag -s v2.1.0 -m "Signed production release"
```

### Recommended use

Use annotated or signed tags for official releases because they provide better metadata, identity verification, auditability, and release traceability.

***

## 26. What does `git remote -v` show?

`git remote -v` lists configured remote repositories and their URLs.

```bash
git remote -v
```

Example output:

```text
origin  https://github.com/company/payment-service.git (fetch)
origin  https://github.com/company/payment-service.git (push)
```

The output shows:

* Remote name, such as `origin`
* Fetch URL
* Push URL

A repository may have different URLs for fetching and pushing.

For example:

```text
origin  https://github.com/company/payment-service.git (fetch)
origin  git@github.com:company/payment-service.git (push)
```

View detailed information about a remote:

```bash
git remote show origin
```

This may display:

* Remote branches
* Tracking relationships
* Default branch
* Push configuration
* Local branches configured for pull

> `origin` is only a conventional remote name. It is not a mandatory Git keyword.

***

## 27. How do you add a remote repository?

Use `git remote add`:

```bash
git remote add origin https://github.com/company/payment-service.git
```

Azure Repos example:

```bash
git remote add origin https://dev.azure.com/company/project/_git/payment-service
```

Verify the configuration:

```bash
git remote -v
```

Push the local branch and establish upstream tracking:

```bash
git push -u origin main
```

### Add another remote

A second remote is useful when working with a fork:

```bash
git remote add upstream https://github.com/original-owner/payment-service.git
```

Typical convention:

* `origin` points to your fork or primary repository.
* `upstream` points to the original repository.

Fetch from the upstream repository:

```bash
git fetch upstream
```

### Change a remote URL

```bash
git remote set-url origin git@github.com:company/payment-service.git
```

### Remove a remote

```bash
git remote remove origin
```

Removing a remote configuration does not delete the actual remote repository.

***

## 28. How do you rename a branch?

Rename the currently checked-out branch:

```bash
git branch -m new-branch-name
```

Rename a specified local branch:

```bash
git branch -m old-branch-name new-branch-name
```

Example:

```bash
git branch -m feature/payment feature/payment-validation
```

If the old branch was already pushed, push the renamed branch:

```bash
git push -u origin feature/payment-validation
```

Delete the old remote branch:

```bash
git push origin --delete feature/payment
```

Verify tracking:

```bash
git branch -vv
```

For a shared branch, inform team members because their local repositories may still reference the old branch name.

For protected branches, such as renaming `master` to `main`, also update:

* Default branch settings
* Branch protection rules
* CI/CD pipeline triggers
* Deployment policies
* Scripts and automation
* Documentation
* Pull requests
* Azure Boards or GitHub integrations

> Renaming a branch locally does not automatically rename the corresponding remote branch.

***

## 29. What is `git revert`, and how is it different from `git reset`?

`git revert` creates a new commit that reverses the changes introduced by an earlier commit.

```bash
git revert <commit-hash>
```

Example:

```bash
git revert a7b31f2
```

Because it creates a new commit, the original commit remains in history.

### Revert a merge commit

```bash
git revert -m 1 <merge-commit-hash>
```

The `-m 1` option identifies the mainline parent. Reverting merge commits requires careful validation because it can affect future merge behavior.

### `git reset`

`git reset` moves the current branch reference to another commit and may also modify the staging area and working directory.

```bash
git reset --soft HEAD~1
git reset --mixed HEAD~1
git reset --hard HEAD~1
```

### Key difference

* `git revert` preserves history and adds a corrective commit.
* `git reset` rewrites the local branch history by moving the branch pointer.

### Recommended usage

Use `git revert` for:

* Shared branches
* Published commits
* Protected branches
* Production rollback commits
* Auditable change reversal

Use `git reset` primarily for:

* Local unpublished commits
* Cleaning up work before pushing
* Correcting private branch history

> For GitHub and Azure Repos shared branches, `git revert` is normally safer because it does not require rewriting remote history.

***

## 30. What is `git reset --hard`, and when should you avoid it?

`git reset --hard` moves the current branch to a specified commit and makes both the staging area and working directory match that commit.

Example:

```bash
git reset --hard HEAD~1
```

This removes the latest commit from the current branch and discards tracked-file changes associated with the current state.

Reset to a specific commit:

```bash
git reset --hard a7b31f2
```

Make a local branch match the remote-tracking branch:

```bash
git fetch origin
git reset --hard origin/main
```

### What it affects

`git reset --hard` changes:

* The current branch pointer
* The staging area
* The tracked files in the working directory

It normally does not remove untracked files. Removing untracked files requires another command, such as:

```bash
git clean -fd
```

This combination is highly destructive:

```bash
git reset --hard
git clean -fd
```

### Avoid it when

* You have uncommitted changes you need.
* You are uncertain which commit should become the new branch tip.
* The branch is shared with other developers.
* The commits have already been pushed.
* You do not have a backup, stash, branch, or reflog reference.

Before running it, consider creating a safety branch:

```bash
git branch backup-before-reset
```

Or create a stash:

```bash
git stash push -u -m "Backup before hard reset"
```

> `git reset --hard` can permanently discard uncommitted tracked-file changes. Use it only after verifying `git status`, the target commit, and the recovery plan.

***

## 31. What is the difference between `git reset`, `git checkout`, and `git revert`?

These commands can all appear to undo changes, but they operate differently.

### `git reset`

`git reset` moves the current branch pointer and optionally changes the staging area and working directory.

Common modes:

```bash
git reset --soft HEAD~1
git reset --mixed HEAD~1
git reset --hard HEAD~1
```

Use it mainly for correcting local, unpublished history.

### `git checkout`

Historically, `git checkout` has been used to:

* Switch branches
* Enter detached HEAD state
* Restore files
* Check out a historical commit

Examples:

```bash
git checkout develop
git checkout a7b31f2
git checkout -- app/config.yaml
```

Modern alternatives are clearer:

```bash
git switch develop
git restore app/config.yaml
```

### `git revert`

`git revert` creates a new commit that reverses another commit:

```bash
git revert a7b31f2
```

It is designed for safe reversal on shared branches.

### Practical selection

* Use `git restore` to discard local file changes.
* Use `git switch` to change branches.
* Use `git reset` to adjust unpublished local history.
* Use `git revert` to reverse a published commit safely.

***

## 32. What is `git stash` used for?

`git stash` temporarily saves uncommitted changes so that the working directory can be cleaned without creating a normal source-code commit.

Basic command:

```bash
git stash
```

Recommended descriptive form:

```bash
git stash push -m "WIP payment validation changes"
```

A common scenario is receiving an urgent task while feature work is incomplete:

```bash
git stash push -u -m "Incomplete payment feature"
git switch main
git switch -c hotfix/login-timeout
```

After the hotfix, return and restore the work:

```bash
git switch feature/payment-validation
git stash pop
```

### Include untracked files

```bash
git stash push -u -m "Include untracked test files"
```

### Include ignored files as well

```bash
git stash push -a -m "Include all local files"
```

### List stashes

```bash
git stash list
```

### Inspect a stash

```bash
git stash show -p stash@{0}
```

### Important considerations

* A stash is local and is not normally pushed to the remote repository.
* Stashes can become confusing if retained for a long time.
* A stash may produce conflicts when reapplied.
* Do not use stashes as a long-term backup system.

> For work that must be shared or preserved remotely, create a temporary branch and commit instead of relying only on a stash.

***

## 33. How do you apply a stash?

There are two common commands.

### Apply without deleting the stash

```bash
git stash apply
```

This applies the latest stash but keeps it in the stash list.

Apply a specific stash:

```bash
git stash apply stash@{2}
```

### Apply and remove the stash

```bash
git stash pop
```

This applies the latest stash and removes it if the operation completes successfully.

Pop a specific stash:

```bash
git stash pop stash@{2}
```

### Delete a stash manually

```bash
git stash drop stash@{0}
```

Delete all stashes:

```bash
git stash clear
```

### Create a branch from a stash

```bash
git stash branch feature/recovered-work stash@{0}
```

This is useful when applying the stash to the current branch causes conflicts.

### `apply` versus `pop`

* Use `apply` when you want to keep the stash as a backup.
* Use `pop` when you want to restore and remove it in one operation.

> In important scenarios, use `git stash apply` first, validate the restored work, and then explicitly drop the stash.

***

## 34. What is `git cherry-pick` used for?

`git cherry-pick` applies the changes introduced by a specific commit onto the currently checked-out branch.

Example:

```bash
git switch release/2.1
git cherry-pick a7b31f2
```

Git creates a new commit on the current branch with a different commit hash, even though the code changes may be similar.

### Common use cases

* Apply a hotfix to multiple release branches.
* Move a specific fix without merging an entire feature branch.
* Recover a useful commit created on the wrong branch.
* Backport a security fix to an older supported version.

### Cherry-pick multiple commits

```bash
git cherry-pick a7b31f2 b8c42d3 c9d53e4
```

Cherry-pick a commit range:

```bash
git cherry-pick a7b31f2^..c9d53e4
```

### Handle conflicts

After resolving conflicting files:

```bash
git add <resolved-files>
git cherry-pick --continue
```

Abort the operation:

```bash
git cherry-pick --abort
```

Skip the problematic commit:

```bash
git cherry-pick --skip
```

### Important consideration

Excessive cherry-picking can duplicate changes across branches and make history difficult to manage. It should not replace a well-designed branching and release strategy.

***

## 35. What is `git bisect` used for?

`git bisect` uses a binary-search process to identify the commit that introduced a defect or regression.

Start the process:

```bash
git bisect start
```

Mark the current commit as bad:

```bash
git bisect bad
```

Mark a known working commit as good:

```bash
git bisect good v2.0.0
```

Git checks out a commit near the middle of the range. Test the application and classify the result:

```bash
git bisect good
```

Or:

```bash
git bisect bad
```

Repeat until Git identifies the first bad commit.

Finish and return to the original branch:

```bash
git bisect reset
```

### Automate bisect with a test script

```bash
git bisect start
git bisect bad HEAD
git bisect good v2.0.0
git bisect run ./test-regression.sh
```

The script should return:

* `0` when the commit is good
* A nonzero value, usually between `1` and `127`, when it is bad
* `125` when the commit cannot be tested

> `git bisect` is particularly effective when the repository has reliable automated tests and the regression range contains many commits.

***

## 36. How do you find who changed a line of code?

Use `git blame`:

```bash
git blame app/payment-service.cs
```

It shows, for each line:

* Commit hash
* Author
* Date
* Line number
* Current line content

Limit output to a line range:

```bash
git blame -L 50,80 app/payment-service.cs
```

View details of the associated commit:

```bash
git show <commit-hash>
```

Ignore whitespace-only changes:

```bash
git blame -w app/payment-service.cs
```

Follow code moved or copied within files:

```bash
git blame -M -C app/payment-service.cs
```

### Appropriate use

`git blame` should be used to understand:

* Why code exists
* Which commit introduced a behavior
* Associated work items or pull requests
* Relevant implementation context
* Potential subject-matter experts

> It should not be used to assign personal blame. In healthy engineering teams, Git history is a troubleshooting and knowledge-discovery tool.

***

## 37. What is `.gitignore` for?

A `.gitignore` file defines patterns for intentionally untracked files that Git should not suggest for staging.

Example:

```gitignore
# Build output
bin/
obj/
dist/

# Dependencies
node_modules/

# Logs
*.log

# Local environment files
.env
.env.local

# IDE settings
.vscode/
.idea/

# Terraform local and sensitive files
.terraform/
*.tfstate
*.tfstate.*
crash.log
```

### Important behavior

`.gitignore` applies only to untracked files. If a file is already tracked, adding it to `.gitignore` does not stop Git from tracking it.

To stop tracking a file while retaining it locally:

```bash
git rm --cached appsettings.Local.json
```

Then add it to `.gitignore` and commit the change.

### Sources of ignore rules

Git can use:

* Repository-level `.gitignore`
* Nested `.gitignore` files
* Repository-specific `.git/info/exclude`
* Global ignore configuration

Configure a global ignore file:

```bash
git config --global core.excludesfile ~/.gitignore_global
```

> `.gitignore` is not a security boundary. If a secret has already been committed, ignoring the file does not remove that secret from history. Rotate the credential immediately and follow an approved history-cleaning process.

***

## 38. How do you ignore changes to a tracked file?

A commonly mentioned command is:

```bash
git update-index --assume-unchanged path/to/file
```

Restore normal tracking behavior:

```bash
git update-index --no-assume-unchanged path/to/file
```

However, `--assume-unchanged` is primarily a performance hint. It tells Git to assume the working-tree file has not changed, and it is not a reliable general-purpose solution for local configuration management.

Another option is:

```bash
git update-index --skip-worktree path/to/file
```

Undo it with:

```bash
git update-index --no-skip-worktree path/to/file
```

List files marked with special index flags:

```bash
git ls-files -v
```

### Recommended design

For local configuration, use a tracked template and an ignored local override:

```text
appsettings.example.json
appsettings.local.json
```

Add the local file to `.gitignore`:

```gitignore
appsettings.local.json
```

This is more reliable than asking individual developers to apply index flags.

> Neither `assume-unchanged` nor `skip-worktree` should be treated as a permanent team-wide configuration strategy. Use environment variables, secret stores, configuration providers, or ignored local override files instead.

***

## 39. What is `git rm` used for?

`git rm` removes a file from both the working directory and the Git index, preparing the deletion for the next commit.

```bash
git rm obsolete-config.xml
git commit -m "Remove obsolete configuration file"
```

Remove a directory recursively:

```bash
git rm -r legacy-scripts/
```

### Stop tracking but keep the local file

```bash
git rm --cached appsettings.Local.json
```

For a directory:

```bash
git rm -r --cached .terraform/
```

Afterward, add the path to `.gitignore`.

### Remove ignored files from tracking after updating `.gitignore`

A broad approach is:

```bash
git rm -r --cached .
git add .
git commit -m "Apply updated ignore rules"
```

This should be reviewed carefully before committing:

```bash
git status
git diff --staged
```

> `git rm` stages the deletion. The deletion is not part of repository history until it is committed.

***

## 40. How do you move or rename a file with Git while preserving history?

Use `git mv`:

```bash
git mv old-name.yaml new-name.yaml
git commit -m "Rename deployment configuration"
```

Move a file into another directory:

```bash
git mv deployment.yaml manifests/deployment.yaml
```

Git does not store a rename as a special permanent rename object. It records the deletion and addition, then detects the rename by comparing content similarity.

Therefore, these commands can produce an equivalent result:

```bash
mv old-name.yaml new-name.yaml
git add old-name.yaml new-name.yaml
```

View detected renames:

```bash
git diff --staged --summary
```

Follow a file’s history across renames:

```bash
git log --follow -- new-name.yaml
```

> `git mv` is preferred because it updates the working directory and staging area in one clear command, reducing accidental omissions.

***

## 41. What is a detached HEAD state?

Normally, `HEAD` points to the currently checked-out branch, and the branch points to a commit.

Normal state:

```text
HEAD -> main -> commit
```

A detached HEAD state occurs when `HEAD` points directly to a commit rather than to a branch.

Example:

```bash
git checkout a7b31f2
```

Or with modern Git:

```bash
git switch --detach a7b31f2
```

Detached HEAD is useful for:

* Inspecting an old version
* Running tests against a historical commit
* Investigating a production issue
* Building a specific commit in CI
* Performing temporary experiments

You can create commits in detached HEAD state, but they are not attached to a named branch.

To preserve those commits:

```bash
git switch -c investigation/recovered-change
```

Return to an existing branch:

```bash
git switch main
```

> Detached HEAD is not an error. The risk arises when valuable commits are created and no branch or tag is created to retain them.

***

## 42. How do you create a patch or export changes?

Git supports multiple patch-generation methods.

### Export uncommitted changes

```bash
git diff > working-changes.patch
```

Apply the patch:

```bash
git apply working-changes.patch
```

### Export staged changes

```bash
git diff --staged > staged-changes.patch
```

### Export differences between commits

```bash
git diff commit1 commit2 > changes.patch
```

### Create email-style patches from commits

Export the latest commit:

```bash
git format-patch -1 HEAD
```

Export the latest three commits:

```bash
git format-patch -3 HEAD
```

Export commits that exist on the current branch but not on `main`:

```bash
git format-patch main..HEAD
```

Apply formatted patches while preserving commit metadata:

```bash
git am 0001-add-payment-validation.patch
```

Abort a failed `git am` operation:

```bash
git am --abort
```

### Difference

* `git diff` creates a content-difference patch without full commit metadata.
* `git format-patch` creates one or more patches containing commit author, message, and related metadata.
* `git apply` applies file changes.
* `git am` applies formatted commits and creates corresponding commits.

***

## 43. What is `git show` used for?

`git show` displays information about Git objects. Its most common use is inspecting a commit.

```bash
git show <commit-hash>
```

It normally shows:

* Commit hash
* Author
* Commit date
* Commit message
* Changed files
* Patch introduced by the commit

Show the latest commit:

```bash
git show HEAD
```

Show only a summary:

```bash
git show --stat HEAD
```

Show only the file names:

```bash
git show --name-only HEAD
```

Show a file as it existed at a specific commit:

```bash
git show a7b31f2:app/config.yaml
```

Inspect a tag:

```bash
git show v2.1.0
```

Show a commit without its patch:

```bash
git show --no-patch --pretty=fuller HEAD
```

> `git show` is particularly useful during incidents to inspect the exact code change introduced by a deployment-related commit.

***

## 44. How do you view the commit history graphically in the CLI?

Use:

```bash
git log --graph --oneline --all
```

A more informative version is:

```bash
git log --graph --oneline --decorate --all
```

Example output:

```text
*   a23bc45 (HEAD -> main) Merge feature/payment-validation
|\
| * b34cd56 Add payment validation tests
| * c45de67 Implement payment validation
|/
* d56ef78 Previous release baseline
```

Useful variations:

```bash
git log --graph --oneline --decorate --all --date-order
```

Show authors and dates:

```bash
git log --graph --all \
  --pretty=format:'%C(auto)%h%d %s %C(black)%C(bold)%cr %C(blue)<%an>'
```

Create an alias:

```bash
git config --global alias.lg "log --graph --oneline --decorate --all"
```

Then run:

```bash
git lg
```

> The graph helps visualize branching, merges, release lines, and divergence between local and remote branches.

***

## 45. How do you squash commits locally before pushing?

Use interactive rebase.

To review the latest four commits:

```bash
git rebase -i HEAD~4
```

Git opens an editor with entries similar to:

```text
pick a1b2c3d Add payment validation
pick b2c3d4e Fix validation test
pick c3d4e5f Correct formatting
pick d4e5f6a Update error handling
```

Keep the first commit and change later commands to `squash` or `fixup`:

```text
pick a1b2c3d Add payment validation
squash b2c3d4e Fix validation test
fixup c3d4e5f Correct formatting
squash d4e5f6a Update error handling
```

Meanings:

* `squash`: Combine with the previous commit and edit the final message.
* `fixup`: Combine with the previous commit and discard the fixup commit’s message.

After saving, Git rewrites the commits.

### If the branch was already pushed

Because commit hashes change, update the remote using:

```bash
git push --force-with-lease
```

### Best practice

Squash local feature-branch commits before requesting final review or use the repository platform’s squash-merge functionality.

> Do not interactively rebase commits that other developers are actively using unless the team has explicitly coordinated the history rewrite.

***

## 46. What is `git rebase`, and when is it helpful?

`git rebase` reapplies commits from one branch onto another base commit.

Suppose the history is:

```text
      C---D  feature
     /
A---B---E---F  main
```

Run:

```bash
git switch feature
git rebase main
```

Git reapplies the feature commits on top of `main`:

```text
A---B---E---F---C'---D'  feature
```

`C'` and `D'` are new commits with new hashes.

### When rebase is helpful

* Updating a local feature branch with the latest `main`
* Maintaining a linear and readable history
* Cleaning local commits before a pull request
* Avoiding unnecessary merge commits
* Preparing a patch series for review

### Resolve rebase conflicts

```bash
git status
```

Resolve the files, stage them, and continue:

```bash
git add <resolved-files>
git rebase --continue
```

Skip a commit:

```bash
git rebase --skip
```

Abort and return to the original state:

```bash
git rebase --abort
```

### Golden rule

Do not rebase shared public history unless everyone affected has coordinated the change.

> Rebase is best suited for cleaning and updating private feature-branch history. Merge is often safer when preserving shared branch history is more important.

***

## 47. What is `git rebase --interactive` used for?

Interactive rebase allows you to edit and reorganize a series of commits before publishing them.

Start an interactive rebase:

```bash
git rebase -i HEAD~5
```

Common actions include:

* `pick`: Keep the commit unchanged.
* `reword`: Change the commit message.
* `edit`: Pause and modify the commit.
* `squash`: Combine with the previous commit and edit the message.
* `fixup`: Combine and discard the selected commit message.
* `drop`: Remove the commit.
* Reorder lines to reorder commits.

Example:

```text
pick a1b2c3d Add API validation
fixup b2c3d4e Fix typo
reword c3d4e5f Add validation tests
drop d4e5f6a Temporary debug logging
```

### Common use cases

* Squash noisy work-in-progress commits
* Remove debugging commits
* Improve commit messages
* Split a large commit
* Reorder dependent commits
* Produce an audit-friendly pull request history

### Split a commit

Mark it as `edit`, then run:

```bash
git reset HEAD^
git add <first-set-of-files>
git commit -m "First logical change"
git add <second-set-of-files>
git commit -m "Second logical change"
git rebase --continue
```

> Interactive rebase rewrites commit hashes. Use it before pushing or only on branches where history rewriting has been explicitly coordinated.

***

## 48. What is a commit hash, and how is it used?

A commit hash is a cryptographic identifier for a Git commit object.

Historically, Git repositories have used SHA-1 identifiers. Modern Git also supports a SHA-256 repository format, although interoperability and hosting-platform support should be verified before adopting it.

A full hash may look like:

```text
c3a7e30d9f6c4cde1692ca1c29b85cfbde234abc
```

A shortened form may be used when it remains unique:

```text
c3a7e30
```

### The commit identity is based on information including

* Project tree
* Parent commit or commits
* Author
* Committer
* Timestamps
* Commit message

Changing any commit metadata or content generates a new commit hash.

### Common uses

Inspect a commit:

```bash
git show c3a7e30
```

Check out a historical commit:

```bash
git switch --detach c3a7e30
```

Create a branch from it:

```bash
git switch -c hotfix/recovery c3a7e30
```

Cherry-pick it:

```bash
git cherry-pick c3a7e30
```

Revert it:

```bash
git revert c3a7e30
```

Compare commits:

```bash
git diff c3a7e30 a7b31f2
```

Reference it in a CI/CD deployment:

```text
Deploy source revision c3a7e30
```

> A commit hash provides precise source-code traceability, which is critical for reproducible builds, release auditing, vulnerability tracking, and incident investigation.

***

## 49. How do you configure your name and email for commits?

Configure the identity globally:

```bash
git config --global user.name "Rajesh Singh"
git config --global user.email "rajesh.singh@example.com"
```

Verify the values:

```bash
git config --global user.name
git config --global user.email
```

Configure a different identity for a particular repository:

```bash
git config user.name "Rajesh Singh"
git config user.email "rajesh.singh@company.com"
```

View the effective configuration:

```bash
git config --list
```

View each setting and its source:

```bash
git config --list --show-origin
```

Inspect the identity on the latest commit:

```bash
git show -s --format=fuller HEAD
```

### Important distinction

The Git author name and email do not authenticate a push. Authentication is handled separately through mechanisms such as:

* SSH keys
* Personal access tokens
* Credential managers
* Federated enterprise authentication
* Managed identities or service connections in automation

### CI/CD recommendation

Automation accounts should use an explicitly configured service identity:

```bash
git config user.name "Build Automation"
git config user.email "build-automation@company.com"
```

Do not allow a shared build agent to accidentally use a previous user’s global configuration.

***

## 50. What are global versus local Git configuration scopes?

Git configuration can exist at several scopes.

### System scope

Applies to all users and repositories on a machine:

```bash
git config --system core.autocrlf true
```

Typical configuration location:

```text
/etc/gitconfig
```

Administrative privileges may be required.

### Global scope

Applies to the current operating-system user:

```bash
git config --global user.name "Rajesh Singh"
git config --global user.email "rajesh.singh@example.com"
```

Typical configuration location:

```text
~/.gitconfig
```

### Local scope

Applies only to the current repository:

```bash
git config user.email "rajesh.singh@company.com"
```

It is stored in:

```text
.git/config
```

### Worktree scope

Git can also maintain configuration for an individual worktree when worktree-specific configuration is enabled.

### Precedence

More specific settings override broader settings:

```text
Worktree > Local > Global > System
```

For example, a local repository email overrides the globally configured email.

### Show effective configuration

```bash
git config --list
```

Show the source file for every setting:

```bash
git config --list --show-origin
```

Show scope information:

```bash
git config --list --show-scope
```

### Senior DevOps use case

A developer may use:

* A personal global identity
* A corporate identity in enterprise repositories
* Special signing and proxy settings for regulated projects
* Dedicated CI identities on build agents

Example local override:

```bash
cd enterprise-payment-platform

git config user.name "Rajesh Singh"
git config user.email "rajesh.singh@company.com"
```

> Use local configuration for repository-specific identity, signing, hooks, line-ending, or workflow requirements. Use global configuration only for settings that should consistently apply to all repositories owned by that user.

***

# Version Control Interview Questions and Answers

## Git, GitHub, and Azure Repos: Questions 51–100

Continuing the Senior DevOps Engineer interview series, these answers focus on Git configuration, authentication, repository maintenance, history recovery, advanced branching, Git LFS, submodules, worktrees, and troubleshooting.

***

## 51. What is `git config --list`?

`git config --list` displays the effective Git configuration available in the current context.

```bash
git config --list
```

It can show settings such as:

* User name and email
* Default branch
* Credential helper
* Line-ending behavior
* Configured aliases
* Merge and rebase preferences
* Proxy settings
* Commit-signing configuration
* Remote and branch configuration

Example output:

```text
user.name=Rajesh Singh
user.email=rajesh.singh@example.com
init.defaultbranch=main
credential.helper=manager
core.autocrlf=true
pull.rebase=true
```

Git configuration can come from multiple scopes. To identify where each setting originated, use:

```bash
git config --list --show-origin
```

To display the scope:

```bash
git config --list --show-scope
```

Display only global settings:

```bash
git config --global --list
```

Display repository-specific settings:

```bash
git config --local --list
```

Display a particular value:

```bash
git config user.email
```

> This command is especially useful when troubleshooting incorrect commit identities, proxy problems, authentication behavior, line-ending changes, or unexpected pull strategies.

***

## 52. How do you set up credential caching for HTTPS?

Git uses credential helpers to securely retrieve or temporarily store credentials used for HTTPS authentication.

View the configured credential helper:

```bash
git config --show-origin --get-all credential.helper
```

### Temporary in-memory caching

On supported Unix-like environments:

```bash
git config --global credential.helper cache
```

Set the cache duration to one hour:

```bash
git config --global credential.helper "cache --timeout=3600"
```

The credentials are stored in memory and removed when the cache expires or the credential-cache daemon stops.

### Git Credential Manager

For Windows and many cross-platform environments, Git Credential Manager is preferred:

```bash
git config --global credential.helper manager
```

Depending on the installation, the helper may be registered as:

```bash
git config --global credential.helper manager-core
```

Git Credential Manager can support:

* Secure operating-system credential storage
* Browser-based authentication
* Personal access tokens
* Multifactor authentication
* Microsoft Entra ID authentication
* GitHub and Azure Repos authentication flows

### Plaintext credential storage

```bash
git config --global credential.helper store
```

This stores credentials persistently, commonly in a plaintext file such as:

```text
~/.git-credentials
```

Because this is not encrypted, it is generally unsuitable for production servers, shared workstations, and enterprise environments.

### Remove a configured helper

```bash
git config --global --unset-all credential.helper
```

> Prefer Git Credential Manager, SSH authentication, or an enterprise-approved secret mechanism. Do not place personal access tokens directly in repository URLs, scripts, pipeline YAML, or shell history.

***

## 53. How do you set up SSH keys for Git authentication?

SSH keys provide public-key authentication between a Git client and a Git hosting platform.

### Step 1: Generate a key pair

Use Ed25519 where supported:

```bash
ssh-keygen -t ed25519 -C "rajesh.singh@example.com"
```

A common key location is:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

The private key must remain confidential. The `.pub` file is the public key that can be uploaded to GitHub, Azure DevOps, or another Git server.

### Step 2: Start the SSH agent

```bash
eval "$(ssh-agent -s)"
```

### Step 3: Add the private key

```bash
ssh-add ~/.ssh/id_ed25519
```

### Step 4: Copy the public key

```bash
cat ~/.ssh/id_ed25519.pub
```

Add the displayed public key to the Git hosting account.

### Step 5: Test authentication

GitHub example:

```bash
ssh -T git@github.com
```

Azure Repos example:

```bash
ssh -T git@ssh.dev.azure.com
```

### Step 6: Clone using an SSH URL

GitHub:

```bash
git clone git@github.com:organization/payment-service.git
```

Azure Repos commonly uses a URL similar to:

```bash
git clone git@ssh.dev.azure.com:v3/organization/project/payment-service
```

### Multiple-account configuration

An SSH configuration file can associate different keys with different hosts:

```text
Host github-corporate
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_corporate
    IdentitiesOnly yes

Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal
    IdentitiesOnly yes
```

Clone using the configured alias:

```bash
git clone git@github-corporate:organization/payment-service.git
```

> Protect private keys using strong file permissions and, where operationally practical, a passphrase. Use separate service identities and keys for CI/CD automation rather than personal developer credentials.

***

## 54. What is `.gitmodules`, and when does it appear?

The `.gitmodules` file stores configuration for Git submodules associated with a parent repository.

It normally appears after adding a submodule:

```bash
git submodule add https://github.com/company/shared-library.git libs/shared-library
```

Example `.gitmodules` content:

```ini
[submodule "libs/shared-library"]
    path = libs/shared-library
    url = https://github.com/company/shared-library.git
```

It contains:

* Submodule name
* Local path
* Remote repository URL
* Optional branch or update configuration

The parent repository does not store all submodule files directly. Instead, it stores a reference to a specific commit in the submodule repository.

The `.gitmodules` file should normally be committed:

```bash
git add .gitmodules libs/shared-library
git commit -m "Add shared library as a submodule"
```

> A submodule provides dependency separation, but it adds operational complexity. Teams must ensure cloning, CI pipelines, credentials, security scanning, and updates all handle submodules correctly.

***

## 55. What is Git LFS, and when would you use it?

Git Large File Storage, or Git LFS, is an extension for managing large binary files without storing every binary version directly in the normal Git object database.

Instead of storing the complete binary file in each commit, Git stores a small pointer file. The corresponding large content is stored in an LFS-compatible storage service.

### Initialize Git LFS

```bash
git lfs install
```

### Track file types

```bash
git lfs track "*.zip"
git lfs track "*.psd"
git lfs track "*.dll"
```

This updates `.gitattributes`, which should be committed:

```bash
git add .gitattributes
git commit -m "Configure Git LFS tracking"
```

### Add and commit a large file

```bash
git add assets/application-package.zip
git commit -m "Add application package using Git LFS"
git push
```

### List LFS-managed files

```bash
git lfs ls-files
```

### Appropriate use cases

Git LFS can be used for:

* Large media assets
* Design files
* Machine-learning models
* Large test datasets
* Vendor binaries that must remain versioned
* Game-development assets

### Operational considerations

* Hosting platforms may impose LFS storage and bandwidth quotas.
* CI agents need Git LFS installed and authenticated.
* Existing large files are not automatically migrated by adding a tracking rule.
* LFS is not a replacement for an artifact repository.
* Build packages should generally be stored in Azure Artifacts, GitHub Packages, JFrog Artifactory, Nexus, or object storage.

> Source repositories should primarily contain source code. Use Git LFS only when large files genuinely require source-style versioning.

***

## 56. What is a pull request or merge request?

A **pull request**, or PR, is a controlled proposal to merge changes from one branch into another.

GitHub and Azure Repos commonly use the term **pull request**. GitLab commonly uses **merge request**. The fundamental concept is similar.

A PR usually contains:

* Source and target branches
* Commit list
* File differences
* Author and reviewers
* Discussion comments
* Linked issues or work items
* Build and test results
* Security and quality checks
* Merge status

### Typical workflow

1. Create a feature branch.
2. Develop and test the change.
3. Push the branch.
4. Open a pull request.
5. Run automated validation.
6. Perform peer review.
7. Resolve comments and conflicts.
8. Obtain approvals.
9. Merge using the approved strategy.
10. Delete the completed feature branch.

### Common branch policies

* Minimum number of reviewers
* Prohibition on self-approval
* Successful CI build
* Unit- and integration-test completion
* Work-item association
* Comment resolution
* Code-owner approval
* Security scanning
* Restricted merge strategies

> A pull request is more than a Git merge mechanism. It is a governance, quality, security, collaboration, and audit control in the software-delivery process.

***

## 57. What are commit-message best practices?

A good commit message explains what changed and why the change was required.

### Recommended structure

```text
Short imperative summary

Additional explanation of the problem, technical decision,
risk, and implementation details.

Refs: AB#12345
```

Example:

```text
Handle payment API timeout gracefully

Return a retryable response when the upstream bank service
exceeds the configured timeout. This prevents the transaction
worker from failing without a valid status.

Refs: AB#12345
```

### Best practices

* Keep the subject concise and meaningful.
* Use imperative language such as `Add`, `Fix`, `Update`, or `Remove`.
* Explain why the change was needed.
* Keep each commit focused on one logical change.
* Reference the relevant issue, ticket, work item, or incident.
* Mention breaking changes when applicable.
* Avoid vague messages such as `changes`, `fix`, or `updated code`.
* Do not include secrets or sensitive customer information.
* Follow organizational conventions, such as Conventional Commits, if adopted.

Example using Conventional Commits:

```text
fix(payment-api): handle upstream timeout

Refs: AB#12345
```

> A useful test is whether another engineer can understand the purpose and impact of the commit six months later without opening every changed file.

***

## 58. What is the difference between `git blame` and `git log -S`?

`git blame` identifies the most recent commit associated with each current line of a file.

```bash
git blame src/payment-service.cs
```

Use it when you want to know:

* Who last changed a particular line
* Which commit introduced the current line
* When the current line was last modified

`git log -S` searches history for commits that changed the number of occurrences of a specific string.

```bash
git log -S "PaymentTimeout" --oneline
```

Search within a particular file:

```bash
git log -S "PaymentTimeout" -- src/payment-service.cs
```

Show associated patches:

```bash
git log -S "PaymentTimeout" -p
```

### Practical difference

* `git blame` starts from the current file and maps existing lines to commits.
* `git log -S` searches historical commits for additions or removals of a string.
* `git log -G` searches patches using a regular expression.

Example:

```bash
git log -G "Payment.*Timeout" -p
```

> Use `git blame` to investigate the current implementation. Use `git log -S` or `git log -G` to locate when a concept, variable, function, or configuration entered or left the history.

***

## 59. How do you re-stage files after partial commits?

Use patch mode to interactively select parts of a file:

```bash
git add -p
```

Apply patch mode to a particular file:

```bash
git add -p src/payment-service.cs
```

Git divides the changes into hunks and offers actions such as:

* `y`: Stage the current hunk
* `n`: Do not stage the current hunk
* `s`: Split the hunk into smaller hunks
* `e`: Manually edit the hunk
* `q`: Quit
* `?`: Display help

After staging selected hunks, inspect them:

```bash
git diff --staged
```

Commit them:

```bash
git commit -m "Add payment validation"
```

Then stage the remaining logical change:

```bash
git add -p
git commit -m "Improve payment error logging"
```

Interactively unstage selected changes:

```bash
git reset -p
```

A modern alternative is:

```bash
git restore --staged -p
```

> Patch staging supports atomic commits when one file contains changes for multiple tickets or logical purposes.

***

## 60. What is `git shortlog`?

`git shortlog` summarizes commit history by author.

```bash
git shortlog
```

A common form shows commit counts and author names:

```bash
git shortlog -s -n
```

Options:

* `-s`: Show only the commit count
* `-n`: Sort numerically by commit count
* `-e`: Include email addresses

Example:

```bash
git shortlog -sne
```

Summarize all branches:

```bash
git shortlog -sne --all
```

Summarize a release range:

```bash
git shortlog -sne v2.0.0..v2.1.0
```

### Use cases

* Generate contributor summaries
* Review release contributions
* Identify duplicate author identities
* Audit inconsistent names or email addresses
* Produce release documentation

> Commit counts should not be used as a standalone productivity metric. Commit size, complexity, testing, review, operational impact, and collaboration are not represented by raw counts.

***

## 61. How do you see which files changed in a commit?

Use:

```bash
git show --name-only <commit>
```

Example:

```bash
git show --name-only a7b31f2
```

Show file names without the patch:

```bash
git show --name-only --format=fuller a7b31f2
```

Show only the names, with minimal commit output:

```bash
git show --name-only --format="" a7b31f2
```

Show status letters:

```bash
git show --name-status a7b31f2
```

Example status letters:

* `A`: Added
* `M`: Modified
* `D`: Deleted
* `R`: Renamed
* `C`: Copied

Show a statistical summary:

```bash
git show --stat a7b31f2
```

Show exact changes:

```bash
git show a7b31f2
```

> `--name-status` is useful in deployment reviews because it shows both affected paths and the type of change.

***

## 62. What is `git archive` used for?

`git archive` creates an archive of files from a specified commit, branch, or tag without including the `.git` directory.

Create a ZIP archive from `main`:

```bash
git archive --format=zip --output=source.zip main
```

Create a compressed tar archive from a release tag:

```bash
git archive --format=tar.gz --output=payment-service-v2.1.0.tar.gz v2.1.0
```

Add a directory prefix inside the archive:

```bash
git archive \
  --format=zip \
  --prefix=payment-service-v2.1.0/ \
  --output=payment-service-v2.1.0.zip \
  v2.1.0
```

Archive a subdirectory:

```bash
git archive --format=zip --output=manifests.zip HEAD:manifests
```

### Use cases

* Create source-release packages
* Export a clean repository snapshot
* Provide code without Git metadata
* Package deployment manifests
* Generate audit evidence for a tagged release

> `git archive` packages tracked content from a Git tree. It generally does not include untracked files, working-directory changes, or independent submodule content.

***

## 63. How do you remove a file from Git history, and why must you be careful?

Removing a file from the latest version does not remove it from earlier commits. Sensitive or oversized content may require history rewriting.

The modern, preferred tool is generally `git filter-repo`.

Example:

```bash
git filter-repo --path config/secrets.json --invert-paths
```

For historical Git installations, `git filter-branch` may be encountered:

```bash
git filter-branch \
  --force \
  --index-filter \
  "git rm --cached --ignore-unmatch config/secrets.json" \
  --prune-empty \
  --tag-name-filter cat \
  -- --all
```

After rewriting, updated branches and tags may need to be force-pushed:

```bash
git push origin --force --all
git push origin --force --tags
```

### Why this is dangerous

History rewriting changes commit hashes for affected commits. This can:

* Disrupt open pull requests
* Break existing clones
* Invalidate commit signatures
* Affect tags and release references
* Require force-pushing
* Cause deleted content to reappear from an old clone
* Affect CI/CD or audit references based on commit hashes

### If a secret was committed

1. Revoke or rotate the secret immediately.
2. Investigate usage and exposure.
3. Remove the secret from the current branch.
4. Rewrite history if required by security policy.
5. Coordinate the force-push.
6. Ask developers to re-clone or carefully reset their repositories.
7. Update secret scanning and preventive controls.

> History removal is not a substitute for credential rotation. Once a secret is published, it must be treated as compromised.

***

## 64. What is `git gc`, and why run it?

`git gc` means Git garbage collection. It optimizes the repository by consolidating objects, compressing data, cleaning unnecessary files, and pruning eligible unreachable objects.

```bash
git gc
```

More aggressive optimization:

```bash
git gc --aggressive
```

Prune unreachable objects immediately:

```bash
git gc --prune=now
```

Use immediate pruning carefully because it can reduce recovery opportunities for recently deleted commits.

### Why run it?

* Reduce repository disk usage
* Improve some Git operations
* Pack loose objects
* Clean temporary data
* Maintain long-lived repositories
* Optimize repositories after extensive history rewriting

Git often runs maintenance automatically, so manual execution is not normally required.

For ongoing maintenance, modern Git also provides:

```bash
git maintenance start
```

> Avoid aggressive garbage collection during active repository operations. Ensure no other process is writing to the repository and preserve recovery references before pruning immediately.

***

## 65. What is `git fsck`?

`git fsck` checks the connectivity and validity of objects in a Git repository.

```bash
git fsck
```

A more comprehensive form is:

```bash
git fsck --full
```

It can identify:

* Missing objects
* Corrupt objects
* Broken references
* Dangling commits
* Dangling trees
* Dangling blobs
* Unreachable objects

Example output might include:

```text
dangling commit a7b31f2...
```

A dangling commit is not necessarily corruption. It may be an older commit left after rebase, reset, amend, or branch deletion.

Inspect it:

```bash
git show a7b31f2
```

Recover it:

```bash
git branch recovered-work a7b31f2
```

> Before attempting repository repair, create a backup and verify whether the remote repository or another clone contains a healthy copy.

***

## 66. What is `git reflog` used for?

`git reflog` records local movements of references such as `HEAD` and branch tips.

```bash
git reflog
```

It can show operations such as:

* Commits
* Resets
* Rebases
* Branch switches
* Amendments
* Merges
* Pulls
* Detached HEAD movements

Example:

```text
a7b31f2 HEAD@{0}: reset: moving to HEAD~2
d8e42a1 HEAD@{1}: commit: Add payment validation
```

Inspect a previous state:

```bash
git show HEAD@{1}
```

Recover it by creating a branch:

```bash
git branch recovered-payment-work HEAD@{1}
```

Show a branch-specific reflog:

```bash
git reflog show feature/payment-validation
```

### Important limitations

* Reflog is local to the repository.
* It is not pushed to GitHub or Azure Repos.
* Entries expire according to Git configuration.
* Garbage collection may eventually prune unreachable objects.

> Reflog is one of the most important Git recovery tools for accidental resets, deleted branches, failed rebases, and amended commits.

***

## 67. How do you recover a lost commit after a reset?

First, inspect the reflog:

```bash
git reflog
```

Suppose the output shows:

```text
a1b2c3d HEAD@{0}: reset: moving to HEAD~2
f4e5d6c HEAD@{1}: commit: Add payment retry logic
```

Inspect the lost commit:

```bash
git show f4e5d6c
```

Create a recovery branch:

```bash
git branch recovery/payment-retry f4e5d6c
```

Switch to it:

```bash
git switch recovery/payment-retry
```

Alternatively, restore the branch pointer directly:

```bash
git reset --hard f4e5d6c
```

The direct reset should be used only after confirming that the current state is no longer required.

If reflog does not reveal the commit, try:

```bash
git fsck --full --no-reflogs --unreachable
```

Then inspect candidate commits with:

```bash
git show <commit-hash>
```

> The safest recovery approach is to create a recovery branch first. This preserves both the recovered commit and the current branch while you validate the result.

***

## 68. How do you view branches that contain a commit?

List local branches containing a commit:

```bash
git branch --contains <commit>
```

Example:

```bash
git branch --contains a7b31f2
```

List remote-tracking branches:

```bash
git branch -r --contains a7b31f2
```

List both local and remote-tracking branches:

```bash
git branch -a --contains a7b31f2
```

Find branches that do not contain the commit:

```bash
git branch --no-contains a7b31f2
```

### Use cases

* Verify whether a hotfix reached a release branch.
* Confirm whether a commit is included in `main`.
* Check whether a branch can be safely deleted.
* Determine which supported versions contain a security fix.
* Investigate deployment discrepancies.

> Fetch the latest remote references before making remote-branch conclusions:

```bash
git fetch --all --prune
```

***

## 69. What is `git describe` used for?

`git describe` generates a human-readable name for a commit based on the nearest reachable tag.

```bash
git describe
```

Example output:

```text
v2.1.0-5-ga7b31f2
```

This means:

* Nearest tag: `v2.1.0`
* Number of commits since that tag: `5`
* Abbreviated commit hash: `a7b31f2`
* The `g` prefix indicates a Git commit identifier

Include lightweight tags:

```bash
git describe --tags
```

Always return a result even when no tag is reachable:

```bash
git describe --tags --always
```

Describe the current commit and include dirty working-tree status:

```bash
git describe --tags --always --dirty
```

### Use cases

* Generate development build versions
* Include source identity in application metadata
* Label non-release packages
* Troubleshoot deployed versions
* Create traceable container-image tags

> For official releases, use an immutable release tag. For development builds, combine `git describe` output with the complete commit hash and CI build number when stronger traceability is required.

***

## 70. What is `git clean`, and when should you use it?

`git clean` removes untracked files from the working directory.

Preview what would be removed:

```bash
git clean -n
```

The `-n` option performs a dry run.

Remove untracked files:

```bash
git clean -f
```

Remove untracked directories:

```bash
git clean -fd
```

Remove ignored files too:

```bash
git clean -fdx
```

Remove only ignored files:

```bash
git clean -fdX
```

Interactively select files:

```bash
git clean -i
```

### Appropriate use cases

* Clean a CI workspace
* Remove generated build files
* Reproduce a fresh checkout
* Remove temporary test output
* Troubleshoot contamination from previous builds

### Critical warning

Files removed by `git clean` are untracked, so Git usually cannot restore them using normal history or reflog.

Always preview first:

```bash
git clean -nd
git clean -ndx
```

> `git clean -fdx` is highly destructive because it removes both untracked and ignored content. This may include local configuration files, downloaded tools, credential files, build caches, and secrets.

***

## 71. How do you configure the default branch name globally?

Configure new repositories to use `main`:

```bash
git config --global init.defaultBranch main
```

Verify:

```bash
git config --global --get init.defaultBranch
```

New repositories initialized afterward will use:

```bash
git init
```

and start with `main` as the initial branch.

This does not rename existing branches.

Rename the current existing branch:

```bash
git branch -m main
```

If changing an existing remote repository, also update:

* The remote default branch
* Branch protection rules
* Pipeline triggers
* Pull-request targets
* Deployment configurations
* Documentation and badges
* Automation scripts

> Standardizing the default branch name reduces automation inconsistencies across new repositories.

***

## 72. What is the purpose of `HEAD` in Git?

`HEAD` represents the currently checked-out position in a Git repository.

Normally, it symbolically references a branch:

```text
HEAD -> main -> commit a7b31f2
```

View its symbolic target:

```bash
git symbolic-ref HEAD
```

Typical output:

```text
refs/heads/main
```

Show the commit currently referenced by `HEAD`:

```bash
git rev-parse HEAD
```

Common relative references include:

```text
HEAD      Current commit
HEAD^     First parent of the current commit
HEAD~1    One generation before HEAD
HEAD~3    Three generations before HEAD
```

Examples:

```bash
git show HEAD
git diff HEAD~1..HEAD
git reset --soft HEAD~1
```

In detached HEAD state, `HEAD` points directly to a commit rather than a branch.

> `HEAD` provides the context for committing, merging, rebasing, comparing, and navigating history.

***

## 73. What does `origin` commonly represent?

`origin` is the conventional name Git assigns to the remote repository from which a repository was cloned.

```bash
git clone https://github.com/company/payment-service.git
```

Git generally creates:

```text
origin
```

View it:

```bash
git remote -v
```

Examples:

```bash
git fetch origin
git pull origin main
git push origin feature/payment-validation
```

Important points:

* `origin` is not a special mandatory Git keyword.
* It can be renamed.
* It may point to any repository.
* It does not necessarily represent the authoritative repository.
* In a fork workflow, `origin` often represents the developer’s fork and `upstream` represents the original project.

Rename it:

```bash
git remote rename origin corporate
```

> Always inspect `git remote -v` before pushing from sensitive environments to ensure that the remote URL is correct.

***

## 74. What is `git show-ref`?

`git show-ref` lists references in a local repository and the object IDs to which they point.

```bash
git show-ref
```

Example:

```text
a7b31f2... refs/heads/main
b8c42d3... refs/remotes/origin/main
c9d53e4... refs/tags/v2.1.0
```

Show only branches:

```bash
git show-ref --heads
```

Show only tags:

```bash
git show-ref --tags
```

Check whether an exact reference exists:

```bash
git show-ref --verify refs/heads/main
```

Suppress normal output and rely on the exit code:

```bash
git show-ref --verify --quiet refs/heads/main
```

This is useful in scripts:

```bash
if git show-ref --verify --quiet refs/heads/main; then
    echo "main branch exists"
fi
```

> `git branch` and `git tag` provide user-friendly lists, while `git show-ref` is useful for low-level inspection and automation involving full reference names.

***

## 75. How do you cherry-pick a range of commits?

To include commits from `A` through `B`, inclusive:

```bash
git cherry-pick A^..B
```

The caret includes commit `A` by starting the range from its parent.

Example:

```bash
git cherry-pick a1b2c3d^..f4e5d6c
```

Without the caret:

```bash
git cherry-pick A..B
```

commit `A` is excluded.

### Recommended workflow

```bash
git switch release/2.1
git pull --ff-only
git switch -c backport/payment-fix
git cherry-pick a1b2c3d^..f4e5d6c
```

If conflicts occur:

```bash
git status
```

Resolve and stage them:

```bash
git add <resolved-files>
git cherry-pick --continue
```

Abort the entire operation:

```bash
git cherry-pick --abort
```

> Validate the commit range with `git log --oneline A^..B` before applying it. Ranges involving merge commits or non-linear history may require more careful selection.

***

## 76. How do you amend a commit?

Amend the latest commit with newly staged changes:

```bash
git add <files>
git commit --amend
```

Change only the latest commit message:

```bash
git commit --amend -m "Corrected commit message"
```

Add staged content without changing the message:

```bash
git commit --amend --no-edit
```

Example:

```bash
git add tests/payment-service-tests.cs
git commit --amend --no-edit
```

Amending creates a new commit with a new commit hash because the content or metadata changes.

If the commit has already been pushed, the remote update usually requires:

```bash
git push --force-with-lease
```

> Amend is ideal for correcting the latest unpublished commit, such as adding a forgotten file, correcting metadata, or improving the message.

***

## 77. When should you avoid amending a pushed commit?

Avoid amending a pushed commit when the commit is part of shared or public history.

Amending changes the commit hash. If other developers have based work on the original commit, they may experience:

* Divergent history
* Duplicate commits
* Rebase conflicts
* Rejected pushes
* Confusing pull-request updates
* Inconsistent audit references

Avoid amending when:

* The commit is on `main` or a release branch.
* Other developers are using the branch.
* The commit has been deployed.
* A release tag points to it.
* Compliance records reference the original hash.
* Branch policies prohibit history rewriting.

Instead, create a new corrective commit:

```bash
git add <files>
git commit -m "Correct configuration from previous change"
git push
```

If amendment is necessary on a private feature branch:

```bash
git push --force-with-lease
```

> Treat published commit hashes as immutable unless the team has explicitly coordinated the rewrite.

***

## 78. What is a merge commit?

A merge commit is a commit that joins two or more lines of development.

A normal non-merge commit usually has one parent. A merge commit typically has two parents:

1. The previous target-branch commit
2. The source-branch commit

Example history:

```text
      C---D  feature
     /     \
A---B---E---M  main
```

`M` is the merge commit.

Create one by merging divergent branches:

```bash
git switch main
git merge feature/payment-validation
```

Inspect parent relationships:

```bash
git show --summary <merge-commit>
```

Display parent hashes:

```bash
git show -s --format="%H %P" <merge-commit>
```

### Benefits

* Preserves branch topology
* Shows when integration occurred
* Groups feature commits under an integration point
* Supports auditing of parallel development

### Consideration

Too many unnecessary merge commits can make history difficult to read. Teams should standardize whether they use merge commits, squash merges, or rebase-based workflows.

***

## 79. What is `git merge --no-ff`?

`git merge --no-ff` forces Git to create a merge commit even when a fast-forward merge is possible.

```bash
git switch main
git merge --no-ff feature/payment-validation
```

Without `--no-ff`, Git might simply move the `main` pointer forward when no divergent commits exist.

With `--no-ff`, the branch relationship remains visible:

```text
      C---D
     /     \
A---B-------M
```

Add a merge message:

```bash
git merge --no-ff feature/payment-validation \
  -m "Merge payment validation feature"
```

### Advantages

* Preserves feature boundaries
* Makes feature-level reversion easier to understand
* Provides an explicit integration event
* Supports some audit workflows

### Disadvantages

* Creates additional merge commits
* Can make high-volume repository history noisy
* May be unnecessary for very small changes

> Use `--no-ff` only when preserving the branch integration point provides meaningful operational or audit value.

***

## 80. How do you list remote branches?

List locally known remote-tracking branches:

```bash
git branch -r
```

List both local and remote-tracking branches:

```bash
git branch -a
```

Refresh them first:

```bash
git fetch --all --prune
git branch -r
```

Query references directly from a remote:

```bash
git ls-remote --heads origin
```

List all remote references:

```bash
git ls-remote origin
```

List remote tags:

```bash
git ls-remote --tags origin
```

### Difference

* `git branch -r` displays local remote-tracking references from the last fetch.
* `git ls-remote` queries the remote server directly.
* `git fetch --prune` updates local references and removes stale tracking branches.

> When investigating whether a branch currently exists on GitHub or Azure Repos, `git ls-remote --heads origin` provides a direct remote query.

***

## 81. How do you set an upstream branch for a local branch?

Push a branch and establish upstream tracking:

```bash
git push -u origin feature/payment-validation
```

The longer form is:

```bash
git push --set-upstream origin feature/payment-validation
```

Afterward, these shorter commands can usually be used:

```bash
git push
git pull
```

Set tracking for an existing remote branch without pushing:

```bash
git branch --set-upstream-to=origin/feature/payment-validation
```

Short form:

```bash
git branch -u origin/feature/payment-validation
```

View tracking relationships:

```bash
git branch -vv
```

Remove the upstream association:

```bash
git branch --unset-upstream
```

> An upstream branch provides the default push, pull, comparison, ahead, and behind relationship for the local branch.

***

## 82. What is `git pull --rebase`, and why use it?

`git pull --rebase` fetches remote changes and then rebases local commits onto the updated remote branch instead of creating a merge commit.

```bash
git pull --rebase origin main
```

Conceptually:

```bash
git fetch origin
git rebase origin/main
```

### Why use it?

* Maintains a more linear history
* Avoids unnecessary merge commits from routine pulls
* Places local work on top of the latest remote work
* Makes feature-branch history easier to review

### If conflicts occur

Resolve the files and stage them:

```bash
git add <resolved-files>
git rebase --continue
```

Abort:

```bash
git rebase --abort
```

Configure it globally:

```bash
git config --global pull.rebase true
```

Configure it only for the current repository:

```bash
git config pull.rebase true
```

### Caution

Rebase recreates local commits with new hashes. It is safe for normal unpublished local commits, but shared history should not be rewritten casually.

> `git pull --rebase` is often preferred on developer feature branches, while protected branch integration should follow the repository’s approved pull-request strategy.

***

## 83. How do you configure automatic CRLF handling?

Git uses `core.autocrlf` to manage line-ending conversion between:

* LF: Common on Linux and macOS
* CRLF: Common on Windows

### Common Windows setting

```bash
git config --global core.autocrlf true
```

This generally converts LF to CRLF during checkout and converts CRLF back to LF when committing.

### Common Linux or macOS setting

```bash
git config --global core.autocrlf input
```

This converts CRLF to LF when committing but does not convert LF to CRLF during checkout.

### Disable automatic conversion

```bash
git config --global core.autocrlf false
```

### Recommended repository-level control

Use `.gitattributes` to establish consistent behavior:

```gitattributes
* text=auto

*.sh text eol=lf
*.yaml text eol=lf
*.yml text eol=lf
*.ps1 text eol=crlf
*.bat text eol=crlf
*.cmd text eol=crlf

*.png binary
*.jpg binary
*.zip binary
```

After introducing normalized rules:

```bash
git add --renormalize .
git status
git diff --staged
```

Review carefully before committing.

> `.gitattributes` is more predictable for cross-platform teams because the policy travels with the repository rather than depending only on each developer’s global configuration.

***

## 84. What is `git blame -L`?

`git blame -L` limits blame output to a selected line range or function.

Show lines 50 through 80:

```bash
git blame -L 50,80 src/payment-service.cs
```

Start from line 50 and display the next 20 lines:

```bash
git blame -L 50,+20 src/payment-service.cs
```

Show a range ending before a pattern:

```bash
git blame -L 50,/return/ src/payment-service.cs
```

In supported source formats, blame a function by name:

```bash
git blame -L :ProcessPayment src/payment-service.cs
```

Ignore whitespace-only changes:

```bash
git blame -w -L 50,80 src/payment-service.cs
```

> The line-range option is useful for large files because it limits the investigation to the specific function or block associated with an incident.

***

## 85. How do you check file differences between branches?

Compare the tips of two branches:

```bash
git diff branchA..branchB
```

Example:

```bash
git diff main..feature/payment-validation
```

Compare only a particular file:

```bash
git diff main..feature/payment-validation -- src/payment-service.cs
```

Display only changed file names:

```bash
git diff --name-only main..feature/payment-validation
```

Display change status:

```bash
git diff --name-status main..feature/payment-validation
```

Display a statistical summary:

```bash
git diff --stat main..feature/payment-validation
```

### Two-dot versus three-dot comparison

Two dots compare the two branch tips:

```bash
git diff main..feature
```

Three dots compare the merge base with the source branch tip:

```bash
git diff main...feature
```

The three-dot comparison is often useful for reviewing what the feature branch introduced relative to its common ancestor with `main`.

> Fetch remote changes before comparison when accuracy against the current remote state is important.

***

## 86. How do you find commits by author?

Search by author name:

```bash
git log --author="Rajesh Singh"
```

Search using an email pattern:

```bash
git log --author="rajesh.singh@company.com"
```

Concise output:

```bash
git log --author="Rajesh Singh" --oneline
```

Limit by date:

```bash
git log \
  --author="Rajesh Singh" \
  --since="2026-07-01" \
  --until="2026-07-24"
```

Search all branches:

```bash
git log --all --author="Rajesh Singh" --oneline
```

Show file-change statistics:

```bash
git log --author="Rajesh Singh" --stat
```

Git treats the author and committer as separate identities. Search by committer when necessary:

```bash
git log --committer="Build Automation"
```

> Author searches may reveal duplicate identities if different email addresses or spelling variations were used. A `.mailmap` file can normalize how contributors appear in reports.

***

## 87. How do you find commits touching a specific file?

Use:

```bash
git log -- <file>
```

Example:

```bash
git log -- src/payment-service.cs
```

Concise output:

```bash
git log --oneline -- src/payment-service.cs
```

Show the actual patches:

```bash
git log -p -- src/payment-service.cs
```

Follow the file across renames:

```bash
git log --follow -- src/payment-service.cs
```

Show change statistics:

```bash
git log --stat -- src/payment-service.cs
```

Search all references:

```bash
git log --all -- src/payment-service.cs
```

Search for changes involving a string in that file:

```bash
git log -S "PaymentTimeout" -p -- src/payment-service.cs
```

> `--follow` works best for a single path and Git’s rename detection is similarity-based, so major rewrites may make earlier history difficult to follow automatically.

***

## 88. What is `git daemon`?

`git daemon` is a lightweight Git server that serves repositories using the native Git protocol.

A basic example is:

```bash
git daemon \
  --reuseaddr \
  --base-path=/srv/git \
  /srv/git
```

It commonly serves repositories through URLs such as:

```text
git://server.example.com/project.git
```

### Characteristics

* Lightweight
* Fast for Git object transfer
* Suitable for simple read-only repository distribution
* Limited authentication and authorization
* No built-in pull-request workflow
* No advanced enterprise governance
* Traffic is not inherently protected like SSH or HTTPS

Repositories may need an export marker:

```bash
touch /srv/git/project.git/git-daemon-export-ok
```

> `git daemon` is rarely the preferred choice for enterprise write access. GitHub, Azure Repos, GitLab, SSH, and HTTPS provide stronger identity, encryption, authorization, audit, and collaboration capabilities.

***

## 89. How do you set up a bare repository, and why?

A bare repository contains Git objects and references but has no checked-out working directory.

Create one:

```bash
git init --bare payment-service.git
```

A shared server location might be:

```bash
mkdir -p /srv/git
cd /srv/git
git init --bare payment-service.git
```

Add it as a remote:

```bash
git remote add origin ssh://git@server/srv/git/payment-service.git
git push -u origin main
```

### Why use a bare repository?

* It can act as a central Git repository.
* Developers can push without interfering with a checked-out working tree.
* Server-side hooks can be installed.
* It is suitable for internal Git hosting and repository mirrors.

Create a bare clone of an existing repository:

```bash
git clone --bare existing-repository payment-service.git
```

For a full mirror including all references:

```bash
git clone --mirror <repository-url> payment-service.git
```

> A normal repository should not usually be used as a shared push target because updating its checked-out branch can create working-tree inconsistencies.

***

## 90. What is the `index.lock` error, and how do you resolve it?

Git creates `.git/index.lock` to prevent multiple processes from writing to the repository index simultaneously.

A typical error is:

```text
fatal: Unable to create '.git/index.lock': File exists.
```

### Possible causes

* Another Git command is currently running.
* An IDE or Git client is performing an operation.
* A previous Git process crashed.
* A pipeline job was terminated.
* Antivirus or filesystem tools are interfering.
* Two automation jobs share the same workspace.

### Safe resolution

First, verify that no Git process is active.

Linux:

```bash
ps -ef | grep git
```

Windows PowerShell:

```powershell
Get-Process git -ErrorAction SilentlyContinue
```

Inspect repository status:

```bash
git status
```

If no Git process is using the repository and the lock is stale, remove it:

Linux:

```bash
rm -f .git/index.lock
```

PowerShell:

```powershell
Remove-Item .git\index.lock -Force
```

Then verify repository integrity:

```bash
git status
git fsck
```

> Never remove `index.lock` while another Git operation is active. Doing so can corrupt the index or interrupt an important repository update.

For CI/CD, allocate a separate workspace per job instead of allowing parallel jobs to use the same checkout.

***

## 91. How do you configure multiple remotes for one repository?

Add additional remotes using distinct names:

```bash
git remote add origin git@github.com:company/payment-service.git
git remote add upstream git@github.com:platform/payment-service.git
```

Verify:

```bash
git remote -v
```

Fetch from all remotes:

```bash
git fetch --all --prune
```

Fetch from a specific remote:

```bash
git fetch upstream
```

Push to a particular remote:

```bash
git push origin main
```

### Common use cases

* Fork workflow
* Migration between Git platforms
* GitHub and Azure Repos replication
* Read-only upstream with a writable fork
* Disaster-recovery mirror
* Separate fetch and push locations

Configure a different push URL:

```bash
git remote set-url --add --push origin \
  git@github.com:company/payment-service.git
```

Add another push destination:

```bash
git remote set-url --add --push origin \
  git@secondary-server:company/payment-service.git
```

Review carefully:

```bash
git remote get-url --all origin
git remote get-url --all --push origin
```

> Multi-remote replication must be designed carefully. Branch policies, pull requests, tags, permissions, Git LFS objects, hooks, and platform metadata may not automatically replicate with normal Git pushes.

***

## 92. How do you push tags to a remote?

Push a specific tag:

```bash
git push origin v2.1.0
```

Push all local tags that are missing remotely:

```bash
git push origin --tags
```

Push annotated tags reachable from pushed commits:

```bash
git push origin --follow-tags
```

Delete a remote tag:

```bash
git push origin --delete v2.1.0
```

Alternative deletion syntax:

```bash
git push origin :refs/tags/v2.1.0
```

Verify remote tags:

```bash
git ls-remote --tags origin
```

### CI/CD recommendation

Push official release tags individually:

```bash
git push origin v2.1.0
```

This reduces the chance of accidentally publishing local test or temporary tags.

> Protect production release tags where the hosting platform supports tag-protection or immutable-release policies.

***

## 93. How do you list all tags?

List local tags:

```bash
git tag -l
```

The shorter form also works:

```bash
git tag
```

Filter by pattern:

```bash
git tag -l "v2.*"
```

Sort by semantic version:

```bash
git tag -l --sort=-version:refname
```

Show tags with annotation subjects:

```bash
git tag -n
```

Show tags pointing to the current commit:

```bash
git tag --points-at HEAD
```

Find tags containing a commit:

```bash
git tag --contains <commit>
```

Display remote tags:

```bash
git ls-remote --tags origin
```

Refresh locally known tags:

```bash
git fetch origin --tags
```

> Local tag lists may not represent the latest remote state until tags have been fetched.

***

## 94. What is `git merge --squash`?

`git merge --squash` combines the net changes from another branch and stages them in the current branch without creating a merge commit and without recording the source branch as a parent.

```bash
git switch main
git merge --squash feature/payment-validation
git commit -m "Add payment validation feature"
```

Suppose the feature branch contains five commits. A squash merge creates one new commit on `main` containing their combined changes.

### Benefits

* Keeps the target-branch history concise
* Removes temporary or noisy feature commits
* Provides one commit per feature or pull request
* Makes simple feature rollback easier

### Limitations

* Does not preserve detailed branch topology.
* The original feature commits remain only on the feature branch.
* Git does not record that the branch was formally merged.
* Repeated squash merges from a long-lived branch can be difficult to manage.

> Squash merging works well with short-lived feature branches and pull requests where the target branch requires a clean, curated history.

***

## 95. How do you perform a shallow clone?

Clone only the latest commit:

```bash
git clone --depth 1 https://github.com/company/payment-service.git
```

Clone a specific branch shallowly:

```bash
git clone \
  --depth 1 \
  --branch main \
  --single-branch \
  https://github.com/company/payment-service.git
```

Increase the available history:

```bash
git fetch --deepen=50
```

Convert to a full clone:

```bash
git fetch --unshallow
```

Check whether the repository is shallow:

```bash
git rev-parse --is-shallow-repository
```

### Benefits

* Faster clone
* Reduced network traffic
* Less disk usage
* Useful for some CI build jobs

### Limitations

A shallow clone may affect:

* History-based version generation
* `git blame`
* `git bisect`
* Merge-base calculations
* Changelog generation
* Security and compliance scanning
* Operations requiring older commits or tags

> Use the smallest depth that still satisfies the build, testing, versioning, and audit requirements.

***

## 96. What is the purpose of `git blame --show-email`?

`git blame --show-email` displays the author’s email address instead of only the author’s name.

```bash
git blame --show-email src/payment-service.cs
```

Limit it to relevant lines:

```bash
git blame --show-email -L 50,80 src/payment-service.cs
```

### Use cases

* Distinguish contributors with similar names
* Identify duplicate author identities
* Contact the relevant technical owner
* Investigate commits created using an incorrect identity
* Support audit and contribution analysis

### Privacy and governance consideration

Email addresses may be personal information. Use this output only for legitimate engineering, operational, or audit purposes and follow organizational privacy requirements.

> Use blame information to find implementation context and expertise, not to assign personal fault.

***

## 97. How do you change the author of past commits?

For the latest commit:

```bash
git commit --amend \
  --author="Rajesh Singh <rajesh.singh@company.com>" \
  --no-edit
```

For several recent commits, start an interactive rebase:

```bash
git rebase -i HEAD~5
```

Change `pick` to `edit` for each commit that requires correction:

```text
edit a1b2c3d Add payment validation
pick b2c3d4e Add tests
edit c3d4e5f Update configuration
```

When Git pauses:

```bash
git commit --amend \
  --author="Rajesh Singh <rajesh.singh@company.com>" \
  --no-edit

git rebase --continue
```

Repeat until the rebase completes.

An alternative is to set environment variables while amending:

```bash
GIT_COMMITTER_NAME="Rajesh Singh" \
GIT_COMMITTER_EMAIL="rajesh.singh@company.com" \
git commit --amend \
  --author="Rajesh Singh <rajesh.singh@company.com>" \
  --no-edit
```

### Important impact

Every amended commit and its descendants receive new hashes. If the branch was pushed:

```bash
git push --force-with-lease
```

> Avoid rewriting shared or released history solely to correct cosmetic identity information. Coordinate the impact and preserve audit records where required.

***

## 98. What is the difference between `credential-store` and `credential-cache`?

Both are Git credential helpers, but they store credentials differently.

### Credential cache

```bash
git config --global credential.helper cache
```

Set a timeout:

```bash
git config --global credential.helper "cache --timeout=3600"
```

Characteristics:

* Stores credentials temporarily in memory
* Credentials expire after a configured period
* Does not normally persist across restarts
* More secure than plaintext persistent storage
* Primarily available in Unix-like environments

### Credential store

```bash
git config --global credential.helper store
```

Characteristics:

* Stores credentials persistently on disk
* Commonly writes to `~/.git-credentials`
* Credentials are generally stored in plaintext
* Survives restarts
* Inappropriate for shared or sensitive systems

Remove stored credentials by editing or securely removing the applicable credential file after confirming its location.

### Preferred enterprise approach

Use an operating-system-backed secure helper such as Git Credential Manager:

```bash
git config --global credential.helper manager
```

> `credential-cache` is temporary and memory-based. `credential-store` is persistent but normally plaintext. For enterprise systems, prefer Git Credential Manager, SSH keys, or a secure secrets-management integration.

***

## 99. How do you use `git worktree`, and what problem does it solve?

`git worktree` allows multiple working directories to be attached to the same repository data. Each worktree can check out a different branch.

This solves the problem of repeatedly stashing changes or cloning the repository multiple times when working on parallel branches.

### Add another worktree

From the primary repository:

```bash
git worktree add ../payment-hotfix hotfix/payment-timeout
```

Create a new branch and worktree:

```bash
git worktree add -b hotfix/payment-timeout \
  ../payment-hotfix \
  main
```

List worktrees:

```bash
git worktree list
```

Remove a worktree:

```bash
git worktree remove ../payment-hotfix
```

Clean stale worktree metadata:

```bash
git worktree prune
```

### Practical use cases

* Develop a feature while investigating a production issue.
* Maintain multiple supported release branches.
* Run comparisons between versions.
* Execute builds for separate branches.
* Review a pull request without disturbing current work.
* Test a hotfix while retaining uncommitted feature changes.

### Important restriction

The same branch normally cannot be checked out simultaneously in multiple worktrees.

> Worktrees share Git objects, so they use less disk and network bandwidth than separate full clones while maintaining independent working directories and indexes.

***

## 100. What is `git submodule update --init --recursive` used for?

This command initializes, fetches, and checks out the commits recorded for submodules, including nested submodules:

```bash
git submodule update --init --recursive
```

The options mean:

* `update`: Check out the commit recorded by the parent repository.
* `--init`: Initialize submodules that have not yet been initialized.
* `--recursive`: Process nested submodules too.

### Clone and initialize in one operation

```bash
git clone --recurse-submodules \
  https://github.com/company/payment-service.git
```

If the repository was already cloned:

```bash
git submodule update --init --recursive
```

### Update submodule URLs after `.gitmodules` changes

```bash
git submodule sync --recursive
git submodule update --init --recursive
```

### Update submodules from their configured remote branches

```bash
git submodule update --remote --recursive
```

This is different from checking out the exact commits recorded by the parent repository. After a remote update, the parent repository must commit the new submodule references if they are intended to be shared.

### CI/CD usage

A pipeline checking out a repository with submodules must ensure:

* Submodule checkout is enabled.
* Credentials can access every submodule repository.
* Nested submodules are handled.
* The required SSH host keys or HTTPS credentials are available.
* The exact referenced submodule commits still exist.

> The parent repository records specific submodule commit IDs. Running `git submodule update --init --recursive` reproduces those defined dependency versions rather than automatically selecting each submodule’s latest commit.

***

# Senior DevOps Quick Revision

## Authentication and credentials

* Use Git Credential Manager or SSH keys for secure authentication.
* Avoid plaintext credential storage.
* Never embed tokens in Git URLs, scripts, or pipeline YAML.
* Use separate service identities for CI/CD automation.

## History and recovery

* Use `git reflog` to recover commits after reset, amend, or rebase.
* Use `git fsck` to inspect repository integrity and unreachable objects.
* Create a recovery branch before performing destructive repair.
* Rotate exposed secrets before rewriting Git history.

## Repository maintenance

* Use `git gc` for repository optimization when necessary.
* Run `git clean -n` before deleting untracked content.
* Resolve `index.lock` only after verifying that no Git process is active.
* Use shallow clones only when the pipeline does not require full history.

## Enterprise workflow

* Protect release tags and critical branches.
* Enforce pull-request policies in GitHub or Azure Repos.
* Use `.gitattributes` for cross-platform line-ending consistency.
* Store build artifacts outside source repositories.
* Use Git LFS only for genuinely version-controlled large files.
* Use worktrees for parallel branch activity.
* Pin submodules to known commit IDs for reproducible builds.

