# Day 5: Git Internals & Advanced Version Control
**Date:** 7th November

## 1. Repository Management: Local vs Remote vs Distributed

### Distributed Version Control System (DVCS)
Git is distributed, meaning every developer has a full copy of the entire repository history on their local machine.
*   **Local Repository:** Stored on your laptop (`.git` folder). No internet needed.
*   **Remote Repository:** Hosted on a server (GitHub/GitLab). Used for sharing code.
*   **Centralized (Old way - SVN):** One central server. If it goes down, no one can work. Git solves this.

### The `.git` Directory (Internals)
When you run `git init`, this folder is created.
*   **HEAD:** Pointer to current branch ref.
*   **config:** Local configuration (remotes, user).
*   **objects/:** Database of blobs, trees, and commits (The DAG).
*   **refs/:** Pointers to commit hashes (tags, heads, remotes).

## 2. Essential Commands Deep Dive

### Starting a Project
1.  **Init:** `git init` (Turns current folder into a repo).
2.  **Clone:** `git clone <url>` (Downloads a repo from remote).
    *   `git clone --depth 1 <url>`: Shallow clone (faster, no history).

### The "Save" Cycle
1.  **Add:** `git add filename` moves a file to the **Staging Area** (Index).
    *   `git add .`: Stage everything.
    *   `git add -p`: Interactive patch staging (stage parts of a file).
2.  **Commit:** `git commit -m "message"` saves the staged changes to the **Local Repo**.
    *   `git commit --amend`: Modify the *last* commit (fix typo).
3.  **Push:** `git push origin main` saves the commits to the **Remote Repo**.
    *   `git push -u origin main`: Set upstream tracking.

### Inspecting Changes
1.  **Log:** `git log` (Show commit history).
    *   `git log --oneline --graph --all`: The "Network Graph" view.
2.  **Diff:** `git diff` (Show what changed but isn't staged yet).
    *   `git diff --staged` (Show what is staged but not committed).
    *   `git diff HEAD~1 HEAD`: Compare last commit with one before.

---

## 3. Workflows: Merge vs Rebase

### Branching
A branch is a parallel version of your repository.
*   **Create:** `git branch feature1`
*   **Switch:** `git checkout feature1` (or `git switch feature1`)
*   **Create & Switch:** `git checkout -b feature1`

### The Merge Strategy (`git merge feature1`)
*   **Creating a Merge Commit:** Preserves history exactly as it happened.
*   **Concept:** "I happened at the same time as you, let's join hands."
*   **Pros:** Non-destructive. True history.
*   **Cons:** "Pollutes" history with merge commits.

### The Rebase Strategy (`git rebase main`)
*   **Rewriting History:** Moves your feature branch commits to the *tip* of main.
*   **Concept:** "I want to pretend I wrote my code *after* your latest changes."
*   **Pros:** Linear, clean history.
*   **Cons:** Dangerous on shared branches (rewrites Hash IDs).
*   **Interactive Rebase (`-i`):**
    ```bash
    git rebase -i HEAD~3
    ```
    Allows you to `squash` (combine 3 commits into 1), `edit`, or `drop`.

---

## 4. Advanced: Recovery & Debugging

### Git Reflog (`git reflog`)
The Safety Net. Git records *every* time `HEAD` moves (checkout, commit, reset).
*   **Scenario:** You accidentally `git reset --hard` and lost your work.
*   **Fix:**
    1.  `git reflog` -> Find the hash `abc1234` *before* the reset.
    2.  `git reset --hard abc1234`.
    3.  **Magic:** Code is back.

### Git Stash (`git stash`)
Save dirty changes without committing.
*   `git stash`: Save.
*   `git stash list`: Show stack.
*   `git stash pop`: Apply and remove from stack.
*   `git stash apply`: Apply and keep in stack.
*   **Scenario:** You are on `main`, start coding, realize you should be on `feature`. Stash -> Checkout -> Pop.

### Git Bisect (`git bisect`)
Binary search for bugs.
*   **Scenario:** v1.0 was good. v2.0 is bad. There are 100 commits in between. Which one broke it?
*   **Steps:**
    1.  `git bisect start`
    2.  `git bisect bad` (Current)
    3.  `git bisect good v1.0` (Old known good)
    4.  Git checks out the middle commit. You test.
    5.  `git bisect bad` or `git bisect good`.
    6.  Repeat until Git says: "Commit xyz is the first bad commit".

---

## 5. Interview Questions
1.  **Q: What is a "Detached HEAD"?**
    *   *A:* When you check out a specific Commit Hash instead of a Branch. You are not on any branch. New commits will be lost if you switch away (unless you create a branch there).
2.  **Q: Difference between `git fetch` and `git pull`?**
    *   *A:* `pull` = `fetch` + `merge`. `fetch` downloads changes but doesn't touch your working code. `pull` updates your code immediately.
3.  **Q: How do you handle large binary files?**
    *   *A:* Use **Git LFS** (Large File Storage). Git tracks a pointer file, while the binary blob goes to a separate store (S3/Artifactory).
4.  **Q: What is `.gitignore`?**
    *   *A:* A file listing patterns to exclude from version control (e.g., `node_modules/`, `.env`, `*.log`).
