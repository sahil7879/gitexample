pipeline {
    agent any
     tools {
        // already Defined the Maven tool to use from Jenkins as a plugin insted of directly installingon server
        maven 'mvn'  // mvn is what we named in tools configuration in Jenkins
    }

    stages {
        stage('Check sensitive git secrets exposure'){
          steps {
            sh 'rm -rf trufflehog || true'
            sh "docker run --rm -v \"$PWD:/pwd\" trufflesecurity/trufflehog:latest github --repo ${env.GITHUB_REPO} --only-verified --json > trufflehog"
            sh 'cat trufflehog'
            }
        }

        stage('Sonar Scan Static Analsis ') {
            steps {
              // make sure to create webhook from sonarqube to jenkins
              // add insallation name 
              withSonarQubeEnv(credentialsId: 'sonacred',installationName: 'sonar') {
              sh 'mvn clean package sonar:sonar -Dsonar.projectKey=test'
              }
            }
        }

        stage('Quality Gates') {
            steps {
              // Implement quality gates criteria timeout 
              timeout(time: 2, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
              }
            }  
        }

        stage ('Source Composition Analysis') {   
          // this stage will take around 20 min on the first run
          // dependency checker for finding vulnerbailities in project dependencies
          steps {
            script {
              def workspaceDir = env.WORKSPACE
              sh 'rm -rf owasp* || true'
              sh 'wget "https://raw.githubusercontent.com/ramannkhanna2/devops-maven-docker/master/owasp-dependency-check.sh" '
              sh 'chmod +x owasp-dependency-check.sh'
              sh 'bash owasp-dependency-check.sh'
              sh "cat $workspaceDir/odc-reports/dependency-check-report.xml"
            }
          }
        }

        stage('Build') {
            steps {
                // Build your project (e.g., Maven)
                sh 'mvn clean package'
            }
        }

        stage('removal') {
            steps {
                // removal of prev containers and images
                sh 'docker rm -f mynewcontainer'
                sh 'docker rmi `docker images`|| echo "Command 3 failed with exit code $?"'
            }
        }

        stage('Docker Build') {
            steps {
                // Build a Docker image
                sh 'docker build -t image-name:version .'
            }
        }

        stage('Security Scanning') {
            steps {
                // Scan the Docker image for security vulnerabilities (e.g., Trivy)
                sh 'trivy --no-progress --exit-code 1 --severity HIGH,CRITICAL image-name:version'
            }
        }

        stage('Deploy') {
            steps {
                // Deploy the Docker image to your environment (e.g., Kubernetes, Docker Swarm)
                sh 'docker run -d -p 8081:8080 --name mynewcontainer image-name:version'
            }
        }

       stage('DAST-Analysis') {
          steps {
            sh 'docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t http://13.231.149.29:8081/devops/ || echo "Command 3 failed with exit code $?"'
           }
          } 
    }
    
    post {
        success {
            // Actions to take on a successful pipeline run
            echo 'Pipeline completed successfully'
        }
        failure {
            // Actions to take on a failed pipeline run
            echo 'Pipeline failed'
        }
    }
}
