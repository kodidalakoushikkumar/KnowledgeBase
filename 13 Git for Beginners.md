### What is Git?

Git is a popular version control system.
It is used for:
- Tracking code changes
- Tracking who made changes
- Coding collaboration

### Configure Git

User Name: `git config --global [user.name] "Your Name"`
Email Address : `git config --global user.email "[you@example.com]"`
Use` --global` to set the value for every repository on your computer.
Use `--local` (the default) to set it only for the current repository.
##### Viewing Your Configuration

```
git config --list
[user.name]=Your Name
user.email=[you@example.com]
core.editor=code --wait
[alias.st]=status
init.defaultbranch=main
```
### Configuration Levels

There are three levels of configuration:
System (all users): `git config --system`
Global (current user): `git config --global`
Local (current repo): `git config --local`
### Git Staging Environment

The staging environment (or staging area) is like a waiting room for your changes.
`git add <file>` - Stage a file
### Git Commit

A commit is like a save point in your project.
It records a snapshot of your files at a certain time, with a message describing what changed.
You can always go back to a previous commit if you need to.
Here are some key commands for commits:
`git commit -m "message"` - Commit staged changes with a message
`git commit -a -m "message"` - Commit all tracked changes (skip staging)
`git log` - See commit history
To view the history of commits for a repository, you can use the git log command.
### Git Tagging
#### Key Commands for Tagging

`git tag <tagname>` - Create a lightweight tag
`git tag -a <tagname> -m "message"` - Create an annotated tag
`git tag <tagname> <commit-hash>` - Tag a specific commit
`git tag `- List tags
`git show <tagname>` - Show tag details
A tag in Git is like a label or bookmark for a specific commit.
Tags are most often used to mark important points in your project history, like releases (v1.0 or v2.0).
Tags are a simple and reliable way to keep track of versions and share them with your team or users.
A lightweight tag is just a name for a commit.
It's quick and simple, but does not store extra information.
An annotated tag stores your name, the date, and a message.
This is recommended for most uses.
Delete a tag locally: `git tag -d v1.0`
### Git Stash
#### Key Commands for Stashing

`git stash` - Stash your changes
`git stash push -m "message"` - Stash with a message
`git stash list` - List all stashes
`git stash branch <branchname>` - Create a branch from a stash

Sometimes you need to quickly switch tasks or fix a bug, but you're not ready to commit your work.
git stash lets you save your uncommitted changes and return to a clean working directory.
You can come back and restore your changes later.

Delete a specific stash when you no longer need it: `git stash drop stash@{0}`
Delete all your stashes at once: `git stash clear`
### Git Branch

In Git, a branch is like a separate workspace where you can make changes and try new ideas without affecting the main project. Think of it as a "parallel universe" for your code.
#### Why Use Branches?

Branches let you work on different parts of a project, like new features or bug fixes, without interfering with the main branch.
#### Common Reasons to Create a Branch

- Developing a new feature
- Fixing a bug
- Experimenting with ideas

So we create a new branch:  `git branch hello-world-images`
Listing All Branches : `git branch`
Switching Between Branches:c`git checkout hello-world-images`
### Git Branch Merge

Merging in Git means combining the changes from one branch into another.
This is how you bring your work together after working separately on different features or bug fixes.
#### Common git merge Options

`git merge` - Merge a branch into your current branch
`git merge --no-ff` - Always create a merge commit
`git merge --squash` - Combine changes into a single commit
`git merge --abort` - Abort a merge in progress

### STEPS FOR BEGINNERS
#### Step 1: Install Git (if not installed)

Check if Git is installed:   `git --version`
#### Step 2: Initialize a local Git repository

Go to your project folder in the terminal:  `cd path/to/your/project`
Initialize Git:  `git init`
This creates a `.git` folder in your project (Git now tracks your files).
#### Step 3: Add files to staging

Add all your project files:  `git add .`
Check what’s staged:  `git status`
#### Step 4: Commit your changes

Make your first commit:  `git commit -m "Initial commit"`
#### Step 5: Create a GitHub repository

1. Go to GitHub and log in.
2. Click **New repository**.
3. Give it a name (same as your project or anything).
4. You can leave “Initialize with README” unchecked (since you already have local files).
5. Click **Create repository**.
#### Step 6: Link local repo to GitHub

GitHub will show you something like: `git remote add origin https://github.com/USERNAME/REPO_NAME.git`
Copy and run this in your terminal. Replace `USERNAME` and `REPO_NAME` with your GitHub info.
#### Step 7: Push your code

For the first push, run:

```
git branch -M main
git push -u origin main
```

- `-M main` ensures your branch is named `main`.
- `-u origin main` sets `origin` as the default remote and `main` as the default branch.
  Now your project is live on GitHub.

#### Step 8: Future updates

Next time you make changes:

```
git add . 
git commit -m "Describe your changes" 
git push
```

No need to re-run `git remote add origin`.
