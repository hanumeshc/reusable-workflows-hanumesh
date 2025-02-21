# Reusable GitHub Actions Workflow for Integrating with Queue Services

## Overview

This document describes how to set up and use a reusable GitHub Actions workflow for building and pushing Docker images, and then sending build data to RabbitMQ. This workflow can be placed in a separate repository or within an existing repository.

## Setting Up

### 1. Repository Setup
You can either:
- Create a separate repository for the reusable workflow.
- Use an existing repository.

### 2. Secrets Management
Secrets like Docker registry login credentials can be passed as parameters or registered under organization secrets. Ensure the following secrets are set up in the appropriate GitHub repositories:

#### Reusable Workflow Repository
Set up the following secrets in the repository containing the reusable workflow:
- `RABBITMQ_USER`: RabbitMQ user
- `RABBITMQ_PASS`: RabbitMQ password

#### Repository Using the Reusable Workflow
Set up the following secrets in the repository that uses the reusable workflow:
- `DOCKER_USERNAME`: Docker Hub username
- `DOCKER_PASSWORD`: Docker Hub password

### 3. Enable Access
In your repository settings:
- Navigate to `Settings -> Actions -> General`
- Enable "Accessible from repositories owned by the user 'org'".

## Workflow Description

### Reusable Workflow: `ssd-github-actions-integration.yaml`

<details>
  <summary>Click to expand/collapse</summary>

```yaml
name: Docker Image CI

on:
  workflow_call:
    inputs:
      image_tag:
        description: 'Image tag'
        required: true
        type: string
      registry:
        description: 'Docker registry'
        required: true
        type: string
      repository:
        description: 'Docker repository'
        required: true
        type: string
      rabbitmq_url:
        description: 'RabbitMQ URL'
        required: true
        type: string
        default: 'http://34.16.91.57:15672'
      rabbitmq_queue:
        description: 'RabbitMQ queue'
        required: true
        type: string
        default: 'githubactions-ssd'
      rabbitmq_exchange:
        description: 'RabbitMQ exchange'
        required: true
        type: string
        default: 'githubactions.events'
      rabbitmq_binding_key:
        description: 'RabbitMQ binding key'
        required: true
        type: string
        default: 'githubactions-ssd'
      org:
        description: 'Organization name'
        required: true
        type: string
      image_sha:
        description: 'Image SHA'
        required: true
        type: string
      visibility:
        description: 'Repository visibility'
        required: true
        type: string
      parent_repo:
        description: 'Parent repository'
        required: true
        type: string
      license_name:
        description: 'License name'
        required: true
        type: string
      caller_workflow_name:
        description: 'Name of the calling workflow'
        required: true
        type: string

jobs:
  rabbitmq_setup:
    runs-on: ubuntu-latest

    steps:
      - name: Create RabbitMQ Exchange and Queue, and Bind them
        run: |
          # Create the exchange
          curl -u ${{ secrets.RABBITMQ_USER }}:${{ secrets.RABBITMQ_PASS }} \
               -H "Content-Type: application/json" \
               -X PUT "${{ inputs.rabbitmq_url }}/api/exchanges/%2F/${{ inputs.rabbitmq_exchange }}" \
               -d '{"type":"direct","durable":true}'

          # Create the queue
          curl -u ${{ secrets.RABBITMQ_USER }}:${{ secrets.RABBITMQ_PASS }} \
               -H "Content-Type: application/json" \
               -X PUT "${{ inputs.rabbitmq_url }}/api/queues/%2F/${{ inputs.rabbitmq_queue }}" \
               -d '{"durable":true}'

          # Bind the queue to the exchange with the binding key
          curl -u ${{ secrets.RABBITMQ_USER }}:${{ secrets.RABBITMQ_PASS }} \
               -H "Content-Type: application/json" \
               -X POST "${{ inputs.rabbitmq_url }}/api/bindings/%2F/e/${{ inputs.rabbitmq_exchange }}/q/${{ inputs.rabbitmq_queue }}" \
               -d '{"routing_key":"${{ inputs.rabbitmq_binding_key }}"}'

      - name: Prepare and Send RabbitMQ Message
        run: |
          echo "Sending message to RabbitMQ"
          
          MESSAGE=$(jq -n --arg image "${{ inputs.registry }}/${{ inputs.repository }}:${{ inputs.image_tag }}" \
                          --arg imgsha "${{ inputs.image_sha }}" \
                          --arg jobId "${{ github.job }}" \
                          --arg buildNumber "${{ github.run_number }}" \
                          --arg gitUrl "${{ github.event.repository.html_url }}" \
                          --arg gitCommit "${{ github.sha }}" \
                          --arg gitBranch "${{ github.ref_name }}" \
                          --arg jobUrl "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}" \
                          --arg buildTime "${{ github.event.repository.updated_at }}" \
                          --arg buildUser "${{ github.actor }}" \
                          --arg visibility "${{ inputs.visibility }}" \
                          --arg parentRepo "${{ inputs.parent_repo }}" \
                          --arg licenseName "${{ inputs.license_name }}" \
                          --arg organization "${{ inputs.org }}" \
                          --arg imageTag "${{ inputs.image_tag }}" \
                          --arg workflowName "${{ inputs.caller_workflow_name }}" \
                          '{image: $image, imageTag: $imageTag, imgsha: $imgsha, jobId: $jobId, buildNumber: $buildNumber, gitUrl: $gitUrl, gitCommit: $gitCommit, gitBranch: $gitBranch, jobUrl: $jobUrl, buildTime: $buildTime, buildUser: $buildUser, visibility: $visibility, parentRepo: $parentRepo, licenseName: $licenseName, organization: $organization, workflowName: $workflowName}')
          
          PAYLOAD=$(jq -n --arg vhost "/" \
                          --arg name "${{ inputs.rabbitmq_exchange }}" \
                          --argjson properties '{}' \
                          --arg routing_key "${{ inputs.rabbitmq_binding_key }}" \
                          --arg delivery_mode "2" \
                          --arg payload "$MESSAGE" \
                          --arg payload_encoding "string" \
                          '{vhost: $vhost, name: $name, properties: $properties, routing_key: $routing_key, delivery_mode: $delivery_mode, payload: $payload, payload_encoding: $payload_encoding}')
          
          curl -u ${{ secrets.RABBITMQ_USER }}:${{ secrets.RABBITMQ_PASS }} \
              -H "Content-Type: application/json" \
              -X POST "${{ inputs.rabbitmq_url }}/api/exchanges/%2F/${{ inputs.rabbitmq_exchange }}/publish" \
              --data-binary "$PAYLOAD" || { echo "Failed to send message"; exit 1; }
```       
</details>


## Explanation for Each Section

### on.workflow_call:
This section defines the inputs and secrets that the reusable workflow will accept. These are the parameters and sensitive information that the workflow needs to execute.

### jobs.rabbitmq_setup:
This section defines the job to be executed. It specifies the runner environment (`ubuntu-latest`) and contains all the steps needed to create the RabbitMQ exchange and queue, bind them, and send the build data to RabbitMQ.

#### Create RabbitMQ Exchange and Queue, and Bind them:
Uses RabbitMQ API to create an exchange and a queue, and binds the queue to the exchange with the specified binding key.

#### Prepare and Send RabbitMQ Message:
Prepares a JSON message containing build details and sends it to the RabbitMQ exchange. This step includes gathering all relevant build data and sending the message using the RabbitMQ API.

## Usage Instructions

### Calling the Reusable Workflow

If you already have a workflow that builds Docker images and you want to use the reusable workflow to set up RabbitMQ and send build data, you need to add the following section under your jobs. Note that the `image_sha`, `visibility`, `parent_repo`, and `license_name` should be generated in your existing workflow.

Here’s an example:

```yaml
name: Use Reusable Workflow

on:
  push:
    branches:
      - main

jobs:
  build_and_push:
    runs-on: ubuntu-latest

    steps:
      - name: Check out the repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Log in to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v2
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ inputs.registry }}/${{ inputs.repository }}:${{ inputs.image_tag }}

      - name: Pull the image to get the digest
        run: docker pull ${{ inputs.registry }}/${{ inputs.repository }}:${{ inputs.image_tag }}

      - name: Get the image digest
        run: echo "IMAGE_SHA=$(docker inspect --format='{{index .RepoDigests 0}}' ${{ inputs.registry }}/${{ inputs.repository }}:${{ inputs.image_tag }} | cut -d':' -f2)" >> $GITHUB_ENV

      - name: Get Repository Metadata
        run: |
          echo "Fetching repository metadata"
          VISIBILITY=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" https://api.github.com/repos/${{ github.repository }} | jq -r '.visibility')
          PARENT_REPO=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" https://api.github.com/repos/${{ github.repository }} | jq -r '.parent.full_name // empty')
          LICENSE_NAME=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" https://api.github.com/repos/${{ github.repository }} | jq -r '.license.spdx_id // empty')
          echo "VISIBILITY=$VISIBILITY" >> $GITHUB_ENV
          echo "PARENT_REPO=$PARENT_REPO" >> $GITHUB_ENV
          echo "LICENSE_NAME=$LICENSE_NAME" >> $GITHUB_ENV

  integration_setup:
    needs: build_and_push
    uses: your-org/reusable-workflows/.github/workflows/ssd-github-actions-integration.yaml@main
    with:
      image_tag: ${{ github.run_id }}
      registry: 'docker.io'
      repository: 'your-repo/your-image'
      org: 'your-org'
      image_sha: ${{ env.IMAGE_SHA }}
      visibility: ${{ env.VISIBILITY }}
      parent_repo: ${{ env.PARENT_REPO }}
      license_name: ${{ env.LICENSE_NAME }}
      caller_workflow_name: ${{ github.workflow }}
```

## Explanation for Each Section

### on.push, on.pull_request:
These sections define the events that trigger the workflow. The workflow will run on pushes and pull requests to the `main` branch.

### jobs.build_and_push:
This section defines the job to be executed. It specifies the runner environment (`ubuntu-latest`) and contains all the steps needed to build and push the Docker image.

#### actions/checkout@v4:
This step checks out the repository to the GitHub Actions runner. The `fetch-depth: 2` option ensures enough history is fetched to allow diffing with the previous commit.

#### docker/setup-buildx-action@v1:
Sets up Docker Buildx, a CLI plugin that extends the docker command with the full support of the features provided by Moby BuildKit builder toolkit.

#### docker/login-action@v1:
Logs into Docker Hub using the provided credentials (`docker_username` and `docker_password`).

#### docker/build-push-action@v2:
Builds and pushes the Docker image to the specified repository. The `context` and `file` parameters specify the Docker build context and the Dockerfile to use, respectively. The `push` parameter indicates that the built image should be pushed to the Docker registry, and `tags` specifies the tag to be applied to the image.

#### Pull the image to get the digest:
This step pulls the built image to ensure it is available and retrieves its digest. The digest is a unique identifier for the image.

#### Get the image digest:
Retrieves the digest of the Docker image and stores it in an environment variable (`IMAGE_SHA`).

#### Get Repository Metadata:
Fetches repository metadata such as visibility, parent repository, and license name using GitHub's API and stores them in environment variables.

### jobs.integration_setup:
This section defines the job that calls the reusable workflow to set up RabbitMQ and send the message.

## GitHub Repository Secrets

Secrets are encrypted environment variables that you create in an organization, repository, or repository environment. These secrets are used to store sensitive information such as passwords, API keys, and tokens. In the context of GitHub Actions, secrets are essential for securely accessing external services, such as Docker Hub or RabbitMQ.

### Steps to Add Secrets to Your Repository

1. **Navigate to Your Repository:**
   - Go to the main page of your repository on GitHub.

2. **Access the Settings:**
   - Click on the `Settings` tab located at the top of the repository page.

3. **Find the Secrets Section:**
   - In the left sidebar, click on `Secrets and variables`, then `Actions`.

4. **Add a New Secret:**
   - Click on the `New repository secret` button.
   - Add each secret by providing a name and a value. For example:
     - `DOCKER_USERNAME`: Your Docker Hub username.
     - `DOCKER_PASSWORD`: Your Docker Hub password.

5. **Save the Secret:**
   - After entering the name and value, click `Add secret`.

### Organizational Secrets

If you have multiple repositories that need the same secrets, you can add them at the organization level:

1. **Navigate to Your Organization:**
   - Go to your organization's main page on GitHub.

2. **Access the Settings:**
   - Click on the `Settings` tab located at the top of the organization page.

3. **Find the Secrets Section:**
   - In the left sidebar, click on `Secrets and variables`, then `Actions`.

4. **Add a New Secret:**
   - Click on the `New organization secret` button.
   - Add each secret by providing a name and a value.

5. **Repository Access:**
   - You can specify which repositories have access to each secret.

## Applying the Workflow to an Entire Organization

If you want to apply the second workflow to all repositories in an organization, you can achieve this by creating a special repository named `.github`. This repository can contain organization-wide workflow templates and other configurations.

### Steps to Set Up

### 1. Create a Repository Named `.github`

- Go to your GitHub organization.
- Create a new repository and name it `.github`. This is a special repository name recognized by GitHub for organization-wide settings.

### 2. Add the Workflow to the `.github` Repository

- In the `.github` repository, create a directory named `workflow-templates`.
- Inside the `workflow-templates` directory, add the `use-dssd-github-actions-integration.yaml` file.

Your repository structure should look like this:

.github
└── workflow-templates
    └── use-dssd-github-actions-integration.yaml
