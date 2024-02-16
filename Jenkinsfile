pipeline{
    agent{
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            metadata:
               lables: 
                  app: test
            spec:
              containers:
              - name: git
                image: bitnami/git:latest
                command:
                - cat
                tty: true
              - name: maven
                image: adoptopenjdk/maven-openjdk11:latest
                command:
                - cat
                tty: true
                volumeMounts:
                - name: cache
                  mountPath: "/root/.M2/repository/"
              - name: sonar-scanner
                image: sonarsource/sonar-scanner-cli
                command:
                - cat
                tty: true
              - name: curl
                image: alpine/curl:latest
                command:
                - cat
                tty: true
              - name: kubectl-helm-cli
                image: shantayya/kubectl-helm-cli:latest
                command:
                - cat
                tty: true
              - name: docker
                image: docker:latest
                command:
                - cat
                tty: true
                volumeMounts:
                - name: docker-sock
                  mountPath: /var/run/docker.sock  
              volumes:
              - name: cache
                persistentVolumeClaim:
                  claimName: maven-cache
              - name: docker-sock
                hostPath:
                  path: /var/run/docker.sock    
                '''
                }
    }
    environment{
        NEXUS_REPOSITORY = "maven-hosted"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "144.126.254.10:8081"
        NEXUS_VERSION = "nexus3"
        NEXUS_CREDENTIAL_ID = "nexus-creds"
        DOCKERHUB_USERNAME = "shantayya"
        APP_NAME = "spring-petclinic"
        IMAGE_NAME = "${DOCKERHUB_USERNAME}"+"/"+"${APP_NAME}"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    options{
        buildDiscarder(logRotator(numToKeepStr: '2'))
    }
    stages{
         stage('Clean Workspace'){
            when { expression { true}}
            steps{
                container('jnlp'){
                    script{
                        sh 'echo "Cleaning workspace"'
                        deleteDir()
                    }
                }       
            }
        }
        stages{
        stage('SCM Checkout'){
            when { expression { true } }
            steps{
                container('git'){
                    git url: 'https://github.com/ShantayyaSwami/spring-petclinic.git',
                    branch: 'master'
                }
            }
            post{
                success{
                    sendStatus("Git Checkout","Success")
                }
                failure{
                    sendStatus("Git Checkout","Failure")
                }
            }
        }
        stage('Build SW'){
            when { expression { true } }
            steps{
                container('maven'){
                    sh 'mvn -Dmaven.test.failure.ignore=true clean package'
                }
            }
            post{
                success{
                    junit "**/target/surefire-reports/*.xml"
                    sendStatus("Build SW", "Success")
                }
                failure{
                    sendStatus("Build SW", "Failed")
                }
            }
        }
        stage('Sonar Scan'){
            when { expression{ true } }
            steps{
                container('maven'){
                withSonarQubeEnv(credentialsId: "sonar", installationName: "sonar"){
                sh ''' mvn -Dmaven.test.failure.ignore=true clean verify sonar:sonar  \
                -Dsonar.projectKey=petclinic \
                -Dsonar.projectName='petclinic' \
                -Dsonar.source=src/main \
                -Dsonar.tests=src/test \
                -Dsonar.language=java 
                '''
                }
              }  
            }
            post{
                success{
                    sendStatus("Sonar Scan","Success")
                }
                failure{
                    sendStatus("Sonar Scan","Failure")
                }
            }
        }
        stage('Quality Gate'){
            when { expression{ true } }
            steps{
                container('maven'){
                        timeout(time: 1, unit:'HOURS'){
                            waitForQualityGate abortPipeline: true
                        }
                }
              }
            post{
                success{
                    sendStatus("Quality Gate","Success")
                }
                failure{
                    sendStatus("Quality Gate","Failure")
                }
            }  
            }
        stage('Push artifact to Nexus'){
            when { expression { true } }
            steps{
                container('curl'){
                    script{
                        pom = readMavenPom file: "pom.xml";
                        withCredentials([usernamePassword(credentialsId: "nexus", usernameVariable: "USR", passwordVariable: "PASS")]){
                            sh "curl -u $USR:$PASS --upload-file target/${pom.artifactId}-${pom.version}.${pom.packaging} \
                            http://$NEXUS_URL/repository/maven-hosted/org/springframework/samples/${pom.artifactId}/${pom.version}/${pom.artifactId}-${pom.version}.${pom.packaging}"
                        }
                }
              }  
        }
        post{
                success{
                    sendStatus("Push build to Nexus","Success")
                }
                failure{
                    sendStatus("Push build to Nexus","Failure")
                }
            }
    }
    stage('Build Docker Image'){
            when { expression { true } }
            steps{
                container('docker'){
                 sh "docker build -t $IMAGE_NAME:$IMAGE_TAG ."
                 sh "docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest"
                 withCredentials([usernamePassword(credentialsId: "docker", usernameVariable: "USR", passwordVariable: "PASS")]){
                    sh "docker login -u $USR -p $PASS"
                    sh "docker push $IMAGE_NAME:$IMAGE_TAG"
                    sh "docker push $IMAGE_NAME:latest"
                 }
              }  
        }
        post{
                success{
                    sendStatus("Docker Build","Success")
                }
                failure{
                    sendStatus("Docker Build","Failure")
                }
            }
    }
    stage('Deploy to K8s Cluster'){
            when { expression { true } }
            steps{
                container('kubectl-helm-cli'){
                   withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                        sh 'kubectl apply -f deployment.yaml'
                }
            }  
        }
        post{
                success{
                    sendStatus("Deployment","Success")
                }
                failure{
                    sendStatus("Deployment","Failure")
                }
            }
    }
  }
 post {
    failure {
        mail to: 'devops-dlkinsadmin@worldpay.com',
        from: "jenkins-notification@worldpay.com",
        subject: "Jenkins pipeline has failed for job ${env.JOB_NAME}",
        body: "Check build logs at ${env.BUILD_URL}"
    }
    success {
        mail to: 'devops-dlkinsadmin@worldpay.com',
        from: 'jenkins-notification@worldpay.com',
        subject: "Jenkins pipeline for job ${env.JOB_NAME} is completed successfully",
        body: "Check build logs at ${env.BUILD_URL}"
    }
  }
 }
}   

void sendStatus(String stage, String status) {
    container('curl') {
        withCredentials([string(credentialsId: 'git_token', variable: 'TOKEN')]) {
            sh "curl -u shantayya:$TOKEN -X POST 'https://api.github.com/repos/ShantayyaSwami/Jenkins-end-end-pipeline/statuses/$SHA_ID' -H 'Accept: application/vnd.github.v3+json' -d '{\"state\": \"$status\",\"context\": \"$stage\", \"description\": \"Continuous-Integration-Jenkins\", \"target_url\": \"$JENKINS_URL/job/$JOB_NAME/$BUILD_NUMBER/console\"}' "
        }
    }
}
