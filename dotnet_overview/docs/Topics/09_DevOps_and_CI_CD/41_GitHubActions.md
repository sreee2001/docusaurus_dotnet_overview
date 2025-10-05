## 41. GitHub Actions

### Short Introduction

GitHub Actions is a CI/CD platform integrated directly into GitHub repositories, allowing developers to automate software workflows including building, testing, and deploying applications. It uses YAML-based workflow files and provides a marketplace of pre-built actions for common development tasks.

### Official Definition

GitHub Actions is a continuous integration and continuous delivery (CI/CD) platform that allows you to automate your build, test, and deployment pipeline. You can create workflows that build and test every pull request to your repository, or deploy merged pull requests to production.

### Setup/Usage with .NET 8+ Code

**Basic .NET Workflow:**

```yaml
# .github/workflows/dotnet.yml
name: .NET Core CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: "8.0.x"
  AZURE_WEBAPP_NAME: "hotel-management-api"
  AZURE_WEBAPP_PACKAGE_PATH: "./published"

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v3
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: dotnet restore

      - name: Build application
        run: dotnet build --no-restore --configuration Release

      - name: Run unit tests
        run: dotnet test --no-build --configuration Release --verbosity normal --collect:"XPlat Code Coverage" --results-directory ./coverage

      - name: Upload coverage reports
        uses: codecov/codecov-action@v3
        with:
          directory: ./coverage
          flags: unittests
          name: codecov-umbrella

      - name: Publish application
        run: dotnet publish --no-build --configuration Release --output ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}

      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: webapp
          path: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}

  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production

    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: webapp
          path: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}

      - name: Deploy to Azure Web App
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: ${{ env.AZURE_WEBAPP_PACKAGE_PATH }}
```

**Advanced Multi-Environment Workflow:**

```yaml
# .github/workflows/advanced-pipeline.yml
name: Advanced CI/CD Pipeline

on:
  push:
    branches: [main, develop, "release/**"]
  pull_request:
    branches: [main, develop]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  DOTNET_VERSION: "8.0.x"

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: "fs"
          scan-ref: "."
          format: "sarif"
          output: "trivy-results.sarif"

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: "trivy-results.sarif"

  build:
    runs-on: ubuntu-latest
    needs: security-scan
    outputs:
      version: ${{ steps.version.outputs.version }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Generate version
        id: version
        run: |
          if [[ "$GITHUB_REF" == "refs/heads/main" ]]; then
            VERSION="1.0.$GITHUB_RUN_NUMBER"
          elif [[ "$GITHUB_REF" == "refs/heads/develop" ]]; then
            VERSION="1.0.$GITHUB_RUN_NUMBER-dev"
          else
            VERSION="1.0.$GITHUB_RUN_NUMBER-${{ github.sha }}"
          fi
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "Generated version: $VERSION"

      - name: Restore dependencies
        run: dotnet restore

      - name: Build application
        run: dotnet build --no-restore --configuration Release -p:Version=${{ steps.version.outputs.version }}

      - name: Run tests with coverage
        run: |
          dotnet test --no-build --configuration Release \
            --collect:"XPlat Code Coverage" \
            --results-directory ./TestResults/ \
            --logger "trx;LogFileName=test-results.trx" \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

      - name: Generate test report
        uses: dorny/test-reporter@v1
        if: success() || failure()
        with:
          name: .NET Tests
          path: "./TestResults/*.trx"
          reporter: dotnet-trx

      - name: Publish test coverage
        uses: codecov/codecov-action@v3
        with:
          directory: ./TestResults
          flags: unittests

      - name: Build Docker image
        run: |
          docker build . --file Dockerfile \
            --tag ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.version }} \
            --tag ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest \
            --build-arg VERSION=${{ steps.version.outputs.version }}

      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push Docker image
        run: |
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.version }}
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: development

    steps:
      - name: Deploy to Development
        run: |
          echo "Deploying version ${{ needs.build.outputs.version }} to development"
          # Add actual deployment steps here

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/heads/release/')
    environment: staging

    steps:
      - name: Deploy to Staging
        run: |
          echo "Deploying version ${{ needs.build.outputs.version }} to staging"
          # Add actual deployment steps here

  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Deploy to Production
        uses: azure/webapps-deploy@v2
        with:
          app-name: "hotel-management-api-prod"
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE_PROD }}
          images: "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build.outputs.version }}"

      - name: Create GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ needs.build.outputs.version }}
          release_name: Release v${{ needs.build.outputs.version }}
          body: |
            Automated release of version ${{ needs.build.outputs.version }}

            Changes in this release:
            ${{ github.event.head_commit.message }}
          draft: false
          prerelease: false
```

### Use Cases

- **Open Source Projects**: Free CI/CD for public repositories
- **GitHub-Centric Development**: Teams already using GitHub for source control
- **Simple to Medium Complexity Pipelines**: Straightforward build, test, deploy workflows
- **Container-Based Deployments**: Excellent Docker and container registry integrations
- **Community-Driven Solutions**: Large marketplace of community actions
- **Cost-Effective CI/CD**: Generous free tier for both public and private repos

### When to Use vs When Not to Use

**Use GitHub Actions when:**

- Source code is hosted on GitHub
- Need cost-effective CI/CD solution
- Want integrated pull request workflows
- Building containerized applications
- Working with open source projects
- Need simple, maintainable workflows

**Consider alternatives when:**

- Using other source control systems (GitLab, Azure Repos)
- Need advanced enterprise features
- Require on-premises CI/CD solutions
- Building extremely complex deployment pipelines
- Need tighter integration with Microsoft ecosystem

### Market Alternatives & Pros/Cons

**Alternatives:**

- **Azure DevOps**: More enterprise features, Microsoft integration
- **GitLab CI/CD**: Integrated with GitLab, comprehensive DevOps platform
- **CircleCI**: Fast builds, good Docker support
- **Jenkins**: Self-hosted, highly customizable
- **Travis CI**: Simple setup, good for open source
- **AWS CodePipeline**: Native AWS integration

**Pros:**

- Deeply integrated with GitHub
- Large marketplace of community actions
- Free for public repositories, generous free tier for private
- Simple YAML syntax
- Excellent container and cloud integrations
- Built-in security scanning capabilities

**Cons:**

- Limited to GitHub ecosystem
- Can become expensive for large private repositories
- Less enterprise features compared to dedicated platforms
- Runner limitations for compute-intensive tasks
- Fewer advanced deployment patterns

### Complete Runnable Sample

**Production-Ready Workflow with Multiple Environments:**

```yaml
# .github/workflows/production-pipeline.yml
name: Production Pipeline

on:
  push:
    branches: [ main, develop ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main, develop ]

env:
  DOTNET_VERSION: '8.0.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: hotel-management-api
  AZURE_RESOURCE_GROUP: rg-hotel-management

jobs:
  validate:
    name: Code Quality & Security
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}

    - name: Restore dependencies
      run: dotnet restore

    - name: Format check
      run: dotnet format --verify-no-changes --verbosity diagnostic

    - name: Lint with EditorConfig
      run: dotnet format --verify-no-changes --include-generated

    - name: Security scan with Snyk
      uses: snyk/actions/dotnet@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high

    - name: OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'hotel-management'
        path: '.'
        format: 'ALL'

    - name: Upload OWASP results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: reports/dependency-check-report.sarif

  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest
    needs: validate

    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}

    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: ~/.nuget/packages
        key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}

    - name: Restore
      run: dotnet restore

    - name: Build
      run: dotnet build --no-restore --configuration Release

    - name: Test
      run: |
        dotnet test --no-build --configuration Release \
          --collect:"XPlat Code Coverage" \
          --results-directory ./coverage \
          --logger "trx;LogFileName=test-results.trx" \
          --logger "html;LogFileName=test-results.html" \
          -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

    - name: Upload test results
      uses: dorny/test-reporter@v1
      if: success() || failure()
      with:
        name: Test Results (${{ matrix.os }})
        path: './coverage/*.trx'
        reporter: dotnet-trx

    - name: Code Coverage Summary
      uses: irongut/CodeCoverageSummary@v1.3.0
      with:
        filename: ./coverage/**/coverage.opencover.xml
        badge: true
        format: markdown
        output: both

    - name: Add Coverage PR Comment
      uses: actions/github-script@v6
      if: github.event_name == 'pull_request'
      with:
        script: |
          const fs = require('fs');
          const coverage = fs.readFileSync('code-coverage-results.md', 'utf8');
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: coverage
          });

  docker-build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: build-and-test
    if: github.event_name != 'pull_request'

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2

    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ github.repository }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=sha,prefix={{branch}}-

    - name: Build and push
      id: build
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        platforms: linux/amd64,linux/arm64

    - name: Sign container image
      uses: sigstore/cosign-installer@v3
      with:
        cosign-release: 'v2.1.1'
    - run: |
        cosign sign --yes ${{ env.REGISTRY }}/${{ github.repository }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}

  deploy-dev:
    name: Deploy to Development
    needs: docker-build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment:
      name: development
      url: https://hotel-management-dev.azurewebsites.net

    steps:
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS_DEV }}

    - name: Deploy to Container App
      uses: azure/container-apps-deploy-action@v1
      with:
        appSourcePath: ${{ github.workspace }}
        acrName: hotelmanagementacr
        containerAppName: hotel-management-dev
        resourceGroup: ${{ env.AZURE_RESOURCE_GROUP }}-dev
        imageToDeploy: ${{ needs.docker-build.outputs.image-tag }}

  deploy-prod:
    name: Deploy to Production
    needs: docker-build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://hotel-management.azurewebsites.net

    steps:
    - name: Azure Login
      uses: azure/login@v1
      with:
awr
        creds: ${{ secrets.AZURE_CREDENTIALS_PROD }}

    - name: Deploy to Container App
      uses: azure/container-apps-deploy-action@v1
      with:
        appSourcePath: ${{ github.workspace }}
        acrName: hotelmanagementacr
        containerAppName: hotel-management-prod
        resourceGroup: ${{ env.AZURE_RESOURCE_GROUP }}-prod
        imageToDeploy: ${{ needs.docker-build.outputs.image-tag }}

    - name: Create Release
      uses: actions/create-release@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ github.run_number }}
        release_name: Release ${{ github.run_number }}
        body: |
          🚀 Production deployment successful!

          **Image:** `${{ needs.docker-build.outputs.image-tag }}`
          **Digest:** `${{ needs.docker-build.outputs.image-digest }}`

          **Changes:**
          ${{ github.event.head_commit.message }}
        draft: false
        prerelease: false
```
