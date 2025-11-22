## Overview

Went through both Dockerfiles and pipeline files to document what changed and why. Main thing is we're moving from Azure ACR to JFrog Artifactory for base images and npm packages. Also cleaned up a lot of security issues and optimized the build process.

Files compared:
- `api/API/OLD-DEVELOP&MASTER/Dockerfile` vs `api/API/NEW FEATURE BRANCH CHNAGES/Dockerfile`
- `api/API/OLD-DEVELOP&MASTER/14031_digital-soh-api.yml` vs `api/API/NEW FEATURE BRANCH CHNAGES/14031_digital-soh-api.yml`

---

## Dockerfile Changes

Key things that changed:
- Base image now pulls from JFrog instead of ACR
- Node upgraded to v22 (was on alpine3.18)
- No more manual cert handling - base image has it
- Better secrets management (no .npmrc in final image)
- Production stage is way smaller now
- Added MetLife compliance labels

### Change 1: Base Image (Lines 1-3)

OLD:
```
FROM acr14031e1dv01.azurecr.io/infra/node:alpine3.18 AS base
USER root
```

NEW:
```
FROM jfrog-artifactory.metlife.com/docker-metlife-base-images-virtual/library/node:22-alpine AS base
```

What changed:
- Switched from ACR to JFrog Artifactory (company standard now)
- Node version bump: alpine3.18 → node:22-alpine
- Removed the `USER root` line - not needed right away since base image already has proper permissions

Why:
- JFrog is the new standard across the company for artifacts
- Node 22 is the latest LTS with security patches
- Base images from JFrog already have the right configs

---

### Change 2: MetLife Labels (Lines 4-15 in NEW)

NEW file has these labels:
```
LABEL com.metlife.image.app.description="Digital SOH API - Node.js application"
LABEL com.metlife.image.app.maintainer="devops@metlife.com"
LABEL com.metlife.image.app.dockerfile="Dockerfile"
LABEL com.metlife.image.app.dpccode="14031"
LABEL com.metlife.image.app.eaicode="digital-soh-api"
LABEL com.metlife.image.app.snow-group="Digital-SOH"
```

OLD file didn't have these.

Why added:
- Governance requirement from security team
- Helps with CMDB tracking and ServiceNow integration
- Makes it easier to identify who owns what in the registry

---

### Change 3: Certificate Handling (Lines 6-15 in OLD - REMOVED)

OLD had:
```
COPY ca-certificates /usr/local/share/ca-certificates/
RUN apk add ca-certificates && update-ca-certificates
ENV NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt
```

NEW: Completely removed

Why:
- JFrog base images already have MetLife certs baked in
- No need to manually copy and install certs anymore
- Reduces image size and one less thing to maintain
- Also cleaned up commented out lines for gcc, g++, python3 etc that were never used

---

### Change 4: DEV Stage (Lines 17-42)

OLD:
```
FROM base AS dev
USER root
RUN node --version
RUN npm --version

ENTRYPOINT ["/app_cache/entrypoint.sh"]
CMD npm run start:dev
WORKDIR /app_cache
COPY --chown=node:node entrypoint.sh package*.json ./
RUN chmod +x ./entrypoint.sh

RUN npm install -g npm@10.8.1
RUN npm install
WORKDIR /app
```

NEW:
```
FROM base AS dev
USER node
RUN node --version && npm --version

ENTRYPOINT ["/app_cache/entrypoint.sh"]
CMD ["npm", "run", "start:dev"]
WORKDIR /app_cache
COPY --chown=node:node entrypoint.sh package*.json ./
RUN chmod +x ./entrypoint.sh

ARG NPM_TOKEN
RUN echo "registry=https://jfrog-artifactory.metlife.com/artifactory/api/npm/remote-npm/" > .npmrc && \
    if [ -n "$NPM_TOKEN" ]; then \
        echo "//jfrog-artifactory.metlife.com/artifactory/api/npm/remote-npm/:_authToken=$NPM_TOKEN" >> .npmrc; \
    fi && \
    npm install --include=dev && \
    rm -f .npmrc
WORKDIR /app
```

What changed:
- Changed from `USER root` to `USER node` - better security, don't need root
- Combined version checks into one line with `&&` - reduces layers
- CMD now uses array syntax `["npm", "run", "start:dev"]` instead of string - better signal handling
- Removed npm upgrade to 10.8.1 - Node 22 already has good npm version
- **BIG ONE**: Added JFrog npm authentication

The JFrog authentication part is important:
- Creates .npmrc file dynamically during build
- Takes NPM_TOKEN as build arg (passed from pipeline)
- Points npm to JFrog registry instead of public npmjs.org
- Deletes .npmrc after install so token doesn't leak into image
- If NPM_TOKEN is empty, still works (for local dev maybe)

---

### Change 5: PROD-BUILD Stage (Lines 44-71)

OLD:
```
FROM base as prod-build

RUN mkdir -p /app && chown node.node /app
WORKDIR /app
USER root 

RUN node --version
RUN npm --version

USER node
COPY --chown=node:node .npmrc package*.json ./
RUN npm install
COPY --chown=node:node . ./

RUN npm run build
```

NEW:
```
FROM base as prod-build

RUN mkdir -p /app && chown node:node /app
WORKDIR /app
USER node

RUN node --version && npm --version

COPY --chown=node:node package*.json ./
ARG NPM_TOKEN
RUN echo "registry=https://jfrog-artifactory.metlife.com/artifactory/api/npm/remote-npm/" > .npmrc && \
    if [ -n "$NPM_TOKEN" ]; then \
        echo "//jfrog-artifactory.metlife.com/artifactory/api/npm/remote-npm/:_authToken=$NPM_TOKEN" >> .npmrc; \
    fi && \
    NODE_ENV=development npm install && \
    rm -f .npmrc

COPY --chown=node:node . ./
RUN rm -f .npmrc
```

Changes:
- Fixed typo: `chown node.node` → `chown node:node` (was using dot instead of colon)
- Removed the `USER root` line - don't need to switch back to root
- Version checks combined into one line again
- No longer copying .npmrc from source - was a security risk
- Same JFrog authentication setup as dev stage
- Uses `NODE_ENV=development` for npm install to get devDependencies (needed for build)
- Extra `rm -f .npmrc` after copying source code - just in case someone has .npmrc in repo
- Removed explicit `npm run build` command (might be in package.json scripts now?)

---

### Change 6: PROD Stage (Lines 73-103) - **BIGGEST CHANGE**

OLD:
```
FROM prod-build AS prod
WORKDIR /app
USER node
ENTRYPOINT ["./entrypoint_prod.sh"]
CMD npm run start

RUN chmod +x ./entrypoint_prod.sh

ARG GIT_COMMIT=latest
ENV GIT_COMMIT $GIT_COMMIT
```

NEW:
```
FROM base AS prod

COPY --chown=node:node package*.json ./
ARG NPM_TOKEN
RUN echo "registry=https://jfrog-artifactory.metlife.com/artifactory/api/npm/remote-npm/" > .npmrc && \
    if [ -n "$NPM_TOKEN" ]; then \
        echo "//jfrog-artifactory.metlife.com/artifactory/api/npm/remote-npm/:_authToken=$NPM_TOKEN" >> .npmrc; \
    fi && \
    npm install --omit=dev && \
    rm -f .npmrc

COPY --chown=node:node --from=prod-build /app/dist ./dist
COPY --chown=node:node --from=prod-build /app/migrations ./migrations
COPY --chown=node:node --from=prod-build /app/migrations_utils ./migrations_utils

COPY --chown=node:node entrypoint_prod.sh ./
RUN chmod +x ./entrypoint_prod.sh

WORKDIR /app
USER node

ARG GIT_COMMIT=latest
ENV GIT_COMMIT $GIT_COMMIT

ENTRYPOINT ["./entrypoint_prod.sh"]
CMD ["npm", "run", "serve"]
```

This is a BIG optimization!

What changed:
- **FROM base instead of FROM prod-build** - this is huge!
  - Old way: prod image contained everything from build (source code, build tools, devDependencies)
  - New way: prod image starts fresh from base, only copies what's needed
  - Result: Image size WAY smaller, more secure

- **npm install --omit=dev** - only production dependencies
  - No jest, no typescript compiler, no dev tools
  - Saves tons of space
  - Less attack surface

- **Selective copying from build stage**
  - Only copies: dist/, migrations/, migrations_utils/
  - Source code stays in build stage, never makes it to prod image
  - If someone breaks into container, they can't see our source

- **CMD changed to "serve"**
  - OLD: `npm run start` (might rebuild stuff)
  - NEW: `npm run serve` (just runs compiled code)
  - Uses array format for better process handling

Why this matters:
- Production image probably 50-60% smaller
- No source code in prod = better security
- Faster deployments (smaller image to push/pull)
- Cleaner separation of concerns

---

## Pipeline YAML Changes (14031_digital-soh-api.yml)

Main changes in the pipeline:
- Using template for Docker builds instead of manual podman commands
- Added Prisma Cloud scanning
- JFrog integration for npm packages
- Better npm install with retry logic
- Java 17 setup for SonarQube
- Removed coverage from tests (runs faster)
- Clean up .npmrc before Docker build

---

### Change 1: Template Repository (Lines 13-17)

NEW only:
```yaml
## CG Requirement
  - repository: templates
    name: DevOps-Patterns-Practices-Examples/Pipeline-Foundation
    type: git
```

Why:
- References the shared templates repo
- Needed for the docker-image-template.yml we use later
- Corporate governance requirement

---

### Change 2: Variable Groups (Lines 24-30)

OLD:
```yaml
variables:
  - group: Veracode
  - group: ARTIFACTORY_CREDENTIALS
  - group: SONAR
```

NEW added these:
```yaml
  - group: JFROG_BASE_IMAGE_PULL
  - group: PRISMA_CLOUD
  - group: NPM_TOKEN_GROUP
```

What these do:
- **JFROG_BASE_IMAGE_PULL**: Creds to pull base images from JFrog
- **PRISMA_CLOUD**: For Twistlock/Prisma Cloud security scanning
- **NPM_TOKEN_GROUP**: Has the NPM_TOKEN we pass to Docker build

All secrets stored in Azure DevOps variable groups (encrypted).

---

### Change 3: pushImage Variable (Lines 40-43)

NEW:
```yaml
- name: pushImage
  value: ${{ ne(variables['Build.Reason'], 'PullRequest') }}
```

What it does:
- Skips pushing images during PR builds
- Saves registry space and build time
- Template needs this variable

---

### Change 4: npm Install (Lines 143-148)

OLD:
```bash
npm install
```

NEW:
```bash
npm install --no-optional --legacy-peer-deps || npm install --no-optional --legacy-peer-deps
```

Changes:
- Added `--no-optional` - skips optional deps, faster builds
- Added `--legacy-peer-deps` - handles peer dependency issues in npm 7+
- Added retry with `||` - if first install fails, tries again
- Helps with flaky network issues

---

### Change 5: Tests and Java Setup (Lines 158-182)

OLD:
```bash
npm run test -- --coverage --coverageReporters="lcov"
```

NEW:
```bash
npm run test
```

Removed the coverage flags - runs faster, coverage might be handled elsewhere.

**NEW: Java 17 Installation** (Lines 164-182)

Added these steps:
```bash
# Download JDK
mkdir builds
curl -SkL "https://download.oracle.com/java/$(JAVA_VERSION)/archive/$(JAVA_JDK_FILE)" --output builds/$(JAVA_JDK_FILE)

# Install it
- task: JavaToolInstaller@0
  inputs:
    versionSpec: '17'
    jdkSourceOption: LocalDirectory
    jdkFile: "$(Build.SourcesDirectory)/builds/$(JAVA_JDK_FILE)"
```

Why:
- SonarQube needs Java to run
- Java 17 is LTS and works with latest SonarQube
- Downloads and caches it locally

---

### Change 6: Docker Build (Lines 328-347) - **MAJOR CHANGE**

OLD (lots of manual commands):
```yaml
# Get ACR creds from key vault
- task: AzureKeyVault@1
  inputs:
    KeyVaultName: 'kyv14031e1dv01' 
    SecretsFilter: 'ACR-PASSWORD,ACR-USERNAME'

# Login
- task: CmdLine@2
  displayName: 'Login to ACR'
  script: |
    podman login $(ACR_SERVER_NAME) --username $(ACR-USERNAME) --password $(ACR-PASSWORD)

# Build
- task: CmdLine@2
  displayName: 'Build Podman Image'
  script: |
    podman build -t $(ACR_SERVER_NAME)/$(ACR_REGISTRY_NAME):$(Build.BuildId) .

# Push
- task: CmdLine@2
  displayName: 'Push Podman Image'
  script: |
    podman image push $(ACR_SERVER_NAME)/$(ACR_REGISTRY_NAME):$(Build.BuildId)

# Logout
- task: CmdLine@2
  displayName: 'Logout of ACR'
  script: |
    podman logout $(ACR_SERVER_NAME)
```

NEW (uses template):
```yaml
# Clean up .npmrc first
- task: Bash@3
  displayName: 'Remove .npmrc file before Docker build'
  inputs:
    script: 'rm -f .npmrc'

# Use template for everything
- template: templates/Build/Docker/docker-image-template.yml@templates
  parameters:
    dockerFile: "Dockerfile"
    docker_registry_name: "acr14031e1dv01.azurecr.io"
    imageName: "us-gssp-digital-soh-api"
    tag: "$(Build.BuildId)"
    build_context: "$(Build.SourcesDirectory)"
    build_args: "--build-arg NPM_TOKEN=$(NPM_TOKEN) --build-arg branch=$(Build.SourceBranchName) --build-arg buildNumber=$(Build.BuildNumber) --build-arg commit=$(Build.SourceVersion)"
    image_scan: true
    target_registry_connection: "SPAZDO14031NP01"
```

This is huge!

Why the template approach is better:
- **Standardization**: Everyone uses same tested template
- **Prisma Cloud scanning**: Template handles it automatically (`image_scan: true`)
- **Better error handling**: Template has retry logic
- **Cleaner pipeline**: All the login/build/push/scan logic is in template
- **Easier maintenance**: Template updates apply to everyone

The .npmrc removal:
- Extra security step to make sure no creds get into image
- Happens right before Docker build

Build args:
- `NPM_TOKEN=$(NPM_TOKEN)` - passes the token to Dockerfile (for JFrog auth)
- `branch`, `buildNumber`, `commit` - for tracking/debugging later

---

## Summary - What We Gained

### Security
- No more hardcoded credentials anywhere
- .npmrc files get created and deleted during build - never in final image
- Containers run as `node` user, not root
- Prisma Cloud scans every image before push
- Production images don't have source code or dev tools
- JFrog base images are pre-approved and maintained

### Performance
- Production images are MUCH smaller (no source, no devDependencies, no build tools)
- Better Docker layer caching
- Only prod dependencies in final image (`npm install --omit=dev`)
- npm install has retry logic for flaky networks

### Easier to Maintain
- Using standard templates instead of custom scripts
- MetLife compliance labels on all images
- Git commit/branch info baked into images for traceability
- Cleaned up commented code

### Compliance
- Meets Corporate Governance requirements
- Labels enable ServiceNow/CMDB integration
- Multiple security scans (Veracode, SonarQube, Prisma Cloud)
- Follows DevOps best practices

---

## Migration Checklist

### Before You Start
- [ ] Get access to JFrog Artifactory (jfrog-artifactory.metlife.com)
- [ ] Make sure NPM_TOKEN_GROUP variable group is set up in Azure DevOps
- [ ] Check JFROG_BASE_IMAGE_PULL creds are configured
- [ ] Check PRISMA_CLOUD variable group exists
- [ ] Get access to Pipeline-Foundation repo

### Dockerfile Changes
- [ ] Update FROM line to use JFrog base image (node:22-alpine)
- [ ] Add the MetLife labels at the top
- [ ] Remove certificate copy/install lines
- [ ] Update dev stage - add JFrog npm auth with NPM_TOKEN
- [ ] Update prod-build stage - add JFrog npm auth
- [ ] Rewrite prod stage - FROM base instead of FROM prod-build
- [ ] Change CMD to array format: ["npm", "run", "serve"]
- [ ] Test locally: `docker build --build-arg NPM_TOKEN=<your-token> -t test:latest .`

### Pipeline Changes
- [ ] Add templates repo reference at top
- [ ] Add new variable groups: JFROG_BASE_IMAGE_PULL, PRISMA_CLOUD, NPM_TOKEN_GROUP
- [ ] Add pushImage variable
- [ ] Update npm install line: `npm install --no-optional --legacy-peer-deps || npm install --no-optional --legacy-peer-deps`
- [ ] Add Java 17 download and install steps (for SonarQube)
- [ ] Simplify test command (remove coverage flags)
- [ ] Add `rm -f .npmrc` step before Docker build
- [ ] Replace all podman commands with docker-image-template.yml call
- [ ] Make sure build_args includes NPM_TOKEN
- [ ] Set image_scan: true

### Testing
- [ ] Create PR and run pipeline
- [ ] Check that npm install works (should use JFrog registry)
- [ ] Check that Docker build succeeds
- [ ] Compare image sizes (prod should be smaller)
- [ ] Check Prisma Cloud scan passes
- [ ] Check SonarQube runs
- [ ] Deploy and test container works
- [ ] Verify no .npmrc in image: `docker run <image> cat .npmrc` (should fail)

---

## Common Issues

### npm install fails with 401 Unauthorized
Check:
- Is NPM_TOKEN set in NPM_TOKEN_GROUP variable group?
- Does the token have access to jfrog-artifactory.metlife.com?
- Is the token expired? (They expire periodically)

### Can't pull base image from JFrog
Check:
- Are JFROG_BASE_IMAGE_PULL creds correct?
- Can you reach jfrog-artifactory.metlife.com from build agent?
- Is the base image path correct? Should be: `jfrog-artifactory.metlife.com/docker-metlife-base-images-virtual/library/node:22-alpine`

### Prisma Cloud scan fails
Check:
- Is PRISMA_CLOUD variable group configured?
- Is `image_scan: true` set in template call?
- Look at Prisma policies - might need exclusion for false positive

### Container won't start in prod
Check:
- Does `npm run serve` exist in package.json?
- Is dist/ folder getting created during build?
- Are migrations/ and migrations_utils/ copied to prod stage?
- Check entrypoint_prod.sh script - is it executable?

### Image size is still big
Check:
- Is prod stage using `FROM base` or `FROM prod-build`? (should be base)
- Is npm install using `--omit=dev` in prod stage?
- Is .npmrc getting deleted after npm install?
- Run `docker history <image>` to see which layers are big

---

## References

### Useful Links
- JFrog Artifactory: https://jfrog-artifactory.metlife.com
- Pipeline Templates: https://dev.azure.com/DevOps-Patterns-Practices-Examples/Pipeline-Foundation
- Docker multi-stage builds: https://docs.docker.com/develop/develop-images/multistage-build/

### Variable Groups (Azure DevOps)
- **Veracode** - Security scanning creds
- **ARTIFACTORY_CREDENTIALS** - Legacy (might not need anymore)
- **SONAR** - SonarQube connection
- **JFROG_BASE_IMAGE_PULL** - For pulling base images from JFrog
- **PRISMA_CLOUD** - Twistlock/Prisma scanning creds
- **NPM_TOKEN_GROUP** - Has the NPM_TOKEN for npm auth



