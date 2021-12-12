pipeline{
    agent any
    environment{
        VERSION = "${env.BUILD_ID}"
    }
    stages{
        stage("sonar quality check"){
            agent {
                docker {
                    image  'openjdk:11'
                }
            }
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                        sh 'chmod +x gradlew'
                        sh './gradlew sonarqube'
                    }

                    timeout(5) {
                        def qg = waitForQualityGate()
                        if(qg.status != 'OK'){
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }
        stage("identifying misconfigs using datree in helm charts"){
            steps{
                script{
                    dir('kubernetes/') {
                        withEnv(['DATREE_TOKEN=ez2nPSrcaZJC4jmNJcGcnj']) {
                           sh 'helm datree test myapp '
                        }
                    }
                }
            }
        }
        stage("docker build & push"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'nexus_pwd', variable: 'nexus_pwd')]) {
                        sh '''
                            docker build -t 34.125.23.250:8083/springapp:${VERSION} .
                            docker login -u admin -p $nexus_pwd 34.125.23.250:8083
                            docker push 34.125.23.250:8083/springapp:${VERSION}
                            docker rmi 34.125.23.250:8083/springapp:${VERSION}
                        '''
                    }
                    
                }
            }
        }
        stage("pushing helm charts to nexus"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'nexus_pwd', variable: 'nexus_pwd')]) {
                        dir('kubernetes/') {
                            sh '''
                                helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ' )
                                tar -czvf myapp-${helmversion}.tgz myapp/
                                curl -u admin:$nexus_pwd http://34.125.23.250:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                            '''
                        }
                    }
                    
                }
            }
        }
    }
    post{
        always{
            echo "SUCCESS"
        }
    }
}