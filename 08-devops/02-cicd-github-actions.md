# CI/CD with GitHub Actions — Spring Boot Workflow

## GitHub Actions Workflow

A YAML file in `.github/workflows/` defines your CI/CD pipeline. GitHub runs it on every push or PR.

## Step 1: Complete Workflow

```yaml
# .github/workflows/build.yml
name: Build and Deploy

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  JAVA_VERSION: '21'
  REGISTRY: ${{ secrets.ECR_REGISTRY }}
  IMAGE_NAME: product-service

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: Determine version
        id: version
        run: |
          VERSION="${{ github.sha }}"
          echo "version=${VERSION:0:8}" >> "$GITHUB_OUTPUT"

      - name: Build and test
        run: ./mvnw verify

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: target/surefire-reports/

  docker:
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name == 'push'

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        env:
          VERSION: ${{ needs.build.outputs.version }}
        run: |
          docker build -t $REGISTRY/$IMAGE_NAME:$VERSION \
                       -t $REGISTRY/$IMAGE_NAME:latest .
          docker push $REGISTRY/$IMAGE_NAME:$VERSION
          docker push $REGISTRY/$IMAGE_NAME:latest

  deploy-staging:
    needs: docker
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging

    steps:
      - name: Deploy to ECS staging
        env:
          VERSION: ${{ needs.build.outputs.version }}
        run: |
          aws ecs update-service \
            --cluster staging \
            --service product-service \
            --force-new-deployment

  deploy-production:
    needs: docker
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Deploy to ECS production
        env:
          VERSION: ${{ needs.build.outputs.version }}
        run: |
          aws ecs update-service \
            --cluster production \
            --service product-service \
            --force-new-deployment
```

## Step 2: Build Matrix (Test Multiple JDK Versions)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java: [17, 21]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: temurin
          cache: maven
      - run: ./mvnw verify
```

The matrix runs your test suite on both JDK 17 and 21 in parallel.

## Step 3: Caching Maven Dependencies

```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: temurin
    cache: maven
```

The `cache: maven` option caches `~/.m2/repository` between runs. First run downloads dependencies; subsequent runs use the cache. This cuts build time by 30-50%.

## Step 4: Secrets Configuration

Store these in your GitHub repository settings (Settings > Secrets and variables > Actions):

| Secret | Value |
|--------|-------|
| `AWS_ACCESS_KEY_ID` | Your AWS access key |
| `AWS_SECRET_ACCESS_KEY` | Your AWS secret key |
| `ECR_REGISTRY` | Your ECR registry URL |

Never put secrets in the workflow file.

## Key Points

- One YAML file handles build, test, Docker push, and deployment
- Use `environment` for production deployments — GitHub requires manual approval
- Cache Maven dependencies with `cache: maven` in setup-java
- Build matrix tests across multiple JDK versions in parallel
- `if` conditions control which jobs run per branch
