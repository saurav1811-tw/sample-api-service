pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  environment {
    APP_NAME = "sample-api-service"
    IMAGE_REGISTRY = "rmkanda"
    GIT_URL = "https://github.com/saurav1811-tw/sample-api-service.git"
  }
  stages {
    stage('Setup') {
      parallel {
        stage('Install Dependencies') {
          steps {
            container('maven') {
              sh './mvnw install -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
            }
          }
        }
        stage('Secrets Scan') {
          steps {
            container('trufflehog') {
              sh 'trufflehog --exclude_paths secrets-exclude.txt ${GIT_URL}'
            }
          }
        }
      }
    }
    stage('Build') {
      steps {
        container('maven') {
          sh './mvnw package -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
        }
      }
    }
    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh './mvnw test'
            }
          }
        }
        stage('OSS Checks') {
          steps {
              container('maven') {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                  sh './mvnw org.owasp:dependency-check-maven:check'
              }
            }
          }
          post{
            always{
              dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
            }
          }
        }
        stage('License Checker') {
          steps {
            container('licensefinder') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh '''#!/bin/bash --login
                    /bin/bash --login
                    rvm use default
                    gem install license_finder
                    license_finder
                  '''
              }
            }
          }
        }
        stage('SAST') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh './mvnw compile spotbugs:check'
              }
            }
          }
          post {
            always {
              recordIssues enabledForFailure: true, tool: spotBugs() 
            }
          }
        }
      }
    }
    stage('Package') {
      steps {
        container('docker-tools') {
          sh "docker build . -t ${APP_NAME}"
        }
      }
    }
    stage('Infra Scans') {
      parallel {
        stage('grype Container Scan') {
          steps {
            container('docker-tools') {
              sh "grype ${APP_NAME}"
            }
          }
        }
        stage('dockle Container Scan') {
          steps {
            container('docker-tools') {
              sh "dockle ${APP_NAME}"
            }
          }
        }
        stage('Kubesec Kubernetes YAML Scan') {
          steps {
            container('docker-tools') {
              sh "kubesec scan k8s.yaml || exit 0"
            }
          }
        }
      }
    }
    stage('Publish') {
      steps {
        container('docker-tools') {
          echo "Publishing docker image"
          // sh "docker push ${APP_NAME}"
        }
      }
    }
    stage('Deploy to Dev') {
      steps {
        container('docker-tools') {
          echo "Deploying the app"
          // sh "kubectl apply -f k8s.yaml"
        }
      }
    }
    stage('Promote to Prod') {
      steps {
        container('docker-tools') {
          echo "Promote to Prod"
        }
      }
    }
  }
}