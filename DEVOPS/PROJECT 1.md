# Architecture & Deployment Guide: Static Portfolio on GitHub Pages

## 1. Architectural Overview

This project follows a **Monolithic Static Web Architecture**. All client-side presentation assets (HTML, CSS) are bundled in a single repository and served directly to the user's browser without external application servers or database dependencies.

```
┌───────────────────────────┐         ┌───────────────────────────────┐
│       Local Machine       │  Git    │      GitHub Repository        │
│  (HTML5 / CSS3 Monolith)  ├────────►│        (main branch)          │
└───────────────────────────┘  Push   └──────────────┬────────────────┘
                                                     │ Automated Build &
                                                     │ Global Edge Distribution
                                                     ▼
                                      ┌───────────────────────────────┐
                                      │         GitHub Pages          │
                                      │   (Hosted Static Site URL)    │
                                      └───────────────────────────────┘
```

## 2. Step-by-Step Hosting Process

### Step 1: Build the Static Site

Create the foundational site files in your local project folder:

- **`index.html`**: Entry point for the application containing semantic markup.
- **`style.css`**: Stylesheet handling visual formatting and layout responsiveness.

### Step 2: Initialize Git Version Control

Track your project code locally using Git:

```
# Navigate to the project directory
cd devops-portfolio

# Initialize a new local Git repository
git init

# Stage all static files for tracking
git add .

# Save the baseline snapshot with a commit message
git commit -m "feat: initial static site build"
```

### Step 3: Create a GitHub Repository

1. Log into [GitHub]([suspicious link removed]).
2. Click the **+** icon in the top right corner and select **New repository**.
3. Set the **Repository name** (e.g., `devops-portfolio`).
4. Select **Public** visibility (required for free GitHub Pages hosting).
5. Leave "Add a README file" **unchecked** so you can push your existing local code.
6. Click **Create repository**.

### Step 4: Connect Remote & Push Code

Link your local repository to GitHub and upload the code:

```
# Set the default branch name to main
git branch -M main

# Add the remote GitHub repository URL
git remote add origin [suspicious link removed]

# Push your local commits to the remote main branch
git push -u origin main
```

### Step 5: Configure GitHub Pages Settings

1. Open your repository page on GitHub.
2. Click on the **Settings** tab in the top navigation bar.
3. On the left sidebar under _Code and automation_, click **Pages**.
4. Under the **Build and deployment** section:
    - **Source**: Choose `Deploy from a branch`.
    - **Branch**: Select `main` and keep the directory set to `/ (root)`.
5. Click **Save**.

### Step 6: Verify Deployment

1. Wait 1–2 minutes for GitHub's automated build runner to process the site.
2. Refresh the **Settings > Pages** page.
3. You will see a banner confirming deployment with your live site URL:  `[suspicious link removed]`