pipeline {
    agent any

    environment {
        EC2_USER   = 'ubuntu'
        EC2_HOST   = '54.206.18.232'          // <-- replace with your EC2 public IP
        SSH_KEY_ID = 'ec2-ssh-key'           // Jenkins credential ID (SSH private key)
        REMOTE_DIR = '/home/ubuntu/petclinic'
        APP_PORT   = '8081'                  // 8081 avoids clashing with Jenkins on 8080
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/spring-projects/spring-petclinic.git'
            }
        }

        stage('Build') {
            steps {
                sh '''
                    chmod +x mvnw
                    ./mvnw clean package -DskipTests
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: [env.SSH_KEY_ID]) {
                    sh '''
                        # Ensure app directory exists on the server
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "mkdir -p ${REMOTE_DIR}"

                        # Copy the built jar
                        scp -o StrictHostKeyChecking=no target/spring-petclinic-*.jar \
                            ${EC2_USER}@${EC2_HOST}:${REMOTE_DIR}/petclinic.jar

                        # Stop old instance (if any) and start the new one
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "
                            pkill -f 'petclinic.jar' || true
                            sleep 2
                            nohup java -jar ${REMOTE_DIR}/petclinic.jar --server.port=${APP_PORT} \
                                > ${REMOTE_DIR}/app.log 2>&1 &
                        "
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    echo "Waiting for PetClinic to start..."
                    sleep 30
                    curl -f http://${EC2_HOST}:${APP_PORT} > /dev/null && echo "PetClinic is UP"
                '''
            }
        }
    }

    post {
        success {
            echo "Deployed successfully: http://${EC2_HOST}:${APP_PORT}"
        }
        failure {
            echo 'Deployment failed - check the stage logs above.'
        }
    }
}







