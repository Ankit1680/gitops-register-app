pipeline {
    agent any

    tools {
        jdk 'java17'
        maven 'maven3'
    }

    parameters {
        string(name: 'REPO_NAME', description: 'Name of the GitHub repository')
        string(name: 'REPO_URL', description: 'GitHub repository URL')
        string(name: 'ORG_NAME', description: 'GitHub organization or user name')
        string(name: 'LANGUAGE', description: 'Primary programming language')
        string(name: 'RELEASE', defaultValue: '1.0.0', description: 'Release version')
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "6thelement"
        DOCKER_PASS = 'docker-cred'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }

    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from SCM') {
            steps {
                git branch: 'main', credentialsId: 'github-token-botu', url: "${params.REPO_URL}"
            }
        }

        stage('Build Application') {
            when {
                expression {
                    params.LANGUAGE.toLowerCase() == 'java'
                }
            }
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test Application') {
            when {
                expression {
                    params.LANGUAGE.toLowerCase() == 'java'
                }
            }
            steps {
                sh 'mvn test'
            }
        }

   
        // stage("SonarQube Analysis") {
        //     steps {
        //         script {
        //             withSonarQubeEnv(credentialsId: 'sonar-token') {
        //                 sh '''
        //                 mvn sonar:sonar \
        //                     -Dsonar.projectKey=botu-springboot-app \
        //                     -Dsonar.projectName="botu-springboot-app" \
        //                     -Dsonar.projectVersion=1.0 \
        //                     -Dsonar.java.binaries=target/classes \
        //                     -Dsonar.surefire.reportsPath=target/surefire-reports \
        //                     -Dsonar.junit.reportPaths=target/surefire-reports
        //                 '''
        //             }
        //         }
        //     }
        // }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image = docker.build("${IMAGE_NAME}")
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        
        // stage("Trivy Scan") {
        //     steps {
        //         script {
        //             sh '''
        //             docker run -v /var/run/docker.sock:/var/run/docker.sock \
        //             aquasec/trivy image ${IMAGE_NAME}:latest --no-progress \
        //             --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table
        //             '''
        //         }
        //     }
        // }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh """
                    curl -v -k --user clouduser:${JENKINS_API_TOKEN} \
                    -X POST \
                    -H 'cache-control: no-cache' \
                    -H 'content-type: application/x-www-form-urlencoded' \
                    --data 'IMAGE_TAG=${IMAGE_TAG}' \
                    'http://jenkins.usebotu.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'
                    """
                }
            }
        }

        stage('Post-Build Info') {
            steps {
                echo "Repo: ${params.REPO_URL}"
                echo "Org: ${params.ORG_NAME}"
                echo "Tech Stack: ${params.LANGUAGE}"
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER

                sh """
                echo "Uploading Jenkins log file to S3..."

                LOG_FILE="/var/lib/jenkins/jobs/${jobName}/builds/${buildNumber}/log"
                S3_BUCKET="s3://jenkinslogs-bucket"

                echo "Looking for log: \$LOG_FILE"

                if [ -f "\$LOG_FILE" ]; then
                    aws s3 cp "\$LOG_FILE" "\$S3_BUCKET/${jobName}/${buildNumber}/log" --only-show-errors
                    echo "Uploaded log to S3: \$S3_BUCKET/${jobName}/${buildNumber}/log"
                else
                    echo "Jenkins log file not found: \$LOG_FILE"
                fi
                """
            }
        }

        success {
            echo "Dispatcher succeeded for ${APP_NAME}"
        }

        failure {
            echo "Dispatcher failed for ${APP_NAME}"
        }
    }
}
