pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    parameters {
        choice(name: 'ENVIRONMENT',
            choices: ['dev', 'stage', 'uat', 'prod'],
            description: 'Environment configuration to update')

        choice(name: 'NODE',
            choices: ['dev', 'stage', 'uat', 'prod'],
            description: 'Node configuration to update')

        choice(name: 'CONFIG_TYPE',
            choices: ['both', 'environment', 'node'],
            description: 'Select configuration type')

        string(name: 'ENV_APPNAME', defaultValue: '',
            description: 'Application name (optional)')
        string(name: 'ENV_VERSION', defaultValue: '',
            description: 'New application version; required for environment updates')
        string(name: 'ENV_REPLICAS', defaultValue: '',
            description: 'Desired replica count (optional)')
        choice(name: 'ENV_LOGLEVEL',
            choices: ['', 'DEBUG', 'INFO', 'WARN', 'ERROR'],
            description: 'Application log level')

        string(name: 'NODE_NAME', defaultValue: '',
            description: 'Node name (optional)')
        string(name: 'NODE_TYPE', defaultValue: '',
            description: 'Node type, e.g. worker (optional)')
        string(name: 'NODE_REGION', defaultValue: '',
            description: 'AWS region (optional)')
        string(name: 'NODE_AZ', defaultValue: '',
            description: 'Availability zone (optional)')

        string(name: 'INSTANCE_TYPE', defaultValue: '',
            description: 'Required for node updates')
        string(name: 'K8S_VERSION', defaultValue: '',
            description: 'Optional Kubernetes version')
        string(name: 'CPU', defaultValue: '',
            description: 'Optional CPU value')
        string(name: 'MEMORY', defaultValue: '',
            description: 'Optional memory value')
    }

    environment {
        GIT_REPO = 'https://github.com/asifshaik6558/flipkart.git'
        GITHUB_REPO = 'asifshaik6558/flipkart'
        BASE_BRANCH = 'main'
        GIT_CREDENTIALS_ID = 'git'
        SONAR_SERVER = 'SonarQube-Server'
        SONAR_SCANNER = 'SonarQube-Scanner'
    }

    stages {
        stage('Checkout Fresh Main') {
            steps {
                deleteDir()

                git branch: 'main',
                    credentialsId: 'git',
                    url: 'https://github.com/asifshaik6558/flipkart.git'

                sh '''
                    set -eu
                    echo "===== Checked out revision ====="
                    git log -1 --oneline
                    echo "===== Configuration files ====="
                    find environment node -maxdepth 1 -type f -name '*.json' | sort
                '''
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    if (!(params.ENVIRONMENT in ['dev', 'stage', 'uat', 'prod'])) {
                        error('Invalid ENVIRONMENT')
                    }

                    if (!(params.NODE in ['dev', 'stage', 'uat', 'prod'])) {
                        error('Invalid NODE')
                    }

                    if (!(params.CONFIG_TYPE in ['both', 'environment', 'node'])) {
                        error('Invalid CONFIG_TYPE')
                    }

                    if (params.CONFIG_TYPE in ['environment', 'both']) {
                        if (!params.ENV_VERSION?.trim()) {
                            error('ENV_VERSION is required for environment updates')
                        }

                        if (params.ENV_REPLICAS?.trim() &&
                            !(params.ENV_REPLICAS.trim() ==~ /[1-9][0-9]*/)) {
                            error('ENV_REPLICAS must be a positive integer')
                        }
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        if (!params.INSTANCE_TYPE?.trim()) {
                            error('INSTANCE_TYPE is required for node updates')
                        }
                    }

                    echo 'Parameter validation passed'
                }
            }
        }

        stage('Read and Update JSON') {
            steps {
                script {
                    def targets = []

                    if (params.CONFIG_TYPE in ['environment', 'both']) {
                        targets.add([
                            type: 'environment',
                            path: "environment/${params.ENVIRONMENT}.json"
                        ])
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        targets.add([
                            type: 'node',
                            path: "node/${params.NODE}.json"
                        ])
                    }

                    targets.each { target ->
                        def path = target.path

                        if (!fileExists(path)) {
                            error("Missing configuration file: ${path}")
                        }

                        def config = readJSON(
                            file: path,
                            returnPojo: true
                        )

                        if (!(config instanceof Map)) {
                            error("Expected a JSON object in ${path}")
                        }

                        if (target.type == 'environment') {
                            def oldVersion = config.version?.toString()

                            if (params.ENV_VERSION.trim() == oldVersion) {
                                error("New version must differ from current version ${oldVersion}")
                            }

                            config.version = params.ENV_VERSION.trim()

                            if (params.ENV_APPNAME?.trim()) {
                                config.appName = params.ENV_APPNAME.trim()
                            }

                            if (params.ENV_REPLICAS?.trim()) {
                                config.replicas =
                                    params.ENV_REPLICAS.trim().toInteger()
                            }

                            if (params.ENV_LOGLEVEL?.trim()) {
                                config.logLevel = params.ENV_LOGLEVEL.trim()
                            }
                        }

                        if (target.type == 'node') {
                            config.instanceType = params.INSTANCE_TYPE.trim()

                            if (params.NODE_NAME?.trim()) {
                                config.nodeName = params.NODE_NAME.trim()
                            }

                            if (params.NODE_TYPE?.trim()) {
                                config.nodeType = params.NODE_TYPE.trim()
                            }

                            if (params.NODE_REGION?.trim()) {
                                config.region = params.NODE_REGION.trim()
                            }

                            if (params.NODE_AZ?.trim()) {
                                config.availabilityZone = params.NODE_AZ.trim()
                            }

                            if (params.K8S_VERSION?.trim()) {
                                if (!(config.kubernetes instanceof Map)) {
                                    config.kubernetes = [:]
                                }
                                config.kubernetes.version = params.K8S_VERSION.trim()
                            }

                            if (params.CPU?.trim() || params.MEMORY?.trim()) {
                                if (!(config.resources instanceof Map)) {
                                    config.resources = [:]
                                }

                                if (params.CPU?.trim()) {
                                    config.resources.cpu = params.CPU.trim()
                                }

                                if (params.MEMORY?.trim()) {
                                    config.resources.memory = params.MEMORY.trim()
                                }
                            }
                        }

                        writeJSON(file: path, json: config, pretty: 4)
                        echo "Updated ${path}"
                    }
                }
            }
        }

        stage('Verify JSON and Review Diff') {
            steps {
                script {
                    sh '''
                        set -eu
                        git diff --check
                        git diff -- environment node
                    '''

                    echo 'Review the diff in the build log before proceeding.'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    script {
                        def scannerHome = tool(
                            name: 'SonarQube-Scanner',
                            type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                        )

                        withEnv(["PATH+SONAR=${scannerHome}/bin"]) {
                            sh '''
                                set -eu
                                sonar-scanner \
                                  -Dsonar.projectKey=flipkart-config-automation \
                                  -Dsonar.projectName=Flipkart-Config-Automation \
                                  -Dsonar.sources=environment,node \
                                  -Dsonar.sourceEncoding=UTF-8
                            '''
                        }
                    }
                }
            }
        }

        stage('Maven Compile') {
            when {
                expression { fileExists('pom.xml') }
            }
            steps {
                sh 'mvn -B clean verify'
            }
        }

        stage('Create ZIP Artifact') {
            steps {
                script {
                    def filesToPackage = []

                    if (params.CONFIG_TYPE in ['environment', 'both']) {
                        filesToPackage.add("environment/${params.ENVIRONMENT}.json")
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        filesToPackage.add("node/${params.NODE}.json")
                    }

                    env.ARTIFACT_NAME =
                        "flipkart-config-${env.BUILD_NUMBER}.zip"

                    def fileArgs = filesToPackage.collect {
                        "'${it}'"
                    }.join(' ')

                    sh """
                        set -eu
                        zip '${env.ARTIFACT_NAME}' ${fileArgs}
                        unzip -l '${env.ARTIFACT_NAME}'
                    """

                    archiveArtifacts(
                        artifacts: env.ARTIFACT_NAME,
                        fingerprint: true
                    )
                }
            }
        }

        stage('Create Feature Branch and Commit') {
            steps {
                script {
                    env.FEATURE_BRANCH =
                        "feature/config-${params.ENVIRONMENT}-${params.NODE}-${env.BUILD_NUMBER}"

                    sh '''
                        set -eu
                        git config user.name "Asif Shaik"
                        git config user.email "asifshaik6558@gmail.com"

                        git switch -c "$FEATURE_BRANCH"

                        if [ "$CONFIG_TYPE" = "environment" ]; then
                            git add -- "environment/$ENVIRONMENT.json"
                        elif [ "$CONFIG_TYPE" = "node" ]; then
                            git add -- "node/$NODE.json"
                        else
                            git add -- "environment/$ENVIRONMENT.json" "node/$NODE.json"
                        fi

                        if git diff --cached --quiet; then
                            echo "No configuration changes to commit"
                            exit 1
                        fi

                        git diff --cached --check
                        git diff --cached --stat
                        git commit -m "Update configuration for $ENVIRONMENT"
                    '''
                }
            }
        }

        stage('Push Feature Branch') {
            steps {
                withCredentials([
                    gitUsernamePassword(
                        credentialsId: 'git',
                        gitToolName: 'Default'
                    )
                ]) {
                    sh '''
                        set -eu
                        git push -u origin "$FEATURE_BRANCH"
                    '''
                }
            }
        }

        stage('Create or Reuse Pull Request') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'git',
                        usernameVariable: 'GITHUB_USER',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        python3 - <<'PY'
import json
import os
import urllib.parse
import urllib.request

repo = os.environ["GITHUB_REPO"]
branch = os.environ["FEATURE_BRANCH"]
base = os.environ["BASE_BRANCH"]
token = os.environ["GITHUB_TOKEN"]

headers = {
    "Authorization": f"Bearer {token}",
    "Accept": "application/vnd.github+json",
    "X-GitHub-Api-Version": "2022-11-28"
}

query = urllib.parse.urlencode({
    "state": "open",
    "head": f"{repo.split('/')[0]}:{branch}",
    "base": base
})

url = f"https://api.github.com/repos/{repo}/pulls?{query}"
request = urllib.request.Request(url, headers=headers)

with urllib.request.urlopen(request, timeout=30) as response:
    pulls = json.load(response)

if pulls:
    print("Reusing open pull request:", pulls[0]["html_url"])
else:
    payload = {
        "title": f"Configuration update: {branch}",
        "head": branch,
        "base": base,
        "body": "Automated configuration update created by Jenkins."
    }

    request = urllib.request.Request(
        f"https://api.github.com/repos/{repo}/pulls",
        data=json.dumps(payload).encode(),
        headers={**headers, "Content-Type": "application/json"},
        method="POST"
    )

    with urllib.request.urlopen(request, timeout=30) as response:
        result = json.load(response)
        print("Pull request created:", result["html_url"])
PY
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Configuration pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the failed stage and console output.'
        }

        always {
            echo "Build result: ${currentBuild.currentResult}"
        }
    }
}
