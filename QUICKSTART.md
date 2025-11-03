# ⚡ Quick Start Guide

Get started with the Azure DevOps Pipeline Creator in **5 minutes** or less!

## ✅ Prerequisites Checklist

- [ ] Azure DevOps organization and project
- [ ] Repository with this pipeline committed
- [ ] Admin or Build Administrator permissions

---

## 🚀 3-Step Setup

### Step 1: Grant Permissions (1 minute)

**Easy Way:**
1. Go to **Project Settings** > **Pipelines**
2. Click **⋮** (three dots) on root `\` folder > **Security**
3. Click **+ Add** > Search for **[YourProject] Build Service**
4. Set **"Create build pipeline"** to **Allow**
5. ✅ Done!

### Step 2: Create the Pipeline (1 minute)

1. **Pipelines** > **New Pipeline**
2. Select your repository
3. **Existing Azure Pipelines YAML file**
4. Select `azure-pipelines-creator.yml`
5. Click **Save** (don't run yet)

### Step 3: Prepare Your YAML Files (2 minutes)

Organize your pipeline YAML files:

```
your-repo/
├── azure-pipelines-creator.yml
└── pipelines/
    ├── ci/
    │   └── build.yml
    └── cd/
        └── deploy.yml
```

**Minimum YAML file** (`pipelines/ci/build.yml`):
```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
  - script: echo "Hello from auto-created pipeline!"
```

---

## 🎯 First Run

### Test Run (Dry Mode)

1. Click **Run pipeline**
2. Set parameters:
   - **gitFolderPath**: `pipelines`
   - **subfolderStructure**: `*`
   - **azureDevOpsFolderPath**: `\\`
   - **dryRun**: `true` ← IMPORTANT!
3. Click **Run**
4. Review logs - should show: `[DRY-RUN] Would create X pipelines`

### Actual Creation

1. Click **Run pipeline** again
2. Change **dryRun** to `false`
3. Click **Run**
4. ✅ Check **Pipelines** - your new pipelines should appear!

---

## 📋 Parameter Guide

| Parameter | What It Does | Common Values |
|-----------|--------------|---------------|
| **gitFolderPath** | Folder with YAML files | `pipelines`, `devops/pipelines` |
| **subfolderStructure** | Which subfolders to scan | `*` (all), `ci,cd` (specific) |
| **azureDevOpsFolderPath** | Where to create in Azure DevOps | `\\` (root), `\\Automated` |
| **dryRun** | Preview without creating | `true` (test), `false` (create) |

---

## ❓ Common First-Run Issues

### ❌ "Access denied. Build Service needs permissions"

**Fix:** Redo Step 1 - Grant "Create build pipeline" permission

### ❌ "No YAML files found"

**Fix:**
```bash
# Verify files exist
git ls-files pipelines/

# Check extensions (.yml or .yaml)
# Ensure files are committed
```

### ❌ Pipelines created in wrong folder

**Fix:** Check `azureDevOpsFolderPath` parameter:
- Use `\\` for root
- Use `\\MyFolder` for custom folder
- Always use backslashes `\`

---

## ✨ What Happens Automatically

🔍 **Self-Contained** - No need to specify:
- ✅ Organization URL (uses current)
- ✅ Project name (uses current)
- ✅ Repository URL (uses current)
- ✅ Branch name (uses current)

🛡️ **Smart Exclusions** - Automatically skips:
- ❌ Folders with "archive" (e.g., `archive/`, `old-archive/`)
- ❌ Folders with "template" (e.g., `templates/`, `my-templates/`)
- ❌ Existing pipelines (no duplicates)

⚡ **Auto-Detection:**
- ✅ Agent queue (finds "Azure Pipelines" automatically)
- ✅ Existing pipelines (prevents duplicates)

---

## 📊 Success Verification

After running, check:

1. **Pipeline Logs:**
   ```
   Found 4 YAML file(s)
   CREATED: backend-build
   CREATED: dev-deploy
   === SUMMARY ===
   Created: 2 | Skipped: 0 | Failed: 0
   ```

2. **Azure DevOps UI:**
   - Navigate to **Pipelines**
   - See your new pipelines in the correct folders
   - Names match your YAML filenames

3. **Re-run Test:**
   - Run pipeline again
   - Should show "SKIP: X (exists)" for all pipelines
   - No duplicates created ✅

---

## 🎓 Next Steps

Now that it's working:

1. ✅ Add more YAML files to `pipelines/` folder
2. ✅ Re-run pipeline - only creates new ones
3. ✅ Organize by subfolder: `ci/`, `cd/`, `infrastructure/`
4. ✅ Use `subfolderStructure: ci,cd` to be selective
5. ✅ Read full [README.md](README.md) for advanced features

---

## 💡 Pro Tips

💡 **Always dry-run first**: Set `dryRun: true` for new configurations

💡 **Incremental updates**: Re-running is safe - won't create duplicates

💡 **Root permissions**: Grant at `\` root to apply to all subfolders

💡 **Naming**: Pipeline names = YAML filename without extension
   - `backend-build.yml` → Pipeline: "backend-build"

💡 **Folders**: Repository structure = Azure DevOps structure
   - `pipelines/ci/build.yml` → `\ci\build`

---

## 🆘 Get Help

- 📖 **Full Documentation**: [README.md](README.md)
- 🐛 **Troubleshooting**: See README.md troubleshooting section
- 📝 **Logs**: Always check pipeline logs for detailed errors
- ✅ **Test Mode**: Use `dryRun: true` to diagnose issues

---

**Time to Value**: ⏱️ 5 minutes  
**Difficulty**: ⭐⭐☆☆☆ Easy  
**One-Time Setup**: Yes - run anytime to add new pipelines

---

**Happy automating! 🎉**