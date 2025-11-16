# 1. CI/CD PIPELINES
### 1.1 Azure DevOps YAML
#### Notes: Use Azure DevOps service connections for Docker registries (`dockerRegistryServiceConnection`), AKS (`kubernetesServiceConnection`), and for Terraform use a service principal or backend like Azure Storage. Secrets are referenced via pipeline variables or variable groups.
#### `A. Basic CI (generic)`
```
trigger:
  - main

pool:
  vmImage: ubuntu-latest

steps:
  - script: echo "Running basic CI checks"
    displayName: Basic Step

  - script: |
      echo "Linting"
      # e.g. npm run lint
    displayName: Lint
```
#### `B. Node.js pipeline (build, test, publish artifact)`
```
trigger:
  - main

pool:
  vmImage: ubuntu-latest

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '18.x'
    displayName: 'Use Node 18'

  - script: |
      npm ci
      npm run build
    displayName: 'Install & Build'

  - script: npm test
    displayName: 'Run Tests'

  - task: PublishBuildArtifacts@1
    inputs:
      PathtoPublish: 'dist'
      ArtifactName: 'drop'
    displayName: 'Publish Artifact'
```
#### `C. .NET pipeline (build & test)`
```
trigger:
  - main

pool:
  vmImage: 'windows-latest'

variables:
  buildConfiguration: 'Release'

steps:
  - task: UseDotNet@2
    inputs:
      packageType: 'sdk'
      version: '7.x'
  - task: NuGetToolInstaller@1

  - script: dotnet restore
    displayName: 'dotnet restore'

  - script: dotnet build --configuration $(buildConfiguration) --no-restore
    displayName: 'dotnet build'

  - script: dotnet test --no-build --verbosity normal
    displayName: 'dotnet test'

  - task: PublishBuildArtifacts@1
    inputs:
      PathtoPublish: '$(Build.SourcesDirectory)/bin/$(buildConfiguration)/net*/publish'
      ArtifactName: 'dotnet-drop'
```
#### `D. Docker build & push (to ACR or Docker Hub)`
```
trigger:
  - main

pool:
  vmImage: ubuntu-latest

variables:
  imageName: myapp
  tag: $(Build.BuildId)

steps:
  - task: Docker@2
    displayName: Build and push image
    inputs:
      command: buildAndPush
      repository: myregistry.azurecr.io/$(imageName)
      dockerfile: Dockerfile
      tags: |
        $(tag)
      containerRegistry: 'MyAcrServiceConnection' # service connection name
```
#### `E. Terraform plan & apply (using CLI task)`
```
trigger:
  - main

pool:
  vmImage: ubuntu-latest

variables:
  tf_version: '1.5.0'

steps:
  - task: UseTerraform@0
    inputs:
      terraformVersion: $(tf_version)

  - script: |
      terraform init -backend-config="storage_account_name=$(tf_state_sa)" -backend-config="container_name=$(tf_state_container)" -backend-config="key=terraform.tfstate"
    env:
      ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
      ARM_CLIENT_ID: $(ARM_CLIENT_ID)
      ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
      ARM_TENANT_ID: $(ARM_TENANT_ID)
    displayName: 'Terraform Init'

  - script: terraform plan -out=plan.tfplan
    displayName: 'Terraform Plan'

  - script: terraform apply -auto-approve plan.tfplan
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
    displayName: 'Terraform Apply (only main)'
```
#### `F. Kubernetes deployment using kubectl`
```
trigger:
  - main

pool:
  vmImage: ubuntu-latest

steps:
  - task: Kubernetes@1
    displayName: 'kubectl apply manifest'
    inputs:
      connectionType: 'Azure Resource Manager'
      azureSubscriptionEndpoint: 'MyAzureServiceConnection'
      azureResourceGroup: 'rg-aks'
      kubernetesCluster: 'aks-cluster-name'
      command: apply
      useConfigurationFile: true
      configuration: |
        k8s/deployment.yaml
        k8s/service.yaml
```
#### `G. Multi-stage pipeline (CI → CD → Prod)`
```
trigger:
  - main

stages:
  - stage: CI
    displayName: CI
    jobs:
      - job: Build
        pool: vmImage: 'ubuntu-latest'
        steps:
          - script: npm ci && npm run build
            displayName: 'Build App'
          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: 'dist'
              ArtifactName: 'drop'

  - stage: CD_Dev
    displayName: Deploy to Dev
    dependsOn: CI
    jobs:
      - deployment: DeployDev
        displayName: DeployToDev
        environment: 'dev'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                - task: Kubernetes@1
                  inputs:
                    connectionType: 'Azure Resource Manager'
                    azureSubscriptionEndpoint: 'MyAzureServiceConnection'
                    azureResourceGroup: 'rg-aks'
                    kubernetesCluster: 'aks-cluster'
                    command: apply
                    useConfigurationFile: true
                    configuration: k8s/deployment-dev.yaml

  - stage: CD_Prod
    displayName: Deploy to Prod
    dependsOn: CD_Dev
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployProd
        environment: 'prod'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                - task: Kubernetes@1
                  inputs:
                    connectionType: 'Azure Resource Manager'
                    azureSubscriptionEndpoint: 'MyAzureServiceConnection'
                    azureResourceGroup: 'rg-aks-prod'
                    kubernetesCluster: 'aks-cluster-prod'
                    command: apply
                    useConfigurationFile: true
                    configuration: k8s/deployment-prod.yaml
```
### 1.2 GITHUB ACTIONS
#### Notes: store secrets in GitHub repo secrets (e.g., `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `TF_VAR_*`, `KUBE_CONFIG_DATA`)
#### `A. Node.js build & test`
```
name: Node CI
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 18
      - name: Install deps
        run: npm ci
      - name: Run tests
        run: npm test
      - name: Build
        run: npm run build
```
#### `B. Docker publish (to Docker Hub)`
```
name: Docker Publish
on:
  push:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myorg/myapp:${{ github.sha }} .

      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Push image
        run: |
          docker tag myorg/myapp:${{ github.sha }} myorg/myapp:latest
          docker push myorg/myapp:${{ github.sha }}
          docker push myorg/myapp:latest
```
#### `C. Terraform workflow (plan + apply with manual approval)`
```
name: Terraform

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  terraform-plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      - name: Terraform Init
        run: terraform init
      - name: Terraform Plan
        run: terraform plan -out=tfplan
      - name: Upload plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: tfplan

  terraform-apply:
    runs-on: ubuntu-latest
    needs: terraform-plan
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: tfplan
      - uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.5.0
      - name: Terraform Apply
        # require manual approval by enabling workflow_dispatch or by using environments with required reviewers
        run: terraform apply -auto-approve tfplan
```
#### For safer production usage, protect the `main` branch and require approvals or use GitHub environments with reviewers for the `terraform-apply` job.
#### `D. Kubernetes deploy (kubectl)`
```
name: Deploy to K8s
on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'latest'

      - name: Configure kubeconfig
        run: |
          echo "${{ secrets.KUBE_CONFIG_DATA }}" | base64 --decode > kubeconfig
          export KUBECONFIG=$PWD/kubeconfig

      - name: kubectl apply
        run: kubectl apply -f k8s/deployment.yaml
```
### 1.3 JENKINS
#### Notes: Jenkins pipelines below assume Jenkins has required plugins (Git, Docker, Kubernetes, Pipeline). Use credentials stored in Jenkins (IDs referenced as e.g., `dockerhub-creds`, `kubeconfig`).
#### `A. Declarative pipeline (simple build & test)`
```
pipeline {
  agent any

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'npm ci'
        sh 'npm run build'
      }
    }

    stage('Test') {
      steps {
        sh 'npm test'
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'dist/**', fingerprint: true
    }
  }
}
```
#### `B. Multibranch pipeline (Jenkinsfile — supports PRs & branches)`
```
pipeline {
  agent any
  triggers { pollSCM('H/5 * * * *') }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Build') {
      when { branch 'main' }
      steps {
        sh 'npm ci'
        sh 'npm run build'
      }
    }

    stage('Test') {
      steps {
        sh 'npm test'
      }
    }
  }

  post {
    success { echo 'Build succeeded' }
    failure { echo 'Build failed' }
  }
}
```
#### `C. Docker build + push (using Jenkins credentials)`
```
pipeline {
  agent any

  environment {
    IMAGE = "myorg/myapp:${env.BUILD_ID}"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Image') {
      steps {
        sh "docker build -t ${IMAGE} ."
      }
    }

    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
          sh "docker push ${IMAGE}"
        }
      }
    }
  }
}
```
#### `D. Kubernetes deploy (kubectl apply using kubeconfig credential)`
```
pipeline {
  agent any
  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Deploy to K8s') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig-file', variable: 'KUBECONFIG_FILE')]) {
          sh 'export KUBECONFIG=$KUBECONFIG_FILE'
          sh 'kubectl apply -f k8s/deployment.yaml'
        }
      }
    }
  }
}
```
### FINAL NOTES & BEST PRACTICES
----------------
* **Secrets**: Never hardcode secrets. Use Azure DevOps variable groups, GitHub secrets, or Jenkins credentials.

* **Immutability**: Tag images with immutable tags (build id, SHA).

* **Branch protections**: Use protected branches and required approvals for production deploys.

* **Idempotence**: Terraform apply / kubectl apply should be idempotent — use `plan` stage before apply.

* **Observability**: Add pipeline steps to publish test reports, code coverage, and container security scans (Trivy/Snyk).

* **Rollback**: Implement automated rollback or blue/green / canary strategies for production deployments.

**If you want, I can:**

* Convert these into a **ready GitHub repo** with example files and README.

* Split Azure DevOps YAML into separate files per pipeline stage and push to a repo.

* Create **pipeline templates** (reusable templates) for Azure DevOps or GitHub Actions.
