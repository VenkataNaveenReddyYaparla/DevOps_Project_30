pipeline {
    agent any
    tools {
        jdk "JDK 21"
        maven "Maven"
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/VenkataNaveenReddyYaparla/DevOps_Project_30.git'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqubeserver') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Blogging-app -Dsonar.projectKey=Blogging-app \
                          -Dsonar.java.binaries=target'''
                }
            }
        }
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        stage('Docker Build & Tag') {
            steps {
                script{
                withDockerRegistry(credentialsId: 'dockertoken', url: 'https://index.docker.io/v1/') {
                sh "docker build -t naveenreddy9/devops_30 ."
                }
                }
            }
        }
        stage('Docker Push Image') {
            steps {
                script{
                withDockerRegistry(credentialsId: 'dockertoken', url: 'https://index.docker.io/v1/') {
                    sh "docker push naveenreddy9/devops_30"
                }
                }
            }
        }

        
    }  // Closing stages
}  // Closing pipeline


