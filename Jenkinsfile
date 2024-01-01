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
        stage('SCM Checkout'){
            when { expression { true}}
            steps{
                container('git'){
                    git branch: 'master',
                    url: 'https://github.com/Shantayya/spring-petclinic.git'
                }       
            }
        }
        stage('Build SW'){
            when { expression { true}}
            steps{
                container('maven'){
                    sh 'mvn -Dmaven.test.failure.ignore=true package'
                }
            }
            post{
                success{
                    junit "**target/surefire-reports/*.xml"
                }
            }
            
        }
        stage('Sonar scan'){
            when { expression { false}}
            steps{
                container('sonar-scanner'){
                    withSonarQubeEnv(credentialsId: 'sonar', installationName: 'sonarserver'){
                        sh ''' /opt/sonar-scanner/bin/sonar-scanner \
                        -Dsonar.projectKey=petclinic \
                        -Dsonar.projectName=petcliic \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/main \
                        -Dsonar.tests=src/test \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.language=java \
                        -Dsonar.sourceEncoding=UTF8 \
                        -Dsonar.java.libraries=target/classes 
                        '''
                   }
                }       
            }
        }
        stage('Wait for QualityGate'){
            when { expression { false}}
            steps{
                container('sonar-scanner'){
                    timeout(time: 1, unit: HOURS){
                        waitForQualityGate abortPipeline: true
                    }
                   }
                }       
            }
         stage('Push maven artifact to Nexus'){
            when { expression { true}}
            steps{
                container('jnlp'){
                        script {
                            pom = readMavenPom file: "pom.xml";
                            filesByGlob = findFiles(glob: "target/*.${pom.packaging}"); 
                            echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                            artifactPath = filesByGlob[0].path;
                            artifactExists = fileExists artifactPath;
                            if(artifactExists) {
                                echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version}";
                                nexusArtifactUploader(
                                    nexusVersion: NEXUS_VERSION,
                                    protocol: NEXUS_PROTOCOL,
                                    nexusUrl: NEXUS_URL,
                                    groupId: pom.groupId,
                                    version: pom.version,
                                    repository: NEXUS_REPOSITORY,
                                    credentialsId: NEXUS_CREDENTIAL_ID,
                                    artifacts: [
                                        [artifactId: pom.artifactId,
                                        classifier: '',
                                        file: artifactPath,
                                        type: pom.packaging],

                                        [artifactId: pom.artifactId,
                                        classifier: '',
                                        file: "pom.xml",
                                        type: "pom"]
                                    ]
                                );

                            } else {
                                error "*** File: ${artifactPath}, could not be found";
                            }
                        }
                    }
                   }
                }
            stage('Push maven artifact to nexus via curl'){
            when { expression { false}}
            steps{
                container('curl'){
                    script{
                        pom = readMavenPom file: "pom.xml";
                        withCredentials([usernamePassword(credentialsId: 'nexus-creds', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                                sh "curl -v -u $USER:$PASS --upload-file ${WORKSPACE}/spring-petclinic/target/${pom.artifactId}-${pom.version}.${pom.packaging} \
                                        http://${NEXUS_URL}/repository/maven-hosted/org/springframework/samples/${pom.artifactId}/${pom.version}/${pom.artifactId}-${pom.version}.${pom.packaging}"
                                }

                        }
                     
                   }
                }       
            } 
            stage('Build Docker Image'){
                when { expression { true}}
                steps{
                    container('docker'){
                        sh "docker build -t $IMAGE_NAME:$IMAGE_TAG ."
                        sh " docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest"
                        withCredentials([usernamePassword(credentialsId: 'docker-creds', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                                sh "docker login -u $USER -p $PASS"
                                sh "docker push $IMAGE_NAME:$IMAGE_TAG"
                                sh "docker push $IMAGE_NAME:latest"
                            }
                            sh "docker rmi $IMAGE_NAME:$IMAGE_TAG"
                            sh "docker rmi $IMAGE_NAME:latest"
                        }
                    }
            }
            stage('Deploy Spring-petclinic to K8s Cluster'){
                when { expression { true}}
                steps{
                    container('kubectl-helm-cli'){
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'kubeconfig', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh "kubectl apply -f deployment.yaml" 
                            }
                        }
                    }
                post{
                    success{
                        mail to: 'shantayyaswami4@gmail.com',
                        from: 'jenkinsadmin@gmail.com',
                        subject: "Jenkins pipeline for the job ${JOB_NAME} completed successfully",
                        body: "Check build logs at ${BUILD_URL}"
                    }
                    failure{
                        mail to: 'shantayyaswami4@gmail.com',
                        from: 'jenkinsadmin@gmail.com',
                        subject: "Jenkins pipeline is failed for the job ${JOB_NAME}",
                        body: "Check build logs at ${BUILD_URL}"
                    }
                }
            }             
    }          
}
