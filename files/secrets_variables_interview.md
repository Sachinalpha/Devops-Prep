# Secrets and Variables Interview Preparation
## For DevOps Engineers (1 Year Experience)

---

## MOST ASKED INTERVIEW QUESTIONS WITH CODE

---

### Q1. What is the difference between variable and secret?

```
Variable:
  stores non-sensitive configuration
  visible in logs
  can be shared freely
  example: app name, port number, environment name

Secret:
  stores sensitive data
  hidden in logs (masked)
  must be protected
  example: passwords, API keys, tokens, certificates
```

---

### Q2. Azure DevOps - Pipeline Variables

```yaml
# method 1 - define in yaml
variables:
  appName: myapp
  environment: dev
  port: "3000"

steps:
  - script: echo $(appName)
  - script: echo $(port)
```

```yaml
# method 2 - variable groups
# create in Azure DevOps Library first
# Pipelines → Library → Variable Groups → New

variables:
  - group: dev-variables        # all variables in group available
  - group: shared-variables     # multiple groups allowed
  - name: localVar
    value: localvalue           # mix group and local variables

steps:
  - script: echo $(db-host)     # from variable group
  - script: echo $(localVar)    # local variable
```

```yaml
# method 3 - set variable at runtime
steps:
  - script: echo "##vso[task.setvariable variable=myVar]hello world"
    displayName: "Set variable"

  - script: echo $(myVar)
    displayName: "Use variable"

# set variable for next stage (isOutput=true)
  - script: echo "##vso[task.setvariable variable=myVar;isOutput=true]hello"
    name: setVar
    displayName: "Set output variable"
```

---

### Q3. Azure DevOps - Secret Variables

```
Method 1 - Pipeline secret variable:
  1. Go to pipeline
  2. Click Edit
  3. Click Variables (top right)
  4. Click New Variable
  5. Enter name and value
  6. CHECK the lock icon "Keep this value secret"
  7. Save

  In pipeline - value is masked in logs:
  steps:
    - script: echo $(mySecret)    # shows *** in logs
```

```yaml
# method 2 - pass secret to script as env variable
steps:
  - script: |
      echo "DB password is: $DB_PASS"
      ./deploy.sh
    displayName: "Deploy"
    env:
      DB_PASS: $(myDatabasePassword)   # map pipeline secret to env variable
      API_KEY: $(myApiKey)
```

```yaml
# method 3 - Azure Key Vault integration
steps:
  - task: AzureKeyVault@2
    displayName: "Get secrets from Key Vault"
    inputs:
      azureSubscription: 'my-service-connection'
      KeyVaultName: 'my-keyvault-name'
      SecretsFilter: 'DB-PASSWORD,API-KEY,CONNECTION-STRING'
      # SecretsFilter: '*'  means get all secrets
      RunAsPreJob: true

  - script: echo $(DB-PASSWORD)
    displayName: "Use KV secret"
    # secret names in KV use hyphens: DB-PASSWORD
    # in pipeline they become: $(DB-PASSWORD)
```

---

### Q4. Azure Key Vault - Store and retrieve secrets

```bash
# create key vault
az keyvault create \
  --name my-keyvault \
  --resource-group my-rg \
  --location eastus

# add a secret
az keyvault secret set \
  --vault-name my-keyvault \
  --name DB-PASSWORD \
  --value "mysecretpassword"

# get a secret
az keyvault secret show \
  --vault-name my-keyvault \
  --name DB-PASSWORD \
  --query value \
  --output tsv

# list all secrets
az keyvault secret list \
  --vault-name my-keyvault

# delete a secret
az keyvault secret delete \
  --vault-name my-keyvault \
  --name DB-PASSWORD

# give service principal access to key vault
az keyvault set-policy \
  --name my-keyvault \
  --spn "service-principal-id" \
  --secret-permissions get list
```

---

### Q5. Terraform - Variables and Secrets

```hcl
# variables.tf

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true    # hides value in terraform output and logs
}

variable "api_key" {
  description = "API key"
  type        = string
  sensitive   = true
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}
```

```hcl
# terraform.tfvars - NEVER commit this to git!

db_password = "mysecretpassword"
api_key     = "myapikey123"
environment = "prod"
```

```bash
# pass secrets via environment variables (more secure)
export TF_VAR_db_password="mysecretpassword"
export TF_VAR_api_key="myapikey123"
terraform apply
# TF_VAR_ prefix tells terraform these are variable values
```

```hcl
# get secret from Azure Key Vault in Terraform

data "azurerm_key_vault" "kv" {
  name                = "my-keyvault"
  resource_group_name = "my-rg"
}

data "azurerm_key_vault_secret" "db_password" {
  name         = "DB-PASSWORD"          # secret name in key vault
  key_vault_id = data.azurerm_key_vault.kv.id
}

# use the secret value
resource "azurerm_linux_virtual_machine" "vm" {
  admin_password = data.azurerm_key_vault_secret.db_password.value
}
```

---

### Q6. Kubernetes - Secrets

```bash
# create secret from command line
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=mysecretpassword \
  --from-literal=DB_USER=admin

# create secret from file
kubectl create secret generic tls-secret \
  --from-file=tls.crt=./cert.crt \
  --from-file=tls.key=./cert.key

# create docker registry secret
kubectl create secret docker-registry acr-secret \
  --docker-server=myacr.azurecr.io \
  --docker-username=myacr \
  --docker-password=myacrpassword
```

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_PASSWORD: bXlzZWNyZXRwYXNzd29yZA==    # must be base64 encoded
  DB_USER: YWRtaW4=

# encode to base64:
# echo -n "mysecretpassword" | base64
# decode from base64:
# echo "bXlzZWNyZXRwYXNzd29yZA==" | base64 -d
```

```yaml
# use secret in pod
spec:
  containers:
    - name: myapp
      image: myapp:1.0
      env:
        # single value from secret
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD

      envFrom:
        # all values from secret as env variables
        - secretRef:
            name: db-secret

      volumeMounts:
        # mount secret as file
        - name: secret-volume
          mountPath: /app/secrets
          readOnly: true

  volumes:
    - name: secret-volume
      secret:
        secretName: db-secret
```

---

### Q7. Docker - Environment variables and secrets

```bash
# pass env variable at runtime
docker run -e DB_PASSWORD=secret myapp

# pass env file
docker run --env-file .env myapp

# .env file (never commit to git)
DB_PASSWORD=secret
API_KEY=mykey
DB_HOST=localhost
```

```yaml
# docker-compose.yml
services:
  app:
    image: myapp
    environment:
      - DB_PASSWORD=${DB_PASSWORD}   # read from host env variable
    env_file:
      - .env                         # read from .env file
    secrets:
      - db_password                  # docker swarm secrets

secrets:
  db_password:
    file: ./db_password.txt          # read from file
```

---

### Q8. Service Principal and credentials

```bash
# create service principal
az ad sp create-for-rbac \
  --name my-service-principal \
  --role Contributor \
  --scopes /subscriptions/your-subscription-id

# output:
# {
#   "appId": "xxx",         # client ID
#   "displayName": "my-sp",
#   "password": "xxx",      # client secret (save this!)
#   "tenant": "xxx"         # tenant ID
# }

# login with service principal
az login \
  --service-principal \
  --username appId \
  --password clientSecret \
  --tenant tenantId

# store SP credentials in Key Vault
az keyvault secret set --vault-name my-kv --name SP-CLIENT-ID --value "appId"
az keyvault secret set --vault-name my-kv --name SP-CLIENT-SECRET --value "password"
az keyvault secret set --vault-name my-kv --name SP-TENANT-ID --value "tenant"
```

---

### Q9. .gitignore - never commit these files

```
# .gitignore

# terraform secrets
*.tfvars              # variable files with secrets
terraform.tfstate     # state file has resource details
terraform.tfstate.backup
.terraform/

# environment files
.env
.env.local
.env.production
*.env

# certificates and keys
*.pem
*.key
*.crt
*.p12
*.pfx
id_rsa
id_rsa.pub

# docker secrets
docker-compose.override.yml

# kubernetes secrets
*-secret.yaml
secrets.yaml

# general
*.secret
credentials
config/secrets/
```

---

### Q10. Azure DevOps - Variable scope

```yaml
# pipeline level variable - available everywhere
variables:
  pipelineVar: "available everywhere"

stages:
  - stage: Build
    variables:
      stageVar: "available in Build stage only"  # stage level
    jobs:
      - job: BuildJob
        variables:
          jobVar: "available in this job only"   # job level
        steps:
          - script: echo $(pipelineVar)  # works
          - script: echo $(stageVar)     # works
          - script: echo $(jobVar)       # works

  - stage: Deploy
    steps:
      - script: echo $(pipelineVar)    # works
      - script: echo $(stageVar)       # FAILS - stage var not available here
      - script: echo $(jobVar)         # FAILS - job var not available here
```

---

### Q11. Passing variables between stages

```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: |
              echo "##vso[task.setvariable variable=imageTag;isOutput=true]$(Build.BuildId)"
            name: setTag
            displayName: "Set image tag"

  - stage: Deploy
    dependsOn: Build
    variables:
      # reference output variable from previous stage
      imageTag: $[ stageDependencies.Build.BuildJob.outputs['setTag.imageTag'] ]
      # format: stageDependencies.StageName.JobName.outputs['stepName.variableName']
    jobs:
      - job: DeployJob
        steps:
          - script: echo "Deploying image tag $(imageTag)"
```

---

### Q12. Azure Key Vault - Access policies vs RBAC

```bash
# old way - access policies
az keyvault set-policy \
  --name my-keyvault \
  --upn user@domain.com \
  --secret-permissions get list set delete

# new way - RBAC (recommended)
# assign role to user/SP on key vault
az role assignment create \
  --role "Key Vault Secrets Officer" \
  --assignee user@domain.com \
  --scope /subscriptions/xxx/resourceGroups/my-rg/providers/Microsoft.KeyVault/vaults/my-keyvault

# common Key Vault RBAC roles:
# Key Vault Administrator          = full access
# Key Vault Secrets Officer        = manage secrets
# Key Vault Secrets User           = read secrets only
# Key Vault Reader                 = read metadata only
```

---

### Common Security Mistakes to Avoid

```
1. Committing .env files or .tfvars to git
2. Storing passwords in plain text in yaml files
3. Using same secret for all environments
4. Not rotating secrets regularly
5. Giving too many permissions to service principals
6. Logging secret values in pipeline output
7. Using default namespace for all resources (no isolation)
8. Not using Key Vault for centralized secret management
9. Hardcoding credentials in Dockerfiles
10. Not scanning code for leaked secrets (use git-secrets or truffleHog)
```

---

### Quick Reference - Where to store secrets

```
Azure Key Vault         = centralized secret storage for Azure
                          all apps and pipelines read from here

Azure DevOps Secrets    = pipeline specific secrets
                          link variable group to Key Vault

Kubernetes Secrets      = secrets for containerized apps
                          use Azure Key Vault Provider for Secrets Store CSI driver
                          to sync KV secrets into K8s secrets

Terraform sensitive var = mark sensitive=true
                          pass via TF_VAR_ env variables
                          never in .tfvars committed to git

Docker secrets          = for docker swarm
                          for K8s use K8s secrets instead
```
