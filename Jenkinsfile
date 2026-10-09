pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stage', 'uat', 'prod'],
            description: 'Select the target environment'
        )

        choice(
            name: 'CONFIG_TYPE',
            choices: ['both', 'environment', 'node'],
            description: 'Select which configuration to update'
        )

        string(
            name: 'INSTANCE_TYPE',
            defaultValue: 'm7i-flex.large',
            description: 'Node instance type'
        )

        string(
            name: 'K8S_VERSION',
            defaultValue: '',
            description: 'Optional Kubernetes version'
        )

        string(
            name: 'CPU',
            defaultValue: '',
            description: 'Optional CPU resource value'
        )

        string(
            name: 'MEMORY',
            defaultValue: '',
            description: 'Optional memory resource value'
        )

        string(
            name: 'ENV_APPNAME',
            defaultValue: 'Flipkart',
            description: 'Environment application name'
        )

        string(
            name: 'ENV_VERSION',
            defaultValue: '1.0.0',
            description: 'Application version'
        )

        string(
            name: 'ENV_REPLICAS',
            defaultValue: '2',
            description: 'Application replica count'
        )

        string(
            name: 'ENV_LOGLEVEL',
            defaultValue: 'INFO',
            description: 'Application log level'
        )

        string(
            name: 'NODE_NAME',
            defaultValue: 'node-01',
            description: 'Node name'
        )

        string(
            name: 'NODE_TYPE',
            defaultValue: 'worker',
            description: 'Node type'
        )

        string(
            name: 'NODE_REGION',
            defaultValue: 'us-east-1',
            description: 'Node region'
        )

        string(
            name: 'NODE_AZ',
            defaultValue: 'us-east-1a',
            description: 'Node availability zone'
        )
    }

    environment {
        GITHUB_REPO = 'asifshaik6558/flipkart'
        BASE_BRANCH = 'main'

        NEXUS_REPOSITORY_URL =
            'http://localhost:8081/repository/flipkart-config-releases'

        CONFIG_CHANGES = 'false'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/asifshaik6558/flipkart.git',
                        credentialsId: 'git'
                    ]]
                ])

                sh '''
                    set -eu
                    echo "Checked out repository:"
                    git log -1 --oneline
                '''
            }
        }

        stage('Validate Parameters') {
            steps {
                script {
                    if (!(params.ENVIRONMENT in
                        ['dev', 'stage', 'uat', 'prod'])) {
                        error('Invalid ENVIRONMENT parameter.')
                    }

                    if (!(params.CONFIG_TYPE in
                        ['both', 'environment', 'node'])) {
                        error('Invalid CONFIG_TYPE parameter.')
                    }

                    if (params.CONFIG_TYPE in ['both', 'node'] &&
                        !params.INSTANCE_TYPE?.trim()) {
                        error(
                            'INSTANCE_TYPE is required for node or both.'
                        )
                    }

                    if (params.ENV_REPLICAS?.trim() &&
                        !(params.ENV_REPLICAS ==~ /[1-9][0-9]*/)) {
                        error('ENV_REPLICAS must be a positive integer.')
                    }
                }

                sh '''
                    set -eu
                    python3 - <<'PY'
import json
import os

env_name = os.environ["ENVIRONMENT"]
config_type = os.environ["CONFIG_TYPE"]

files = []

if config_type in ("environment", "both"):
    files.append(f"environment/{env_name}.json")

if config_type in ("node", "both"):
    files.append(f"node/{env_name}.json")

for path in files:
    if not os.path.isfile(path):
        raise SystemExit(f"Required configuration file not found: {path}")

    with open(path, encoding="utf-8") as f:
        json.load(f)

    print(f"Valid JSON: {path}")
PY
                '''
            }
        }

        stage('Update Configuration') {
            steps {
                sh '''
                    set -eu

                    python3 - <<'PY'
import json
import os

env_name = os.environ["ENVIRONMENT"]
config_type = os.environ["CONFIG_TYPE"]

def load_json(path):
    with open(path, encoding="utf-8") as f:
        return json.load(f)

def save_json(path, data):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2)
        f.write("\\n")
    print(f"Updated: {path}")

if config_type in ("environment", "both"):
    path = f"environment/{env_name}.json"
    data = load_json(path)

    data["environment"] = env_name
    data["appName"] = os.environ["ENV_APPNAME"]
    data["version"] = os.environ["ENV_VERSION"]
    data["replicas"] = int(os.environ["ENV_REPLICAS"])
    data["logLevel"] = os.environ["ENV_LOGLEVEL"]

    data.setdefault("resources", {})

    cpu = os.environ.get("CPU", "").strip()
    memory = os.environ.get("MEMORY", "").strip()

    if cpu:
        data["resources"]["cpu"] = cpu
    if memory:
        data["resources"]["memory"] = memory

    save_json(path, data)

if config_type in ("node", "both"):
    path = f"node/{env_name}.json"
    data = load_json(path)

    data["environment"] = env_name
    data["nodeName"] = os.environ["NODE_NAME"]
    data["nodeType"] = os.environ["NODE_TYPE"]
    data["region"] = os.environ["NODE_REGION"]
    data["availabilityZone"] = os.environ["NODE_AZ"]
    data["instanceType"] = os.environ["INSTANCE_TYPE"]

    data.setdefault("kubernetes", {})
    data.setdefault("resources", {})

    version = os.environ.get("K8S_VERSION", "").strip()
    cpu = os.environ.get("CPU", "").strip()
    memory = os.environ.get("MEMORY", "").strip()

    if version:
        data["kubernetes"]["version"] = version
    if cpu:
        data["resources"]["cpu"] = cpu
    if memory:
        data["resources"]["memory"] = memory

    save_json(path, data)
PY
                '''
            }
        }

        stage('Verify JSON and Diff') {
            steps {
                sh '''
                    set -eu
                    python3 - <<'PY'
import json
import os

env_name = os.environ["ENVIRONMENT"]
config_type = os.environ["CONFIG_TYPE"]

files = []

if config_type in ("environment", "both"):
    files.append(f"environment/{env_name}.json")

if config_type in ("node", "both"):
    files.append(f"node/{env_name}.json")

for path in files:
    with open(path, encoding="utf-8") as f:
        json.load(f)
    print(f"Verified JSON: {path}")
PY

                    echo "===== Configuration diff ====="
                    git diff -- environment node || true
                '''
            }
        }

        stage('Check Configuration Changes') {
            steps {
                script {
                    def filesToCheck = []

                    if (params.CONFIG_TYPE in ['environment', 'both']) {
                        filesToCheck.add(
                            "environment/${params.ENVIRONMENT}.json"
                        )
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        filesToCheck.add(
                            "node/${params.ENVIRONMENT}.json"
                        )
                    }

                    def diffStatus = sh(
                        script: "git diff --quiet -- ${filesToCheck.join(' ')}",
                        returnStatus: true
                    )

                    if (diffStatus == 0) {
                        env.CONFIG_CHANGES = 'false'
                        currentBuild.description =
                            'CONFIG_CHANGES=false'

                        echo 'No configuration changes detected.'
                        echo 'Git stages will be skipped.'
                    } else if (diffStatus == 1) {
                        env.CONFIG_CHANGES = 'true'
                        currentBuild.description =
                            'CONFIG_CHANGES=true'

                        echo 'Configuration changes detected.'
                        echo 'Git stages will run.'
                    } else {
                        error(
                            "Git diff failed with exit code ${diffStatus}"
                        )
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQube-Scanner'

                    withSonarQubeEnv('SonarQube-Server') {
                        withEnv(["SONAR_SCANNER_HOME=${scannerHome}"]) {
                            sh '''
                                set -eu
                                "$SONAR_SCANNER_HOME/bin/sonar-scanner" \
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
                    def filesToArchive = []

                    if (params.CONFIG_TYPE in ['environment', 'both']) {
                        filesToArchive.add(
                            "environment/${params.ENVIRONMENT}.json"
                        )
                    }

                    if (params.CONFIG_TYPE in ['node', 'both']) {
                        filesToArchive.add(
                            "node/${params.ENVIRONMENT}.json"
                        )
                    }

                    env.ZIP_NAME =
                        "flipkart-config-${params.ENVIRONMENT}-${env.BUILD_NUMBER}.zip"

                    withEnv([
                        "ZIP_NAME=${env.ZIP_NAME}",
                        "ARCHIVE_FILES=${filesToArchive.join(' ')}"
                    ]) {
                        sh '''
                            set -eu

                            python3 - <<'PY'
import os
import zipfile

zip_name = os.environ["ZIP_NAME"]
files = os.environ["ARCHIVE_FILES"].split()

with zipfile.ZipFile(
    zip_name, "w", compression=zipfile.ZIP_DEFLATED
) as archive:
    for path in files:
        if not os.path.isfile(path):
            raise SystemExit(f"Missing file: {path}")
        archive.write(path)

with zipfile.ZipFile(zip_name) as archive:
    print(f"Created: {zip_name}")
    print(f"Contents: {archive.namelist()}")
PY
                        '''
                    }

                    archiveArtifacts(
                        artifacts: env.ZIP_NAME,
                        fingerprint: true
                    )
                }
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USERNAME',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        curl --fail --silent --show-error \
                          --user "$NEXUS_USERNAME:$NEXUS_PASSWORD" \
                          --upload-file "$ZIP_NAME" \
                          "$NEXUS_REPOSITORY_URL/$ZIP_NAME"

                        echo "Artifact uploaded to Nexus"
                    '''
                }
            }
        }

        stage('Create Git Feature Branch') {
            when {
                expression {
                    currentBuild.description == 'CONFIG_CHANGES=true'
                }
            }

            steps {
                script {
                    env.FEATURE_BRANCH =
                        "feature/update-${params.ENVIRONMENT}-${env.BUILD_NUMBER}"

                    withEnv(["FEATURE_BRANCH=${env.FEATURE_BRANCH}"]) {
                        sh '''
                            set -eu

                            git config user.name "Asif Shaik"
                            git config user.email "asifshaik6558@gmail.com"

                            git switch -c "$FEATURE_BRANCH"

                            git add -- \
                              "environment/$ENVIRONMENT.json" \
                              "node/$ENVIRONMENT.json"

                            git diff --cached --check

                            if git diff --cached --quiet; then
                                echo "Expected configuration changes were not staged."
                                exit 1
                            fi

                            git commit -m "Update $ENVIRONMENT configuration"
                        '''
                    }
                }
            }
        }

        stage('Push Feature Branch') {
            when {
                expression {
                    currentBuild.description == 'CONFIG_CHANGES=true'
                }
            }

            steps {
                script {
                    env.FEATURE_BRANCH =
                        "feature/update-${params.ENVIRONMENT}-${env.BUILD_NUMBER}"
                }

                withCredentials([
                    usernamePassword(
                        credentialsId: 'git',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        AUTH=$(printf '%s:%s' \
                          "$GIT_USERNAME" "$GIT_TOKEN" | base64 | tr -d '\\n')

                        git -c http.extraheader="AUTHORIZATION: basic $AUTH" \
                          push "https://github.com/$GITHUB_REPO.git" \
                          "$FEATURE_BRANCH"

                        echo "Feature branch pushed successfully"
                    '''
                }
            }
        }

        stage('Create GitHub Pull Request') {
            when {
                expression {
                    currentBuild.description == 'CONFIG_CHANGES=true'
                }
            }

            steps {
                script {
                    env.FEATURE_BRANCH =
                        "feature/update-${params.ENVIRONMENT}-${env.BUILD_NUMBER}"
                }

                withCredentials([
                    usernamePassword(
                        credentialsId: 'git',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        python3 - <<'PY'
import json
import os

payload = {
    "title": "Update " + os.environ["ENVIRONMENT"] + " configuration",
    "head": os.environ["FEATURE_BRANCH"],
    "base": os.environ["BASE_BRANCH"],
    "body": (
        "Automated configuration update from Jenkins build "
        + os.environ["BUILD_NUMBER"]
        + ".\\n\\nArtifact: "
        + os.environ["ZIP_NAME"]
    )
}

with open("pull-request.json", "w", encoding="utf-8") as f:
    json.dump(payload, f)
PY

                        HTTP_CODE=$(curl --silent --show-error \
                          --output pr-response.json \
                          --write-out '%{http_code}' \
                          --user "$GIT_USERNAME:$GIT_TOKEN" \
                          --header 'Accept: application/vnd.github+json' \
                          --header 'X-GitHub-Api-Version: 2022-11-28' \
                          --header 'Content-Type: application/json' \
                          --request POST \
                          --data @pull-request.json \
                          "https://api.github.com/repos/$GITHUB_REPO/pulls")

                        if [ "$HTTP_CODE" -lt 200 ] ||
                           [ "$HTTP_CODE" -ge 300 ]; then
                            echo "GitHub pull request creation failed."
                            cat pr-response.json
                            exit 1
                        fi

                        python3 - <<'PY'
import json

with open("pr-response.json", encoding="utf-8") as f:
    result = json.load(f)

print("Pull request created successfully.")
print("PR number:", result.get("number"))
print("PR URL:", result.get("html_url"))
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
            echo 'Pipeline failed. Check Console Output for the failed stage.'
        }

        always {
            echo "Final build result: ${currentBuild.currentResult}"
        }
    }
}
