@Library('Jenkins-Shared_library')
def gv
pipeline {
    agent any
    environment{
        AWS_ACCESS_KEY_ID = credentials('jenkins-aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws-secret-access-key-id')
        KUBE_NAMESPACE = 'development'		
        INGRESS_NAMESPACE = 'ingress-nginx'	
    }
    tools{
        gradle '8.10'
    }
    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout') {
            steps {
                checkout scm
            }
        }       
         stage('Lint') {
            steps {
                script {
                  // Run the gradle check task, which includes linting
                    //LintApp functions is avaliable in the jenkins-shared-library
                    
                    lintApp()
                }
            }
            post {
                always {
                    // Archive the linting report if any
                    archiveArtifacts artifacts: '**/build/reports/**', allowEmptyArchive: true
                }
            }
        }
        stage('Test') {
            steps {
                script {
                    //testApp functions is avaliable in the jenkins-shared-library
                    testApp()
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh './gradlew sonar '
                }
            }
        }
        stage('Quality Gate') {
            steps {
                // Wait for SonarQube quality gate result
                timeout(time: 10, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Build') {
            steps {
                script {
                    //buildJar functions is avaliable in the jenkins-shared-library
                    buildJar()
                }
            }
        }
        stage('Package') {
            steps {
                script {
                    //packageApp functions is avaliable in the jenkins-shared-library
                    packageApp()
                }
            }
        }
        stage('Build and Push Image '){
            steps {
                script {
                     //buildImage , pushImage functions is avaliable in the jenkins-shared-library,
                     // and they takes a varaiable 'image name'
                     buildImage 'my-spring-boot-app'
                     dockerLogin()
                     pushImage 'my-spring-boot-app:v3'

                }
            }
        }
        stage('Deploying App to Development environment on EKS Cluster ') {
            steps {
                script{
                    // Create the development Namespace if it doesn't Exist 
                    // || true used not stop the pipeline if the Development namespace is already Exist

                    sh 'kubectl create namespace ${KUBE_NAMESPACE} || true'     

                    // Create k8s secret to allow deployment resource to access dockerhub and pull the image 
                    withCredentials([usernamePassword(credentialsId: 'DockerHub_Credientials', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                            sh "kubectl create secret docker-registry my-registry-key \
                                --docker-server=docker.io \
                                --docker-username=$USER \
                                --docker-password=$PASS \
                                --namespace=${KUBE_NAMESPACE} || true"
                        }
                    // deploying the spring boot application to EKS Cluster
                    echo "deploying to development Environment EKS Cluster "
                    sh 'kubectl apply -f ./kubernetes/deployment.yaml -n ${KUBE_NAMESPACE}'
                    sh 'kubectl apply -f ./kubernetes/service.yaml -n ${KUBE_NAMESPACE}'    
                }
            }
        }
        stage('Deploy Nginx Ingress Controller') {
            steps {
                script {

                    //|| true used not stop the pipeline if the development namespace is already Exist
                    sh '''
                    kubectl create namespace ${INGRESS_NAMESPACE} || true
                    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
                    helm repo update
                    helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ${INGRESS_NAMESPACE} || true
                    '''
                }
            }
        }
    }
}