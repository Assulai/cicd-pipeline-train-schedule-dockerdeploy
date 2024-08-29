pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build("farajassulai/train-schedule")
                    app.inside {
                        // Check if the application is running inside the Docker container
                        sh 'echo $(curl localhost:8080)'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                withCredentials([usernamePassword(credentialsId: 'webserver_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        // Pull the new Docker image on the production server
                        sh "sshpass -p '${USERPASS}' ssh -o StrictHostKeyChecking=no ${USERNAME}@${prod_ip} \"docker pull farajassulai/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                            // Stop and remove the existing container
                            sh "sshpass -p '${USERPASS}' ssh -o StrictHostKeyChecking=no ${USERNAME}@${prod_ip} \"docker stop train-schedule\""
                            sh "sshpass -p '${USERPASS}' ssh -o StrictHostKeyChecking=no ${USERNAME}@${prod_ip} \"docker rm train-schedule\""
                        } catch (err) {
                            echo "Caught error: ${err}"
                        }
                        // Run the new container
                        sh "sshpass -p '${USERPASS}' ssh -o StrictHostKeyChecking=no ${USERNAME}@${prod_ip} \"docker run --restart always --name train-schedule -p 8080:8080 -d farajassulai/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    }
}
