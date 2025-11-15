# AZURE DEVOPS – YAML PIPELINE SAMPLES
### Build & Test (Node.js)
```
trigger:
  - main

pool:
  vmImage: ubuntu-latest

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '18.x'

  - script: |
      npm install
      npm run build
    displayName: Build Project

  - script: npm test
    displayName: Run Tests
```
### Publish Artifact
```
steps:
  - task: PublishBuildArtifacts@1
    inputs:
      PathtoPublish: 'dist'
      ArtifactName: 'drop'
```
### Multi-Stage Pipeline (CI + CD)
```
stages:
  - stage: CI
    jobs:
      - job: Build
        pool:
          vmImage: ubuntu-latest
        steps:
          - script: echo "Build Stage"

  - stage: CD
    dependsOn: CI
    jobs:
      - deployment: DeployApp
        environment: dev
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to Dev Environment"
```
