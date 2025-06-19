# BASICS 

Git is a distributed version control system designed for speed, data integrity, and collaboration.
🔧 High-Level Git Architecture
```
          +-----------------+
          |   Working Tree  |  ← you edit files here
          +-----------------+
                   |
                   v
          +-----------------+
          |     Staging     |  ← you add files here using `git add`
          |    (Index)      |
          +-----------------+
                   |
                   v
          +-----------------+
          |     Local       |  ← you commit here using `git commit`
          |   Repository    |
          +-----------------+
                   |
            +--------------+
            |   Remotes    |  ← sync via `git push` / `git pull`
            +--------------+

```

🧠 Components Explained
1. Working Directory (Working Tree)
- This is your actual project folder.
- You can: - Modify files   - Delete files   - Create new files
- Files here are untracked, modified, or clean
💬 "Where you code and change things"

2. Staging Area (Index)
- Temporary holding area for changes you want to commit.
- You use git add <file> to move changes here.
- Think of it like: “These are the changes I want to save next”
💬 "Prepares the snapshot for the next commit"

3. Local Repository (.git folder)
- Once you commit, changes are stored in the local repo:
   * Each commit is a snapshot, not a diff.
   * Stored in .git/objects, .git/refs, etc.
- Contains:
   * Commits  
   * Branches
   * Tags
   * HEAD pointer
   * History (via reflog)
💬 "Your project's full history lives here"

4. Remote Repository
  - Shared repo (usually on GitHub, GitLab, Bitbucket).
  - You push and pull from here.
  - Enables collaboration across developers.
💬 "The central/shared copy"

📂 Inside .git Directory
| Folder/File     | Purpose                                 |
| --------------- | --------------------------------------- |
| `.git/objects/` | All commits, trees, blobs, tags as SHA1 |
| `.git/refs/`    | Pointers to branches/tags               |
| `.git/HEAD`     | Points to current branch                |
| `.git/index`    | The staging area                        |
| `.git/config`   | Repo-specific config                    |
| `.git/hooks/`   | Client-side hooks (e.g. pre-commit)     |


🧱 Git Objects
Git stores everything as 4 object types:
| Object Type | Example       | Description                                  |
| ----------- | ------------- | -------------------------------------------- |
| **Blob**    | File contents | Stores the file's content                    |
| **Tree**    | Directory     | Maps filenames → blobs/trees                 |
| **Commit**  | Snapshot      | Points to tree + parent commit               |
| **Tag**     | Tag           | Marks a specific commit (like a version tag) |

=========================================================================================

Here are **commonly asked Git interview questions** tailored for someone with **1 year of DevOps experience**, along with **precise, scenario-based answers**:

---

### 🔹 1. **What is the difference between `git pull` and `git fetch`?**

**Question Intent:** Test your understanding of syncing remote changes.

**Answer:**

* `git fetch`: Downloads changes from the remote but does **not** apply them to your current branch.
* `git pull`: Does a `git fetch` **+** `git merge` — applies the downloaded changes immediately.

✅ **Best Practice:** In CI/CD or automated scripts, prefer `git fetch` + manual merge to avoid unexpected changes.

---

### 🔹 2. **How do you resolve a merge conflict in Git?**

**Answer:**

* When merging branches (e.g., `git merge feature-xyz`), if two branches modify the same line, Git throws a conflict.
* You:

  1. Manually edit the conflict markers `<<<<<<<`, `=======`, `>>>>>>>`
  2. Run `git add <file>`
  3. Then `git commit` (or `git merge --continue`)

✅ Use `git merge --abort` if you want to back out of the merge safely.

---

### 🔹 3. **What is `reflog` and when did you use it?**

**Answer:**

* `git reflog` shows the **history of HEAD movements**, including commits, resets, rebases.
* It’s useful when you:

  * Accidentally run `git reset --hard`
  * Lose your branch pointer
  * Want to recover deleted commits

```bash
git reflog
git checkout HEAD@{2}
```

---

### 🔹 4. **Explain the Git commit lifecycle in a CI/CD pipeline context.**

**Answer:**

* **Dev writes code → `git add` → `git commit` → `git push`**
* CI tool (e.g., Jenkins, GitHub Actions) **listens to push**
* Trigger: `on: push`
* Pulls latest commit: `git checkout <commit SHA>`
* Builds, tests, deploys

✅ Git commit is the **trigger point** for DevOps automation.

---

### 🔹 5. **How would you revert a deployed commit in production?**

**Answer:**

* Use `git revert <commit>` to generate a new commit that undoes a previous one.
* Safe for production because it doesn't rewrite history.

```bash
git revert abc123
git push origin main
```

✅ Avoid `reset` in production workflows as it can break shared history.

🔙 git reset (3 modes)
1️⃣ --soft: Move HEAD only (keep index & working directory)




---

### 🔹 6. **What is a detached HEAD and how do you fix it?**

**Answer:**

* Happens when you check out a specific commit (not a branch):

  ```bash
  git checkout abc123
  ```
* HEAD is detached → changes are not tied to a branch.

**Fix:**

```bash
git checkout -b hotfix/temp-fix
```

---

### 🔹 7. **How do you squash commits before merging a branch?**

**Answer:**

* Use interactive rebase:

  ```bash
  git rebase -i HEAD~3
  ```
* Mark all except the first as `squash` or `s`
* Then `git push --force-with-lease`

✅ Useful in DevOps to keep deployment branches clean.

---

### 🔹 8. **What is the `.gitignore` file used for?**

**Answer:**

* It tells Git which files/folders to ignore (e.g., logs, secrets, build artifacts)
* Example:

  ```
  *.log
  node_modules/
  .env
  ```

✅ Helps prevent leaking credentials or unnecessary files in the repo.

---

### 🔹 9. **How do you rollback a Git tag-based deployment?**

**Answer:**

* Checkout an older tag:

  ```bash
  git checkout tags/v1.0.1
  ```
* Redeploy or re-tag:

  ```bash
  git checkout -b rollback-v1.0.1
  ```

✅ Useful for rolling back production in a controlled way.

---

### 🔹 10. **Have you handled Git in team conflicts?**

**Answer (Sample):**

> Yes, during a team merge into `main`, we had a conflict in `app.py`. I used `git diff` to analyze changes, collaborated with the other dev, manually resolved it, staged via `git add`, and completed the merge.

---

## ✅ BONUS Tips for 1-Year DevOps Interviews

| Concept                     | Likely Question                                  |
| --------------------------- | ------------------------------------------------ |
| CI/CD Flow                  | How Git triggers your pipeline                   |
| `git revert` vs `git reset` | Which to use in shared repo?                     |
| SSH vs HTTPS in Git         | How do you connect Git to your CI/CD system?     |
| Git tags                    | How do you deploy using tags?                    |
| Git hooks                   | Have you ever used `pre-commit` or `post-merge`? |

---

Let me know if you want:

* Questions **only on GitOps/GitHub Actions**
* Git-related **error resolution scenarios** (e.g., "non-fast-forward", "push rejected")
* A **one-page Git cheat sheet** for DevOps engineers

Want to go into branching, cherry-pick, or multi-dev workflow next?


# =========================================================================================

 **Git commands with their real-world use cases and scenarios**, especially relevant to **DevOps**, **collaboration**, and **troubleshooting**:

---

### 🔁 `git revert`

| ✅ Purpose                  | **Safely undo a specific commit** by creating a new commit |
| -------------------------- | ---------------------------------------------------------- |
| ✅ Safe on shared branches? | Yes                                                        |

#### 🔧 Use Cases:

* You deployed a bad commit to production and need to roll it back.
* You want to **undo a change**, but keep your Git history intact for auditing.

```bash
git revert <commit-hash>
```

🧠 Think of it as an "anti-commit" that negates the selected commit.

→ Adds a new commit that undoes a previous one without touching history.
```
Before:
A -- B -- C -- D   ← main (bug here)

After `git revert D`:
A -- B -- C -- D -- D'   ← main
                        ↑
                   (revert of D)

```
✅ Use case: Revert a bad production config but keep history clean.


---

### 🔄 `git rebase`

| ✅ Purpose                  | **Rewrites commit history** by placing your commits on a new base |
| -------------------------- | ----------------------------------------------------------------- |
| ❌ Safe on shared branches? | No (it rewrites history!)                                         |

#### 🔧 Use Cases:

* You're working on `feature-A` and want to integrate latest `main` cleanly:

```bash
git checkout feature-A
git rebase main
```

* You want to **squash** 5 commits into 1 before a pull request:

```bash
git rebase -i HEAD~5
```

* Clean up commit messages before merging for **cleaner history**.

→ Moves your feature commits to the top of latest main. History is rewritten.
```
Before:
main:    A -- B -- C
feature:           \ 
                    D -- E

After `git rebase main` on feature:
main:    A -- B -- C
                      \
feature (rebased):      D' -- E'

```
✅ Use case: Bring your feature branch up-to-date cleanly before PR.


---

### 🔙 `git reset`

| ✅ Purpose                  | Move HEAD (and optionally staging & working directory) to an earlier state |
| -------------------------- | -------------------------------------------------------------------------- |
| ❌ Safe on shared branches? | No (destructive if `--hard`)                                               |

#### 🔧 Use Cases:

* You accidentally committed secrets or debug code and want to undo it:

```bash
git reset --soft HEAD~1
```

* You want to **unstage files** but keep changes:

```bash
git reset HEAD <file>
```

* You want to wipe out local changes:

```bash
git reset --hard HEAD
```
-------------------------------------------------------------------

🔙 git reset (3 modes)
----------------------
1️⃣ --soft: Move HEAD only (keep index & working directory)
```
HEAD: moves back, files stay staged.

A -- B -- C (HEAD)
       ↑
    reset to B
```

2️⃣ --mixed (default): Move HEAD & unstage files
```
HEAD: moves back, changes stay but **unstaged**
```

3️⃣ --hard: Move HEAD & discard all changes
```
A -- B (HEAD now here)
All changes in working/staged area are **gone**
```
⚠️ Dangerous but useful for full reset.


---

### 🍒 `git cherry-pick`

| ✅ Purpose                  | Apply a **specific commit** from another branch |
| -------------------------- | ----------------------------------------------- |
| ✅ Safe on shared branches? | Yes                                             |

#### 🔧 Use Cases:

* You fixed a bug in `main` and want to apply it to `release/1.0`:

```bash
git checkout release/1.0
git cherry-pick <bugfix-commit>
```

* You’re working on a hotfix and only want **1 commit** from a long feature branch.

→ Copy a single commit from one branch to another.
```
dev:     A -- B -- C
                      ↑
prod:    X -- Y       |  ← cherry-pick C
            \         ↓
             ---------C'

```
✅ Use case: Bug fix applied from dev to production without full merge.


---

### 🔍 `git reflog`

| ✅ Purpose                  | View local history of `HEAD` changes (undo destructive commands) |
| -------------------------- | ---------------------------------------------------------------- |
| ✅ Safe on shared branches? | Yes (read-only)                                                  |

#### 🔧 Use Cases:

* You accidentally deleted or reset a branch and want to recover it:

```bash
git reflog
git checkout <sha-from-reflog>
```

* You ran a `git reset --hard` and want to go back to the previous state.

→ Shows history of what HEAD pointed to — even deleted commits!
```
HEAD log:
0. HEAD@{0} — reset to B
1. HEAD@{1} — commit C
2. HEAD@{2} — commit B

You can do:
git checkout HEAD@{1}
```
✅ Use case: Undo a bad reset, recover a lost branch, or rollback force-push.


---

🧪 Bonus: Combined Workflow
Imagine this real-world CI/CD scenario:
```
A -- B -- C -- D          ← main
        \      \
         E -- F  hotfix  ← you cherry-pick D into hotfix

```
Then you:
 - Fix a build on hotfix → revert
 - Clean history on feature → rebase
 - Accidentally reset → reflog to recover

**🧩 Quick Reference Summary Table**

| Command           | Description                                          | When to Use                                                   |
| ----------------- | ---------------------------------------------------- | ------------------------------------------------------------- |
| `git revert`      | Create a new commit to undo an earlier one           | Safely undo a deployed or shared commit                       |
| `git rebase`      | Reapply commits onto another base (rewrites history) | Clean up messy history before a pull request                  |
| `git reset`       | Move HEAD to earlier point (can discard data)        | Undo recent commits locally or remove staging/working changes |
| `git cherry-pick` | Copy a commit from one branch to another             | Apply a bugfix from one branch into another                   |
| `git reflog`      | Show all HEAD movements (recover lost commits)       | Undo destructive operations like reset or branch delete       |

---

**🧪 Real DevOps Scenarios:**

1. **CI/CD pipeline fails because of a bad config pushed to main**
   → `git revert <commit>`
   → Safe fix without rewriting history.

2. **PR with 10 random commits, team asks to squash before merge**
   → `git rebase -i HEAD~10`

3. **You reset a branch and now it’s empty**
   → `git reflog` → checkout previous commit

4. **Cherry-pick a fix into production without waiting for full merge**
   → `git cherry-pick <sha>`

---

Would you like a **Git mind map**, **cheat sheet**, or **project workflow case study** next?


=========================================================================================

# Confusion b/w rebase , reset , revert 

Absolutely — let’s **use a single, relatable use case** to compare `git reset`, `git rebase`, and `git revert` side-by-side, and clarify:

> ✅ **When** you can use each (before/after commit, before/after push)
> 🔄 **What** they do to history and files
> 🧠 **Why** you'd pick one over another

---

## 🧪 **Use Case**: You’re fixing a bug and make messy commits

You’re on a branch called `fix/login-bug`.
You make three commits:

```plaintext
A -- B -- C (WIP fix - not working)
           \
            D (added debug prints)
             \
              E (final fix done)
```

Now you want to clean this up before merging.

---

## 🛠️ Tool 1: `git reset`

### 🔹 Description:

Moves the branch pointer backward. Can unstage or delete commits depending on flags.

### ✅ Use:

* **After commits**
* **Before push** (never on public/shared history unless you're 100% sure)

### 💡 Scenario:

You want to **discard** WIP and debug commits and make **one clean commit**.

```bash
git reset --soft C    # keep changes staged
git commit -m "Fix login issue cleanly"
```

Now:

```plaintext
A -- B -- C -- F (clean fix)
```

---

## 🛠️ Tool 2: `git rebase -i`

### 🔹 Description:

Rewrites history: squashes, reorders, edits commits.

### ✅ Use:

* **After commits**
* **Before push** only!

### 💡 Scenario:

You want to **combine messy commits** into one but **preserve the history cleanly**.

```bash
git rebase -i C
# squash D and E into one
```

Now:

```plaintext
A -- B -- C -- F (clean fix)
```

Looks same as reset, but **doesn’t unstage anything** — it's a cleaner rewrite tool.

---

## 🛠️ Tool 3: `git revert`

### 🔹 Description:

Creates a new commit that undoes another commit without touching history.

### ✅ Use:

* **After commits**
* ✅ **After push** — safe for shared history

### 💡 Scenario:

You already pushed a bad commit `E`, and want to undo it in production.

```bash
git revert E
```

Now:

```plaintext
A -- B -- C -- D -- E -- E' (undo E)
```

---

## 🧭 Summary Table (Same Use Case)

| Tool     | When to use                    | Rewrites History? | File Changes?              | Push-safe? |
| -------- | ------------------------------ | ----------------- | -------------------------- | ---------- |
| `reset`  | Before push, clean all commits | ✅ Yes             | Yes                        | ❌ No       |
| `rebase` | Before push, squash/reorder    | ✅ Yes             | No (rewrites commits only) | ❌ No       |
| `revert` | After push, undo safely        | ❌ No              | Adds undo commit           | ✅ Yes      |

---

## 🔚 Final Recommendation

* ✅ **Use `reset`**: If you're *alone* on a branch and haven't pushed.
* ✅ **Use `rebase -i`**: To clean up commit history before PR.
* ✅ **Use `revert`**: When bad commit is already pushed, and you're on production/shared history.

---

Would you like this as a **printable cheat sheet** or an **interactive example repo** setup?

=========================================================================================

# As learnt , reset & rebase should be used after commit but **before push** . Can't I use them **after push**

Absolutely — let me walk you through a **real-world scenario** step-by-step where using `reset`, `rebase`, or `revert` *after push* can either break things or fix them safely.

---

## 🔧 Scenario: You and teammate are fixing a bug

You both work on the same branch `feature/login-fix`.

---

### ✅ Step 1: You make messy commits

You write some test code:

```bash
git checkout -b feature/login-fix
echo "debug logs" >> login.js
git commit -am "WIP: added debug logs"
echo "final fix" >> login.js
git commit -am "Fix login bug"
git push origin feature/login-fix
```

So the commit tree looks like:

```
A -- B -- C  ← you pushed this
```

* `B`: debug commit
* `C`: final fix

---

### 👥 Step 2: Teammate pulls and works

Your teammate pulls `feature/login-fix`, sees both commits `B` and `C`, and starts working.

---

### ⚠️ Step 3: You want to clean it up

You realize `B` is messy and decide to remove it using `rebase -i`:

```bash
git rebase -i A
# squash B into C → now only one clean commit: D
```

Now your history looks like:

```
A -- D
```

You then do:

```bash
git push --force
```

---

### 💥 What your teammate sees

Your teammate still has:

```
A -- B -- C
```

When they try to pull:

```bash
git pull origin feature/login-fix
```

Git complains:

```
fatal: refusing to merge unrelated histories
```

Because the remote is now:

```
A -- D
```

and their local is:

```
A -- B -- C
```

It’s like two people edited a Google Doc, and one *rewrote the entire history* and saved over it. The other person now can’t merge.

---

## 🔁 What could you have done instead?

### ✅ Option 1: Use `revert` (safe for shared branches)

Instead of rebasing or resetting:

```bash
git revert B
```

It creates a new commit `B'` that undoes `B`. New history:

```
A -- B -- C -- B'
```

Teammate can pull safely. Everyone’s happy. You removed bad code *without rewriting history*.

---

### ✅ Option 2: Coordinate and use `rebase + force-push`

If you really need to squash/rewrite:

* Inform your teammate: "I’m rewriting branch history."
* After you `push --force`, they must do:

```bash
git fetch origin
git reset --hard origin/feature/login-fix
```

⚠️ But this **destroys any local work** — not recommended without coordination.

---

## 🧠 TL;DR (Real-World Rule)

| Action         | Safe after push?  | Use case                                           |
| -------------- | ----------------- | -------------------------------------------------- |
| `reset`        | ❌ No              | Cleanup your own local commits before push         |
| `rebase -i`    | ❌ No              | Squash or edit commits before sharing with team    |
| `revert`       | ✅ Yes             | Safely undo mistakes after push                    |
| `push --force` | ⚠️ Only with care | Only use when working alone or with full agreement |

---

=========================================================================================

# Diffrence b/w reset ( hard, soft , mmixed)

You're absolutely right — let's break down `git reset` **clearly**, with **real examples**, covering:

---

### ✅ Summary of `git reset` types

| Type                | Keeps your files? | Keeps staging (index)? | Description                                |
| ------------------- | ----------------- | ---------------------- | ------------------------------------------ |
| `--soft`            | ✅ Yes             | ✅ Yes                  | Moves HEAD only — leaves everything staged |
| `--mixed` (default) | ✅ Yes             | ❌ No                   | Moves HEAD, un-stages files                |
| `--hard`            | ❌ No              | ❌ No                   | Resets HEAD, staging, and file changes     |

---

## 🔧 Real Use Case to Understand All 3

Imagine you're working on a feature:

```bash
git checkout -b feature/home-banner
echo "banner 1" > banner.js
git add banner.js
git commit -m "WIP: Added basic banner"

echo "banner 2" >> banner.js
git add banner.js
git commit -m "Added styles"
```

Current state (HEAD at commit `B`):

```
A -- B  ← HEAD (you pushed nothing yet)
```

---

### 🚀 You now want to **fix the last 2 commits** — here’s how reset works:

---

### 🟡 `git reset --soft HEAD~1`

* **Moves HEAD back** to `A`
* **Keeps** your changes in **staging**
* You can now squash both changes together

```bash
git reset --soft HEAD~1
git commit -m "Clean banner module"
```

📌 Use when you want to **combine commits**.

---

### 🟠 `git reset --mixed HEAD~1` (default)

* **Moves HEAD back** to `A`
* **Unstages** your files
* Keeps changes in working directory

```bash
git reset HEAD~1
```

Now:

* Changes still there
* But `git status` shows files as modified

📌 Use this when you want to **re-edit or redo** staging

---

### 🔴 `git reset --hard HEAD~1`

* **Moves HEAD back** to `A`
* **Wipes** staging and **file content**
* Completely removes changes

```bash
git reset --hard HEAD~1
```

⚠️ No way to recover unless you use `git reflog`.

📌 Use only when you want to **nuke changes** and start fresh

---

## 🔥 Danger After Push?

If you've already pushed commits and then do:

```bash
git reset --hard HEAD~1
git push --force
```

👉 That will rewrite history and **break teammates' clones** unless coordinated.

---

### 🧪 When to Use Each?

| Scenario                                        | Recommended Reset Type |
| ----------------------------------------------- | ---------------------- |
| Want to squash last few commits before push     | `--soft`               |
| Want to unstage files but keep changes          | `--mixed`              |
| Want to delete local work completely            | `--hard`               |
| Want to undo latest commit *but keep code*      | `--mixed` or `--soft`  |
| Want to undo *everything*, as if never happened | `--hard`               |

---

Would you like a visual Git diagram to compare these actions commit-by-commit?



=========================================================================================

## 🔁 Branching & Merging Scenarios

### **1. Hotfix vs. Long-Running Feature Branch**

**Scenario:** You need to urgently fix production but a teammate is working on the same file in a long-running feature.

**Solution:**
Great follow-up — let's break down **how this approach helps prevent issues**, especially when someone else is also working on the **same file** in a **long-running feature branch**.

---

🔍 The Problem

* You urgently need to patch production (e.g., fix a bug in `UserService.js`)
* Your teammate is also modifying `UserService.js` in a feature branch (`feature-xyz`)
* You **don’t want to delay the fix**, but you also **don’t want to break their work**

---

✅ Why This Strategy Works
 
🔹 1. **Isolation via Hotfix Branch**

```bash
git checkout main
git checkout -b hotfix/urgent-bug
```

You isolate your fix from both:

* the `main` branch (clean history),
* and `feature-xyz` (to avoid merge conflicts *now*).

This keeps your hotfix **small, focused, and risk-free**.

---

 🔹 2. **Safe Deployment Path**

You make the fix, test it, and **merge only the hotfix into production**:

```bash
git commit -m "Fix: urgent prod issue"
git push origin hotfix/urgent-bug
# Create PR → merge to main → deploy
```

You’re not touching your teammate’s `feature-xyz` branch yet.

---

 🔹 3. **Later Synchronization (merge hotfix into feature)**

Once the fix is live:

```bash
git checkout feature-xyz
git merge main
```

* Now your teammate **receives the hotfix**
* If there's a **conflict**, it happens **in their isolated branch**, not in main
* They can resolve it with full context of their work

This avoids disrupting the production line while still syncing all changes.

---

 🧠 Without This Approach...

* If you merged directly into the feature branch or cherry-picked back and forth, you might:

  * Cause **conflicts in production**
  * Accidentally **overwrite teammate’s changes**
  * Introduce **inconsistent state** across branches

---

 ✅ Summary: How It Prevents Issues

| Issue                           | How This Strategy Helps                                   |
| ------------------------------- | --------------------------------------------------------- |
| Teammate modifying same file    | You isolate your fix in a separate branch                 |
| Production needs urgent patch   | You deploy from a clean branch without blocking           |
| Conflict risk in feature branch | Merging `main → feature-xyz` happens later and explicitly |
| Clean history                   | Small atomic fix, easy to revert or track                 |

---

If you'd like, I can draw a quick **before/after Git graph** to visualize how this reduces conflict risk.


---------------------------------------------------------------------------------------------------------------

### **2. Restart Merge Conflict Resolution**

**Scenario:** You messed up resolving a merge conflict.

**Solution:**

* If merge is still in progress:
  `git merge --abort` → resets to pre-merge state
* If changes were staged:
  `git reset --merge` or use `git reflog` to go back

**Why:** `--abort` is the safest undo for unresolved merges.

---

### **3. Sensitive Info (API Key) Committed & Pushed**

**Solution:**

1. Remove file:
   `git rm --cached .env`
2. Rebuild history:
   `git filter-repo --path .env --invert-paths`
3. Force push:
   `git push origin --force`
4. Invalidate credentials immediately.

**Why:** It's not enough to delete from latest commit—must erase from full history.

Let’s break down both of these critical Git scenarios with **real examples**, clear **commands**, and the **reasoning behind them** so you fully understand how and why to act.

---

 🧪 Scenario:

You accidentally committed an API key in `.env`:

```bash
git add .env
git commit -m "add secrets"
git push origin main
```

Now `.env` (with your keys) is publicly in Git history. Even if you `rm .env`, the key is still **recoverable** from Git’s history!

---

##### 🔥 Why this is dangerous:

* Secrets in Git history can be extracted from **any past commit**
* GitHub or any mirror **never deletes history** unless you force it

---

##### ✅ What You **Must** Do (Step-by-step):

```bash
# 1. Remove the file from index (but keep it locally)
git rm --cached .env

# 2. Rebuild the full Git history **without** the file
git filter-repo --path .env --invert-paths
```

> `filter-repo` rewrites history and **removes all traces** of `.env` from every commit.

```bash
# 3. Push rewritten history
git push --force
```

```bash
# 4. Rotate/disable exposed API keys immediately
```

---

##### 📌 Why not just delete the latest `.env`?

Because older commits still **contain it**. You must **erase it from history**, not just from the working directory.

---

##### 💡 Extra Tips:

* Install filter-repo: `brew install git-filter-repo` or via [GitHub](https://github.com/newren/git-filter-repo)
* Always add `.env` to `.gitignore` going forward

---


### **4. Teammate Force-Pushed Rebased Branch**

**Symptoms:** Your local branch says it's behind/ahead unexpectedly.

**Solution:**

* Backup:
  `git branch backup-my-work`
* Fetch latest:
  `git fetch origin`
* Rebase your changes:
  `git rebase origin/feature-branch`
* Or reset to their version:
  `git reset --hard origin/feature-branch`

**Why:** You can’t push without syncing after a force-push.



##### 🧪 Scenario:

Your teammate rebased a shared branch (`feature/login`) and ran:

```bash
git push --force origin feature/login
```

Now, you try to pull, and Git says:

> Your branch is behind by 3 commits, ahead by 2.

This happens because your **local branch has diverged**.

---

##### 🔥 Why it happens:

Rebase rewrites commit history → Git sees their version and yours as two different branches, even if they look similar.

---

##### ✅ What to Do:

```bash
# 1. Backup your local work in case
git branch backup-my-work
```

---

##### Option A: Rebase your work on top of theirs

```bash
git fetch origin
git rebase origin/feature/login
```

This makes your commits sit *on top* of their rebased history.

---

##### Option B: Reset to teammate’s version (⚠️ overwrites your changes)

```bash
git fetch origin
git reset --hard origin/feature/login
```

> Use this only if you're okay discarding local changes.

---

##### 🚀 Why it's required:

You **can’t push** your version unless you’re aligned. Git will reject the push due to the branch history conflict.

---

##### 💡 Tip:

If you're working on shared branches like `develop` or `main`, **avoid rebasing or force-pushing**. Use `merge` unless you coordinate.



---

## 🔍 Git Reset / Reflog / Recovery Scenarios

### **5. Recover After `git reset --hard`**

**Solution:**

* Run:
  `git reflog`
* Find lost commit hash, e.g., `abc123`
* Recover:
  `git checkout abc123` or
  `git cherry-pick abc123` onto current branch

**Why:** Reflog tracks all HEAD movements.

---

### **6. Teammate Force-Pushed, You Lost Commits**

**Solution:**

* `git reflog` to find commit before overwrite
* Recover by checking out:
  `git checkout <old-commit>`
  or create a temp branch:
  `git checkout -b recovery <old-commit>`

**Why:** Git doesn't delete immediately; reflog is a time machine.

---

## 📂 Repo Management Scenarios

### **7. .git Folder Deleted**

**Solution:**

* If `.git/` is deleted → repo is no longer tracked.
* Options:

  * Re-clone:
    `git clone git@github.com:org/repo.git`
  * Or recover `.git/` from backup

**Why:** Without `.git/`, it's just a folder of files.

---

### **8. Wrong Default Branch (master vs. main)**

**Solution:**

* Fetch all branches:
  `git fetch origin`
* Checkout:
  `git checkout main`
* Set upstream:
  `git branch --set-upstream-to=origin/main`

**Why:** This sets correct tracking and avoids confusion with `origin/HEAD`.

---

## 🧪 Advanced Git Scenarios

### **9. Split Large Commit**

**Solution:**

* Undo commit but keep changes staged:
  `git reset HEAD~1`
* Stage in chunks:
  `git add -p` (interactive patch)
* Commit logically.

**Why:** `git add -p` lets you select individual changes by hunk.

---

### **10. Find Breaking Commit**

**Solution:**

* Run:
  `git bisect start`
  `git bisect bad`
  `git checkout <old-known-good-commit>`
  `git bisect good`
* Git walks through commits → you test and mark good/bad until root cause is found.

**Why:** This is binary search for bugs in version history.

---

### **11. Remove Specific File Type from History**

**Solution:**

```bash
git filter-repo --path-glob '*.mp4' --invert-paths
```

* Then:
  `git push --force`
  Ask team to re-clone.

**Why:** Filter-repo is faster & safer than `filter-branch`.

---

### **12. Clean Up Deleted Remote Branches**

**Solution:**

```bash
git fetch -p
git branch -vv | grep 'gone'
git branch -d branchname
```

**Why:** `-p` prunes remote-tracking branches that no longer exist.

---

## 🧠 Conceptual & Team Scenarios

### **13. Rebase vs Merge**

| Rebase                  | Merge                                |
| ----------------------- | ------------------------------------ |
| Rewrites history        | Preserves history                    |
| Cleaner linear log      | Better for tracking feature branches |
| Ideal for local cleanup | Better for shared/public branches    |

**Best Practice:**

* Use `rebase` locally before pushing
* Use `merge` when integrating team branches

---

### **14. Teammate Pushed to Main Instead of Feature**

**Solution:**

* Revert using:
  `git revert <bad_commit_sha>`
* Do not reset or rebase if it's already pushed

**Why:** Revert is non-destructive and safe for public branches.

---

### **15. Preserve Uncommitted Changes Temporarily**

**Options:**

1. `git stash`
2. `git checkout -b wip/save && git add . && git commit -m "WIP"`

**Why:** Safe to test new code without losing current state.

---

## 🧾 Bonus Quick Answers

* **Squash before merging:**
  `git rebase -i HEAD~N`
  Use `squash` in editor

* **Delete remote branch:**
  `git push origin --delete branchname`

* **Track remote branch:**
  `git branch --set-upstream-to=origin/branchname`

* **Undo git pull with merge:**
  `git reset --hard ORIG_HEAD`

* **Compare two branches:**
  `git diff branch1..branch2`
  or
  `git log branch1..branch2`

---

Would you like these exported as a PDF, markdown sheet, or in quiz form? I can format them for interviews or prep sheets.
