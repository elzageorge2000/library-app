pipeline {
    agent any
    tools {
        maven 'maven-3.9.6'
    }

    stages {
        stage('Maven Build') {
            steps {
                // Run maven build
                sh 'mvn clean package -DskipTests'
                echo 'Maven build Completed'
            }
        }
        stage('JUnit Test') {
            steps {
                // Run unit tests
                script {
                    try {
                        sh 'mvn clean test surefire-report:report' 
                    } catch (err) {
                        currentBuild.result = 'FAILURE'
                        echo 'Unit tests failed!'
                        error 'Unit tests failed!'
                    }
                }
                echo 'JUnit test Completed'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                // Run SonarQube analysis
                withSonarQubeEnv('SonarQube') {
                    sh '''mvn clean verify sonar:sonar -Dsonar.projectKey=ci-cd-employees -Dsonar.projectName='ci-cd-employees' -Dsonar.host.url=http://localhost:9000 -Dsonar.token=sqp_b2e02e39466407dbb52e36ace6615f4aaf2287e9''' //port 9000 is default for sonar
                    echo 'SonarQube Analysis Completed'
                }
            }
        }
        stage('Copy artifact to EC2') {
            steps {
                // Copy jar file to EC2 instance
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'ansible',
                            transfers: [
                                sshTransfer(
                                    cleanRemote: false,
                                    excludes: '',
                                    execCommand: '',
                                    execTimeout: 120000,
                                    flatten: false,
                                    makeEmptyDirs: false,
                                    noDefaultExcludes: false,
                                    patternSeparator: '[, ]+',
                                    remoteDirectory: '//opt//elza',
                                    remoteDirectorySDF: false,
                                    removePrefix: 'target',
                                    sourceFiles: 'target/*.jar'
                                )
                            ],
                            usePromotionTimestamp: false,
                            useWorkspaceInPromotion: false,
                            verbose: false
                        )
                    ]
                )
                echo 'Copying Jar to EC2 Completed'
            }
        }
        stage('Deploy Docker Container') {
            steps {
                //Using Ansible to deploy the container in EC2
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'ansible',
                            transfers: [
                                sshTransfer(
                                    cleanRemote: false,
                                    excludes: '',
                                    execCommand: '''
                                        cd /opt/elza/
                                        ansible-playbook docker-deploy.yml
                                    ''',
                                    execTimeout: 120000,
                                    flatten: false,
                                    makeEmptyDirs: false,
                                    noDefaultExcludes: false,
                                    patternSeparator: '[, ]+',
                                    remoteDirectory: '',
                                    remoteDirectorySDF: false,
                                    removePrefix: '',
                                    sourceFiles: ''
                                )
                            ],
                            usePromotionTimestamp: false,
                            useWorkspaceInPromotion: false,
                            verbose: false
                        )
                    ]
                )
                echo 'Deployment Completed'
            }
        }

    }
    post {
        failure {
            // This block will execute if any of the previous stages fail, including unit tests
            echo 'One or more stages have failed!'
            echo 'Pipeline Aborted'
        }
        always {
            // Publish test report
            junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
        }
    }
}
