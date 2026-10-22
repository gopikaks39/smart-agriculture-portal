
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    parameters {
        choice(
            name: 'RELEASE_MODE',
            choices: ['STABLE', 'FAULTY_ROLLBACK_DEMO'],
            description: 'Deploy stable image or demonstrate failed rollout and rollback'
        )
        booleanParam(
            name: 'DEPLOY_TO_KUBERNETES',
            defaultValue: true,
            description: 'Run the Kubernetes deployment stage'
        )
    }

    environment {
        DOCKER_IMAGE = 'gopikaks/smart-agriculture-portal'
        K8S_DEPLOYMENT = 'smart-agriculture'
        K8S_NAMESPACE = 'default'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                bat 'git --version'
            }
        }

        stage('Validate and Test') {
            parallel {
                stage('Check required files') {
                    steps {
                        bat 'if not exist index.html exit /b 1'
                        bat 'if not exist Dockerfile exit /b 1'
                        bat 'if not exist deployment.yaml exit /b 1'
                        bat 'if not exist Jenkinsfile exit /b 1'
                    }
                }

                stage('Dashboard smoke test') {
                    steps {
                        powershell '''
                            $ErrorActionPreference = "Stop"
                            $html = Get-Content -Raw "index.html"
                            $required = @(
                                "Smart Agriculture Dashboard",
                                "F-101", "F-102", "F-103", "F-104", "F-105",
                                "Prioritized irrigation recommendations",
                                "Water conservation plan"
                            )
                            foreach ($item in $required) {
                                if (-not $html.Contains($item)) {
                                    throw "Missing required content: $item"
                                }
                            }
                            if ($html -notmatch 'moistures*<\s*30') {
                                throw "Missing irrigation threshold below 30%."
                            }
                            Write-Output "Dashboard smoke test passed."
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build --pull -t %DOCKER_IMAGE%:%BUILD_NUMBER% -t %DOCKER_IMAGE%:latest .'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASS'
                    )
                ]) {
                    bat 'echo %DH_PASS%| docker login -u %DH_USER% --password-stdin'
                    bat 'docker push %DOCKER_IMAGE%:%BUILD_NUMBER%'
                    bat 'docker push %DOCKER_IMAGE%:latest'
                }
            }
            post {
                always {
                    bat 'docker logout'
                }
            }
        }

        stage('Deploy and verify') {
            when {
                expression { params.DEPLOY_TO_KUBERNETES }
            }
            steps {
                script {
                    withCredentials([
                        file(
                            credentialsId: 'kuberconfig',
                            variable: 'KUBECONFIG_FILE'
                        )
                    ]) {
                        if (params.RELEASE_MODE == 'STABLE') {
                            bat 'set KUBECONFIG=%KUBECONFIG_FILE%&& kubectl apply -f deployment.yaml'
                            bat 'set KUBECONFIG=%KUBECONFIG_FILE%&& kubectl set image deployment/%K8S_DEPLOYMENT% portal=%DOCKER_IMAGE%:%BUILD_NUMBER% -n %K8S_NAMESPACE%'
                            bat 'set KUBECONFIG=%KUBECONFIG_FILE%&& kubectl rollout status deployment/%K8S_DEPLOYMENT% -n %K8S_NAMESPACE% --timeout=120s'
                            bat 'set KUBECONFIG=%KUBECONFIG_FILE%&& kubectl get deployment,pods,service -l app=smart-agriculture -n %K8S_NAMESPACE%'
                        } else {
                            echo 'Injecting deliberate faulty image tag.'

                            bat 'set KUBECONFIG=%KUBECONFIG_FILE%&& kubectl apply -f deployment.yaml'
                            bat 'set KUBECONFIG=%KUBECONFIG_FILE%&& kubectl set image deployment/%K8S_DEPLOYMENT% portal=%DOCKER_IMAGE%:deliberately-broken-tag -n %K8S_NAMESPACE%'

                            int result = bat(
                                returnStatus: true,
                                script: 'set KUBECONFIG=%KUBECONFIG_FILE%&& kubectl rollout status deployment/%K8S_DEPLOYMENT% -n %K8S_NAMESPACE% --timeout=90s'
                            )

                            if (result == 0) {
                                error('The faulty rollout unexpectedly succeeded; verify that the bad tag does not exist.')
                            }

                            echo 'Failed rollout detected. Rolling back to the previous revision.'

                            bat 'set KUBECONFIG=%KUBECONFIG_FILE%&& kubectl rollout undo deployment/%K8S_DEPLOYMENT% -n %K8S_NAMESPACE%'
                            bat 'set KUBECONFIG=%KUBECONFIG_FILE%&& kubectl rollout status deployment/%K8S_DEPLOYMENT% -n %K8S_NAMESPACE% --timeout=120s'
                            bat 'set KUBECONFIG=%KUBECONFIG_FILE%&& kubectl get deployment,pods,service -l app=smart-agriculture -n %K8S_NAMESPACE%'

                            echo 'Rollback verified: previous stable release is available.'
                        }
                    }
                }
            }
        }

        stage('Archive artifacts') {
            steps {
                archiveArtifacts artifacts: 'index.html,Dockerfile,Jenkinsfile,deployment.yaml', fingerprint: true
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Inspect Console Output for the first error.'
        }
        always {
            echo 'Build finished.'
        }
    }
}
