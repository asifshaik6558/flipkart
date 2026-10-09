pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stage', 'uat', 'prod'],
            description: 'Environment to update'
        )

        choice(
            name: 'CONFIG_TYPE',
            choices: ['both', 'environment', 'node'],
            description: 'Which configuration to update'
        )

        string(
            name: 'INSTANCE_TYPE',
            defaultValue: '',
            description: 'Required for node updates'
        )

        string(
            name: 'K8S_VERSION',
            defaultValue: '',
            description: 'Optional Kubernetes version'
        )

        string(
            name: 'CPU',
            defaultValue: '',
            description: 'Optional CPU value'
        )

        string(
            name: 'MEMORY',
            defaultValue: '',
            description: 'Optional memory value'
        )

        string(
            name: 'ENV_APPNAME',
            defaultValue: '',
            description: 'Optional application name'
        )

        string(
            name: 'ENV_VERSION',
            defaultValue: '',
            description: 'Optional new application version'
        )

        string(
            name: 'ENV_REPLICAS',
            defaultValue: '',
            description: 'Optional replica count'
        )

        string(
            name: 'ENV_LOGLEVEL',
            defaultValue: '',
            description: 'Optional log level: DEBUG, INFO, WARN, ERROR'
        )

        string(
            name: 'NODE_NAME',
            defaultValue: '',
            description: 'Optional node name'
        )

        string(
            name: 'NODE_TYPE',
            defaultValue: '',
            description: 'Optional node type'
        )

        string(
            name: 'NODE_REGION',
            defaultValue: '',
            description: 'Optional AWS region'
        )

        string(
            name: 'NODE_AZ',
            defaultValue: '',
            description: 'Optional availability zone'
        )
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
        stage('Checkout GitHub Repository') {
            steps {
                deleteDir()

                git branch: 'main',
                    credentialsId: 'git',
                    url: 'https://github.com/asifshaik6558/flipkart.git'

                sh '''
                    set -eu
                    echo "===== Current commit ====="
                    git log -1 --oneline
                    echo "===== Repository files ====="
                    find environment node -maxdepth 1 \
                        -type f -name '*.json' | sort
                '''
            }
        }

        stage('Display Parameters') {
            steps {
                echo "Environment: ${params.ENVIRONMENT}"
                echo "Configuration type: ${params.CONFIG_TYPE}"
                echo "Instance type: ${params.INSTANCE_TYPE}"
                echo "Kubernetes version: ${params.K8S_VERSION}"
                echo "CPU: ${params.CPU}"
                echo "Memory: ${params.MEMORY}"
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    if (!(params.ENVIRONMENT in
                        ['dev', 'stage', 'uat', 'prod'])) {
                        error('Invalid environment selected')
                    }

                    if (!(params.CONFIG_TYPE in
                        ['both', 'environment', 'node'])) {
                        error('Invalid configuration type')
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        if (!params.INSTANCE_TYPE?.trim()) {
                            error(
                                'INSTANCE_TYPE is required for node updates'
                            )
                        }
                    }

                    if (params.ENV_REPLICAS?.trim()) {
                        if (!(params.ENV_REPLICAS.trim() ==~
                            /[1-9][0-9]*/)) {
                            error(
                                'ENV_REPLICAS must be a positive integer'
                            )
                        }
                    }

                    if (params.ENV_LOGLEVEL?.trim()) {
                        if (!(params.ENV_LOGLEVEL.trim() in
                            ['DEBUG', 'INFO', 'WARN', 'ERROR'])) {
                            error('Invalid ENV_LOGLEVEL')
                        }
                    }

                    echo 'Parameter validation passed'
                }
            }
        }

        stage('Read Configuration Files') {
            steps {
                script {
                    if (params.CONFIG_TYPE in
                        ['environment', 'both']) {
                        def envFile =
                            "environment/${params.ENVIRONMENT}.json"

                        if (!fileExists(envFile)) {
                            error("Missing file: ${envFile}")
                        }

                        def envConfig = readJSON(
                            file: envFile,
                            returnPojo: true
                        )

                        if (!(envConfig instanceof Map)) {
                            error("Invalid JSON object: ${envFile}")
                        }

                        echo "Validated JSON file: ${envFile}"
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        def nodeFile =
                            "node/${params.ENVIRONMENT}.json"

                        if (!fileExists(nodeFile)) {
                            error("Missing file: ${nodeFile}")
                        }

                        def nodeConfig = readJSON(
                            file: nodeFile,
                            returnPojo: true
                        )

                        if (!(nodeConfig instanceof Map)) {
                            error("Invalid JSON object: ${nodeFile}")
                        }

                        echo "Validated JSON file: ${nodeFile}"
                    }
                }
            }
        }

        stage('Update JSON Configuration') {
            steps {
                script {
                    if (params.CONFIG_TYPE in
                        ['environment', 'both']) {
                        def envFile =
                            "environment/${params.ENVIRONMENT}.json"

                        def config = readJSON(
                            file: envFile,
                            returnPojo: true
                        )

                        if (params.ENV_VERSION?.trim()) {
                            if (params.ENV_VERSION.trim() ==
                                config.version?.toString()) {
                                error(
                                    'New version must differ from current version'
                                )
                            }

                            config.version = params.ENV_VERSION.trim()
                        }

                        if (params.ENV_APPNAME?.trim()) {
                            config.appName = params.ENV_APPNAME.trim()
                        }

                        if (params.ENV_REPLICAS?.trim()) {
                            config.replicas =
                                params.ENV_REPLICAS.trim().toInteger()
                        }

                        if (params.ENV_LOGLEVEL?.trim()) {
                            config.logLevel =
                                params.ENV_LOGLEVEL.trim()
                        }

                        writeJSON(
                            file: envFile,
                            json: config,
                            pretty: 4
                        )

                        echo "Updated local configuration: ${envFile}"
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        def nodeFile =
                            "node/${params.ENVIRONMENT}.json"

                        def config = readJSON(
                            file: nodeFile,
                            returnPojo: true
                        )

                        config.instanceType =
                            params.INSTANCE_TYPE.trim()

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
                            config.availabilityZone =
                                params.NODE_AZ.trim()
                        }

                        if (params.K8S_VERSION?.trim()) {
                            if (!(config.kubernetes instanceof Map)) {
                                config.kubernetes = [:]
                            }

                            config.kubernetes.version =
                                params.K8S_VERSION.trim()
                        }

                        if (params.CPU?.trim() ||
                            params.MEMORY?.trim()) {
                            if (!(config.resources instanceof Map)) {
                                config.resources = [:]
                            }

                            if (params.CPU?.trim()) {
                                config.resources.cpu =
                                    params.CPU.trim()
                            }

                            if (params.MEMORY?.trim()) {
                                config.resources.memory =
                                    params.MEMORY.trim()
                            }
                        }

                        writeJSON(
                            file: nodeFile,
                            json: config,
                            pretty: 4
                        )

                        echo "Updated local configuration: ${nodeFile}"
                    }
                }
            }
        }

        stage('Verify Updated JSON') {
            steps {
                script {
                    sh 'git diff --check'

                    if (params.CONFIG_TYPE in
                        ['environment', 'both']) {
                        def envFile =
                            "environment/${params.ENVIRONMENT}.json"

                        def config = readJSON(
                            file: envFile,
                            returnPojo: true
                        )

                        echo "Verified environment file: ${envFile}"
                        echo "Application: ${config.appName}"
                        echo "Version: ${config.version}"
                        echo "Replicas: ${config.replicas}"
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        def nodeFile =
                            "node/${params.ENVIRONMENT}.json"

                        def config = readJSON(
                            file: nodeFile,
                            returnPojo: true
                        )

                        echo "Verified node file: ${nodeFile}"
                        echo "Instance type: ${config.instanceType}"
                        echo "Node name: ${config.nodeName}"
                        echo "Kubernetes: ${config.kubernetes?.version}"
                        echo "CPU: ${config.resources?.cpu}"
                        echo "Memory: ${config.resources?.memory}"
                    }

                    sh '''
                        echo "===== Selected configuration diff ====="
                        git diff -- environment node
                    '''
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool(
                        name: 'SonarQube-Scanner',
                        type: 'hudson.plugins.sonar.SonarRunnerInstallation'
                    )

                    withSonarQubeEnv('SonarQube-Server') {
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

        stage('Maven Compilation') {
            when {
                expression {
                    fileExists('pom.xml')
                }
            }
            steps {
                sh 'mvn -B clean verify'
            }
        }

        stage('Create ZIP Artifact') {
            steps {
                script {
                    def filesToPackage = []

                    if (params.CONFIG_TYPE in
                        ['environment', 'both']) {
                        filesToPackage.add(
                            "environment/${params.ENVIRONMENT}.json"
                        )
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        filesToPackage.add(
                            "node/${params.ENVIRONMENT}.json"
                        )
                    }

                    env.ARTIFACT_NAME =
                        "flipkart-config-${env.BUILD_NUMBER}.zip"

                    env.PACKAGE_FILES = filesToPackage.join(',')

                    sh '''
                        set -eu

                        python3 - <<'PY'
import os
import zipfile

artifact = os.environ["ARTIFACT_NAME"]
files = os.environ["PACKAGE_FILES"].split(",")

with zipfile.ZipFile(
    artifact, "w", compression=zipfile.ZIP_DEFLATED
) as archive:
    for path in files:
        if not os.path.isfile(path):
            raise FileNotFoundError(path)
        archive.write(path)

print("Created artifact:", artifact)
print("Included files:")
with zipfile.ZipFile(artifact) as archive:
    for name in archive.namelist():
        print(" -", name)
PY
                    '''

                    archiveArtifacts(
                        artifacts: env.ARTIFACT_NAME,
                        fingerprint: true
                    )
                }
            }
        }

        stage('Create Git Feature Branch') {
            steps {
                script {
                    env.FEATURE_BRANCH =
                        "feature/update-${params.ENVIRONMENT}-${env.BUILD_NUMBER}"

                    sh '''
                        set -eu

                        git config user.name "Asif Shaik"
                        git config user.email "asifshaik6558@gmail.com"

                        git switch -c "$FEATURE_BRANCH"

                        if [ "$CONFIG_TYPE" = "environment" ]; then
                            git add -- \
                                "environment/$ENVIRONMENT.json"
                        elif [ "$CONFIG_TYPE" = "node" ]; then
                            git add -- \
                                "node/$ENVIRONMENT.json"
                        else
                            git add -- \
                                "environment/$ENVIRONMENT.json" \
                                "node/$ENVIRONMENT.json"
                        fi

                        if git diff --cached --quiet; then
                            echo "No configuration changes to commit."
                            exit 1
                        fi

                        git diff --cached --check
                        git diff --cached --stat

                        git commit \
                            -m "Update $ENVIRONMENT configuration"
                    '''

                    echo "Prepared branch: ${env.FEATURE_BRANCH}"
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

        stage('Create GitHub Pull Request') {
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
owner = repo.split("/")[0]

headers = {
    "Authorization": f"Bearer {token}",
    "Accept": "application/vnd.github+json",
    "X-GitHub-Api-Version": "2022-11-28"
}

query = urllib.parse.urlencode({
    "state": "open",
    "head": f"{owner}:{branch}",
    "base": base
})

request = urllib.request.Request(
    f"https://api.github.com/repos/{repo}/pulls?{query}",
    headers=headers
)

with urllib.request.urlopen(request, timeout=30) as response:
    pulls = json.load(response)

if pulls:
    print("Existing pull request:", pulls[0]["html_url"])
else:
    payload = {
        "title": f"Configuration update: {branch}",
        "head": branch,
        "base": base,
        "body": "Automated configuration update created by Jenkins."
    }

    request = urllib.request.Request(
        f"https://api.github.com/repos/{repo}/pulls",
        data=json.dumps(payload).encode("utf-8"),
        headers={
            **headers,
            "Content-Type": "application/json"
        },
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
            echo 'Pipeline failed. Check the failed stage in Console Output.'
        }

        always {
            echo "Final build result: ${currentBuild.currentResult}"
        }
    }
}
