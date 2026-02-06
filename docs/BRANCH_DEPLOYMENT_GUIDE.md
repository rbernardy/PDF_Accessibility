# Branch Deployment Guide

## Understanding SOURCE_VERSION in deploy.sh

### What is SOURCE_VERSION?

`SOURCE_VERSION` tells AWS CodeBuild which **branch, tag, or commit** to use when cloning your GitHub repository for deployment.

### Previous Behavior (Hardcoded)

**Before the fix:**
```bash
SOURCE_VERSION="main"  # Always deploys from main branch
```

**Problem:**
- If you're working on branch `dev` and run `./deploy.sh`
- CodeBuild would still pull code from `main` branch
- Your local changes on `dev` wouldn't be deployed (unless you pushed to `main`)

### New Behavior (Auto-detect)

**After the fix:**
```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "main")
SOURCE_VERSION="${SOURCE_VERSION:-$CURRENT_BRANCH}"
```

**How it works:**
1. Detects your current Git branch automatically
2. Uses that branch for deployment
3. Falls back to `main` if Git detection fails
4. Can be overridden with environment variable

## Usage Examples

### Example 1: Deploy from Current Branch (Automatic)

```bash
# You're on branch 'dev'
git checkout dev
git add .
git commit -m "My changes"
git push origin dev

# Deploy from 'dev' branch (auto-detected)
./deploy.sh
# ✅ CodeBuild will use 'dev' branch
```

### Example 2: Deploy from Specific Branch (Override)

```bash
# You're on branch 'dev' but want to deploy from 'main'
SOURCE_VERSION=main ./deploy.sh
# ✅ CodeBuild will use 'main' branch
```

### Example 3: Deploy from Specific Tag

```bash
# Deploy from a specific release tag
SOURCE_VERSION=v1.0.0 ./deploy.sh
# ✅ CodeBuild will use tag 'v1.0.0'
```

### Example 4: Deploy from Specific Commit

```bash
# Deploy from a specific commit SHA
SOURCE_VERSION=abc123def456 ./deploy.sh
# ✅ CodeBuild will use commit 'abc123def456'
```

## Workflow Recommendations

### Development Workflow

```bash
# 1. Create a feature branch
git checkout -b feature/folder-structure-fix

# 2. Make your changes
# ... edit files ...

# 3. Commit and push to YOUR branch
git add .
git commit -m "Add folder structure preservation"
git push origin feature/folder-structure-fix

# 4. Deploy from your feature branch
./deploy.sh
# ✅ Automatically deploys from 'feature/folder-structure-fix'

# 5. Test in AWS

# 6. If successful, merge to main
git checkout main
git merge feature/folder-structure-fix
git push origin main

# 7. Deploy production from main
git checkout main
./deploy.sh
# ✅ Deploys from 'main'
```

### Multi-Environment Workflow

```bash
# Development environment
git checkout dev
./deploy.sh  # Deploys dev branch to dev environment

# Staging environment
git checkout staging
./deploy.sh  # Deploys staging branch to staging environment

# Production environment
git checkout main
./deploy.sh  # Deploys main branch to production
```

## Important Notes

### 1. You MUST Push Your Branch to GitHub

CodeBuild clones from GitHub, not your local machine:

```bash
# ❌ WRONG - Changes only local
git commit -m "My changes"
./deploy.sh  # CodeBuild won't see your changes!

# ✅ CORRECT - Push to GitHub first
git commit -m "My changes"
git push origin your-branch-name
./deploy.sh  # CodeBuild will see your changes
```

### 2. Branch Must Exist on GitHub

```bash
# Check if your branch exists on GitHub
git branch -r | grep your-branch-name

# If not, push it
git push -u origin your-branch-name
```

### 3. CodeBuild Needs Access to Your Branch

Make sure:
- Your GitHub repository is accessible to AWS CodeBuild
- The branch is not protected in a way that blocks CodeBuild
- Your AWS credentials have permission to access the repository

## Troubleshooting

### Issue: "Branch not found" error

**Symptom:**
```
Error: Source version 'my-branch' not found
```

**Solution:**
```bash
# Verify branch exists on GitHub
git ls-remote --heads origin

# Push your branch if missing
git push origin my-branch
```

### Issue: Deploying old code

**Symptom:** Changes don't appear after deployment

**Causes & Solutions:**

1. **Forgot to push to GitHub:**
   ```bash
   git push origin your-branch-name
   ```

2. **CodeBuild using wrong branch:**
   ```bash
   # Check which branch was used (look at deploy.sh output)
   # It shows: "Branch: <branch-name>"
   
   # Force specific branch
   SOURCE_VERSION=your-branch-name ./deploy.sh
   ```

3. **Cached build:**
   ```bash
   # Delete CodeBuild project and redeploy
   aws codebuild delete-project --name PDFAccessibility-Build
   ./deploy.sh
   ```

### Issue: Want to deploy from main while on different branch

**Solution:**
```bash
# You're on 'dev' but want to deploy 'main'
SOURCE_VERSION=main ./deploy.sh
```

## Verification

After running `./deploy.sh`, check the output:

```
🏗️  Creating CodeBuild project...
   Name: PDFAccessibility-Build
   Repository: https://github.com/rbernardy/PDF_Accessibility.git
   Branch: your-branch-name  ← Verify this is correct!
   Buildspec: buildspec-unified.yml
   Solution: PDF-to-PDF Remediation
```

## Best Practices

### 1. Always Verify Your Branch Before Deploying

```bash
# Check current branch
git branch --show-current

# Check if it's pushed to GitHub
git status

# If behind, pull first
git pull origin your-branch-name

# Then deploy
./deploy.sh
```

### 2. Use Descriptive Branch Names

```bash
# ✅ Good
feature/folder-structure-preservation
bugfix/javascript-scoping-issue
hotfix/404-error-fix

# ❌ Bad
test
fix
my-branch
```

### 3. Tag Production Releases

```bash
# After successful deployment to production
git tag -a v1.0.0 -m "Folder structure preservation release"
git push origin v1.0.0

# Later, redeploy exact version
SOURCE_VERSION=v1.0.0 ./deploy.sh
```

### 4. Keep Main Branch Stable

```bash
# Develop on feature branches
git checkout -b feature/new-feature

# Test thoroughly before merging to main
./deploy.sh  # Test on feature branch

# Only merge to main when stable
git checkout main
git merge feature/new-feature
git push origin main
```

## Environment Variable Reference

```bash
# Override source version
SOURCE_VERSION=branch-name ./deploy.sh

# Override deployment type
DEPLOYMENT_TYPE=pdf2html ./deploy.sh

# Combine multiple overrides
SOURCE_VERSION=v1.0.0 DEPLOYMENT_TYPE=pdf2pdf ./deploy.sh
```

## Summary

✅ **Now**: deploy.sh automatically uses your current branch
✅ **Flexible**: Can override with SOURCE_VERSION environment variable
✅ **Safe**: Falls back to 'main' if detection fails
✅ **Transparent**: Shows which branch is being deployed

**Remember:** Always push your branch to GitHub before deploying!
