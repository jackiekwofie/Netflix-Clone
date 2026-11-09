pipeline {
    agent any

    tools{
        jdk 'jdk17'
        nodejs 'NodeJS16'
    }

    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }

    stages {
        stage('clean workspace') {
            steps{
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps{
                git branch: 'main', url: 'https://github.com/Chriscloudaz/Netflix-Clone.git'
            }
        }

        stage("Sonarqube Analysis ") {
            steps{
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }

        stage("Quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token' 
                }
            } 
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push") {
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=${params.TMDB_API} -t netflix ."
                       sh "docker tag netflix chriscloudaz/netflix:latest "
                       sh "docker push chriscloudaz/netflix:latest"
                    }
                }
            }
        }
        
        stage("TRIVY") {
            steps{
                sh "trivy image chriscloudaz/netflix:latest > trivyimage.txt" 
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://0D07D2E983FC86AFCE6D7BFFEAEA6B04.yl4.eu-north-1.eks.amazonaws.com') {
                            sh 'kubectl apply -f deployment.yml'
                            sh 'kubectl apply -f service.yml'
                            sh 'kubectl get svc -n webapps'
                        }
                    }
                }
            }
        }
    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'chriscloudaz@gmail.com',                               
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
