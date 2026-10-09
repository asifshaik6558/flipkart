pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stage', 'uat', 'prod'],
            description: 'Select environment'
        )

        choice(
            name: 'CONFIG_TYPE',
            choices: ['both', 'environment', 'node'],
            description: 'Select configuration type'
        )

        string(
            name: 'INSTANCE_TYPE',
            defaultValue: '',
            description: 'Mandatory for node updates'
        )

        string(
            name: 'K8S_VERSION',
            defaultValue: '',
            description: 'Optional: Kubernetes version'
        )

        string(
            name: 'CPU',
            defaultValue: '',
            description: 'Optional: CPU value'
        )

        string(
            name: 'MEMORY',
            defaultValue: '',
            description: 'Optional: Memory value'
        )
    }

    environment {
        GIT_REPO = 'https://github.com/asifshaik6558/flipkart.git'
        GITHUB_REPO = 'asifshaik6558/flipkart'
        BASE_BRANCH = 'main'
        GIT_CREDENTIALS_ID = 'git'
    }

    stages {

        stage('Checkout GitHub Repository') {
            steps {
                git branch: "${BASE_BRANCH}",
                    credentialsId: "${GIT_CREDENTIALS_ID}",
                    url: "${GIT_REPO}"
            }
        }

        stage('Display Parameters') {
            steps {
                echo "Environment: ${params.ENVIRONMENT}"
                echo "Config Type: ${params.CONFIG_TYPE}"
                echo "Instance Type: ${params.INSTANCE_TYPE}"
                echo "Kubernetes Version: ${params.K8S_VERSION}"
                echo "CPU: ${params.CPU}"
                echo "Memory: ${params.MEMORY}"
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    if (!params.ENVIRONMENT?.trim()) {
                        error('ENVIRONMENT is mandatory')
                    }

                    if (!params.CONFIG_TYPE?.trim()) {
                        error('CONFIG_TYPE is mandatory')
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        if (!params.INSTANCE_TYPE?.trim()) {
                            error(
                                'INSTANCE_TYPE is mandatory for node updates'
                            )
                        }
                    }

                    echo 'All mandatory parameters validated successfully'
                }
            }
        }

        stage('Read Configuration Files') {
            steps {
                script {
                    def types = params.CONFIG_TYPE == 'both'
                        ? ['environment', 'node']
                        : [params.CONFIG_TYPE]

                    types.each { type ->
                        def path = "${type}/${params.ENVIRONMENT}.json"

                        if (!fileExists(path)) {
                            error("Configuration file missing: ${path}")
                        }

                        def config = readJSON(
                            file: path,
                            returnPojo: true
                        )

                        if (!(config instanceof Map)) {
                            error("Expected a JSON object in ${path}")
                        }

                        echo "Validated JSON file: ${path}"
                    }
                }
            }
        }

        stage('Update JSON Configuration') {
            steps {
                script {
                    def types = params.CONFIG_TYPE == 'both'
                        ? ['environment', 'node']
                        : [params.CONFIG_TYPE]

                    types.each { type ->
                        def path = "${type}/${params.ENVIRONMENT}.json"

                        def config = readJSON(
                            file: path,
                            returnPojo: true
                        )

                        if (type == 'node') {
                            config.instanceType =
                                params.INSTANCE_TYPE.trim()

                            if (params.K8S_VERSION?.trim()) {
                                if (!(config.kubernetes instanceof Map)) {
                                    config.kubernetes = [:]
                                }

                                config.kubernetes.version =
                                    params.K8S_VERSION.trim()
                            }

                            if (params.CPU?.trim()) {
                                if (!(config.resources instanceof Map)) {
                                    config.resources = [:]
                                }

                                config.resources.cpu = params.CPU.trim()
                            }

                            if (params.MEMORY?.trim()) {
                                if (!(config.resources instanceof Map)) {
                                    config.resources = [:]
                                }

                                config.resources.memory =
                                    params.MEMORY.trim()
                            }
                        }

                        if (type == 'environment') {
                            echo "Environment configuration selected: ${path}"
                        }

                        writeJSON(
                            file: path,
                            json: config,
                            pretty: 4
                        )

                        echo "Updated local configuration: ${path}"
                    }
                }
            }
        }

        stage('Verify Updated JSON') {
            steps {
                script {
                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        def path =
                            "node/${params.ENVIRONMENT}.json"

                        def config = readJSON(
                            file: path,
                            returnPojo: true
                        )

                        echo "Updated Instance Type: ${config.instanceType}"
                        echo "Updated Kubernetes Version: ${config.kubernetes?.version}"
                        echo "Updated CPU: ${config.resources?.cpu}"
                        echo "Updated Memory: ${config.resources?.memory}"
                    }

                    if (params.CONFIG_TYPE in ['environment', 'both']) {
                        def path =
                            "environment/${params.ENVIRONMENT}.json"

                        def config = readJSON(
                            file: path,
                            returnPojo: true
                        )

                        echo "Verified environment configuration: ${path}"
                    }
                }
            }
        }

        stage('Create Git Feature Branch') {
            steps {
                script {
                    def branchName =
                        "feature/update-${params.ENVIRONMENT}-${env.BUILD_NUMBER}"

                    env.FEATURE_BRANCH = branchName

                    sh '''
                        set -eu

                        git config user.name "Asif Shaik"
                        git config user.email "asifshaik6558@gmail.com"

                        git switch -c "$FEATURE_BRANCH"

                        git add environment/*.json node/*.json

                        if git diff --cached --quiet; then
                            echo "No configuration changes detected"
                            exit 1
                        fi

                        git commit -m "Update ${ENVIRONMENT} configuration"
                    '''

                    echo "Feature branch prepared: ${branchName}"
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
import urllib.request
import urllib.error

repo = "asifshaik6558/flipkart"
branch = os.environ["FEATURE_BRANCH"]
token = os.environ["GITHUB_TOKEN"]
base = os.environ["BASE_BRANCH"]

payload = {
    "title": f"Update configuration: {branch}",
    "head": branch,
    "base": base,
    "body": "Automated configuration update by Jenkins."
}

request = urllib.request.Request(
    f"https://api.github.com/repos/{repo}/pulls",
    data=json.dumps(payload).encode(),
    headers={
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json",
        "X-GitHub-Api-Version": "2022-11-28",
        "Content-Type": "application/json"
    },
    method="POST"
)

try:
    with urllib.request.urlopen(request, timeout=30) as response:
        result = json.load(response)
        print("Pull Request created:", result["html_url"])
except urllib.error.HTTPError as error:
    print("GitHub API returned HTTP", error.code)
    print(error.read().decode())
    raise
PY
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Configuration update and Pull Request workflow completed.'
        }

        failure {
            echo 'Pipeline failed. Review the stage logs to identify the issue.'
        }

        always {
            echo "Build result: ${currentBuild.currentResult}"
        }
    }
}
