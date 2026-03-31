# Import Azure DevOps Pipelines Automatically

A self-contained Azure Pipeline that discovers YAML files in your repository and creates Azure DevOps pipeline definitions for each one -- no manual clicking required.

## Overview

Managing dozens (or hundreds) of pipeline YAML files by hand is tedious. This project provides a single "creator" pipeline (`azure-pipelines-creator.yml`) that you run once. It scans a folder in your repo, finds every `.yml` / `.yaml` file, and registers a corresponding pipeline in Azure DevOps while preserving the folder hierarchy.

Everything is **self-contained** -- the pipeline reads its organisation, project, repo, and branch from built-in Azure DevOps variables, so there's nothing extra to configure.

## Key Features

- **Self-Contained** -- derives org/project/repo/branch from the running pipeline context
- **Smart Discovery** -- recursively finds `.yml` and `.yaml` files, automatically excluding `archive` and `template` folders
- **Duplicate Protection** -- checks for existing pipelines before creation; re-running is always safe
- **Folder Hierarchy** -- mirrors sub-folder structure (e.g. `pipelines/ci/` -> `\ci\` in Azure DevOps)
- **Queue Auto-Detection** -- locates the "Azure Pipelines" hosted pool automatically
- **Dry-Run Mode** -- preview what would be created without touching anything
- **Permission Guidance** -- surfaces specific fix instructions when the Build Service lacks permissions
- **Summary Report** -- prints created / skipped / failed counts at the end

## Repository Structure

```
import-ado-yml-pipelines-automatically/
  azure-pipelines-creator.yml        # The creator pipeline
  QUICKSTART.md
  README.md
  pipelines/
    ci/
      backend-build.yml               # .NET 8 build + test
      frontend-build.yml              # Node 20 build + lint + test
    cd/
      dev-deploy.yml                  # Deploy to Dev (App Service)
      prod-deploy.yml                 # Deploy to Prod (slot swap)
    infrastructure/
      provision.yml                   # Bicep validate + deploy
```

The `pipelines/` folder ships with sample YAML files covering CI, CD, and infrastructure provisioning.
Replace or extend them with your own pipelines -- the creator will pick them up automatically.

## Prerequisites

1. An **Azure DevOps project** where you have Build Administrator (or equivalent) permissions.
2. The repository must contain `azure-pipelines-creator.yml` and at least one pipeline YAML file under the scan folder.
3. The **Build Service** account must have the **"Create build pipeline"** permission (see Setup below).

## Quick Setup

### 1 -- Grant Build Service Permissions

1. Go to **Project Settings > Pipelines** in the left sidebar.
2. Click the three-dot menu (**...**) next to any folder (or at root) and select **Security**.
3. Add **[YourProject] Build Service ([YourOrg])**.
4. Set **"Create build pipeline"** to **Allow**.

Granting this at the root `\` folder propagates to all sub-folders.

### 2 -- Create the Creator Pipeline

1. Navigate to **Pipelines > New Pipeline**.
2. Select your repository.
3. Choose **Existing Azure Pipelines YAML file**.
4. Pick `azure-pipelines-creator.yml`.
5. Click **Save** (or **Run** to execute straight away).

### 3 -- Run the Pipeline

1. Click **Run pipeline**.
2. Fill in the parameters (see table below).
3. Click **Run** and check the logs for the summary.

## Parameters

The creator pipeline exposes four parameters:

| Parameter | Description | Default | Example |
|---|---|---|---|
| `gitFolderPath` | Folder containing your pipeline YAML files (relative to repo root) | `pipelines` | `devops/pipelines` |
| `subfolderStructure` | Comma-separated sub-folder names to scan, or `*` for all | `*` | `ci,cd` |
| `azureDevOpsFolderPath` | Target folder in Azure DevOps where pipelines are created | `\\` | `\\Automated` |
| `dryRun` | When `true`, logs what would happen without creating anything | `false` | `true` |

These are declared at the top of `azure-pipelines-creator.yml`:

```yaml
parameters:
  - name: gitFolderPath
    displayName: 'Git Folder Path (where YAML files are stored)'
    type: string
    default: 'pipelines'

  - name: subfolderStructure
    displayName: 'Subfolder Structure to Scan (e.g., ci,cd or * for all)'
    type: string
    default: '*'

  - name: azureDevOpsFolderPath
    displayName: 'Azure DevOps Root Folder Path for Created Pipelines'
    type: string
    default: '\\'

  - name: dryRun
    displayName: 'Dry Run (Preview without creating)'
    type: boolean
    default: false
```

### Automatic Exclusions

Any folder whose path contains **"archive"** or **"template"** (case-insensitive) is skipped. This is controlled by the following snippet inside the inline PowerShell script:

```powershell
$excludePatterns = @('*archive*', '*template*')
$yamlFiles = Get-ChildItem -Path $searchPath -Include *.yml,*.yaml -Recurse -File |
    Where-Object { $path = $_.FullName; -not ($excludePatterns | Where-Object { $path -like $_ }) }
```

## How It Works

The pipeline runs a single PowerShell task with the following steps:

### 1. Initialise variables from the pipeline context

```powershell
$repositoryUrl      = "$(Build.Repository.Uri)"
$targetBranch       = "$(Build.SourceBranchName)"
$organizationUrl    = "$(System.CollectionUri)"
$projectName        = "$(System.TeamProject)"
```

### 2. Configure the Azure DevOps CLI

```powershell
az devops configure --defaults organization="$organizationUrl" project="$projectName" --use-git-aliases true
```

### 3. Discover YAML files

The script calls `Get-ChildItem` to find all `.yml` / `.yaml` files under the scan folder, filters out excluded patterns, and optionally restricts to specific sub-folders:

```powershell
if ($subfolderStructure -ne '*') {
    $allowedFolders = $subfolderStructure -split ',' | ForEach-Object { $_.Trim() }
    $yamlFiles = $yamlFiles | Where-Object {
        $rel = $_.FullName.Substring($searchPath.Length + 1)
        $first = ($rel -split '\\')[0]
        $allowedFolders -contains $first
    }
}
```

### 4. Build a lookup of existing pipelines

```powershell
$existing = @{}
$pipelines = az pipelines list --output json | ConvertFrom-Json
$pipelines | ForEach-Object { $existing["$($_.path)/$($_.name)".ToLower()] = $_ }
```

### 5. Detect the hosted queue

```powershell
$queues = az pipelines queue list --output json 2>$null | ConvertFrom-Json
$queue  = $queues | Where-Object { $_.name -eq 'Azure Pipelines' } | Select-Object -First 1
if ($queue) { $queueId = $queue.id }
```

### 6. Create each pipeline (or log dry-run)

For every discovered YAML file the script calculates the target folder, checks the lookup map, and either skips (duplicate), logs (dry-run), or creates:

```powershell
$args = @('pipelines','create','--name',$name,'--repository',$repositoryUrl,
          '--branch',$targetBranch,'--yaml-path',$yamlPath,'--skip-run')
if ($queueId -gt 0)       { $args += '--queue-id';    $args += $queueId }
if ($targetFolder -ne '\') { $args += '--folder-path'; $args += $targetFolder }

$result = & az $args 2>&1
```

### 7. Print summary

```
=== SUMMARY ===
Created: 3 | Skipped: 1 | Failed: 0
```

## Folder Mapping Example

Given the repository structure shipped in this repo and a run with `gitFolderPath = pipelines`, `subfolderStructure = *`, and `azureDevOpsFolderPath = \\`:

| Repository Path | Azure DevOps Pipeline (name / folder) |
|---|---|
| `pipelines/ci/backend-build.yml` | `backend-build` in `\ci` |
| `pipelines/ci/frontend-build.yml` | `frontend-build` in `\ci` |
| `pipelines/cd/dev-deploy.yml` | `dev-deploy` in `\cd` |
| `pipelines/cd/prod-deploy.yml` | `prod-deploy` in `\cd` |
| `pipelines/infrastructure/provision.yml` | `provision` in `\infrastructure` |

## Sample Pipeline Files

The `pipelines/` folder includes working examples you can use as starting points.

### CI -- .NET Backend Build (`pipelines/ci/backend-build.yml`)

Builds a .NET 8 solution, runs unit tests with code coverage, and publishes results:

```yaml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - src/backend/**

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - task: UseDotNet@2
            inputs:
              version: '8.x'
          - task: DotNetCoreCLI@2
            inputs:
              command: 'build'
              projects: 'src/backend/**/*.csproj'
              arguments: '--configuration Release'
          - task: DotNetCoreCLI@2
            inputs:
              command: 'test'
              projects: 'src/backend/**/*Tests.csproj'
              arguments: '--configuration Release --collect:"XPlat Code Coverage"'
```

### CI -- Node.js Frontend Build (`pipelines/ci/frontend-build.yml`)

Installs dependencies, lints, runs tests, and builds the frontend:

```yaml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - src/frontend/**

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'
          - task: Npm@1
            inputs:
              command: 'install'
              workingDir: 'src/frontend'
          - task: Npm@1
            inputs:
              command: 'custom'
              customCommand: 'run lint'
              workingDir: 'src/frontend'
          - task: Npm@1
            inputs:
              command: 'custom'
              customCommand: 'run build'
              workingDir: 'src/frontend'
```

### CD -- Dev Deployment (`pipelines/cd/dev-deploy.yml`)

Downloads build artifacts and deploys to an Azure App Service with a smoke test:

```yaml
trigger: none   # Manually triggered

stages:
  - stage: DeployDev
    jobs:
      - deployment: DeployJob
        environment: 'dev'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: DownloadBuildArtifacts@1
                  inputs:
                    artifactName: 'frontend-build'
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: '$(azureSubscription)'
                    appName: 'myapp-dev'
                    package: '$(System.ArtifactsDirectory)/frontend-build'
```

### Infrastructure -- Bicep Provisioning (`pipelines/infrastructure/provision.yml`)

Validates and deploys Bicep templates with parameterised environment and deployment mode:

```yaml
parameters:
  - name: environment
    type: string
    default: 'dev'
    values: [dev, staging, prod]
  - name: deploymentMode
    type: string
    default: 'Incremental'
    values: [Incremental, Complete]

trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure/**
```

## Sample Output

```
Repository: https://dev.azure.com/myorg/myproject/_git/myrepo | Branch: main
Found 5 YAML file(s)

SKIP: frontend-build (exists)
CREATED: backend-build
CREATED: dev-deploy
CREATED: prod-deploy
CREATED: provision

=== SUMMARY ===
Created: 4 | Skipped: 1 | Failed: 0
```

## Troubleshooting

### "Access denied. Build Service needs Create build pipeline permissions"

1. Go to **Project Settings > Pipelines**.
2. Open **Security** (click the three-dot menu on a folder if needed).
3. Add **[YourProject] Build Service ([YourOrg])**.
4. Set **"Create build pipeline"** to **Allow**.
5. Granting at the root `\` applies to all sub-folders.

### "Cannot find a hosted pool queue"

The queue is normally auto-detected. If it fails, the pipeline attempts creation without a queue ID and Azure DevOps will assign the default queue. Check **Project Settings > Agent Pools** if issues persist.

### "No YAML files found"

- Confirm `gitFolderPath` matches a folder in your repo.
- Check that files use `.yml` or `.yaml` extensions.
- Files must be committed to the branch the pipeline runs against.
- Try `subfolderStructure: *` to scan all sub-folders.

### All pipelines show "SKIP (exists)"

This is expected when every pipeline has already been created. The creator never creates duplicates. To recreate a pipeline, delete it from Azure DevOps first, then re-run.

## Best Practices

1. **Always dry-run first** -- set `dryRun: true` when trying a new configuration.
2. **Keep pipeline YAML in a dedicated folder** (e.g. `pipelines/`) to keep the scan targeted.
3. **Use descriptive, lowercase, hyphen-separated names** -- the file name becomes the pipeline name.
4. **Grant permissions at root level** so new sub-folders are covered automatically.
5. **Re-run after adding new YAML files** -- existing pipelines are skipped, only new ones are created.

## Use Cases

**Bulk creation** -- Register 50+ pipelines from a complex folder structure in a single run.

**Multi-environment layout** -- Organise by environment and let the creator mirror the structure:

```
pipelines/
  dev/      ->  \dev\
  staging/  ->  \staging\
  prod/     ->  \prod\
```

**Incremental updates** -- Commit new YAML files, re-run the creator, and only the new pipelines are added.

## License

This project is provided as-is. See the repository for licence details.

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
