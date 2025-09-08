# OiPub Actions

## GitHub Actions Workflows

This repository uses several reusable GitHub Actions workflows for CI/CD automation. Below is a summary of each workflow and its purpose:

### 1. Dotnet Version (`dotnet-version.job.yml`)

- **Purpose:** Updates the version in `AppVersion.cs` for .NET projects.
- **Inputs:** `version` (string, required)
- **Main Steps:**
  - Updates the version string in the source file.
  - Commits and pushes the change.

### 2. Git Version (`git-version.job.yml`)

- **Purpose:** Generates a semantic version based on git history.
- **Inputs:** `baseBranch` (string, default: `main`)
- **Outputs:** `version` (semantic version)
- **Main Steps:**
  - Uses `codacy/git-version` to determine the next version.

### 3. NPM Version (`npm-version.job.yml`)

- **Purpose:** Bumps the version in `package.json` for Node.js projects.
- **Inputs:** `version` (string, required)
- **Main Steps:**
  - Updates the version in `package.json`.
  - Commits and pushes the change.

### 4. Git Tag (`git-tag.job.yml`)

- **Purpose:** Creates and pushes a git tag for releases.
- **Inputs:** `version` (string, required)
- **Main Steps:**
  - Checks if the tag exists.
  - Creates and pushes the tag if it does not.

### 5. Docker Image Build (`do-docker-image.job.yml`)

- **Purpose:** Builds and pushes Docker images to DigitalOcean Container Registry.
- **Inputs:** `imageRegistry`, `imageName`, `imageTag` (all required)
- **Secrets:** `DO_PAT` (DigitalOcean Personal Access Token)
- **Main Steps:**
  - Builds and pushes the Docker image.

### 6. Deployment (`deploy.job.yml`)

- **Purpose:** Updates deployment repositories with new Docker image tags.
- **Inputs:** `deploymentRepo`, `serviceName`, `imageRegistry`, `imageName`, `imageTag` (all required)
- **Secrets:** `GITHUB_PAT` (GitHub Personal Access Token)
- **Main Steps:**
  - Clones the deployment repository.
  - Updates the Docker Compose file with the new image tag.

---

**Usage:**  
These workflows are designed to be called from other workflows using `workflow_call` and require specific inputs and secrets as described above.
