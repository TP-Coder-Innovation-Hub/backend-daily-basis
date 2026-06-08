# CI/CD with Jenkins — Jenkinsfile for Spring Boot

## Jenkins Pipeline

A Jenkinsfile defines your build pipeline as code. It lives in your repository alongside your application code.

## When Jenkins vs GitHub Actions

| Jenkins | GitHub Actions |
|---------|---------------|
| Self-hosted, full control | Managed, zero infrastructure |
| Complex enterprise pipelines | Simple YAML workflows |
| Existing Jenkins infrastructure | New projects, open source |
| Custom plugins and integrations | Native GitHub integration |

## Step 1: Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        IMAGE_NAME = 'product-service'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    tools {
        jdk 'JDK-21'
        maven 'Maven-3.9'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Integration Test') {
            steps {
                sh 'mvn verify -P integration-test'
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar',
                    fingerprint: true
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                        ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-registry',
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASS')]) {
                    sh """
                        echo ${REGISTRY_PASS} | docker login \
                            ${DOCKER_REGISTRY} -u ${REGISTRY_USER} \
                            --password-stdin
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${DOCKER_REGISTRY}/${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            when { branch 'develop' }
            steps {
                sh """
                    kubectl set image deployment/${IMAGE_NAME} \
                        ${IMAGE_NAME}=${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                        -n staging
                """
            }
        }

        stage('Deploy to Production') {
            when { branch 'main' }
            steps {
                input message: 'Deploy to production?',
                    ok: 'Deploy'
                sh """
                    kubectl set image deployment/${IMAGE_NAME} \
                        ${IMAGE_NAME}=${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                        -n production
                """
            }
        }
    }

    post {
        failure {
            mail to: 'team@example.com',
                subject: "Build ${env.JOB_NAME} #${env.BUILD_NUMBER} failed",
                body: "Check console output: ${env.BUILD_URL}"
        }
        success {
            slackSend color: 'good',
                message: "Build ${env.JOB_NAME} #${env.BUILD_NUMBER} succeeded"
        }
    }
}
```

## Step 2: Multibranch Pipeline

Create a multibranch pipeline job in Jenkins:

- It scans your repository for branches with a Jenkinsfile
- Each branch gets its own pipeline
- PR branches are built and tested automatically
- `main` deploys to production, `develop` deploys to staging

## Step 3: Dockerfile for Spring Boot

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Multi-stage build: the JDK is only in the build stage. The final image uses the smaller JRE.

## Key Points

- Jenkinsfile lives in your repo — pipeline changes go through code review
- Multibranch pipelines give each branch its own build
- Use `when` conditions to control which stages run per branch
- Manual approval (`input` step) for production deployments
- Always run tests before packaging — failing fast saves time
