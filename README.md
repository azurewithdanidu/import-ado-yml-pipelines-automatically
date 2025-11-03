# Azure DevOps Pipeline Creator

Automated solution for creating multiple Azure DevOps pipelines from YAML files in your repository with a single click.

## Overview

This self-contained Azure Pipeline automatically discovers and creates Azure DevOps pipelines from YAML files in your repository. It's **fully self-contained** - no external parameters for organization, project, repository, or branch needed. Everything is derived from the running pipeline's context.

## ✨ Key Features

- 🚀 **Self-Contained**: Uses built-in Azure DevOps variables - no org/project/repo/branch parameters needed
- 🔍 **Smart Discovery**: Automatically finds YAML files and excludes archive/template folders
- 🛡️ **Duplicate Protection**: Checks for existing pipelines before creation
- 📁 **Hierarchy Preservation**: Maintains folder structure from repository to Azure DevOps
- 🎯 **Queue Auto-Detection**: Automatically finds and uses the correct agent pool
- 🔐 **Permission Guidance**: Detailed error messages with step-by-step fix instructions
- 🧪 **Dry Run Mode**: Preview changes before creating anything
- ⚡ **Optimized**: Compact 138-line pipeline (~6,700 characters)
- 📊 **Summary Reports**: Clear statistics of created/skipped/failed pipelines

## Prerequisites

1. **Azure DevOps Project** with admin or Build Administrator permissions
2. **Repository** containing this pipeline YAML file
3. **Permissions** - the Build Service account needs "Create build pipeline" permission (see Setup below)

## 🚀 Quick Setup (3 Steps)

### Step 1: Grant Build Service Permissions

1. Go to **Project Settings** > **Pipelines** (left sidebar)
2. Click **⋮** (three dots) next to any folder or at the root level
3. Click **Security**
4. Add: **[YourProject] Build Service ([YourOrg])**
5. Set **"Create build pipeline"** permission to **Allow**
6. ✅ Done! This applies to all subfolders

### Step 2: Create the Pipeline

1. In Azure DevOps, go to **Pipelines** > **New Pipeline**
2. Select your repository
3. Choose **Existing Azure Pipelines YAML file**
4. Select `azure-pipelines-creator.yml`
5. Click **Save** (or **Run** to execute immediately)

### Step 3: Run the Pipeline

1. Click **Run pipeline**
2. Configure parameters:
   - **gitFolderPath**: `pipelines` (folder containing sample YAML files) in this case for this exmple you can have your own fodler
   - **subfolderStructure**: `*` (scan all) or `ci,cd` (specific folders)
   - **azureDevOpsFolderPath**: `\\` (root) or `\\MyPipelines` (custom folder)
   - **dryRun**: `true` (preview) or `false` (create)
3. Click **Run**
4. Check the logs for summary!

## 📋 Pipeline Parameters

| Parameter | Description | Default | Example |
|-----------|-------------|---------|---------|
| **gitFolderPath** | Folder containing pipeline YAML files (relative to repo root) | `pipelines` | `devops/pipelines` |
| **subfolderStructure** | Comma-separated subfolders to scan, or `*` for all | `*` | `ci,cd` |
| **azureDevOpsFolderPath** | Target folder in Azure DevOps for created pipelines | `\\` | `\\Automated` |
| **dryRun** | Preview mode - shows what would be created without creating | `false` | `true` |

### 🔍 What Gets Excluded Automatically

The pipeline automatically excludes:
- Folders containing **"archive"** (e.g., `archive/`, `old-archive/`, `archive-2024/`)
- Folders containing **"template"** (e.g., `templates/`, `my-templates/`, `template-backup/`)
- Case-insensitive matching

## 📂 Folder Structure Example

### Your Repository
```
your-repo/
├── azure-pipelines-creator.yml  ← This pipeline
└── pipelines/
    ├── ci/
    │   ├── backend-build.yml
    │   └── frontend-build.yml
    ├── cd/
    │   ├── dev-deploy.yml
    │   └── prod-deploy.yml
    ├── archive/                ← EXCLUDED
    │   └── old-pipeline.yml
    └── templates/              ← EXCLUDED
        └── base-template.yml
```

### Result in Azure DevOps (with `azureDevOpsFolderPath: \\`)
```
\ci\
  ├── backend-build
  └── frontend-build
\cd\
  ├── dev-deploy
  └── prod-deploy
```

## 🔧 How It Works

1. **Initialization**: Sets up variables from built-in pipeline context
   - Repository: `$(Build.Repository.Uri)`
   - Branch: `$(Build.SourceBranchName)`
   - Organization: `$(System.CollectionUri)`
   - Project: `$(System.TeamProject)`

2. **Azure CLI Setup**: Configures CLI defaults with organization and project

3. **YAML Discovery**: Scans for `.yml` and `.yaml` files
   - Applies subfolder filtering if specified
   - Excludes folders containing "archive" or "template"

4. **Duplicate Detection**: Retrieves existing pipelines and creates lookup map

5. **Queue Detection**: Automatically finds "Azure Pipelines" queue or first available queue

6. **Pipeline Creation**: For each YAML file:
   - Calculates target folder (preserves hierarchy)
   - Checks if pipeline exists
   - Creates pipeline with `az pipelines create`
   - Handles permission errors with detailed instructions

7. **Summary**: Reports created/skipped/failed counts

## 📊 Sample Output

```
Repository: https://dev.azure.com/myorg/myproject/_git/myrepo | Branch: main
Found 4 YAML file(s)

SKIP: frontend-build (exists)
CREATED: backend-build
CREATED: dev-deploy
CREATED: prod-deploy

=== SUMMARY ===
Created: 3 | Skipped: 1 | Failed: 0
```

## ❗ Troubleshooting

### Issue: "Access denied. Build Service needs Create build pipeline permissions"

**Solution:**
1. Go to **Project Settings** > **Pipelines**
2. Click **Security** (you may need to click ⋮ on a folder first)
3. Add: **[YourProject] Build Service ([YourOrg])**
4. Grant **"Create build pipeline"** = **Allow**
5. Tip: Grant at root `\` to apply to all folders

### Issue: "Cannot find a hosted pool queue"

**Solution:** This is usually auto-detected. If it fails:
1. Pipeline will attempt creation without queue ID
2. Azure DevOps will use default queue
3. If issues persist, check Project Settings > Agent Pools

### Issue: "No YAML files found"

**Solutions:**
- Verify `gitFolderPath` exists in your repo
- Check files have `.yml` or `.yaml` extension
- Files must be committed to the current branch
- Try `subfolderStructure: *` to scan all folders

### Issue: "Pipeline already exists" for all pipelines

**This is expected behavior!**
- The pipeline won't create duplicates
- Re-running is safe
- To recreate: Delete existing pipelines in Azure DevOps first

## 💡 Best Practices

1. ✅ **Always test with `dryRun: true` first**
2. ✅ **Keep pipeline YAML files in dedicated folder** (e.g., `pipelines/`)
3. ✅ **Use descriptive names** (lowercase, hyphen-separated)
4. ✅ **Grant permissions at root level** (applies to all subfolders)
5. ✅ **Re-run after adding new YAML files** (won't duplicate existing)
6. ✅ **Review logs** for excluded files/folders

## 🎯 Use Cases

### Use Case 1: Bulk Pipeline Creation
Create 50+ pipelines from a repository with complex folder structure in one run.

### Use Case 2: Multi-Environment Setup
Organize pipelines by environment:
```
pipelines/
├── dev/     → \Dev\*
├── staging/ → \Staging\*
└── prod/    → \Prod\*
```

### Use Case 3: Incremental Updates
Add new YAML files to repo, re-run pipeline - only new pipelines are created.

### Use Case 4: Monorepo Management
Manage hundreds of microservice pipelines with consistent naming and structure.

## 🔒 Security Considerations

- Uses `$(System.AccessToken)` with Build Service permissions only
- No personal access tokens required
- All actions logged in Azure DevOps audit logs
- Pipeline YAML files should be code-reviewed
- Consider branch protection for pipeline definitions

## ⚠️ Limitations

1. **Creates Only**: Doesn't update or delete existing pipelines
2. **Same Repository**: All pipelines reference the repository where this pipeline runs
3. **Current Branch**: All created pipelines use the branch where this pipeline runs
4. **No Validation**: Doesn't validate YAML syntax before creation
5. **Sequential Processing**: Processes files one at a time (not parallel)

## 🤝 Contributing

Suggestions and improvements welcome! This is a community tool.

## 📄 License

MIT License - Use freely in your Azure DevOps environments

---

**Version**: 2.0.0 (Simplified & Self-Contained)  
**Last Updated**: November 2025  
**Compatibility**: Azure DevOps Server 2020+ and Azure DevOps Services  
**Pipeline Size**: 138 lines / ~6,700 characters

---

## 📚 Additional Resources

- [Azure Pipelines Documentation](https://docs.microsoft.com/azure/devops/pipelines/)
- [Azure CLI DevOps Extension](https://docs.microsoft.com/cli/azure/devops)
- [YAML Schema Reference](https://docs.microsoft.com/azure/devops/pipelines/yaml-schema)

---

**Questions?** Check the troubleshooting section or review pipeline logs for detailed error messages.

````
