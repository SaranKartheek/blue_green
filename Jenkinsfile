pipeline {
    agent any
    tools {
        jdk 'java-17'
        maven 'maven3'
    }
    environment {
        scanner_home = tool 'sonar-scanner'
        image_name = "saranvemireddy/bankapp"
        tag = "${params.docker_tag}"
        kube_namespace = "webapps"
    }
    parameters {
        choice(name:'deploy_env', choices: ['blue','green'], description: 'choose in which env to deploy')
        choice(name:'docker_tag', choices: ['blue','green'], description: 'choose docker image tag for deployment')
        booleanParam(name:'switch_traffic', defaultValue: false, description: 'switch traffic between blue and green environments')
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'gitcreds', url: 'https://github.com/SaranKartheek/blue_green.git'
            }
        }
         stage('Compile') {
            steps {
                script {
                    sh "mvn compile"
                }
            }
        }
         stage('Test') {
            steps {
                script {
                    sh "mvn test -DskipTests=true"
                }
            }
        }
         stage('Trivy FS Scan') {
            steps {
                script {
                    sh "trivy fs --format table -o fs.html ."
                }
            }
        }
         stage('Sonarqube Analysis') {
            steps {
               script {
                    withSonarQubeEnv('sonarserver') {
                        sh "$scanner_home/bin/sonar-scanner -Dsonar.projectKey=bluegreen -Dsonar.projectName=bluegreen -Dsonar.java.binaries=."
                    }
               }
            }
        }
         stage('Quality Gates') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonartoken'
                }
            }
        }
         stage('Build') {
            steps {
                script {
                    sh "mvn package -DskipTests=true"
                }
            }
        }
        stage('Publish to Nexus') {
            steps {
               script {
                   withMaven(globalMavenSettingsConfig: 'maven_settings.xml', jdk: 'java-17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                       sh "mvn deploy -DskipTests=true"
                   }
               }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockercreds') {
                        sh "docker build -t ${image_name}:${tag} ."
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                script {
                    sh "trivy image --format table -o image.html ${image_name}:${tag}"
                }
            }
        }
        stage('Push Image to Docker Hub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockercreds') {
                        sh "docker push ${image_name}:${tag}"
                    }
                }
            }
        }
        stage('Deploy MySQL Deployment and service') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'bluegreen.us-east-1.eksctl.io', contextName: '', credentialsId: 'kubecreds', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://6295519D6DEA7638C572AD7E6D4AFCA7.gr7.us-east-1.eks.amazonaws.com') {
                        sh "kubectl apply -f mysql-ds.yml -n ${kube_namespace}"
                    }
                }
            }
        }
        stage('Deploy Bankapp service') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'bluegreen.us-east-1.eksctl.io', contextName: '', credentialsId: 'kubecreds', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://6295519D6DEA7638C572AD7E6D4AFCA7.gr7.us-east-1.eks.amazonaws.com') {
                        sh "kubectl apply -f bankapp-service.yml -n ${kube_namespace}"
                    }
                }
            }
        }
        stage('Deploy Bankapp application') {
            steps {
                script {
                    def deploymentFile = ""
                    if(params.deploy_env == 'blue') {
                        deploymentFile = "app-deployment-blue.yml"
                    }
                    else {
                        deploymentFile = "app-deployment-green.yml"
                    }
                    withKubeConfig(caCertificate: '', clusterName: 'bluegreen.us-east-1.eksctl.io', contextName: '', credentialsId: 'kubecreds', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://6295519D6DEA7638C572AD7E6D4AFCA7.gr7.us-east-1.eks.amazonaws.com') {
                        sh "kubectl apply -f ${deploymentFile} -n ${kube_namespace}"
                    }
                }
            }
        }
        stage('Switch Traffic') {
            when {
                expression { return params.switch_traffic }
            }
            steps {
                script {
                    def newEnv = params.deploy_env
                    withKubeConfig(caCertificate: '', clusterName: 'bluegreen.us-east-1.eksctl.io', contextName: '', credentialsId: 'kubecreds', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://6295519D6DEA7638C572AD7E6D4AFCA7.gr7.us-east-1.eks.amazonaws.com') {
                        sh '''
                            kubectl patch service bankapp-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"bankapp\\", \\"version\\": \\"''' + newEnv + '''\\"}}}" -n ${kube_namespace}
                        '''
                    }
                }
            }
        }
        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.deploy_env
                    withKubeConfig(caCertificate: '', clusterName: 'bluegreen.us-east-1.eksctl.io', contextName: '', credentialsId: 'kubecreds', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://6295519D6DEA7638C572AD7E6D4AFCA7.gr7.us-east-1.eks.amazonaws.com') {
                        sh '''
                            kubectl get pods -l version=${verifyEnv} -n ${kube_namespace}
                            kubectl get svc bankapp-service -n ${kube_namespace}
                        '''
                    }
                }
            }
        }
    }
}
