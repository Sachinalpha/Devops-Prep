# Azure DevOps Pipelines Interview Preparation
## For DevOps Engineers (1 Year Experience)

---

## MOST ASKED INTERVIEW QUESTIONS WITH CODE

---

### Q1. What is Azure DevOps Pipeline?

```
Azure DevOps Pipeline automates:
- building your code
- testing your code
- deploying your code

You define pipeline in azure-pipelines.yml file
Push code → pipeline triggers → builds → tests → deploys
```

---

### Q2. Basic Pipeline Structure

```yaml
# azure-pipelines.yml

name: MyPipeline_$(Date:yyyyMMdd)_$(Rev:.r)
# name = pipeline run name
# $(Date:yyyyMMdd) = date like 20260313
# $(Rev:.r) = revision number 1, 2, 3

trigger:
- main          # run when code is pushed to main branch

pool:
  vmImage: 'ubuntu-latest'   # microsoft hosted agent

stages:
  - stage: Build
    displayName: "Build Stage"
    jobs:
      - job: BuildJob
        displayName: "Build Job"
        steps:
          - checkout: self    # checkout source code
          - script: echo "Hello World"
            displayName: "Print Hello"
```

---

### Q3. What is trigger in Azure Pipelines?

```yaml
# trigger on specific branches
trigger:
- main
- develop
- release/*    # wildcard - all branches starting with release/

# trigger on specific paths
trigger:
  branches:
    include:
    - main
  paths:
    include:
    - src/*       # only trigger if files in src/ changed
    exclude:
    - docs/*      # dont trigger if only docs changed

# disable automatic trigger
trigger: none   # run manually only

# pull request trigger
pr:
- main          # run when PR is created targeting main
```

---

### Q4. What are variables in Azure Pipelines?

```yaml
# method 1 - inline variables
variables:
  appName: myapp
  environment: dev
  buildConfig: Release

steps:
  - script: echo $(appName)      # use with $(variableName)
    displayName: "Print app name"
  - script: echo $(environment)

---

# method 2 - variable groups (from Azure DevOps library)
variables:
  - group: my-variable-group    # contains multiple variables
  - name: appName
    value: myapp

---

# method 3 - stage/job level variables
stages:
  - stage: Build
    variables:
      stageName: BuildStage     # only available in this stage
    jobs:
      - job: BuildJob
        variables:
          jobName: BuildJob     # only available in this job
        steps:
          - script: echo $(stageName) $(jobName)

---

# method 4 - runtime variables (set during pipeline run)
steps:
  - script: echo "##vso[task.setvariable variable=myVar]hello"
    displayName: "Set variable"
  - script: echo $(myVar)
    displayName: "Use variable"
```

---

### Q5. What are secrets/secret variables?

```yaml
# NEVER put secrets directly in yaml file
# Use Azure DevOps secret variables or Key Vault

# method 1 - pipeline secret variable
# Set in Azure DevOps UI:
# Pipelines → Edit → Variables → Add → check "Keep this value secret"
# Then use in pipeline:
steps:
  - script: echo $(mySecretVar)   # value is masked in logs
    env:
      SECRET_VALUE: $(mySecretVar)   # pass as env variable to script

---

# method 2 - Azure Key Vault
steps:
  - task: AzureKeyVault@2
    inputs:
      azureSubscription: 'my-service-connection'
      KeyVaultName: 'my-keyvault'
      SecretsFilter: '*'           # fetch all secrets
      RunAsPreJob: true            # fetch before other steps

  - script: echo $(mySecret)      # now available as variable
    displayName: "Use KV secret"

---

# method 3 - variable groups linked to Key Vault
# in Azure DevOps Library → Variable Groups → Link to Key Vault
variables:
  - group: keyvault-linked-group   # all KV secrets available as variables
```

---

### Q6. What is dependsOn in stages?

```yaml
stages:
  - stage: Build
    displayName: "Build"
    jobs:
      - job: BuildJob
        steps:
          - script: echo "building"

  - stage: Test
    displayName: "Test"
    dependsOn: Build        # Test runs after Build completes
    jobs:
      - job: TestJob
        steps:
          - script: echo "testing"

  - stage: Deploy
    displayName: "Deploy"
    dependsOn: Test         # Deploy runs after Test completes
    jobs:
      - job: DeployJob
        steps:
          - script: echo "deploying"

# parallel stages (run at same time)
  - stage: Lint
    dependsOn: []           # empty array = no dependency = runs in parallel
```

---

### Q7. Publish and Download Artifacts

```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - checkout: self

          # copy files to staging directory
          - task: CopyFiles@2
            displayName: "Copy Files"
            inputs:
              SourceFolder: '$(Build.SourcesDirectory)'
              Contents: '**'
              TargetFolder: '$(Build.ArtifactStagingDirectory)/drop'

          # publish artifact
          - task: PublishBuildArtifacts@1
            displayName: "Publish Artifact"
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)/drop'
              ArtifactName: 'drop'           # artifact name
              publishLocation: 'Container'

  - stage: Deploy
    dependsOn: Build
    jobs:
      - job: DeployJob
        steps:
          # download artifact
          - task: DownloadBuildArtifacts@1
            displayName: "Download Artifact"
            inputs:
              buildType: 'current'
              downloadType: 'single'
              artifactName: 'drop'            # must match publish name
              downloadPath: '$(System.ArtifactsDirectory)'

          - script: ls $(System.ArtifactsDirectory)/drop
            displayName: "List downloaded files"
```

---

### Q8. What are predefined variables?

```yaml
# Azure DevOps provides many built-in variables

steps:
  - script: |
      echo "Build ID: $(Build.BuildId)"
      echo "Build Number: $(Build.BuildNumber)"
      echo "Source Branch: $(Build.SourceBranchName)"
      echo "Source Directory: $(Build.SourcesDirectory)"
      echo "Staging Directory: $(Build.ArtifactStagingDirectory)"
      echo "Agent Name: $(Agent.Name)"
      echo "Agent OS: $(Agent.OS)"
      echo "Pipeline Name: $(Build.DefinitionName)"
      echo "Commit ID: $(Build.SourceVersion)"
      echo "Repo Name: $(Build.Repository.Name)"
      echo "System Team Project: $(System.TeamProject)"
      echo "Artifacts Directory: $(System.ArtifactsDirectory)"
```

---

### Q9. Conditions in Pipeline

```yaml
steps:
  - script: echo "This always runs"
    condition: always()               # runs even if previous step failed

  - script: echo "This runs on success"
    condition: succeeded()            # default, runs if previous step succeeded

  - script: echo "This runs on failure"
    condition: failed()               # runs only if previous step failed

  - script: echo "Deploy to prod"
    condition: and(succeeded(), eq(variables['Build.SourceBranchName'], 'main'))
    # runs only if previous step succeeded AND branch is main

  - script: echo "Dev deploy"
    condition: eq(variables['environment'], 'dev')
    # runs only if environment variable equals dev
```

---

### Q10. Deploy to Azure App Service

```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '18.x'
            displayName: "Install Node.js"

          - script: |
              npm install
              npm run build
            displayName: "Build App"

          - task: ArchiveFiles@2
            inputs:
              rootFolderOrFile: '$(Build.SourcesDirectory)'
              includeRootFolder: false
              archiveType: 'zip'
              archiveFile: '$(Build.ArtifactStagingDirectory)/app.zip'
            displayName: "Zip App"

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'webapp'

  - stage: Deploy
    dependsOn: Build
    jobs:
      - job: DeployJob
        steps:
          - task: DownloadBuildArtifacts@1
            inputs:
              artifactName: 'webapp'
              downloadPath: '$(System.ArtifactsDirectory)'

          - task: AzureWebApp@1
            displayName: "Deploy to App Service"
            inputs:
              azureSubscription: 'my-service-connection'   # service connection name
              appName: 'my-webapp'                         # app service name
              package: '$(System.ArtifactsDirectory)/webapp/app.zip'
```

---

### Q11. Multi environment deployment with approval

```yaml
stages:
  - stage: DeployDev
    displayName: "Deploy to Dev"
    jobs:
      - deployment: DeployDev        # deployment job (not regular job)
        displayName: "Deploy to Dev"
        environment: 'dev'           # environment name in Azure DevOps
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to dev"

  - stage: DeployProd
    displayName: "Deploy to Prod"
    dependsOn: DeployDev
    jobs:
      - deployment: DeployProd
        displayName: "Deploy to Prod"
        environment: 'prod'          # set approval in Azure DevOps Environments
        # go to: Environments → prod → Approvals and checks → Add approval
        # pipeline will PAUSE here and wait for manual approval
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to prod"
```

---

### Q12. Docker build and push in pipeline

```yaml
variables:
  imageRepository: 'myapp'
  containerRegistry: 'myacr.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'

stages:
  - stage: Build
    jobs:
      - job: DockerBuild
        steps:
          - task: AzureCLI@2
            displayName: "Build and Push Docker Image"
            inputs:
              azureSubscription: 'my-service-connection'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                az acr login --name myacr
                docker build -t $(containerRegistry)/$(imageRepository):$(tag) .
                docker push $(containerRegistry)/$(imageRepository):$(tag)
```

---

### Q13. Terraform in Azure Pipeline

```yaml
stages:
  - stage: Terraform
    jobs:
      - job: TerraformJob
        steps:
          - task: TerraformInstaller@1
            displayName: "Install Terraform"
            inputs:
              terraformVersion: 'latest'

          - task: AzureCLI@2
            displayName: "Terraform Init Plan Apply"
            inputs:
              azureSubscription: 'my-service-connection'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                cd terraform
                terraform init
                terraform plan -out=tfplan
                terraform apply -auto-approve tfplan
```

---

### Q14. Pipeline templates (reusable pipelines)

```yaml
# templates/build-steps.yml - reusable template

parameters:
  - name: nodeVersion
    type: string
    default: '18.x'

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: ${{ parameters.nodeVersion }}

  - script: npm install
    displayName: "Install dependencies"

  - script: npm test
    displayName: "Run tests"
```

```yaml
# azure-pipelines.yml - using template

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - template: templates/build-steps.yml   # include template
            parameters:
              nodeVersion: '18.x'                 # pass parameter

  - stage: BuildOld
    jobs:
      - job: BuildJob
        steps:
          - template: templates/build-steps.yml
            parameters:
              nodeVersion: '16.x'                 # different parameter
```

---

### Q15. Common pipeline predefined variables

```
$(Build.BuildId)              = unique build number like 123
$(Build.BuildNumber)          = pipeline run name
$(Build.SourceBranch)         = refs/heads/main
$(Build.SourceBranchName)     = main
$(Build.SourcesDirectory)     = /home/vsts/work/1/s
$(Build.ArtifactStagingDirectory) = /home/vsts/work/1/a
$(System.ArtifactsDirectory)  = /home/vsts/work/1/a
$(Agent.WorkFolder)           = /home/vsts/work
$(System.DefaultWorkingDirectory) = /home/vsts/work/1/s
```

---

### Common Mistakes to Avoid

```
1. Hardcoding secrets in yaml file
2. Not using dependsOn correctly
3. Wrong indentation in yaml (causes errors)
4. Not using artifact publish/download for multi stage
5. Space in job name (use BuildJob not Build Job)
6. Not adding approval for prod environment
7. Using wrong service connection name
```
