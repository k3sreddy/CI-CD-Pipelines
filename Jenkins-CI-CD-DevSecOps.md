# Jenkins CI/CD + DevSecOps Deployment Playbook

## 📦 Project Structure (Java API Application)

```
java-api-app/
├── Jenkinsfile
├── Dockerfile
├── pom.xml
├── src/
│   ├── main/
│   │   └── java/...         # Application code
│   └── test/
│       └── java/...         # Unit & integration tests
├── tests/
│   ├── api/
│   │   └── postman_collection.json
│   └── security/
│       └── owasp-zap-baseline.conf
├── scripts/
│   ├── generate-sbom.sh
│   ├── run-trivy.sh
│   └── run-anchore.sh
└── README.md
```

---

## 🧪 Jenkinsfile (CI/CD Pipeline)

```groovy
pipeline {
  agent any

  environment {
    REGISTRY = "registry.example.com/java-api"
    IMAGE_TAG = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        git 'https://github.com/example/java-api-app.git'
      }
    }

    stage('Build') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
    }

    stage('Unit Test') {
      steps {
        sh 'mvn test'
        junit 'target/surefire-reports/*.xml'
      }
    }

    stage('Code Quality - SonarQube') {
      steps {
        withSonarQubeEnv('SonarQube') {
          sh 'mvn sonar:sonar'
        }
      }
    }

    stage('Generate SBOM') {
      steps {
        sh './scripts/generate-sbom.sh'
        archiveArtifacts artifacts: 'sbom/*.json', fingerprint: true
      }
    }

    stage('Build Docker Image') {
      steps {
        sh 'docker build -t $REGISTRY:$IMAGE_TAG .'
      }
    }

    stage('Image Scan - Trivy') {
      steps {
        sh './scripts/run-trivy.sh $REGISTRY:$IMAGE_TAG'
      }
    }

    stage('Image Scan - Anchore/Snyk') {
      steps {
        sh './scripts/run-anchore.sh $REGISTRY:$IMAGE_TAG'
      }
    }

    stage('Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-registry-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
          sh "echo $PASS | docker login -u $USER --password-stdin $REGISTRY"
          sh 'docker push $REGISTRY:$IMAGE_TAG'
        }
      }
    }

    stage('API Tests') {
      steps {
        sh 'newman run tests/api/postman_collection.json --reporters cli,junit --reporter-junit-export target/api-test-results.xml'
        junit 'target/api-test-results.xml'
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
      publishHTML(target: [reportDir: 'target/site', reportFiles: 'index.html', reportName: 'Project Report'])
    }
  }
}
```

---

## 🐳 Dockerfile

```Dockerfile
FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
COPY target/java-api.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 🧰 SBOM Generation Script (`scripts/generate-sbom.sh`)

```bash
#!/bin/bash
mkdir -p sbom
syft . -o cyclonedx-json > sbom/sbom.json
```

---

## 🛡️ Trivy Scan (`scripts/run-trivy.sh`)

```bash
#!/bin/bash
image=$1
mkdir -p trivy-reports
trivy image --severity HIGH,CRITICAL --format json --output trivy-reports/${image##*/}.json $image
```

---

## 🔐 Anchore CLI Scan (`scripts/run-anchore.sh`)

```bash
#!/bin/bash
image=$1
anchore-cli image add $image
anchore-cli image wait $image
anchore-cli image vuln $image all --json > anchore-${image##*/}.json
```

---

## 🔍 OPA Gatekeeper Policies (example)

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: container-must-have-owner
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    labels: ["owner"]
```

---

## 👮 Falco Runtime Monitoring

- Deploy Falco DaemonSet in cluster:

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco
```

- Example Rule: Alert on unexpected binary execution

```yaml
- rule: Unexpected Binary Execution
  desc: Detect execution of untrusted binaries
  condition: spawned_process and proc.name in ("nc", "nmap", "curl")
  output: "Untrusted binary executed (user=%user.name command=%proc.cmdline)"
  priority: WARNING
```

---

## 📜 CI-Generated Compliance Artifacts

- **SonarQube**: Code smells, vulnerabilities, coverage
- **JUnit**: Unit + API test results
- **SBOM**: CycloneDX JSON artifacts
- **Trivy/Anchore**: Image CVE reports
- **Falco**: Runtime security events
- **OPA**: Admission audit logs

These artifacts should be archived per build and retained for minimum 6 years for CERT-IN.

---

## 📦 Optional GitOps with ArgoCD

- Sync Jenkins-deployed images to ArgoCD manifests using `kustomize edit set image`
- Use ArgoCD image updater plugin or webhook-based syncs

---
