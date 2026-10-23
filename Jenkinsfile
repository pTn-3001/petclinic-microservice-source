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

// Render a rich HTML dashboard summary onto the Jenkins build status page and echo to console log
def updateBuildDashboard() {
    def servicesList = (env.DETECTED_SERVICES ?: '').split(',').findAll { it.trim() }
    if (!servicesList) {
        currentBuild.description = "<b>No services selected for build.</b>"
        echo "[BUILD DASHBOARD] No application services require a build."
        return
    }

    def stripHtml = { str -> str ? str.replaceAll('<[^>]*>', '').trim() : '' }

    def rows = ''
    def consoleRows = ''

    servicesList.each { svc ->
        def cov = fileExists("target/.metrics/${svc}-coverage.txt") ? readFile("target/.metrics/${svc}-coverage.txt").trim() : '<i>Pending...</i>'
        def sonar = fileExists("target/.metrics/${svc}-sonar.txt") ? readFile("target/.metrics/${svc}-sonar.txt").trim() : '<i>Pending...</i>'
        def owasp = fileExists("target/.metrics/${svc}-owasp.txt") ? readFile("target/.metrics/${svc}-owasp.txt").trim() : '<i>Pending...</i>'
        def trivy = fileExists("target/.metrics/${svc}-trivy.txt") ? readFile("target/.metrics/${svc}-trivy.txt").trim() : '<i>Pending...</i>'

        rows += """
        <tr>
            <td style="padding: 8px 12px; font-weight: bold; border: 1px solid #dee2e6;">${svc}</td>
            <td style="padding: 8px 12px; border: 1px solid #dee2e6;">${cov}</td>
            <td style="padding: 8px 12px; border: 1px solid #dee2e6;">${sonar}</td>
            <td style="padding: 8px 12px; border: 1px solid #dee2e6;">${owasp}</td>
            <td style="padding: 8px 12px; border: 1px solid #dee2e6;">${trivy}</td>
        </tr>"""

        def covText = stripHtml(cov)
        def sonarText = fileExists("target/.metrics/${svc}-sonar.txt") ? 'Configured' : 'Pending'
        def owaspText = stripHtml(owasp)
        def trivyText = stripHtml(trivy)

        consoleRows += String.format("  %-22s | %-12s | %-14s | %-20s | %-18s\n", svc, covText, sonarText, owaspText, trivyText)
    }

    currentBuild.description = """
    <div style="font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif; font-size: 13px; margin: 10px 0;">
        <div style="margin-bottom: 8px;">
            <b>Branch:</b> <code>${env.BRANCH_NAME ?: 'N/A'}</code> | 
            <b>Commit:</b> <code>${env.SHORT_SHA ?: 'N/A'}</code> | 
            <b>Tag:</b> <code>${env.IMAGE_TAG ?: 'N/A'}</code>
        </div>
        <table style="border-collapse: collapse; width: 100%; max-width: 850px; background: #fff; box-shadow: 0 1px 3px rgba(0,0,0,0.1);">
            <thead>
                <tr style="background-color: #f1f3f5; color: #495057; text-align: left;">
                    <th style="padding: 8px 12px; border: 1px solid #dee2e6;">Service</th>
                    <th style="padding: 8px 12px; border: 1px solid #dee2e6;">🧪 Test Coverage</th>
                    <th style="padding: 8px 12px; border: 1px solid #dee2e6;">🔍 SonarQube</th>
                    <th style="padding: 8px 12px; border: 1px solid #dee2e6;">🛡️ OWASP CVEs</th>
                    <th style="padding: 8px 12px; border: 1px solid #dee2e6;">🐳 Trivy Scan</th>
                </tr>
            </thead>
            <tbody>
                ${rows}
            </tbody>
        </table>
    </div>"""

    echo """
========================================================================================================
 [BUILD DASHBOARD] Metrics Summary (Branch: ${env.BRANCH_NAME ?: 'N/A'} | Tag: ${env.IMAGE_TAG ?: 'N/A'})
--------------------------------------------------------------------------------------------------------
  Service                | Coverage     | SonarQube      | OWASP CVEs           | Trivy Scan        
--------------------------------------------------------------------------------------------------------
${consoleRows}========================================================================================================
"""
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
                    } else if (env.BRANCH_NAME == 'main') {
                        // Recent commit on main: verify whether a parent commit exists to safely handle root commits
                        def hasParent = sh(
                            script: 'git rev-parse --verify HEAD^ >/dev/null 2>&1',
                            returnStatus: true
                        ) == 0

                        def diffCommand = hasParent
                            ? 'git diff --name-only HEAD^ HEAD'
                            : 'git diff-tree --root --no-commit-id --name-only -r HEAD'

                        def diffOutput = sh(
                            script: diffCommand,
                            returnStdout: true
                        ).trim()

                        changedFiles = diffOutput ? diffOutput.readLines().findAll { it.trim() } : []
                    } else {
                        // Feature branch or PR: compare against the merge-base with origin/main
                        sh 'git fetch origin main'

                        def diffOutput = sh(
                            script: 'git diff --name-only origin/main...HEAD',
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
                        path == 'Jenkinsfile' ||
                        path == 'Dockerfile' ||
                        path == 'docker-compose.yml' ||
                        path.startsWith('docker/') ||
                        path.startsWith('.github/') ||
                        path.startsWith('scripts/')
                    }

                    // Documentation and IDE/repository meta files safe to ignore (no build required)
                    def isDocumentation = { path ->
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
                        sh 'rm -rf target/.metrics && mkdir -p target/.metrics'
                        updateBuildDashboard()
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

                        // Extract JaCoCo test coverage percentage
                        def coverageFile = "${directory}/target/site/jacoco/index.html"
                        def coverageText = 'N/A'

                        if (fileExists(coverageFile)) {
                            def coverageHtml = sh(
                                script: """
                                    grep -o -E 'Total</td><td class="bar">[^<]*</td><td>[0-9]+%' "${coverageFile}" 2>/dev/null | grep -o -E '[0-9]+%' | head -1 || echo ""
                                """,
                                returnStdout: true
                            ).trim()

                            if (coverageHtml) {
                                def coverage = coverageHtml.replace('%', '').toFloat() / 100
                                coverageText = "${Math.round(coverage * 100)}%"
                            }
                        } else if (fileExists("${directory}/target/site/jacoco/jacoco.csv")) {
                            coverageText = sh(
                                script: """
                                    awk -F, 'NR>1 {m+=\$4; c+=\$5} END {t=m+c; if(t>0) printf "%d%%", (c/t)*100; else print "N/A"}' "${directory}/target/site/jacoco/jacoco.csv" 2>/dev/null || echo "N/A"
                                """,
                                returnStdout: true
                            ).trim()
                        }

                        echo ">> [TEST COVERAGE] Service: ${service} -> ${coverageText}"
                        writeFile file: "target/.metrics/${service}-coverage.txt", text: coverageText
                    }

                    // Refresh Jenkins UI dashboard with test coverage metrics
                    updateBuildDashboard()
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
                        sourcePattern: '**/src/main/java',
                        allowEmptyResults: true
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

                        // Run SonarQube code quality scan
                        withCredentials([
                            string(
                                credentialsId: 'sonarqube-token', 
                                variable: 'SONARQUBE_TOKEN'
                            )
                        ]) {
                            sh "./mvnw -f ${directory}/pom.xml org.sonarsource.scanner.maven:sonar-maven-plugin:5.5.0.6356:sonar -Dsonar.token=\$SONARQUBE_TOKEN -Dsonar.host.url=${SONAR_HOST_URL}"
                        }

                        // Record SonarQube dashboard link for Jenkins UI
                        def sonarUrl = "${SONAR_HOST_URL}/dashboard?id=${directory}"
                        writeFile file: "target/.metrics/${service}-sonar.txt", text: "<a href='${sonarUrl}' target='_blank' style='color:#1a73e8;text-decoration:none;'>Dashboard ↗</a>"
                        echo ">> [SONARQUBE] Service: ${service} -> ${sonarUrl}"

                        // Run OWASP Dependency-Check to identify known CVE vulnerabilities in dependencies
                        sh "./mvnw -B -ntp -f ${directory}/pom.xml org.owasp:dependency-check-maven:13.0.0:check -DautoUpdate=false -Dformat=ALL -DfailBuildOnCVSS=11"

                        // Extract OWASP vulnerability counts from JSON report
                        def owaspJson = "${directory}/target/dependency-check-report.json"
                        def owaspText = '0 issues'
                        def rawOwasp = '0 vulnerabilities'

                        if (fileExists(owaspJson)) {
                            def vulnCount = sh(
                                script: """
                                    jq '[.dependencies[]?.vulnerabilities[]?] | length' "${owaspJson}" 2>/dev/null || echo "0"
                                """,
                                returnStdout: true
                            ).trim()

                            rawOwasp = "${vulnCount} vulnerabilities"
                            if (vulnCount && vulnCount != '0') {
                                owaspText = "<span style='color:#d93025;font-weight:bold;'>${vulnCount} vulnerabilities</span>"
                            } else {
                                owaspText = "<span style='color:#188038;'>0 vulnerabilities</span>"
                            }
                        } else {
                            owaspText = "Completed"
                            rawOwasp = "Completed"
                        }

                        echo ">> [OWASP DEPENDENCY-CHECK] Service: ${service} -> ${rawOwasp}"
                        writeFile file: "target/.metrics/${service}-owasp.txt", text: owaspText
                    }

                    // Refresh Jenkins UI dashboard with SonarQube and OWASP findings
                    updateBuildDashboard()
                }
            }
            post {
                always {
                    // Publish OWASP Dependency-Check report to Jenkins UI
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        dependencyCheckPublisher(
                            pattern: '**/target/dependency-check-report.xml',
                            allowEmptyResults: true
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
                        '''

                        services.each { service ->
                            def directory = serviceMap[service]

                            echo "Building Docker image for service: ${service}"

                            // Build the Docker image via Maven Spring Boot buildDocker profile
                            sh "./mvnw clean install -pl ${directory} -am -P buildDocker -Ddocker.image.prefix=${DOCKERHUB_USERNAME}"

                            // Define the Docker image repository and versioned tag
                            def imageName = "${DOCKERHUB_USERNAME}/${directory}"
                            def taggedImageName = "${imageName}:${env.IMAGE_TAG}"

                            echo "Pushing image: ${taggedImageName}"

                            // Tag the local image with the computed version tag
                            sh "docker tag ${imageName}:latest ${taggedImageName}"
                            
                            // Scan the Docker image for container vulnerabilities using Trivy
                            sh "trivy image --exit-code 0 --format json --output trivy-report-${service}.json ${taggedImageName}"

                            // Extract Trivy vulnerabilities summary (Critical and High)
                            def trivyJson = "trivy-report-${service}.json"
                            def trivyText = 'Scanned'

                            def rawTrivy = 'Scanned'
                            if (fileExists(trivyJson)) {
                                def crit = sh(
                                    script: "jq '[.Results[]?.Vulnerabilities[]? | select(.Severity==\"CRITICAL\")] | length' ${trivyJson} 2>/dev/null || echo 0",
                                    returnStdout: true
                                ).trim()
                                def high = sh(
                                    script: "jq '[.Results[]?.Vulnerabilities[]? | select(.Severity==\"HIGH\")] | length' ${trivyJson} 2>/dev/null || echo 0",
                                    returnStdout: true
                                ).trim()

                                rawTrivy = "Critical: ${crit}, High: ${high}"
                                if (crit != '0' || high != '0') {
                                    trivyText = "<span style='color:#d93025;font-weight:bold;'>Crit: ${crit}, High: ${high}</span>"
                                } else {
                                    trivyText = "<span style='color:#188038;'>0 Crit, 0 High</span>"
                                }
                            }

                            echo ">> [TRIVY CONTAINER SCAN] Service: ${service} -> ${rawTrivy}"
                            writeFile file: "target/.metrics/${service}-trivy.txt", text: trivyText

                            // Push the versioned Docker image to Docker Hub (skip for Pull Request builds)
                            if (!env.CHANGE_ID) {
                                sh "docker push ${taggedImageName}"

                                // If building on main branch without a release tag, also publish as latest
                                if (env.BRANCH_NAME == 'main' && !env.TAG_NAME) {
                                    def latestImageName = "${imageName}:latest"
                                    echo "Pushing image: ${latestImageName}"
                                    sh "docker tag ${taggedImageName} ${latestImageName}"
                                    sh "docker push ${latestImageName}"
                                }
                            } else {
                                echo "Pull Request build detected (CHANGE_ID=${env.CHANGE_ID}). Skipping Docker Hub push."
                            }
                        }
                    }

                    // Refresh Jenkins UI dashboard with Trivy scan results
                    updateBuildDashboard()
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

        stage('Update Deployment Repository') {
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
                                echo "services:" > "${targetFile}"
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
if "services" not in data:
    data["services"] = {}
if "${svc}" not in data["services"]:
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
            script {
                updateBuildDashboard()
            }
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