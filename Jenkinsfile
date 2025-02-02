pipeline {
    agent none

    stages {
        stage('worker-build') {
            when {
                changeset '**/worker/**'
            }
            agent {
                docker {
                    image 'maven:3.6.1-jdk-11-slim'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                echo 'Compiling worker app'
                dir('worker') {
                    sh 'mvn compile'
                }
            } 
        }
        stage('worker-test') {
            when {
                changeset '**/worker/**'
            }
            agent {
                docker {
                    image 'maven:3.6.1-jdk-11-slim'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                echo 'Running Unit Tests on worker app'
                dir('worker') {
                    sh 'mvn clean test'
                }
            }
        }
        stage('worker-package') {
            when {
                branch 'master'
                changeset '**/worker/**'
            }
            agent {
                docker {
                    image 'maven:3.6.1-jdk-11-slim'
                    args '-v $HOME/.m2:/root/.m2'
                }
            }
            steps {
                echo 'Packaging worker app'
                dir('worker') {
                    sh 'mvn package -DskipTests'
                    archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
                }
            }
        }
        stage('worker-docker-package') {
            agent any
            when {
                changeset "**/worker/**"
                branch 'master'
            }
            steps {
                echo 'Packaging worker app with docker'
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'jerry871002-dockerhub') {
                        def workerImage = docker.build("jerry871002/worker:v${env.BUILD_ID}", './worker')
                        workerImage.push()
                        workerImage.push("${env.BUILD_ID}")
                        workerImage.push('latest')
                    }
                }
            }
        }

        stage('result-test') {
            agent {
                docker {
                    image 'node:8.16.0-alpine'
                }
            }
            when {
                changeset '**/result/**'
            }
            steps {
                echo 'Running Unit Tests on result app'
                dir('result') {
                    sh 'npm install'
                    sh 'npm test'
                }

            }
        }
        stage('result-docker-package') {
            agent any
            when {
                changeset '**/result/**'
                branch 'master'
            }
            steps {
                echo 'Packaging result app with docker'
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'jerry871002-dockerhub') {
                        def resultImage = docker.build("jerry871002/result:v${env.BUILD_ID}", './result')
                        resultImage.push()
                        resultImage.push("${env.BUILD_ID}")
                        resultImage.push('latest')
                    }
                }
            }
        }

        stage('vote-build') {
            agent {
                docker {
                    image 'python:2.7.16-slim'
                    args '--user root'
                }
            }
            when {
                changeset '**/vote/**'
            }
            steps {
                echo 'Compiling vote app'
                dir('vote') {
                    sh 'pip install -r requirements.txt'
                }
            }
        }
        stage('vote-test') {
            agent {
                docker {
                    image 'python:2.7.16-slim'
                    args '--user root'
                }

            }
            when {
                changeset '**/vote/**'
            }
            steps {
                echo 'Running Unit Tests on vote app'
                dir('vote') {
                    sh 'pip install -r requirements.txt'
                    sh 'nosetests -v'
                }

            }
        }
        stage('vote-integration'){ 
            agent any 
            when { 
                changeset "**/vote/**" 
                branch 'master' 
            } 
            steps { 
                echo 'Running Integration Tests on vote app' 
                dir('vote') { 
                    sh 'sh integration_test.sh' 
                } 
            } 
        }
        stage('vote-docker-package') {
            agent any
            steps {
                echo 'Packaging vote app with docker'
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'jerry871002-dockerhub') {
                        def voteImage = docker.build("jerry871002/vote:${env.BUILD_ID}", "./vote")
                        voteImage.push()
                        voteImage.push("${env.BUILD_ID}")
                        voteImage.push("latest")
                    }
                }

            }
        }

        stage('sonarqube') {
            agent any
            when{
                branch 'master'
            }
            environment{
                sonarpath = tool 'SonarScanner'
            }
            steps {
                echo 'Running Sonarqube Analysis'
                withSonarQubeEnv('sonar-instavote') {
                    sh "${sonarpath}/bin/sonar-scanner -Dproject.settings=sonar-project.properties -Dorg.jenkinsci.plugins.durabletask.BourneShellScript.HEARTBEAT_CHECK_INTERVAL=86400"
                }
            }
        }
        stage("quality-gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('deploy to dev') {
            agent any
            when {
                branch 'master'
            }
            steps {
                echo 'Deploy instavote app with docker compose'
                sh 'docker-compose up -d'
            }
        }
    }

    post {
        always {
            echo 'Building multibranch pipeline for worker is completed..'
        }
    } 
}