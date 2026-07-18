# Google Cloud Run Deployment Setup Guide

This guide will help you set up the GitHub Actions workflow for deploying your Spring Boot + React application to Google Cloud Run.

## Prerequisites

1. Google Cloud Project with the following APIs enabled:
   - Cloud Run API
   - Artifact Registry API
   - Cloud Build API (optional, if using Cloud Build)

2. Google Cloud SDK installed locally for setup steps

## Google Cloud Setup Steps

### 1. Create Artifact Registry Repository

```bash
# Replace with your project details
gcloud artifacts repositories create YOUR_ARTIFACT_REGISTRY \
  --repository-format=docker \
  --location=YOUR_REGION \
  --description="Docker repository for Spring Boot application"
```

Example:
```bash
gcloud artifacts repositories create primefocus-workflow \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker repository for Spring Boot application"
```

### 2. Configure IAM Permissions

Grant the Cloud Run Service Account the necessary permissions:

```bash
# Get your project number
PROJECT_NUMBER=$(gcloud projects describe YOUR_PROJECT_ID --format='value(projectNumber)')

# Grant Cloud Run Service Account permissions to pull from Artifact Registry
gcloud artifacts repositories add-iam-policy-binding YOUR_ARTIFACT_REGISTRY \
  --location=YOUR_REGION \
  --member="serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

### 3. Create Service Account and Key

#### 3a. Create Service Account for Deployments

```bash
gcloud iam service-accounts create github-deploy-sa \
  --display-name="GitHub Deployment Service Account" \
  --project=YOUR_PROJECT_ID
```

#### 3b. Grant Permissions to Service Account

```bash
# Grant permissions to deploy to Cloud Run
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:github-deploy-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/run.developer"

# Grant permissions to push to Artifact Registry
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:github-deploy-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"
```

#### 3c. Create Service Account Key

```bash
gcloud iam service-accounts keys create github-deploy-key.json \
  --iam-account=github-deploy-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

**Important**: Keep this key file secure and never commit it to your repository!

## GitHub Secrets Configuration

Add the following secrets to your GitHub repository (Settings → Secrets and variables → Actions):

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `GCP_PROJECT_ID` | Your Google Cloud Project ID | `prime-focus-services` |
| `GCP_REGION` | Google Cloud region | `us-east5` |
| `GCP_SERVICE_NAME` | Cloud Run service name | `primefocus-workflow` |
| `GCP_ARTIFACT_REGISTRY` | Artifact Registry repository name | `docker` |
| `GCP_SA_KEY` | Service account key JSON content | Contents of `github-deploy-key.json` |

### Adding the Service Account Key

1. Open the `github-deploy-key.json` file you created earlier
2. Copy the entire contents of the file
3. In your GitHub repository, go to Settings → Secrets and variables → Actions
4. Click "New repository secret"
5. Name it `GCP_SA_KEY`
6. Paste the entire contents of the JSON file
7. Click "Add secret"



## React Frontend Setup

The Dockerfile expects the React frontend to be located at:
```
Prime_Vision_Focus_GiveCamp/PVF_React_Frontend/
```

Make sure this directory structure exists before running the workflow, or modify the Dockerfile to match your actual frontend location.

## Usage

1. Go to the Actions tab in your GitHub repository
2. Select "Build and Deploy to Google Cloud Run" workflow
3. Click "Run workflow"
4. Select the environment (dev/staging/prod)
5. Optionally specify a custom image tag (defaults to git SHA)
6. Click "Run workflow"

## Troubleshooting

### Permission Denied Errors
- Verify the service account has the correct IAM roles
- Check that the Workload Identity Federation is properly configured
- Ensure the GitHub repository matches the one configured in the principalSet

### Build Failures
- Ensure the React frontend directory exists and contains package.json
- Check that all dependencies are properly defined in package.json
- Verify the Gradle build works locally: `./gradlew bootJar`

### Deployment Failures
- Verify Cloud Run API is enabled
- Check that the service name doesn't already exist (or update existing service)
- Ensure the region is correct and supported by Cloud Run
