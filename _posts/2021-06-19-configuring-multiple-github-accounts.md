---
layout: post
title: "Configuring Multiple GitHub Accounts"
date: 2021-06-19
tags: [Git, GitHub, SSH]
comments: true
---

When working on multiple projects, you might want to use different GitHub accounts for each project. Here's how you can configure Git to use multiple GitHub accounts on the same machine.

### Step 1: Generate SSH Keys for Each Account

First, generate a new SSH key for each GitHub account. You can do this by running the following command in your terminal:

```bash
ssh-keygen -t ed25519 -C "<your_email@example.com>" -f ~/.ssh/id_ed25519_<account_name>
```

specifying a different file name for each account (e.g., `id_ed25519_personal`, `id_ed25519_work`).

### Step 2: Add SSH Keys to GitHub Accounts

Copy the contents of each public key file (e.g., `~/.ssh/id_ed25519_personal.pub` and `~/.ssh/id_ed25519_work.pub`) and add them to the respective GitHub accounts under **Settings > SSH and GPG keys**.

### Step 3: Configure SSH Config File

Create or edit the SSH config file located at `~/.ssh/config` to specify which key to use for each GitHub account:

```bash
# Personal GitHub account
Host github.com-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal
    IdentiesOnly yes
# Work GitHub account
Host github.com-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
    IdentitiesOnly yes
```

### Step 4: Clone Repositories Using the Correct Host

When cloning repositories, use the appropriate host defined in your SSH config file. For example:

```bash
# For personal projects
git clone git@github.com-personal:username/repo.git
# For work projects
git clone git@github.com-work:username/repo.git
```

### Step 5: Set Git User for Each Repository

Finally, set the Git user name and email for each repository to match the corresponding GitHub account:

```bash
cd /path/to/your/repo
git config user.name "Your Name"
git config user.email "your_email@example.com"
```

Repeat this for each repository associated with different GitHub accounts.

With these steps, you can easily manage multiple GitHub accounts on the same machine without conflicts.
