// Define a mapping of service names to their corresponding directories in the repository
def serviceDirs() {
    return [
        'admin-server': 'spring-petclinic-admin-server',
        'api-gateway': 'spring-petclinic-api-gateway',
        'config-server': 'spring-petclinic-config-server',
        'discovery-server': 'spring-petclinic-discovery-server',
        'vets-service': 'spring-petclinic-vets-service',
        'visits-service': 'spring-petclinic-visits-service',
        'customers-service': 'spring-petclinic-customers-service',
        'genai-service': 'spring-petclinic-genai-service'
    ]
}

pipeline {
    agent {
        label 'built-in'
    }

    environment {
        SONAR_HOST_URL = 'http://localhost:9000'
    }

    stages {
        stage('Detect Changes') {
            steps {
                script {
                    // ==============================
                    // 1. Get the short SHA of the current commit
                    // ==============================
                    env.SHORT_SHA = sh(
                        script: 'git rev-parse --short=8 HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Current commit SHA: ${env.SHORT_SHA}"

                    // ==============================
                    // 2. Determine the image tag based on the build trigger
                    // ==============================
                    if (env.TAG_NAME) {
                        // If the build is triggered by a tag, use the tag name as the image tag
                        env.IMAGE_TAG = env.TAG_NAME
                    } else {
                        // If the build is triggered by a branch, use the short SHA as the image tag
                        env.IMAGE_TAG = env.SHORT_SHA
                    }

                    echo "Image tag: ${env.IMAGE_TAG}"

                    // ==============================
                    // 3. Detect changed services based on the modified files in the repository
                    // ==============================

                    def serviceDirs = serviceDirs()
                    // Get the list of changed files between the last two commits\
                    def changedFiles = sh(
                        script: 'git diff --name-only HEAD~1 HEAD',
                        returnStdout: true
                    ).trim()
                    def filesList = changedFiles ? changedFiles.split('\n').toList() : []
                    echo "Changed files: ${filesList}"

                    // Check which services have changed based on the modified files
                    def detectedServices = []
                    serviceDirs.each { serviceName, serviceDir -> 
                        if (filesList.any { it.startsWith(serviceDir) }) {
                            detectedServices.add(serviceName)
                        }
                    }

                    // Set environment variables based on the detected services
                    if (detectedServices.isEmpty()) {
                        env.DETECTED_SERVICES = ''
                        echo "No services have changed."
                    } else {
                        env.DETECTED_SERVICES = detectedServices.join(',')
                        echo "Detected changed services: ${detectedServices.join(', ')}"
                    }
                }
            }
        }

        stage('Test') {
            when {
                expression { return env.DETECTED_SERVICES?.trim() }
            }
            steps {
                script {
                    def serviceDirs = serviceDirs()
                    def services = env.DETECTED_SERVICES.split(',')

                    services.each { service ->
                        def directory = serviceDirs[service]

                        echo "Running unit tests for service: ${service}"

                        sh "./mvnw -pl ${directory} -am org.jacoco:jacoco-maven-plugin:prepare-agent test org.jacoco:jacoco-maven-plugin:report"
                    }
                }
            }
            post {
                always {
                    junit(
                        testResults: '**/target/surefire-reports/*.xml',
                        allowEmptyResults: true
                    )
                    
                    archiveArtifacts(
                        artifacts: '**/target/site/jacoco/*.html',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Static Analysis') {
            when {
                expression { return env.DETECTED_SERVICES?.trim() }
            }
            steps {
                script {
                    def serviceDirs = serviceDirs()
                    def services = env.DETECTED_SERVICES.split(',')

                    services.each { service ->
                        def directory = serviceDirs[service]

                        echo "Running static analysis for service: ${service}"

                        withCredentials([
                            string(
                                credentialsId: 'sonarqube-token', 
                                variable: 'SONARQUBE_TOKEN'
                                )]) {
                            sh "./mvnw -f ${directory}/pom.xml sonar:sonar -Dsonar.login=\$SONARQUBE_TOKEN -Dsonar.host.url=${SONAR_HOST_URL}"
                        }

                        sh "./mvnw -f ${directory}/pom.xml org.owasp:dependency-check-maven:13.0.0:check -Dformat=HTML,XML -DfailBuildOnCVSS=11"
                    }
                }
            }
            post {
                always {
                    archiveArtifacts(
                        artifacts: '**/target/dependency-check-report.*',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Build & Push') {
            when {
                expression { return env.DETECTED_SERVICES?.trim() }
            }
            steps {
                script {
                    def serviceDirs = serviceDirs()
                    def services = env.DETECTED_SERVICES.split(',')

                    echo "Building only the changed services: ${services.join(', ')}"
                    echo "Current branch: ${env.BRANCH_NAME}"

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-creds',
                            usernameVariable: 'DOCKERHUB_USERNAME',
                            passwordVariable: 'DOCKERHUB_PASSWORD'
                        )
                    ]) {
                        sh '''
                            echo "$DOCKERHUB_PASSWORD" |
                                docker login \
                                -u "$DOCKERHUB_USERNAME" \
                                --password-stdin
                        '''

                        services.each { service ->
                            def directory = serviceDirs[service]

                            echo "Building Docker image for service: ${service}"

                            // Build the Docker image
                            sh "./mvnw clean install -pl ${directory} -am -P buildDocker -Ddocker.image.prefix=${DOCKERHUB_USERNAME}"

                            // Define the image name
                            def imageName = "${DOCKERHUB_USERNAME}/${directory}"
                            def taggedImageName = "${imageName}:${env.IMAGE_TAG}"

                            echo "Pushing image: ${taggedImageName}"

                            // Create tags for the Docker image
                            sh "docker tag ${imageName}:latest ${taggedImageName}"
                            
                            // Scan the Docker image for vulnerabilities using Trivy
                            sh "trivy image --exit-code 0 --format json --output trivy-report-${service}.json ${taggedImageName}"

                            // Push the Docker image to Docker Hub
                            sh "docker push ${taggedImageName}"


                            if (env.BRANCH_NAME == 'main' && !env.TAG_NAME) {
                                def latestImageName = "${imageName}:latest"
                                echo "Pushing image: ${latestImageName}"
                                sh "docker tag ${taggedImageName} ${latestImageName}"
                                sh "docker push ${latestImageName}"
                            }
                        }
                    }
                }
            }
            post {
                always {
                    archiveArtifacts(
                        artifacts: 'trivy-report-*.json',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        // stage('Deploy') {
        //     steps {
                
        //     }
        // }
    }

    post {
        always {
            echo "Build #${BUILD_NUMBER} finished."
        }
        
        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed.'
        }
    }
}