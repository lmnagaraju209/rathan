# MetLife SOH Functions - Project Guide

Hey there! So you're working on the 14031_metlife-soh-functions project. Let me walk you through what this thing does and how everything fits together. I've been working with this setup for a while, so I'll share what I've learned along the way.

## What This Project Is

This is an Azure Function App written in Node.js 22 for MetLife's Employee Digital platform. Specifically, it's part of the SOH (Service of Health) API operations. Basically, it's a simple HTTP-triggered function that handles GET and POST requests.

The project number "14031" is MetLife's internal project identifier - you'll see it everywhere in resource names, pipeline names, and variable groups. The "soh-functions" part tells you it's related to the Service of Health functionality.

## Project Structure - What's Actually Important

Here's what you're working with:

```
14031_metlife-soh-functions/
├── HttpTrigger/index.js          # This is where your actual function logic lives
├── index.js                      # Entry point - just registers the HttpTrigger
├── build-pipelines-ci.yml        # The pipeline entry point (pretty simple, just extends a template)
├── 14031_digital-soh-function.yml # The real pipeline config with all the steps
├── host.json                     # Function host settings (extension bundles, logging)
├── package.json                  # Dependencies - pretty minimal, just @azure/functions
└── local.settings.json          # Don't commit this! It's for local dev only
```

The `HttpTrigger/index.js` file is where you'll spend most of your time. It's a straightforward HTTP handler that accepts GET and POST requests. The `index.js` at the root just imports and registers it - that's the Azure Functions v4 way of doing things.

## Tech Stack - The Real Deal

We're using:
- **Node.js 22.20.0** - This is pinned in the pipeline, so make sure your local version matches or you'll have "works on my machine" issues
- **Azure Functions Runtime v4** - The latest major version, uses the new programming model
- **@azure/functions v4.0.0** - The SDK for Node.js

The pipeline uses Linux agents (`Auto-Scaling-Agents-Linux`), so if you're developing on Windows, test your paths and file handling. I've seen issues with path separators before.

## Build Pipeline - How It Actually Works

The build pipeline uses a template pattern that MetLife uses across projects. Here's how it actually works:

### The Entry Point (`build-pipelines-ci.yml`)

This file is pretty minimal - it just sets up the pipeline name format, agent pool, and which branches trigger builds. Then it extends a template from another repo (`14031_pipeline-template`). This is MetLife's standard pattern - they keep common pipeline templates in a separate repo so multiple projects can reuse them.

The key line is:
```yaml
extends:
  template: build-pipelines/Gulp/14031_digital-soh-function.yml@14031_pipeline-template
```

This tells Azure DevOps to use the template file from the `14031_pipeline-template` repo, specifically from the `build-pipelines/Gulp/` directory. The template repo is referenced as a resource at the top of the file.

The pipeline name format `$(SourceBranchName)_$(Date:yyyyMMdd)$(Rev:.r)` means you'll see builds like `feat_simple_function_20251016.4`. The `.r` is a revision number that increments if multiple builds happen on the same day.

**Important:** There's also a `14031_digital-soh-function.yml` file in this repo, but that's likely just a local copy for reference. The actual pipeline uses the template from the `14031_pipeline-template` repo. If you need to change pipeline behavior, you'll need to update the template in that repo (or get the DevOps team to do it).

### The Template Pipeline (`14031_digital-soh-function.yml` in template repo)

The actual pipeline logic lives in the template repo at `build-pipelines/Gulp/14031_digital-soh-function.yml`. This is where all the action happens. Let me break down what actually runs:

**1. Environment Setup**
- First thing it does is print all environment variables. Super helpful when debugging - you can see exactly what the pipeline sees
- Then it installs Node.js 22.20.0. This is important - if your local version doesn't match, you might get different behavior

**2. SonarQube Analysis**
- This runs on every build, including PRs. The project key is `Employee_Digital_14031_metlife_soh_functions` - you'll need this if you're looking at SonarQube results
- It excludes a bunch of stuff (tests, coverage, node_modules, config files) which is normal
- The quality gate will block your PR if it fails, so pay attention to the SonarQube comments on your PRs
- One thing I've noticed: it excludes `.yml` files from analysis, so your pipeline files won't show up in SonarQube

**3. Dependency Installation**
- Runs `npm install` - pretty standard
- Downloads a secure `.npmrc` file using the `DownloadSecureFile@1` task. This is for accessing MetLife's private npm registry (Artifactory). The file is named `.npmrc` and gets copied to the working directory
- If this step fails, you probably don't have access to the Artifactory credentials variable group, or the secure file isn't configured in the library

**4. Testing**
- This step is optional - it only runs if you have a `test` script in `package.json`
- It continues on error, so test failures won't break your build (which is both good and bad, depending on your perspective)

**5. Veracode Security Scanning**
- This prepares files for Veracode SAST scanning
- Uses `CopyFiles@2` to copy source files (JS/TS files, package.json) to a staging directory, excluding test files, node_modules, coverage, etc.
- Creates a zip file named `repo_$(Build.BuildId).zip` in the veracode staging directory
- The zip gets published as an artifact (`repo-zip`), but the actual Veracode scan happens elsewhere (probably in a separate pipeline or manually)
- Also publishes the veracode directory as a build artifact for SCA (Software Composition Analysis) scanning
- Note: `VERACODE_PIPELINE_SCAN_ENABLED` is set to `0`, so automated scanning in the pipeline is disabled. The artifacts are just prepared for manual scanning

**6. Build Artifact Creation**
- This is the important part - it creates a zip file with your function code
- Uses `ArchiveFiles@2` to create `azure_function_$(Build.BuildId).zip` in the `build/` directory
- Excludes all the stuff you don't need in production: node_modules, .git, test files, coverage, and the build directory itself (to avoid recursive zipping)
- The zip is published as a pipeline artifact named `azure-function-deployment`
- This artifact is what the release pipeline will deploy
- **Important conditions**: This only runs if it's NOT a pull request AND the branch is in the allowed list (`IsBranchesIncluded`). PR builds skip artifact publishing to save resources

### Pipeline Variables - What You Need to Know

The template defines several variables. Here are the key ones and what they actually mean:

- `NODE_TOOL_VERSION: 22.20.0` - Don't change this unless you know what you're doing. Other projects might depend on this version
- `FUNCTION_APP_NAME: func14031-metlife-soh-functions` - This is your function app name in Azure
- `FUNCTION_APP_RESOURCE_GROUP: rg14031-metlife-soh-functions` - The resource group where everything lives
- `AZ_SUBSCRIPTION: SPAZDO14031NP01` - The Azure subscription ID. You'll need access to this subscription to deploy
- `BUILD_SONAR_PROJECT_KEY_NAME: Employee_Digital_14031_metlife_soh_functions` - SonarQube project identifier
- `VERACODE_IMPORT_RESULTS: true` - Whether to import Veracode scan results
- `VERACODE_PIPELINE_SCAN_ENABLED: 0` - Disables automated Veracode scanning in the pipeline
- `IsPullRequest` - Automatically detects if this is a PR build
- `IsBranchesIncluded` - Checks if the current branch is in the allowed list (master, develop, feat/*, release/*, etc.)

The variable groups (`Veracode`, `ARTIFACTORY_CREDENTIALS`, `SONAR`) are managed by the DevOps team. If you need access or something's not working, talk to them.

### Template Parameters

The template also accepts parameters:
- `build_app` (default: "Nodejs") - Not really used in this pipeline, but kept for consistency with other templates
- `BUILD_REPOSITORY_LOCALPATH` (default: `$(Build.Repository.LocalPath)`) - Used for the test step working directory

### When Does the Pipeline Run?

The template defines triggers that work in combination:

**Branch Triggers:**
- `master`
- `develop`
- `feat/simple_function`
- `feature/feature-*` (any branch starting with `feature/feature-`)
- `develop-de1`, `develop-de2`, `develop-ops` (team-specific dev branches)
- `release/release-*` (any release branch)
- `release/hotfix-*` (any hotfix branch)

**Path Triggers:**
- `HttpTrigger/*` - Any changes in the HttpTrigger directory
- `package.json` - Dependency changes
- `host.json` - Host configuration changes
- `local.settings.json` - Local settings (though this shouldn't be committed)
- `index.js` - Main entry point changes

The path triggers are smart - if you only change a README, the pipeline won't run. But if you touch any of those key files, it will.

**Pull Requests:**
- PR builds run automatically (pre-merge validation)
- PR builds skip artifact publishing (saves resources, since you can't deploy from a PR anyway)
- SonarQube analysis runs differently for PRs - it compares against the target branch instead of doing a full branch analysis

## Release Pipeline - Getting Code to Azure

The release pipeline is where things get interesting. It's a classic Azure DevOps release pipeline (not YAML-based like the build), so it's configured through the UI.

### How Releases Work

When you look at a release (like "Release-12" in your screenshot), you'll see:

1. **Artifacts** - These are the build outputs from the CI pipeline
   - `14031_metlife-soh-func...` - This is your function app artifact (the zip file)
   - `14031_digital-soh-ops` - This is a separate artifact, probably from another repo that handles operations/infrastructure

2. **Stages** - Each environment (DEV, QA, UAT, PROD) is a stage
   - DEV (USD) is the development environment
   - You can see it succeeded in your screenshot

### The DEV (USD) Stage Tasks

Here's what actually happens when you deploy to DEV:

**1. Servicing Ops: Pipeline Environment Variable Mapping**
- This is a PowerShell script that maps environment-specific variables
- It probably sets things like connection strings, API endpoints, etc. based on which environment you're deploying to
- Runs first because other tasks depend on these variables

**2. EventHub Create & Config Hubs**
- Creates and configures Azure Event Hub namespaces and hubs
- Uses the `$(EH_NAMESPACE)` variable
- This is infrastructure setup - if the Event Hub already exists, it probably just updates the configuration

**3. Azure Key Vault (First Time)**
- Retrieves secrets from Key Vault before deploying the function
- You'll see this task twice - once before deployment (for secrets needed during deployment) and once after (for runtime secrets)
- The Key Vault name comes from `$(KEYVAULT_NAME)` variable

**4. Download Build Artifacts**
- Downloads the `azure-function-deployment` artifact from the build pipeline
- This is the zip file with your function code

**5. Apply Settings**
- Uses Azure CLI to set application settings on the Function App
- Things like connection strings, API keys, environment-specific configs
- This runs before deployment so the function has the right settings when it starts

**6. Function App Deploy**
- Actually deploys your code to Azure
- Uses the Azure Functions Deploy task
- This is where your zip file gets uploaded and extracted

**7. Azure Key Vault (Second Time)**
- Retrieves additional secrets that the function needs at runtime
- These get added as application settings that the function can access

### Why Some Tasks Are Disabled

You might notice some tasks show as "Disabled" in the pipeline view. This usually happens when:
- The task is conditionally disabled based on environment (e.g., skip Event Hub creation in production if it already exists)
- The task failed validation or has missing required variables
- It's an optional task that's not needed for this particular deployment

Don't worry about disabled tasks unless your deployment is failing - they're usually intentional.

### Release Triggers

Releases can be triggered:
- **Manually** - Click the "+ Deploy" button (most common for DEV)
- **Scheduled** - Like the 12:00 PM daily schedule you saw
- **Automatically** - When new artifacts are available (usually only for lower environments)

For production, you'll almost always need manual approval. The release pipeline has approval gates between stages.

## Deployment Architecture - The Big Picture

Here's how everything connects:

```
Your Code
    ↓
Azure DevOps Build Pipeline
    ↓ (creates artifacts)
Release Pipeline
    ↓ (deploys to)
Azure Cloud:
  - Function App (func14031-metlife-soh-functions)
  - Event Hub (for messaging)
  - Key Vault (for secrets)
  - Storage Account (required by Functions runtime)
```

The Function App lives in:
- **Subscription**: SPAZDO14031NP01
- **Resource Group**: rg14031-metlife-soh-functions
- **Name**: func14031-metlife-soh-functions

You'll need access to this subscription to see or modify the deployed function. If you don't have access, ask your Azure admin.

## Configuration - The Important Parts

### Local Development

For local development, you need `local.settings.json` (don't commit this!):

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "node"
  }
}
```

The `UseDevelopmentStorage=true` tells it to use the Azure Storage Emulator. If you don't have that installed, you can use a real storage account connection string instead.

### Host Configuration

The `host.json` file is pretty standard:
- Extension Bundle v4.x - This gives you access to triggers and bindings
- Application Insights sampling - Enabled, but excludes Request type (to reduce noise)
- Standard logging setup

You probably won't need to touch this unless you're adding new trigger types.

### Function Configuration

The HTTP trigger is configured in code (in `HttpTrigger/index.js`):
- Methods: GET, POST
- Auth Level: Function (requires a function key)
- Route: `/api/HttpTrigger` (the `/api` prefix is added automatically by Azure Functions)

In production, you'll get a function key that you need to include in requests. For local dev, it's usually open (depending on your `local.settings.json`).

## Development Workflow - How I Actually Work

Here's my typical workflow:

1. **Pull the latest code**
   ```bash
   git checkout develop
   git pull
   ```

2. **Create a feature branch**
   ```bash
   git checkout -b feat/my-feature
   ```

3. **Install dependencies** (if package.json changed)
   ```bash
   npm install
   ```

4. **Start the function locally**
   ```bash
   npm start
   # or
   func start
   ```
   This starts on `http://localhost:7071` by default.

5. **Test it**
   ```bash
   # Quick test
   curl "http://localhost:7071/api/HttpTrigger?name=Test"
   ```

6. **Make changes, test locally, commit**

7. **Push and create a PR**
   - The build pipeline will run automatically
   - SonarQube will analyze your code
   - Fix any issues it finds

8. **After PR is approved and merged**
   - Build pipeline runs again
   - Artifacts are created
   - You can trigger a release manually or wait for the scheduled one

### Testing the Deployed Function

Once deployed, your function will be at:
```
https://func14031-metlife-soh-functions.azurewebsites.net/api/HttpTrigger
```

You'll need the function key to call it. Get it from:
- Azure Portal → Function App → Functions → HttpTrigger → Function Keys
- Or use Azure CLI: `az functionapp function keys list ...`

Then call it:
```bash
curl "https://func14031-metlife-soh-functions.azurewebsites.net/api/HttpTrigger?code=YOUR_KEY&name=Test"
```

## Common Issues and How to Fix Them

### Build Pipeline Fails

**"Node version mismatch"**
- Make sure you're using Node 22.20.0 locally, or at least test with that version
- The pipeline uses this exact version, so differences can cause issues

**"SonarQube quality gate failed"**
- Check the SonarQube comments on your PR
- Usually it's code smells or coverage issues
- You can view the full report in SonarQube (ask for access if you don't have it)

**"npm install failed"**
- Probably an Artifactory access issue
- Check that you have the `ARTIFACTORY_CREDENTIALS` variable group access
- Or it might be a network issue - retry the build

**"Artifact publish failed"**
- Usually a permissions issue
- Check that the build service account has permission to publish artifacts
- This is usually a DevOps team thing

### Release Pipeline Fails

**"Key Vault access denied"**
- The service principal used by the release pipeline needs "Get" permission on the Key Vault
- This is an Azure permissions issue - talk to your Azure admin

**"Function App deployment failed"**
- Check that the Function App name is correct
- Verify the resource group exists and the service principal has access
- Check the deployment logs - they usually tell you what went wrong

**"Event Hub creation failed"**
- Probably a permissions issue
- The service principal needs Contributor role on the resource group or Event Hub namespace
- Or the Event Hub might already exist and there's a conflict

### Local Development Issues

**"Function won't start"**
- Check Node.js version: `node --version` should be 22.x
- Make sure `npm install` completed successfully
- Check that port 7071 isn't already in use
- Look at the error message - Azure Functions Core Tools usually gives helpful errors

**"Can't install dependencies"**
- If you're behind a corporate firewall, you might need to configure npm proxy
- Or you might need the `.npmrc` file for Artifactory access (the pipeline downloads this automatically, but locally you need to set it up)

**"Function works locally but not in Azure"**
- Check application settings in Azure Portal - make sure all required settings are there
- Check the function logs in Application Insights
- Verify the function key is correct
- Check that the function is actually deployed (sometimes deployments fail silently)

## Branching Strategy - How We Actually Work

- **`master`** - Production code. Only merge here when you're ready to deploy to prod
- **`develop`** - Main development branch. Most feature work happens here
- **`develop-de1`, `develop-de2`, `develop-ops`** - Team-specific dev branches. Not sure if these are still used, but they're in the pipeline config
- **`feat/*`** - Feature branches. Create these from `develop`, work on your feature, then PR back to `develop`
- **`release/*`** - Release branches. For preparing a release (version bumps, final testing, etc.)
- **`release/hotfix-*`** - Hotfix branches. For urgent production fixes

The pipeline triggers on most of these branches, so you'll get builds automatically.

## Things I Wish I Knew When I Started

1. **The pipeline template is in a different repo** - The actual pipeline logic is in `14031_pipeline-template` repo at `build-pipelines/Gulp/14031_digital-soh-function.yml`. The file in this repo is just a reference copy. If you need to change pipeline behavior, you'll need to update the template repo (or get the DevOps team to do it)

2. **SonarQube will block your PR** - The quality gate is set to wait (`sonar.qualitygate.wait=true`), so your PR won't merge if SonarQube fails. Don't ignore it, fix the issues early

3. **PR builds don't publish artifacts** - This is intentional. The `IsPullRequest` variable prevents artifact publishing on PRs, which saves build resources. Only merged builds create deployable artifacts

4. **The npmrc file is a secure file** - It's stored in the Azure DevOps library as a secure file, not in the repo. The pipeline downloads it automatically. For local dev, you might need to get this file from someone or configure Artifactory access manually

5. **SonarQube exclusions are extensive** - It excludes a lot of stuff (yml files, d.ts files, coverage, tests, etc.). This is normal - you don't want to analyze generated files or test code

6. **The function key changes** - If you're testing against Azure, the key might rotate, so don't hardcode it

7. **Application Insights is your friend** - When something breaks in Azure, check the logs there first

8. **The release pipeline is UI-based** - You can't version control it like the build pipeline, so changes need to be made in Azure DevOps UI

9. **Disabled tasks are usually fine** - Don't panic if you see disabled tasks, they're often conditionally disabled

10. **The test step continues on error** - Tests can fail and the build will still succeed. This is both good (doesn't block on flaky tests) and bad (you might miss real failures). Check test results manually if you're unsure

## Getting Help

If you're stuck:
1. Check the build/release logs - they usually tell you what's wrong
2. Look at Application Insights for runtime errors
3. Check Azure Portal for the Function App status and logs
4. Ask the DevOps team about pipeline issues
5. Ask the Azure admin about permission/resource issues

The project is in the **MetLife-Global / Employee_Digital** organization in Azure DevOps. 

The pipeline template repo is **14031_pipeline-template** and the actual template file is at `build-pipelines/Gulp/14031_digital-soh-function.yml`. If you need to modify the pipeline, that's where you'll need to make changes (or coordinate with the DevOps team).

## Quick Reference

**Function App Name**: `func14031-metlife-soh-functions`  
**Resource Group**: `rg14031-metlife-soh-functions`  
**Subscription**: `SPAZDO14031NP01`  
**Local URL**: `http://localhost:7071/api/HttpTrigger`  
**Azure URL**: `https://func14031-metlife-soh-functions.azurewebsites.net/api/HttpTrigger`  
**Node Version**: 22.20.0  
**Functions Runtime**: v4

That's about it! If you have questions or run into issues, the logs are usually pretty helpful. Good luck!
