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
                    def serviceMap = serviceDirs()

                    // Store both full commit SHA and 8-character short SHA
                    env.GIT_SHA = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    env.SHORT_SHA = env.GIT_SHA.take(8)

                    // Set Docker image tag: use release tag name if triggered by a tag, otherwise fallback to short SHA
                    env.IMAGE_TAG = env.TAG_NAME?.trim()
                        ? env.TAG_NAME.trim()
                        : env.SHORT_SHA

                    echo "Commit: ${env.GIT_SHA} (${env.SHORT_SHA})"
                    echo "Image tag: ${env.IMAGE_TAG}"

                    def changedFiles = []

                    if (env.TAG_NAME?.trim()) {
                        // For release tags, all microservices must be built and released together
                        changedFiles = []
                        echo "Release tag detected: build all services."
                    } else if (env.CHANGE_ID) {
                        // Pull Request build: if Jenkins created a merge commit with target branch (HEAD^1), diff against HEAD^1; otherwise diff against origin/main
                        def isMergeCommit = sh(
                            script: 'git rev-parse --verify HEAD^2 >/dev/null 2>&1',
                            returnStatus: true
                        ) == 0

                        def diffCommand
                        if (isMergeCommit) {
                            diffCommand = 'git diff --name-only HEAD^1 HEAD'
                        } else {
                            sh 'git fetch origin +refs/heads/main:refs/remotes/origin/main'
                            diffCommand = 'git diff --name-only origin/main...HEAD'
                        }

                        def diffOutput = sh(
                            script: diffCommand,
                            returnStdout: true
                        ).trim()

                        changedFiles = diffOutput ? diffOutput.readLines().findAll { it.trim() } : []
                    } else {
                        // Branch push (main or dev/*): compare current commit (HEAD) with previous commit on the SAME branch
                        def hasParent = sh(
                            script: 'git rev-parse --verify HEAD^ >/dev/null 2>&1',
                            returnStatus: true
                        ) == 0

                        def prevCommit = env.GIT_PREVIOUS_COMMIT?.trim()
                        def usePrevCommitOnMain = false

                        // On main, if multiple commits were merged before Jenkins ran (coalesced build), include all commits since last build
                        if (env.BRANCH_NAME == 'main' && prevCommit && prevCommit != env.GIT_SHA) {
                            usePrevCommitOnMain = sh(
                                script: "git cat-file -e ${prevCommit}^{commit} >/dev/null 2>&1",
                                returnStatus: true
                            ) == 0
                        }

                        def diffCommand
                        if (usePrevCommitOnMain) {
                            echo "Comparing current commit (${env.SHORT_SHA}) with previous build commit (${prevCommit.take(8)}) on branch main"
                            diffCommand = "git diff --name-only ${prevCommit} HEAD"
                        } else if (hasParent) {
                            echo "Comparing current commit (${env.SHORT_SHA}) with previous commit (HEAD^) on branch ${env.BRANCH_NAME}"
                            diffCommand = 'git diff --name-only HEAD^ HEAD'
                        } else {
                            diffCommand = 'git diff-tree --root --no-commit-id --name-only -r HEAD'
                        }

                        def diffOutput = sh(
                            script: diffCommand,
                            returnStdout: true
                        ).trim()

                        changedFiles = diffOutput ? diffOutput.readLines().findAll { it.trim() } : []
                    }

                    echo "Changed files: ${changedFiles}"

                    // Global configuration and build files that impact all services or the CI/CD pipeline
                    def isGlobalFile = { path ->
                        path == 'pom.xml' ||
                        path.startsWith('.mvn/') ||
                        path == 'mvnw' ||
                        path == 'mvnw.cmd' ||
                        path == 'Dockerfile' ||
                        path == 'docker-compose.yml' ||
                        path.startsWith('docker/') ||
                        path.startsWith('.github/') ||
                        path.startsWith('scripts/')
                    }

                    // Documentation, pipeline definition, and IDE/repository meta files safe to ignore (no service build required)
                    def isDocumentation = { path ->
                        path == 'Jenkinsfile' ||
                        path == 'README.md' ||
                        path == 'LICENSE' ||
                        path == 'CONTRIBUTING.md' ||
                        path == '.gitignore' ||
                        path == '.gitattributes' ||
                        path == '.editorconfig' ||
                        path.startsWith('.devcontainer/') ||
                        path.startsWith('docs/')
                    }

                    def belongsToService = { path ->
                        serviceMap.values().any { directory ->
                            path == directory ||
                            path.startsWith("${directory}/")
                        }
                    }

                    // Strict validation: fail fast if any changed file is unclassified
                    def unknownFiles = changedFiles.findAll { path ->
                        !belongsToService(path) &&
                        !isGlobalFile(path) &&
                        !isDocumentation(path)
                    }

                    if (unknownFiles) {
                        error(
                            "Unrecognized changed paths: ${unknownFiles}. " +
                            "Please define an explicit handling rule."
                        )
                    }

                    def buildAll =
                        env.TAG_NAME?.trim() ||
                        changedFiles.any { isGlobalFile(it) }

                    def detectedServices

                    if (buildAll) {
                        detectedServices = serviceMap.keySet().toList()
                    } else {
                        detectedServices = serviceMap.findAll { service, directory ->
                            changedFiles.any { path ->
                                path == directory ||
                                path.startsWith("${directory}/")
                            }
                        }.keySet().toList()
                    }

                    // Export comma-separated list of affected services for downstream stages
                    env.DETECTED_SERVICES = detectedServices.join(',')

                    if (detectedServices.isEmpty()) {
                        echo 'No application services require a build.'
                    } else {
                        echo "Services selected: ${env.DETECTED_SERVICES}"
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
                    def serviceMap = serviceDirs()
                    def services = env.DETECTED_SERVICES.split(',')

                    services.each { service ->
                        def directory = serviceMap[service]

                        echo "Running unit tests for service: ${service}"

                        sh "./mvnw -pl ${directory} -am org.jacoco:jacoco-maven-plugin:prepare-agent test org.jacoco:jacoco-maven-plugin:report"

                        // Extract JaCoCo test coverage percentage directly via shell to keep Blue Ocean steps clean
                        def coverageText = sh(
                            script: """
                                if [ -f "${directory}/target/site/jacoco/jacoco.csv" ]; then
                                    awk -F, 'NR>1 {m+=\$4; c+=\$5} END {t=m+c; if(t>0) printf "%d%%", (c/t)*100; else print "0%"}' "${directory}/target/site/jacoco/jacoco.csv" 2>/dev/null || echo "N/A"
                                elif [ -f "${directory}/target/site/jacoco/index.html" ]; then
                                    cov=\$(grep -o -E 'Total</td><td class="bar">[^<]*</td><td class="ctr2">[0-9]+%' "${directory}/target/site/jacoco/index.html" 2>/dev/null | grep -o -E '[0-9]+%' | head -1)
                                    echo "\${cov:-N/A}"
                                else
                                    echo "N/A"
                                fi
                            """,
                            returnStdout: true
                        ).trim()

                        echo "[TEST COVERAGE] ${service}: ${coverageText}"
                    }
                }
            }
            post {
                always {
                    // Collect JUnit test results and display trend charts on the build overview
                    junit(
                        testResults: '**/target/surefire-reports/*.xml',
                        allowEmptyResults: true
                    )
 
                    // Record code coverage metrics and trend reports via JaCoCo plugin
                    jacoco(
                        execPattern: '**/target/jacoco.exec',
                        classPattern: '**/target/classes',
                        sourcePattern: '**/src/main/java'
                    )
 
                    // Archive JaCoCo HTML reports and XML coverage summary artifacts
                    archiveArtifacts(
                        artifacts: '**/target/site/jacoco/*.html, **/target/site/jacoco/jacoco.xml',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Static Analysis') {
            when {
                expression { return env.DETECTED_SERVICES?.trim() }
            }
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                script {
                    def serviceMap = serviceDirs()
                    def services = env.DETECTED_SERVICES.split(',')

                    services.each { service ->
                        def directory = serviceMap[service]

                        echo "Running static analysis for service: ${service}"

                        // Run SonarQube code quality scan (explicitly set projectKey to match dashboard URL)
                        withCredentials([
                            string(
                                credentialsId: 'sonarqube-token', 
                                variable: 'SONARQUBE_TOKEN'
                            )
                        ]) {
                            sh "./mvnw -f ${directory}/pom.xml org.sonarsource.scanner.maven:sonar-maven-plugin:5.5.0.6356:sonar -Dsonar.projectKey=${directory} -Dsonar.projectName=${directory} -Dsonar.token=\$SONARQUBE_TOKEN -Dsonar.host.url=${SONAR_HOST_URL}"
                        }

                        def sonarUrl = "${SONAR_HOST_URL}/dashboard?id=${directory}"
                        echo "[SONARQUBE] ${service}: ${sonarUrl}"

                        // Run OWASP Dependency-Check to identify known CVE vulnerabilities in dependencies
                        sh "./mvnw -B -ntp -f ${directory}/pom.xml org.owasp:dependency-check-maven:13.0.0:check -DautoUpdate=false -Dformat=ALL -DfailBuildOnCVSS=11"

                        // Extract OWASP vulnerability counts from JSON report
                        def owaspSummary = sh(
                            script: """
                                if [ -f "${directory}/target/dependency-check-report.json" ]; then
                                    count=\$(jq '[.dependencies[]?.vulnerabilities[]?] | length' "${directory}/target/dependency-check-report.json" 2>/dev/null || echo "0")
                                    echo "\${count} vulnerabilities"
                                else
                                    echo "Completed"
                                fi
                            """,
                            returnStdout: true
                        ).trim()

                        echo "[OWASP DEPENDENCY-CHECK] ${service}: ${owaspSummary}"
                    }
                }
            }
            post {
                always {
                    // Publish OWASP Dependency-Check report to Jenkins UI
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        dependencyCheckPublisher(
                            pattern: '**/target/dependency-check-report.xml'
                        )
                    }

                    // Archive OWASP dependency check security reports
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
                    def serviceMap = serviceDirs()
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

                            for i in 1 2 3; do
                                docker pull eclipse-temurin:17 && break || (echo "Retrying base image pull ($i/3)..." && sleep 3)
                            done
                        '''

                        services.each { service ->
                            def directory = serviceMap[service]

                            echo "Building Docker image for service: ${service}"

                            // Build the Docker image via Maven Spring Boot buildDocker profile (with retry on transient network/EOF errors)
                            sh """
                                for i in 1 2 3; do
                                    ./mvnw clean install -pl ${directory} -am -P buildDocker -Ddocker.image.prefix=${DOCKERHUB_USERNAME} && break || {
                                        if [ "\$i" -eq 3 ]; then exit 1; fi
                                        echo "Retrying Maven Docker build for ${service} (\$i/3)..."
                                        sleep 3
                                    }
                                done
                            """

                            // Define the Docker image repository and versioned tag
                            def imageName = "${DOCKERHUB_USERNAME}/${directory}"
                            def taggedImageName = "${imageName}:${env.IMAGE_TAG}"

                            // Tag the local image with the computed version tag
                            sh "docker tag ${imageName}:latest ${taggedImageName}"
                            
                            // Scan the Docker image for container vulnerabilities using Trivy
                            sh "trivy image --exit-code 0 --format json --output trivy-report-${service}.json ${taggedImageName}"

                            // Extract Trivy vulnerabilities summary (Critical and High)
                            def trivySummary = sh(
                                script: """
                                    if [ -f "trivy-report-${service}.json" ]; then
                                        crit=\$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' "trivy-report-${service}.json" 2>/dev/null || echo 0)
                                        high=\$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="HIGH")] | length' "trivy-report-${service}.json" 2>/dev/null || echo 0)
                                        echo "Critical: \${crit}, High: \${high}"
                                    else
                                        echo "Scanned"
                                    fi
                                """,
                                returnStdout: true
                            ).trim()

                            echo "[TRIVY CONTAINER SCAN] ${service} (${taggedImageName}): ${trivySummary}"

                            // Push the versioned Docker image to Docker Hub (skip for Pull Request builds)
                            if (!env.CHANGE_ID) {
                                echo "Pushing image: ${taggedImageName}"
                                sh """
                                    for i in 1 2 3; do
                                        docker push ${taggedImageName} && break || {
                                            if [ "\$i" -eq 3 ]; then exit 1; fi
                                            echo "Retrying docker push for ${taggedImageName} (\$i/3)..."
                                            sleep 3
                                        }
                                    done
                                """

                                // If building on main branch without a release tag, also publish as latest
                                if (env.BRANCH_NAME == 'main' && !env.TAG_NAME) {
                                    def latestImageName = "${imageName}:latest"
                                    echo "Pushing image: ${latestImageName}"
                                    sh """
                                        docker tag ${taggedImageName} ${latestImageName}
                                        for i in 1 2 3; do
                                            docker push ${latestImageName} && break || {
                                                if [ "\$i" -eq 3 ]; then exit 1; fi
                                                echo "Retrying docker push for ${latestImageName} (\$i/3)..."
                                                sleep 3
                                            }
                                        done
                                    """
                                }
                            } else {
                                echo "Pull Request build detected (CHANGE_ID=${env.CHANGE_ID}). Skipping Docker Hub push."
                            }
                        }
                    }
                }
            }
            post {
                always {
                    // Archive Trivy container vulnerability scan reports
                    archiveArtifacts(
                        artifacts: 'trivy-report-*.json',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Update GitOps') {
            when {
                allOf {
                    expression { return env.DETECTED_SERVICES?.trim() || env.TAG_NAME?.trim() }
                    anyOf {
                        branch 'main'
                        buildingTag()
                    }
                }
            }
            steps {
                script {
                    // Determine the target environment based on whether a release tag is present
                    def targetEnv = env.TAG_NAME?.trim() ? 'staging' : 'dev'

                    // Define the path to the Helm values.yaml file for the target environment
                    def targetFile = "envs/${targetEnv}/values.yaml"

                    echo "Updating deployment repository for environment: ${targetEnv}"

                    // Determine which services to update: all services for a release tag, or only detected services for a branch build
                    def servicesToUpdate = env.TAG_NAME?.trim() ? serviceDirs().keySet().toList() : env.DETECTED_SERVICES.split(',').toList()

                    // Initialize a list to hold messages about updated services for logging purposes
                    def servicesUpdatedMsg = []

                    // Clone the deployment repository and update the Helm values.yaml file with the new image tags for the affected services
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'github-source-creds',
                            usernameVariable: 'GITHUB_USERNAME',
                            passwordVariable: 'GITHUB_TOKEN'
                        )
                    ]) {
                        sh """
                            set -e
                            rm -rf deploy-repo
                            git clone https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/pTn-3001/petclinic-microservice-deploy.git deploy-repo
                            cd deploy-repo
                            mkdir -p envs/${targetEnv}
                            if [ ! -f "${targetFile}" ]; then
                                echo "services: {}" > "${targetFile}"
                            fi
                        """

                        // Modify the values.yaml file for each service to update, enabling the service and setting its Docker image tag
                        servicesToUpdate.each { svc ->
                            servicesUpdatedMsg.add("${svc}=${env.IMAGE_TAG}")
                            sh """
                                cd deploy-repo
                                python3 -c '
import yaml, os
file_path = "${targetFile}"
with open(file_path, "r") as f:
    data = yaml.safe_load(f) or {}
if not isinstance(data.get("services"), dict):
    data["services"] = {}
if not isinstance(data["services"].get("${svc}"), dict):
    data["services"]["${svc}"] = {}
data["services"]["${svc}"]["enabled"] = True
data["services"]["${svc}"]["tag"] = "${env.IMAGE_TAG}"
with open(file_path, "w") as f:
    yaml.dump(data, f, default_flow_style=False)
'
                            """
                        }

                        // Commit and push the changes to the deployment repository, handling potential merge conflicts with retries
                        sh """
                            cd deploy-repo

                            helm lint charts/petclinic -f ${targetFile}

                            if git diff --quiet ${targetFile}; then
                                echo "No changes detected in ${targetFile}. Skipping commit."
                                exit 0
                            fi

                            git config user.name "ptn-jenkins"
                            git config user.email "ptn-jenkins@petclinic.local"

                            git add ${targetFile}

                            COMMIT_MSG="ci(${targetEnv}): update ${servicesUpdatedMsg.join(' ')} source=${env.SHORT_SHA} build=#${env.BUILD_NUMBER} [skip ci]"
                            git commit -m "\$COMMIT_MSG"

                            for i in 1 2 3; do
                                git pull --rebase origin main && git push origin main && break || sleep 2
                            done

                            cd ..
                            rm -rf deploy-repo
                        """
                    }
                }
            }
        }
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