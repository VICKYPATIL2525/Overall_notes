# Git Clone — Private Repository Guide

**Purpose:** Clone a private GitHub repository using a Personal Access Token (PAT) — for both the main branch and specific branches.

---

## Table of Contents

1. [Why PAT is Needed](#why-pat-is-needed)
2. [Generating a Personal Access Token](#generating-a-personal-access-token)
3. [Clone the Main Branch](#clone-the-main-branch)
4. [Clone a Specific Branch](#clone-a-specific-branch)
5. [Important Security Note](#important-security-note)

---

## Why PAT is Needed

Private repositories require authentication. GitHub no longer accepts passwords over HTTPS — instead you use a **Personal Access Token (PAT)** embedded in the clone URL.

---

## Generating a Personal Access Token

1. Go to the GitHub repository
2. Click **Settings** (top-right of your profile, not the repo)
3. Scroll to the bottom → click **Developer settings**
4. Go to **Personal access tokens** → **Tokens (classic)**
5. Click **Generate new token (classic)**
6. Select the required scopes (at minimum: `repo`)
7. Click **Generate token** — copy it immediately, it is shown only once
8. If the repo belongs to an organization, click **Authorize SSO** next to the token if that option appears

---

## Clone the Main Branch

```bash
git clone https://<username>:<your-token>@github.com/<org>/<repo>.git
```

Example:
```bash
git clone https://your-username:ghp_yourtoken@github.com/Your-Org/Your-Repo.git
```

---

## Clone a Specific Branch

```bash
git clone -b <branch-name> https://<username>:<your-token>@github.com/<org>/<repo>.git
```

Example:
```bash
git clone -b initial-setup https://your-username:ghp_yourtoken@github.com/Your-Org/Your-Repo.git
```

---

## Important Security Note

- **Never commit the token** into any file that gets pushed to GitHub
- Tokens embedded in URLs can be exposed in shell history — clear it after use:
```bash
history -c
```
- If a token is accidentally pushed, revoke it immediately from GitHub → Developer settings → Personal access tokens
- For long-term use, consider setting up SSH keys instead — no token in the URL needed
