# Lab 23 — Git

This lab walks through Git from `init` to merging, conflict resolution, tagging, rewriting history, and pushing to a remote. It maps to CompTIA Linux+ XK0-006 Objective 4.4 (Given a scenario, use version control systems). You will create a real working tree, a separate bare "origin" repo on the same machine, and push between them.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install Git and set identity

```bash
apt update && apt install -y git
git --version
git config --global user.name  "Linux Plus Learner"
git config --global user.email "learner@example.com"
git config --global init.defaultBranch main
git config --global --list
```

The user.name/email values appear in every commit you make. `init.defaultBranch` ensures new repos start on `main`.

---

## Step 2 — Initialise a repo and make the first commit

```bash
mkdir -p /root/lab && cd /root/lab
git init lab && cd lab
cat > README.md <<'EOF'
# Lab repo
Demo project for Linux+ XK0-006.
EOF
cat > .gitignore <<'EOF'
*.log
*.tmp
__pycache__/
EOF
git status
git add README.md .gitignore
git commit -m "initial commit: readme and gitignore"
git log --oneline --graph --decorate
```

`.gitignore` keeps build artifacts out of the repo. `git log --oneline --graph` is the quickest way to visualise history.

---

## Step 3 — Branch, modify, merge

```bash
git switch -c feature/banner
echo "Welcome to the lab" > banner.txt
git add banner.txt
git commit -m "feat: add welcome banner"

git switch main
git merge --no-ff feature/banner -m "merge feature/banner"
git log --oneline --graph --all
```

`--no-ff` forces a merge commit so the branch topology stays visible in history.

---

## Step 4 — Create and resolve a conflict

```bash
git switch -c feature/edit
sed -i 's/Welcome to the lab/Welcome, learner/' banner.txt
git commit -am "tweak banner wording"

git switch main
sed -i 's/Welcome to the lab/Hello from main/' banner.txt
git commit -am "main edits banner"

git merge feature/edit || true
cat banner.txt
# Resolve by keeping a combined line
cat > banner.txt <<'EOF'
Hello learner, welcome to the lab
EOF
git add banner.txt
git commit -m "merge feature/edit and resolve banner conflict"
git log --oneline --graph --all
```

Conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) flag the disputed region. You edit the file to the desired final content, `git add`, and commit.

---

## Step 5 — diff, stash, tag

```bash
echo "draft note" >> README.md
git diff
git stash
git status
git stash pop
git diff

git tag v0.1
git tag --list
```

`git stash` shelves uncommitted work so you can switch context, then `stash pop` restores it. Tags mark releases.

---

## Step 6 — Rewrite history: reset and rebase

```bash
echo "oops" > scratch.txt
git add scratch.txt
git commit -m "WIP scratch"
git log --oneline | head

git reset --soft HEAD~1
git status
git restore --staged scratch.txt
rm scratch.txt

# Linear-history rebase demo
git switch -c topic/rebase
echo "line A" >> README.md && git commit -am "topic: line A"
echo "line B" >> README.md && git commit -am "topic: line B"
git switch main
echo "main change" >> README.md && git commit -am "main: change"
git switch topic/rebase
git rebase main || true
# Resolve any conflict the same way as before, then:
git add README.md 2>/dev/null || true
git rebase --continue 2>/dev/null || true
git switch main
git merge --ff-only topic/rebase || git merge topic/rebase -m "merge topic/rebase"
```

`reset --soft HEAD~1` undoes a commit but keeps the changes staged. Rebase replays commits onto a new base for a cleaner history; squash merge collapses many commits into one on merge.

---

## Step 7 — Add a remote and push/pull

```bash
git init --bare /tmp/origin.git
cd /root/lab/lab
git remote add origin /tmp/origin.git
git push -u origin main
git push origin v0.1

# Simulate a teammate pushing
git clone /tmp/origin.git /tmp/teammate
cd /tmp/teammate
echo "teammate edit" >> README.md
git -c user.email=team@example.com -c user.name=Team commit -am "team edit"
git push

cd /root/lab/lab
git fetch
git log --oneline origin/main | head
git pull --rebase
git log --oneline --graph | head
```

A bare repo (`/tmp/origin.git`) has no working tree and is what services like GitHub host. `git pull --rebase` keeps history linear by replaying your local commits on top of fetched ones.

---

## Step 8 — Cleanup

```bash
cd /
rm -rf /root/lab /tmp/origin.git /tmp/teammate
apt remove -y --purge git
apt autoremove -y
```

This removes both the working repo and the bare remote.

---

## What you learned
- How to configure Git identity and initialise a repository with a sensible `.gitignore`.
- Branching, merging, conflict resolution, and inspecting history with `log --oneline --graph`.
- Stashing, tagging, and rewriting history with `reset --soft` and `rebase`.
- How to push to a remote (here a local bare repo) and pull a teammate's changes with rebase.
- The concept of a squash merge for collapsing a feature branch into a single commit on `main`.

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- Git — https://git-scm.com/doc
- Pro Git book (free) — https://git-scm.com/book
