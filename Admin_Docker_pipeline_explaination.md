# Migration Changes: Dockerfile and Pipeline Updates

This document outlines the changes made to the Dockerfile and Azure DevOps pipeline YAML file as part of the migration to align with container governance requirements and JFrog Artifactory integration.

## Overview

The migration involved two main areas:
1. **Dockerfile**: Updated to use JFrog Artifactory base images, implement proper multi-stage builds, and add MetLife compliance labels
2. **Pipeline YAML**: Integrated template-based Docker builds, added required variable groups, and enhanced quality gates

---

## Dockerfile Changes

### Base Image and Labels

**OLD (Line 1-2):**
```dockerfile
#FROM node:22-alpine AS base
FROM acr14031e1dv01.azurecr.io/infra/node:22-alpine AS base
```

**NEW (Line 1-11):**
```dockerfile
FROM jfrog-artifactory.metlife.com/docker-metlife-base-images-virtual/library/node:22-alpine AS base

LABEL com.metlife.image.app.description="Digital SOH Admin - Node.js React application"
LABEL com.metlife.image.app.maintainer="devops@metlife.com"
LABEL com.metlife.image.app.dockerfile="Dockerfile"
LABEL com.metlife.image.app.dpccode="14031"
LABEL com.metlife.image.app.eaicode="digital-soh-admin"
LABEL com.metlife.image.app.snow-group="Digital-SOH"
```

**Why:** Container governance requires all base images to come from JFrog Artifactory instead of public registries or ACR. The MetLife labels are mandatory for image tracking and compliance. These labels help identify the application, maintainer, and associated service now group for proper asset management.

### Base Stage Setup

**OLD (Line 3-7):**
```dockerfile
WORKDIR /app

RUN mkdir -p /app && chown node:node /app
USER node
```

**NEW (Line 13-21):**
```dockerfile
USER root
RUN apk add --no-cache git && \
    mkdir -p /app && \
    chown -R node:node /app

USER node
WORKDIR /app
```

**Why:** Git is required for husky hooks that run during npm install. The old approach didn't install git, which could cause build failures when husky tries to set up git hooks. Using `chown -R` ensures all subdirectories have correct ownership.

### Multi-Stage Build Structure

**OLD:** Single stage build that did everything in one stage.

**NEW:** Introduced separate `prod-build` and `prod` stages.

**Why:** Multi-stage builds separate the build environment from the runtime environment. This means:
- Build tools and dependencies stay in the build stage
- Only production dependencies and built artifacts go into the final image
- Smaller final image size (no dev dependencies, no source code)
- Better security (fewer attack surfaces in production image)

### NPM Installation and Authentication

**OLD (Line 23):**
```dockerfile
RUN npm install
```

**NEW (Line 35-50):**
```dockerfile
COPY --chown=node:node package*.json ./

ARG NPM_TOKEN
RUN echo "registry=https://jfrog-artifactory.metlife.com/artifactory/api/npm/remote-npm/" > .npmrc && \
    if [ -n "$NPM_TOKEN" ]; then \
        echo "//jfrog-artifactory.metlife.com/artifactory/api/npm/remote-npm/:_authToken=$NPM_TOKEN" >> .npmrc; \
    fi && \
    mkdir -p /tmp/dummy-bin && \
    echo '#!/bin/sh' > /tmp/dummy-bin/husky && \
    echo 'exit 0' >> /tmp/dummy-bin/husky && \
    chmod +x /tmp/dummy-bin/husky && \
    PATH="/tmp/dummy-bin:$PATH" NODE_ENV=development npm install && \
    rm -rf /tmp/dummy-bin && \
    rm -f .npmrc
```

**Why:** 
- NPM packages must be pulled from JFrog Artifactory, not public npm registry
- The NPM_TOKEN build argument provides authentication for private packages
- Husky git hooks fail in Docker builds, so we create a dummy husky script that exits successfully
- `.npmrc` is removed after install to prevent credential leakage in the image layers
- Copying `package*.json` first improves Docker layer caching

### Build Stage Separation

**OLD:** Everything happened in one stage, including build and runtime setup.

**NEW (Line 27-117):** Separate `prod-build` stage that:
- Installs all dependencies (including dev dependencies)
- Copies source code
- Sets all React environment variables
- Runs `npm run build`

**Why:** This separation allows us to:
- Keep build tools out of the final image
- Only copy the built artifacts (`dist` folder) to the production stage
- Reduce final image size significantly

### Source Code Copying

**OLD (Line 25):**
```dockerfile
COPY --chown=node:node . ./
```

**NEW (Line 52-54):**
```dockerfile
COPY --chown=node:node . ./
RUN rm -f .npmrc
```

**Why:** After copying source code, we explicitly remove any `.npmrc` file that might have been in the source tree. This prevents accidentally including credentials in the image.

### Build Output Directory

**OLD (Line 88):**
```dockerfile
RUN npm run build
```

The old Dockerfile didn't explicitly copy the build output. It assumed the build happened in place.

**NEW (Line 147):**
```dockerfile
COPY --chown=node:node --from=prod-build /app/dist ./dist
```

**Why:** Webpack outputs to the `dist` directory, not `build`. The old approach was incorrect. We now explicitly copy the built artifacts from the build stage to the production stage.

### Production Dependencies Only

**OLD:** Installed all dependencies including dev dependencies in the final image.

**NEW (Line 132-144):**
```dockerfile
COPY --chown=node:node package*.json ./
ARG NPM_TOKEN
RUN echo "registry=https://jfrog-artifactory.metlife.com/artifactory/api/npm/remote-npm/" > .npmrc && \
    if [ -n "$NPM_TOKEN" ]; then \
        echo "//jfrog-artifactory.metlife.com/artifactory/api/npm/remote-npm/:_authToken=$NPM_TOKEN" >> .npmrc; \
    fi && \
    mkdir -p /tmp/dummy-bin && \
    echo '#!/bin/sh' > /tmp/dummy-bin/husky && \
    echo 'exit 0' >> /tmp/dummy-bin/husky && \
    chmod +x /tmp/dummy-bin/husky && \
    PATH="/tmp/dummy-bin:$PATH" npm install --omit=dev && \
    rm -rf /tmp/dummy-bin && \
    rm -f .npmrc
```

**Why:** The production stage only installs runtime dependencies using `--omit=dev`. This significantly reduces the final image size and removes build tools that aren't needed at runtime.

### Gettext Installation

**OLD (Line 13-14):**
```dockerfile
USER root
RUN apk add gettext
```

**NEW (Line 125-126):**
```dockerfile
USER root
RUN apk add --no-cache gettext
```

**Why:** The `--no-cache` flag prevents apk from storing the package index locally, reducing image size. Gettext is needed for `envsubst` if environment variable substitution is required in entrypoint scripts.

### File Copying Order

**OLD (Line 92-94):**
```dockerfile
COPY --chown=node:node package*.json .npmrc entrypoint_prod.sh ./
COPY --chown=node:node server.js ./
```

**NEW (Line 149-152):**
```dockerfile
COPY --chown=node:node server.js ./
COPY --chown=node:node entrypoint_prod.sh ./
RUN chmod +x ./entrypoint_prod.sh
```

**Why:** We no longer copy `.npmrc` in the production stage since we create it dynamically. The entrypoint script is copied separately and made executable.

### GIT_COMMIT Environment Variable

**NEW (Line 157-158):**
```dockerfile
ARG GIT_COMMIT=latest
ENV GIT_COMMIT $GIT_COMMIT
```

**Why:** This allows the application to know which git commit it was built from, useful for debugging and version tracking. The default value prevents build failures if the argument isn't provided.

### CMD Format

**OLD (Line 101):**
```dockerfile
CMD npm run serve
```

**NEW (Line 164):**
```dockerfile
CMD ["npm", "run", "serve"]
```

**Why:** Using the exec form (JSON array) is preferred as it doesn't invoke a shell, reducing overhead and potential security issues.

### Removed Duplicate Environment Variable

**OLD (Line 43-45, 63-65):**
The old Dockerfile had `REACT_APP_PING_END_SESSION_URL` defined twice.

**NEW:** Only defined once (Line 76-78).

**Why:** Duplicate definitions were causing confusion and potential issues. Each environment variable should be defined only once.

---

## Pipeline YAML Changes

### Template Repository Reference

**OLD:** No template repository reference.

**NEW (Line 17-20):**
```yaml
## CG Requirement #####################################################
  - repository: templates
    name: DevOps-Patterns-Practices-Examples/Pipeline-Foundation
    type: git
#######################################################################
```

**Why:** Container governance requires using standardized Docker build templates instead of custom build steps. This ensures consistent security scanning, image tagging, and registry handling across all projects.

### Variable Groups

**OLD (Line 20-23):**
```yaml
    - group: Veracode
    - group: ARTIFACTORY_CREDENTIALS
    - group: SONAR
```

**NEW (Line 26-33):**
```yaml
    - group: Veracode
    - group: ARTIFACTORY_CREDENTIALS
    - group: SONAR
## CG Requirement #####################################################
    - group: JFROG_BASE_IMAGE_PULL # JFROG credentials
    - group: PRISMA_CLOUD #For Prisma/Twistlock credentials
    - group: NPM_TOKEN_GROUP #For NPM token
```

**Why:** 
- `JFROG_BASE_IMAGE_PULL`: Provides credentials to pull base images from JFrog Artifactory
- `PRISMA_CLOUD`: Required for container image scanning (Twistlock/Prisma Cloud integration)
- `NPM_TOKEN_GROUP`: Provides NPM authentication token for pulling packages from JFrog Artifactory

### Push Image Variable

**NEW (Line 43-45):**
```yaml
## CG Requirement #####################################################
    - name: pushImage ## Push image for all branches except pull requests
      value: ${{ ne(variables['Build.Reason'], 'PullRequest') }}
#######################################################################
```

**Why:** The template uses this variable to determine whether to push the image to the registry. Pull requests typically don't push images to avoid cluttering the registry with temporary builds.

### SonarQube Version Update

**OLD (Line 77):**
```yaml
      - task: SonarQubePrepare@7
```

**NEW (Line 92):**
```yaml
      - task: SonarQubePrepare@7
```

Actually, both use @7, but the old file in the OLD folder shows @5. The change from @5 to @7 was made to use the latest SonarQube task version with better features and bug fixes.

**Why:** Updated to the latest SonarQube extension version for improved functionality and security.

### NPM Install Changes

**OLD (Line 126-135):**
```yaml
      - bash: |
          echo "Node version :: $(node --version)"
          echo "npm version :: $(npm --version)"
          echo "npx version :: $(npx --version)"

          echo 'NPM:: Install and clean cache...'
          npm cache clean --force
          npm install
          #npm install --save-dev jest-junit
```

**NEW (Line 141-151):**
```yaml
      - bash: |
          echo "Node version :: $(node --version)"
          echo "npm version :: $(npm --version)"
          echo "npx version :: $(npx --version)"

          echo 'NPM: Installing dependencies...'
          npm install --save-dev jest-junit
          #npm install --save-dev jest-cobertura-reporter
          #npm run patch-script
```

**Why:** 
- Removed `npm cache clean --force` as it's not necessary and slows down builds
- Added `jest-junit` as a dev dependency to generate JUnit XML test reports for Azure DevOps
- This enables test result publishing in the pipeline

### Linting Step

**NEW (Line 153-158):**
```yaml
      - bash: |
          echo 'Running ESLint...'
          npm run lint
        workingDirectory: $(System.DefaultWorkingDirectory)
        displayName: 'Run Linting'
        continueOnError: true
```

**Why:** Added linting step to catch code quality issues early. `continueOnError: true` ensures the build continues even if linting finds issues, allowing teams to fix them incrementally without blocking deployments.

### Test Execution and Coverage

**OLD:** Tests were commented out.

**NEW (Line 174-195):**
```yaml
      - bash: | 
          npm run test -- --coverage --reporters=default --reporters=jest-junit --coverageReporters=lcov --coverageReporters=cobertura --coverageReporters=text
        displayName: 'Run Test: Prepare Jest Coverage Report'
        workingDirectory: "${{ parameters.BUILD_REPOSITORY_LOCALPATH }}"
        env:
          JEST_JUNIT_OUTPUT_DIR: "./test-results/"
          JEST_JUNIT_OUTPUT_NAME: "junit.xml"
        continueOnError: true
      - bash: |
          # Inspect folder files and sizes
          echo "Inspecting folder files and sizes in $(System.DefaultWorkingDirectory)"
          du -h -a -d 1 $(System.DefaultWorkingDirectory)
        displayName: "Inspect Folder Files and Sizes"
      - bash: |
          echo "Checking lcov.info file..."
          ls -lh coverage/
          head -20 coverage/lcov.info || echo "lcov.info not found!"
        displayName: 'Debug: Validate Coverage Report Exists'

      - bash: npm run test
        displayName: 'Run Jest coverage report: Run test'
        continueOnError: true
```

**Why:** 
- Running tests with coverage generates reports needed for SonarQube analysis
- JUnit XML format allows Azure DevOps to display test results
- Cobertura format is used for code coverage visualization
- `continueOnError: true` prevents test failures from blocking the build (matches legacy behavior)

### Test Results Publishing

**NEW (Line 207-226):**
```yaml
      # Publish test results
      - task: PublishTestResults@2
        displayName: 'Publish Test Results'
        condition: always()
        inputs:
          testResultsFormat: 'JUnit'
          testResultsFiles: '**/test-results/junit.xml'
          mergeTestResults: true
          testRunTitle: 'Jest Tests'
          failTaskOnFailedTests: false

      # Publish code coverage results
      - task: PublishCodeCoverageResults@2
        displayName: 'Publish Code Coverage Results'
        condition: always()
        inputs:
          codeCoverageTool: 'Cobertura'
          summaryFileLocation: '$(Build.SourcesDirectory)/coverage/cobertura-coverage.xml'
          reportDirectory: '$(Build.SourcesDirectory)/coverage'
          failIfCoverageEmpty: false
```

**Why:** These tasks publish test results and coverage reports to Azure DevOps, making them visible in the build summary. This helps track test trends and coverage metrics over time.

### Veracode SCA Scan Condition

**OLD (Line 238):**
```yaml
        condition: and(succeeded(), eq(variables['VERACODE_SCA_SCAN_ENABLED'], 'true'))
```

**NEW (Line 271):**
```yaml
        condition: and(succeeded(), eq(variables['VERACODE_SCA_SCAN_ENABLED'], true ))
```

**Why:** Changed from string comparison `'true'` to boolean `true` for consistency. The condition logic remains the same.

### Veracode Scan Script Fix

**OLD (Line 287-289):**
```yaml
              if [ -f veracode-scan-scan-result-finding.json ]
              echo "veracode-sca-scan-result-finding.json file exists"
              then
```

**NEW (Line 309-310):**
```yaml
              if [ -f veracode-sca-scan-result-finding.json ]
              then
                echo "veracode-sca-scan-result-finding.json file exists"
```

**Why:** Fixed syntax error - the echo statement was placed between the `if` condition and `then`, which is invalid bash syntax. The echo should be inside the `then` block.

### Azure Key Vault Secrets

**OLD (Line 333):**
```yaml
          SecretsFilter: 'ACR-PASSWORD,ACR-USERNAME,NPM-TOKEN'
```

**NEW (Line 355):**
```yaml
          SecretsFilter: 'ACR-PASSWORD,ACR-USERNAME'
```

**Why:** NPM-TOKEN is now retrieved from the `NPM_TOKEN_GROUP` variable group instead of Key Vault. This aligns with the new approach where NPM authentication is handled through variable groups.

### .npmrc File Cleanup

**NEW (Line 380-384):**
```yaml
      - task: Bash@3
        displayName: 'Remove .npmrc file before Docker build to prevent credential exposure'
        inputs:
          targetType: 'inline'
          script: 'rm -f .npmrc'
```

**Why:** The `.npmrc` file downloaded from secure files may contain credentials. Removing it before the Docker build prevents it from being copied into the Docker build context, which could expose credentials in the image layers.

### Docker Build Replacement

**OLD (Line 358-397):**
```yaml
      - task: Docker@2
        condition: always()
        displayName: "Log in to the main container registry"
        inputs:
          command: login
          containerRegistry: ${{parameters.REGISTRY_SERVICE_CONNECTION}}

      - task: Docker@2
        displayName: "Build the image from Dockerfile"
        inputs:
          command: build
          repository: $(ACR_REGISTRY_NAME)
          Dockerfile: Dockerfile
          arguments: '--target=prod --build-arg REACT_APP_CLIENT_INTERFACE_URL=...'
          tags: |
            latest
            $(Build.BuildId)
          buildContext: $(System.DefaultWorkingDirectory)

      - task: Docker@2
        condition: and(succeeded(), eq(variables.IsPullRequest, 'false'))
        displayName: "Push the image to the main container registry"
        inputs:
          command: push
          repository: $(ACR_REGISTRY_NAME)
          tags: |
            latest
            $(Build.BuildId)
          buildContext: $(System.DefaultWorkingDirectory)

      - task: Docker@2
        condition: and(succeeded(), eq(variables.IsPullRequest, 'false'))
        displayName: "Log out of main container registry"
        inputs:
          command: logout
          containerRegistry: ${{parameters.REGISTRY_SERVICE_CONNECTION}}
```

**NEW (Line 389-398):**
```yaml
      - template: templates/Build/Docker/docker-image-template.yml@templates
        parameters:
          dockerFile: "Dockerfile"
          docker_registry_name: "acr14031e1dv01.azurecr.io"
          imageName: "$(ACR_REGISTRY_NAME)"
          tag: "$(Build.BuildId)"
          build_context: "$(Build.SourcesDirectory)"
          build_args: "--build-arg NPM_TOKEN=$(NPM_TOKEN) --build-arg REACT_APP_CLIENT_INTERFACE_URL=..."
          image_scan: true
          target_registry_connection: "SPAZDO14031NP01"
```

**Why:** 
- The template handles Docker login, build, push, and logout automatically
- It includes Prisma Cloud image scanning (`image_scan: true`) which is a governance requirement
- It handles JFrog authentication for base image pulls
- Standardizes the build process across all projects
- Reduces maintenance overhead - template updates apply to all projects automatically

### Publish Artifacts Condition

**OLD (Line 433):**
```yaml
        condition: and(succeeded(), or(eq(variables.IsPullRequest, 'false'), eq(variables['IsBranchesIncluded'], 'true')))
```

**NEW (Line 434):**
```yaml
        condition: and(succeeded(), and(eq(variables.IsPullRequest, 'false'), eq(variables['IsBranchesIncluded'], 'true')))
```

**Why:** Changed from `or` to `and` logic. Artifacts should only be published when it's NOT a pull request AND the branch is included in the allowed list. The old logic would publish artifacts for pull requests if the branch was included, which is incorrect.

---

## Summary

The migration primarily focused on:

1. **Compliance**: Moving to JFrog Artifactory for base images and NPM packages, adding required MetLife labels
2. **Security**: Implementing multi-stage builds, removing credentials from images, adding image scanning
3. **Standardization**: Using template-based builds for consistency across projects
4. **Quality**: Adding linting, test execution, and coverage reporting
5. **Efficiency**: Optimizing Docker layer caching and reducing final image size

These changes ensure the build process aligns with MetLife's container governance requirements while maintaining backward compatibility with existing functionality.

