# Azure Pipelines YAML Rehberi (TR/EN)

Bu doküman, Azure Pipelines YAML dosyalarında en sık kullanılan temel komutları, ortama göre koşullu çalışma örneklerini ve zamanlı çalıştırma (schedule) yapısını Türkçe ve İngilizce olarak özetler.

## İçerik / Contents

- [Türkçe Bölüme Git](#turkce)
- [Go to English Section](#english)

---

## Türkçe

### 1) Temel Azure Pipelines YAML Komutları

- `trigger`: Hangi branch'lerde otomatik çalışacağını belirler.
- `pr`: Pull Request tetiklemelerini tanımlar.
- `pool`: Agent havuzunu veya imajını seçer.
- `variables`: Sabitler, gizli değişken referansları ve değişken grupları.
- `stages` / `jobs` / `steps`: Pipeline yapısının hiyerarşisi.
- `script` ve `task`: Komut satırı veya hazır task çalıştırma.
- `dependsOn` / `condition`: Bağımlılık ve koşul bazlı akış.

```yaml
trigger:
  branches:
    include:
      - main
      - develop

pr:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

variables:
  buildConfiguration: 'Release'

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: echo "Build başlıyor"
            displayName: Build Start
          - task: DotNetCoreCLI@2
            displayName: Restore
            inputs:
              command: restore
              projects: '**/*.csproj'
```

### 1.1) Hangi Teknolojiye Göre Build Yapılacağını Belirleme

Build komutu, projenin teknolojisine göre değişir. En kritik nokta doğru agent (`windows-latest` veya `ubuntu-latest`) ve doğru task/komutu seçmektir.

- **.NET 8 (SDK-style, cross-platform)**: `UseDotNet@2` + `DotNetCoreCLI@2`
- **.NET Framework 4.7**: Genelde `windows-latest` + `NuGetCommand@2` + `VSBuild@1` (MSBuild tabanlı)
- **MSBuild (genel)**: `MSBuild@1` veya `VSBuild@1`
- **Docker Compose**: Docker/Compose komutları veya Docker task'ları

```yaml
jobs:
  - job: BuildByTech
    strategy:
      matrix:
        dotnet8:
          vmImage: 'ubuntu-latest'
          buildTech: 'dotnet8'
        netframework47:
          vmImage: 'windows-latest'
          buildTech: 'netfx47'
        dockercompose:
          vmImage: 'ubuntu-latest'
          buildTech: 'dockercompose'
    pool:
      vmImage: $(vmImage)
    steps:
      - task: UseDotNet@2
        condition: eq(variables['buildTech'], 'dotnet8')
        inputs:
          packageType: sdk
          version: '8.0.x'

      - task: DotNetCoreCLI@2
        condition: eq(variables['buildTech'], 'dotnet8')
        inputs:
          command: build
          projects: '**/*.csproj'

      - task: NuGetCommand@2
        condition: eq(variables['buildTech'], 'netfx47')
        inputs:
          command: restore
          restoreSolution: '**/*.sln'

      - task: VSBuild@1
        condition: eq(variables['buildTech'], 'netfx47')
        inputs:
          solution: '**/*.sln'
          msbuildArgs: '/p:Configuration=Release'

      - script: docker compose -f docker-compose.yml build
        condition: eq(variables['buildTech'], 'dockercompose')
        displayName: Docker Compose Build
```

### 2) Environment Bazlı İşlem (Dosya/Parametre Seçimi)

Aşağıdaki örneklerde branch veya environment değişkenine göre farklı parametre/dosya seçimi yapılır:

```yaml
variables:
  - name: environmentName
    value: 'dev'

  - ${{ if eq(variables['Build.SourceBranchName'], 'main') }}:
    - name: environmentName
      value: 'prod'

  - ${{ if eq(variables['environmentName'], 'prod') }}:
    - group: vg-prod
  - ${{ if ne(variables['environmentName'], 'prod') }}:
    - group: vg-nonprod

steps:
  - script: |
      echo "Seçilen environment: $(environmentName)"
      if [ "$(environmentName)" = "prod" ]; then
        cp config/appsettings.prod.json config/appsettings.json
      else
        cp config/appsettings.dev.json config/appsettings.json
      fi
    displayName: Ortama göre dosya seç
```

Deployment job ile `environment` kullanım örneği:

```yaml
stages:
  - stage: Deploy
    jobs:
      - deployment: DeployWeb
        environment: $(environmentName)
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploy target: $(environmentName)"
```

### 3) Zamanlı Çalıştırma (Scheduled Runs)

Azure Pipelines'ta `schedules` ile cron tabanlı çalıştırma yapılabilir:

```yaml
schedules:
  - cron: "0 3 * * 1-5"
    displayName: Weekday 03:00 Build
    branches:
      include:
        - main
    always: true
```

- `cron`: UTC formatında zamanlama.
- `branches.include`: Hangi branch için schedule tetikleneceği.
- `always: true`: Kod değişmese bile çalıştırır.

### 4) Test Süreçleri (CI Test Stage)

Build sonrasında testleri ayrı bir stage/job olarak çalıştırmak iyi bir pratiktir:

```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: dotnet build --configuration Release
            displayName: Build

  - stage: Test
    dependsOn: Build
    condition: succeeded()
    jobs:
      - job: UnitTests
        steps:
          - script: dotnet test --configuration Release --collect:"XPlat Code Coverage"
            displayName: Run Unit Tests
          - task: PublishTestResults@2
            inputs:
              testResultsFormat: VSTest
              testResultsFiles: '**/*.trx'
              failTaskOnFailedTests: true
```

### 5) Deploy Makinesi ve Yetkili Kullanıcı Bilgisi (Azure Environment Değerlerinden Alma)

Bu örnekte hedef makine ve yetkili kullanıcı bilgisi pipeline içinde sabit yazılmaz; Azure DevOps tarafında environment'a göre yönetilen variable group'lardan çekilir. Şifre/anahtar gibi hassas veriler secret olarak tutulmalıdır.

```yaml
variables:
  - name: environmentName
    value: 'dev'

  - ${{ if eq(variables['Build.SourceBranchName'], 'main') }}:
    - name: environmentName
      value: 'prod'

  - ${{ if eq(variables['environmentName'], 'prod') }}:
    - group: env-prod-deploy-vars
  - ${{ if ne(variables['environmentName'], 'prod') }}:
    - group: env-dev-deploy-vars

steps:
  - script: |
      echo "Deploy env: $(environmentName)"
      echo "Target host: $(TARGET_HOST)"
      echo "Deploy user: $(DEPLOY_USER)"
      # TARGET_HOST, DEPLOY_USER ve DEPLOY_SSH_KEY secret/safe şekilde Azure'dan gelir.
      ssh $(DEPLOY_USER)@$(TARGET_HOST) "sudo systemctl restart my-app"
    displayName: Environment bazlı deploy bağlantısı
```

Örnek variable group içeriği:
- `env-dev-deploy-vars`: `TARGET_HOST=10.10.20.15`, `DEPLOY_USER=dev_deployer`
- `env-prod-deploy-vars`: `TARGET_HOST=10.10.10.25`, `DEPLOY_USER=prod_deployer`

Azure DevOps UI üzerinde variable group oluşturma (kısa):
1. `Pipelines` > `Library` bölümüne gidin.
2. `+ Variable group` seçeneğine tıklayın.
3. Grup adını girin (örneğin `env-dev-deploy-vars`).
4. Değişkenleri ekleyin: `TARGET_HOST`, `DEPLOY_USER`, gerekiyorsa `DEPLOY_SSH_KEY`.
5. Hassas değerleri `Keep this value secret` ile secret olarak işaretleyin.
6. Pipeline izinlerini verin (`Pipeline permissions`) ve kaydedin.

### 6) Uçtan Uca Örnek (Stage Branch + .NET 10 + Backup + Deploy + Restart)

Bu örnek aşağıdakileri birlikte yapar:
- `stage` branch güncellenince otomatik tetiklenir.
- `environment` olarak `stage` kullanır.
- `.NET 10` ile build ve publish alır.
- xUnit unit test ve integration test çalıştırır.
- Uzak makine bağlantı bilgilerini variable group üzerinden alır.
- Deploy öncesi uzak sunucuda mevcut dosyaların yedeğini alır.
- Yeni sürümü kopyalar, ardından servisi yeniden başlatır.

```yaml
trigger:
  branches:
    include:
      - stage

pr:
  branches:
    include:
      - stage

pool:
  vmImage: ubuntu-latest

variables:
  - group: env-stage-deploy-vars
  - name: buildConfiguration
    value: 'Release'
  - name: dotnetSdkVersion
    value: '10.0.x'
  - name: environmentName
    value: 'stage'
  - name: artifactName
    value: 'app-publish'

stages:
  - stage: Build_Test_Publish
    displayName: Build, Test and Publish
    jobs:
      - job: BuildAndTest
        displayName: Build and Run Tests
        steps:
          - checkout: self

          - task: UseDotNet@2
            displayName: Install .NET SDK
            inputs:
              packageType: sdk
              version: $(dotnetSdkVersion)

          - task: DotNetCoreCLI@2
            displayName: Restore
            inputs:
              command: restore
              projects: '**/*.sln'

          - task: DotNetCoreCLI@2
            displayName: Build
            inputs:
              command: build
              projects: '**/*.sln'
              arguments: '--configuration $(buildConfiguration) --no-restore'

          - task: DotNetCoreCLI@2
            displayName: Run xUnit Unit Tests
            inputs:
              command: test
              projects: '**/*UnitTests*.csproj'
              arguments: '--configuration $(buildConfiguration) --no-build --logger trx'

          - task: DotNetCoreCLI@2
            displayName: Run Integration Tests
            inputs:
              command: test
              projects: '**/*IntegrationTests*.csproj'
              arguments: '--configuration $(buildConfiguration) --no-build --logger trx'

          - task: PublishTestResults@2
            displayName: Publish Test Results
            inputs:
              testResultsFormat: VSTest
              testResultsFiles: '**/*.trx'
              failTaskOnFailedTests: true

          - task: DotNetCoreCLI@2
            displayName: Publish App
            inputs:
              command: publish
              projects: '**/YourApp.csproj'
              publishWebProjects: false
              arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/publish'
              zipAfterPublish: false

          - task: PublishPipelineArtifact@1
            displayName: Publish Artifact
            inputs:
              targetPath: '$(Build.ArtifactStagingDirectory)/publish'
              artifact: '$(artifactName)'

  - stage: Deploy_Stage
    displayName: Deploy to Stage
    dependsOn: Build_Test_Publish
    condition: succeeded()
    jobs:
      - deployment: DeployToStage
        displayName: Backup, Deploy and Restart
        environment: stage
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: $(artifactName)

                - script: |
                    sudo apt-get update
                    sudo apt-get install -y sshpass rsync
                  displayName: Install deployment tools

                - script: |
                    set -e
                    NOW_UTC=$(date -u +"%Y%m%d_%H%M%S")
                    echo "Backup timestamp: $NOW_UTC"

                    sshpass -p "$(DEPLOY_PASSWORD)" ssh -o StrictHostKeyChecking=no "$(DEPLOY_USER)@$(TARGET_HOST)" "
                      mkdir -p $(BACKUP_ROOT) &&
                      if [ -d $(DEPLOY_PATH) ] && [ \"\$(ls -A $(DEPLOY_PATH) 2>/dev/null)\" ]; then
                        tar -czf $(BACKUP_ROOT)/app_backup_${NOW_UTC}.tar.gz -C $(DEPLOY_PATH) .
                      fi
                    "
                  displayName: Backup existing files on remote server

                - script: |
                    set -e
                    sshpass -p "$(DEPLOY_PASSWORD)" ssh -o StrictHostKeyChecking=no "$(DEPLOY_USER)@$(TARGET_HOST)" "mkdir -p $(DEPLOY_PATH)"

                    sshpass -p "$(DEPLOY_PASSWORD)" rsync -avz --delete \
                      -e "ssh -o StrictHostKeyChecking=no" \
                      "$(Pipeline.Workspace)/$(artifactName)/" \
                      "$(DEPLOY_USER)@$(TARGET_HOST):$(DEPLOY_PATH)/"
                  displayName: Deploy new files to remote server

                - script: |
                    sshpass -p "$(DEPLOY_PASSWORD)" ssh -o StrictHostKeyChecking=no "$(DEPLOY_USER)@$(TARGET_HOST)" \
                      "sudo systemctl restart $(SERVICE_NAME) && sudo systemctl status $(SERVICE_NAME) --no-pager"
                  displayName: Restart application service
```

Beklenen `env-stage-deploy-vars` değişkenleri:
- `TARGET_HOST`
- `DEPLOY_USER`
- `DEPLOY_PASSWORD` (secret)
- `DEPLOY_PATH` (ör. `/var/www/my-app`)
- `BACKUP_ROOT` (ör. `/var/backups/my-app`)
- `SERVICE_NAME` (ör. `my-app`)

Not: `YourApp.csproj` yolunu kendi proje dosyanla değiştir.

### 7) Kısa İpuçları

- Gizli bilgiler için variable group + secret variable kullanın.
- Reusable yapı için `template` dosyalarına bölün.
- Ortam geçişlerinde `condition` ve branch kontrollerini açık yazın.

---

## English

### 1) Core Azure Pipelines YAML Commands

- `trigger`: Defines branches for CI execution.
- `pr`: Defines Pull Request triggers.
- `pool`: Selects the agent pool/image.
- `variables`: Constants, secret refs, and variable groups.
- `stages` / `jobs` / `steps`: Pipeline hierarchy.
- `script` and `task`: Run shell commands or built-in tasks.
- `dependsOn` / `condition`: Dependency and condition-based flow.

```yaml
trigger:
  branches:
    include:
      - main
      - develop

pr:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

variables:
  buildConfiguration: 'Release'

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: echo "Build started"
            displayName: Build Start
          - task: DotNetCoreCLI@2
            displayName: Restore
            inputs:
              command: restore
              projects: '**/*.csproj'
```

### 1.1) Choosing Build Technology

Your build implementation should match your stack. The key is selecting the correct agent (`windows-latest` vs `ubuntu-latest`) and the right tasks/commands.

- **.NET 8 (SDK-style, cross-platform)**: `UseDotNet@2` + `DotNetCoreCLI@2`
- **.NET Framework 4.7**: usually `windows-latest` + `NuGetCommand@2` + `VSBuild@1` (MSBuild-based)
- **MSBuild (generic)**: `MSBuild@1` or `VSBuild@1`
- **Docker Compose**: Docker/Compose commands or Docker tasks

```yaml
jobs:
  - job: BuildByTech
    strategy:
      matrix:
        dotnet8:
          vmImage: 'ubuntu-latest'
          buildTech: 'dotnet8'
        netframework47:
          vmImage: 'windows-latest'
          buildTech: 'netfx47'
        dockercompose:
          vmImage: 'ubuntu-latest'
          buildTech: 'dockercompose'
    pool:
      vmImage: $(vmImage)
    steps:
      - task: UseDotNet@2
        condition: eq(variables['buildTech'], 'dotnet8')
        inputs:
          packageType: sdk
          version: '8.0.x'

      - task: DotNetCoreCLI@2
        condition: eq(variables['buildTech'], 'dotnet8')
        inputs:
          command: build
          projects: '**/*.csproj'

      - task: NuGetCommand@2
        condition: eq(variables['buildTech'], 'netfx47')
        inputs:
          command: restore
          restoreSolution: '**/*.sln'

      - task: VSBuild@1
        condition: eq(variables['buildTech'], 'netfx47')
        inputs:
          solution: '**/*.sln'
          msbuildArgs: '/p:Configuration=Release'

      - script: docker compose -f docker-compose.yml build
        condition: eq(variables['buildTech'], 'dockercompose')
        displayName: Docker Compose Build
```

### 2) Environment-Based Logic (Selecting Files/Parameters)

These examples pick different params/files based on branch or environment:

```yaml
variables:
  - name: environmentName
    value: 'dev'

  - ${{ if eq(variables['Build.SourceBranchName'], 'main') }}:
    - name: environmentName
      value: 'prod'

  - ${{ if eq(variables['environmentName'], 'prod') }}:
    - group: vg-prod
  - ${{ if ne(variables['environmentName'], 'prod') }}:
    - group: vg-nonprod

steps:
  - script: |
      echo "Selected environment: $(environmentName)"
      if [ "$(environmentName)" = "prod" ]; then
        cp config/appsettings.prod.json config/appsettings.json
      else
        cp config/appsettings.dev.json config/appsettings.json
      fi
    displayName: Select file by environment
```

Deployment job `environment` usage:

```yaml
stages:
  - stage: Deploy
    jobs:
      - deployment: DeployWeb
        environment: $(environmentName)
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploy target: $(environmentName)"
```

### 3) Scheduled Runs

Use `schedules` for cron-based execution:

```yaml
schedules:
  - cron: "0 3 * * 1-5"
    displayName: Weekday 03:00 Build
    branches:
      include:
        - main
    always: true
```

- `cron`: Schedule in UTC.
- `branches.include`: Branch scope for schedule trigger.
- `always: true`: Runs even without code changes.

### 4) Test Processes (CI Test Stage)

Running tests in a dedicated stage/job after build is a solid practice:

```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: dotnet build --configuration Release
            displayName: Build

  - stage: Test
    dependsOn: Build
    condition: succeeded()
    jobs:
      - job: UnitTests
        steps:
          - script: dotnet test --configuration Release --collect:"XPlat Code Coverage"
            displayName: Run Unit Tests
          - task: PublishTestResults@2
            inputs:
              testResultsFormat: VSTest
              testResultsFiles: '**/*.trx'
              failTaskOnFailedTests: true
```

### 5) Deploy Target Machine and Authorized User (Read from Azure Environment Values)

In this sample, target machine and deploy user are not hardcoded in YAML; they are loaded from environment-specific variable groups managed in Azure DevOps. Keep passwords/keys as secrets.

```yaml
variables:
  - name: environmentName
    value: 'dev'

  - ${{ if eq(variables['Build.SourceBranchName'], 'main') }}:
    - name: environmentName
      value: 'prod'

  - ${{ if eq(variables['environmentName'], 'prod') }}:
    - group: env-prod-deploy-vars
  - ${{ if ne(variables['environmentName'], 'prod') }}:
    - group: env-dev-deploy-vars

steps:
  - script: |
      echo "Deploy env: $(environmentName)"
      echo "Target host: $(TARGET_HOST)"
      echo "Deploy user: $(DEPLOY_USER)"
      # TARGET_HOST, DEPLOY_USER, and DEPLOY_SSH_KEY come from Azure-managed variables.
      ssh $(DEPLOY_USER)@$(TARGET_HOST) "sudo systemctl restart my-app"
    displayName: Environment-based deploy connection
```

Example variable groups:
- `env-dev-deploy-vars`: `TARGET_HOST=10.10.20.15`, `DEPLOY_USER=dev_deployer`
- `env-prod-deploy-vars`: `TARGET_HOST=10.10.10.25`, `DEPLOY_USER=prod_deployer`

Create variable groups in Azure DevOps UI (quick):
1. Go to `Pipelines` > `Library`.
2. Click `+ Variable group`.
3. Enter the group name (for example `env-dev-deploy-vars`).
4. Add variables: `TARGET_HOST`, `DEPLOY_USER`, and optionally `DEPLOY_SSH_KEY`.
5. Mark sensitive values as secret using `Keep this value secret`.
6. Grant access via `Pipeline permissions` and save.

### 6) End-to-End Sample (Stage Branch + .NET 10 + Backup + Deploy + Restart)

This sample does all of the following:
- Automatically triggers when the `stage` branch is updated.
- Uses `stage` as the deployment environment.
- Builds and publishes with `.NET 10`.
- Runs xUnit unit tests and integration tests.
- Reads remote server credentials from a variable group.
- Backs up existing files on the remote server before deployment.
- Deploys the new version and restarts the application service.

```yaml
trigger:
  branches:
    include:
      - stage

pr:
  branches:
    include:
      - stage

pool:
  vmImage: ubuntu-latest

variables:
  - group: env-stage-deploy-vars
  - name: buildConfiguration
    value: 'Release'
  - name: dotnetSdkVersion
    value: '10.0.x'
  - name: environmentName
    value: 'stage'
  - name: artifactName
    value: 'app-publish'

stages:
  - stage: Build_Test_Publish
    displayName: Build, Test and Publish
    jobs:
      - job: BuildAndTest
        displayName: Build and Run Tests
        steps:
          - checkout: self

          - task: UseDotNet@2
            displayName: Install .NET SDK
            inputs:
              packageType: sdk
              version: $(dotnetSdkVersion)

          - task: DotNetCoreCLI@2
            displayName: Restore
            inputs:
              command: restore
              projects: '**/*.sln'

          - task: DotNetCoreCLI@2
            displayName: Build
            inputs:
              command: build
              projects: '**/*.sln'
              arguments: '--configuration $(buildConfiguration) --no-restore'

          - task: DotNetCoreCLI@2
            displayName: Run xUnit Unit Tests
            inputs:
              command: test
              projects: '**/*UnitTests*.csproj'
              arguments: '--configuration $(buildConfiguration) --no-build --logger trx'

          - task: DotNetCoreCLI@2
            displayName: Run Integration Tests
            inputs:
              command: test
              projects: '**/*IntegrationTests*.csproj'
              arguments: '--configuration $(buildConfiguration) --no-build --logger trx'

          - task: PublishTestResults@2
            displayName: Publish Test Results
            inputs:
              testResultsFormat: VSTest
              testResultsFiles: '**/*.trx'
              failTaskOnFailedTests: true

          - task: DotNetCoreCLI@2
            displayName: Publish App
            inputs:
              command: publish
              projects: '**/YourApp.csproj'
              publishWebProjects: false
              arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/publish'
              zipAfterPublish: false

          - task: PublishPipelineArtifact@1
            displayName: Publish Artifact
            inputs:
              targetPath: '$(Build.ArtifactStagingDirectory)/publish'
              artifact: '$(artifactName)'

  - stage: Deploy_Stage
    displayName: Deploy to Stage
    dependsOn: Build_Test_Publish
    condition: succeeded()
    jobs:
      - deployment: DeployToStage
        displayName: Backup, Deploy and Restart
        environment: stage
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: $(artifactName)

                - script: |
                    sudo apt-get update
                    sudo apt-get install -y sshpass rsync
                  displayName: Install deployment tools

                - script: |
                    set -e
                    NOW_UTC=$(date -u +"%Y%m%d_%H%M%S")
                    echo "Backup timestamp: $NOW_UTC"

                    sshpass -p "$(DEPLOY_PASSWORD)" ssh -o StrictHostKeyChecking=no "$(DEPLOY_USER)@$(TARGET_HOST)" "
                      mkdir -p $(BACKUP_ROOT) &&
                      if [ -d $(DEPLOY_PATH) ] && [ \"\$(ls -A $(DEPLOY_PATH) 2>/dev/null)\" ]; then
                        tar -czf $(BACKUP_ROOT)/app_backup_${NOW_UTC}.tar.gz -C $(DEPLOY_PATH) .
                      fi
                    "
                  displayName: Backup existing files on remote server

                - script: |
                    set -e
                    sshpass -p "$(DEPLOY_PASSWORD)" ssh -o StrictHostKeyChecking=no "$(DEPLOY_USER)@$(TARGET_HOST)" "mkdir -p $(DEPLOY_PATH)"

                    sshpass -p "$(DEPLOY_PASSWORD)" rsync -avz --delete \
                      -e "ssh -o StrictHostKeyChecking=no" \
                      "$(Pipeline.Workspace)/$(artifactName)/" \
                      "$(DEPLOY_USER)@$(TARGET_HOST):$(DEPLOY_PATH)/"
                  displayName: Deploy new files to remote server

                - script: |
                    sshpass -p "$(DEPLOY_PASSWORD)" ssh -o StrictHostKeyChecking=no "$(DEPLOY_USER)@$(TARGET_HOST)" \
                      "sudo systemctl restart $(SERVICE_NAME) && sudo systemctl status $(SERVICE_NAME) --no-pager"
                  displayName: Restart application service
```

Expected variables in `env-stage-deploy-vars`:
- `TARGET_HOST`
- `DEPLOY_USER`
- `DEPLOY_PASSWORD` (secret)
- `DEPLOY_PATH` (for example `/var/www/my-app`)
- `BACKUP_ROOT` (for example `/var/backups/my-app`)
- `SERVICE_NAME` (for example `my-app`)

Note: Replace `YourApp.csproj` with your actual project path.

### 7) Quick Tips

- Use variable groups + secret variables for sensitive values.
- Split reusable logic into YAML templates.
- Keep environment switch conditions explicit and readable.
