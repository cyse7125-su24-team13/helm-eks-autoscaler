pipeline {
    agent any
    environment {
        GITHUB_CREDENTIALS = credentials('github-token')
        GITHUB_REPO = 'cyse7125-su24-team13/helm-eks-autoscaler'
        GITHUB_API_URL = 'https://api.github.com'
        NODE_VERSION = '20' // Specify the Node.js version you want to install
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    def scmVars = checkout scm
                    def commitId = scmVars.GIT_COMMIT
                    env.GIT_COMMIT_ID = commitId
                    def credentialsParts = GITHUB_CREDENTIALS.split(':')
                    if (credentialsParts.length != 2) {
                        error 'Invalid GitHub credentials format. Expected format: username:token'
                    }
                    env.GITHUB_USER = credentialsParts[0]
                    env.GITHUB_TOKEN = credentialsParts[1]
                }
            }
        }
        stage('Commit Message Lint') {
            when {
                not {
                    branch 'main'
                }
            }
            steps {
                script {
                    checkCommitMessages()
                }
            }
        }

        stage('Install Helm') {
            when {
                not {
                    branch 'main'
                }
            }
            steps {
                sh '''
                    if ! command -v helm &> /dev/null
                    then
                        echo "Helm not found, installing..."
                        curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
                    else
                        echo "Helm is already installed."
                    fi
                '''
            }
        }

        stage('Helm Lint') {
            when {
                not {
                    branch 'main'
                }
            }
            steps {
                script {
                    def status = sh(script: "helm lint charts/cve-processor", returnStatus: true)
                    if (status != 0) {
                        error "Helm lint failed"
                    } else {
                        echo "Helm lint passed"
                    }
                }
            }
        }

        stage('Helm Template') {
            when {
                not {
                    branch 'main'
                }
            }
            steps {
                script {
                    def status = sh(script: "helm template charts/cve-processor", returnStatus: true)
                    if (status != 0) {
                        error "Helm template failed"
                    } else {
                        echo "Helm template passed"
                    }
                }
            }
        }

        stage('Install Node.js and Dependencies') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    # Install Node.js from NodeSource
                    curl -fsSL https://deb.nodesource.com/setup_${NODE_VERSION} | sudo -E bash -
                    sudo apt-get install -y nodejs

                    # Verify Node.js and npm versions
                    node -v
                    npm -v

                    npm install semantic-release@latest \
                                @semantic-release/changelog@latest \
                                @semantic-release/commit-analyzer@latest \
                                @semantic-release/release-notes-generator@latest \
                                @semantic-release/git@latest \
                                @semantic-release/github@latest \
                                @semantic-release/exec@latest --legacy-peer-deps
                '''
            }
        }

        stage('Semantic Release') {
            when {
                branch 'main'
            }
            steps {
                script {
                    echo "Running semantic-release..."
                    def releaseOutput = sh(script: "npx semantic-release", returnStdout: true).trim()
                    echo "Semantic release output: ${releaseOutput}"
                    def newVersionMatcher = (releaseOutput =~ /The next release version is (\d+\.\d+\.\d+)/)
                    if (newVersionMatcher.find()) {
                        def newVersion = newVersionMatcher.group(1)
                        echo "Extracted new version: ${newVersion}"
                        env.NEW_VERSION = newVersion // Store the new version in an environment variable
                    } else {
                        echo "No new version found from semantic-release."
                        env.NEW_VERSION = null
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                def commitId = env.GIT_COMMIT_ID
                def status = 'success'
                def description = 'Helm lint and template successful.'
                notifyGithub(commitId, status, description)
            }
        }

        failure {
            script {
                def commitId = env.GIT_COMMIT_ID
                def status = 'failure'
                def description = 'Helm lint or template failed.'
                notifyGithub(commitId, status, description)
            }
        }
    }
}

def notifyGithub(commitId, status, description) {
    def context = 'helm'
    def url = "${env.GITHUB_API_URL}/repos/${env.GITHUB_REPO}/statuses/${commitId}"

    def payload = [
        state       : status,
        target_url  : env.BUILD_URL,
        description : description,
        context     : context
    ]

    def response = sh(
        script: """#!/bin/bash
        curl -s -H "Authorization: token ${env.GITHUB_TOKEN}" \\
             -H "Content-Type: application/json" \\
             -d '${groovy.json.JsonOutput.toJson(payload)}' \\
             ${url}
        """,
        returnStdout: true
    ).trim()

    echo "GitHub API response: ${response}"
}

def checkCommitMessages() {
    def commitMessages = sh(script: 'git log --pretty=format:%s origin/main..HEAD', returnStdout: true).trim().split('\n')
    def conventionalCommitRegex = /^(feat|fix|docs|style|refactor|perf|test|chore|build|ci|revert|wip)(\(.+\))?: .{1,50}/

    if (commitMessages.size() == 0 || (commitMessages.size() == 1 && commitMessages[0] == '')) {
        echo "No new commits to validate."
    } else {
        for (commitMessage in commitMessages) {
            if (commitMessage.startsWith("Merge ")) {
                echo "Merge commit detected, skipping Conventional Commits check for this commit."
            } else if (!commitMessage.matches(conventionalCommitRegex)) {
                error("Commit message does not follow the Conventional Commits specification. Commit message: ${commitMessage}")
            } else {
                echo "Commit message is valid: ${commitMessage}"
            }
        }
    }
}
