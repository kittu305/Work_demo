pipeline {
    agent any
    
    stages {
        /*stage('Checkout') {
            steps {
                // this Checkout will clone your source code from the github repository
                // You may need to provide credentials or SSH keys for authentication
                // This step is not required when this is a non-declarative pipeline 
                git credentialsId: 'hhk', url: 'git@github.com:blueed/Queue_App_UI.git'
            }
        }*/
        
        stage('Installing Dependencies') {
            steps {
                // Installing project dependencies, remenber to install LTS [18] version of node in node system
                // sh 'curl -s https://deb.nodesource.com/setup_18.x'
                // sh 'apt install nodejs -y'
                // sh 'npm install'
                sh 'npm install expo-cli --force'
            }
        }
        
        stage('Build') {
            steps {
                // build web application for production
                sh 'npm run web-build:prod'
            } 
        }
        
        stage('web hosting') {
            steps {
                // Most probably this step is not needed when you bulding application for android or ios platforms
                // Make sure you give 777 permisions to source and destination folders
                sh 'cd /home'
                sh 'cp -r "/var/lib/jenkins/workspace/QUEUE/web-build"/* "/var/www/QUEUE"/'
            }  
        }
        
        stage('Building docker immage') {
            steps {
                // executing shell commands to build custom docker immages
                // Make sure you give 777 permision to docker [sudo chmod 777 /var/run/docker.sock]
                // sudo usermod -a -G docker jenkins--this will taakeout the permision issue with docker.sock
                sh "docker build -t queue_${env.BUILD_NUMBER} ."
            } 
        }

        stage('Scaning with Trivy') {
            environment {
                // location of trivy template
                TRIVY_TEMPLATE = '@/usr/local/share/trivy/templates/html.tpl'
            }    
            steps {
                // install trivy (https://aquasecurity.github.io/trivy/v0.20.2/getting-started/installation/)
                // output file will be saves to workspace (QUEUE) folder
                script {
                    sh "trivy image -f template -t ${TRIVY_TEMPLATE} -o report_${env.BUILD_NUMBER}.html queue_${env.BUILD_NUMBER}"
                    def trivyExitCode = sh(script: "trivy image -f template -t ${TRIVY_TEMPLATE} -o report_${env.BUILD_NUMBER}.html queue_${env.BUILD_NUMBER}", returnStatus: true)
                    // sh "trivy image -f template -t ${TRIVY_TEMPLATE} -o ${TRIVY_REPORT} --exit-code 1 --severity CRITICAL queue_${env.BUILD_NUMBER}"
                    if (trivyExitCode != 0) {
                        // Additional commands to run if trivy fails
                        sh 'time trivy image --clear-cache'
                        sh 'time trivy image --download-db-only --no-progress --cache-dir .trivycache/'
                        sh "time trivy image --timeout 30m --cache-dir .trivycache/ --severity UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL --no-progress -f template -t ${TRIVY_TEMPLATE} -o report_${env.BUILD_NUMBER}.html queue_${env.BUILD_NUMBER}"
                    }    
                }
            }
        }

        stage('Docker login') {
            environment {
                // calling configured creds from jenkins using env varibles
                registry = "master-node:5000"
                docker_creds = credentials('Docker_creds')
            }
            steps {
                // Docker loginig in using creds(k8smaster.io:5000)
                sh 'docker login -u $docker_creds_USR -p $docker_creds_PSW https://master-node:5000'
            }
        }  
            
        stage('Pushing to registry') {
            steps {
                sh "docker tag queue_${env.BUILD_NUMBER} master-node:5000/queue_ui:${env.BUILD_NUMBER}"
                sh "docker push master-node:5000/queue_ui:${env.BUILD_NUMBER}"
                sh "docker rmi -f queue_${env.BUILD_NUMBER}"
                sh "docker rmi -f master-node:5000/queue_ui:${env.BUILD_NUMBER}"
            }
        }
    }

    post {
        always {
            //copying artifacts and storing them for backup
            //sh 'cd /var/lib/jenkins/workspace/QUEUE/web-build/'
            sh "zip -r /var/lib/jenkins/workspace/QUEUE/web-build/QUEUE_${env.BUILD_NUMBER}.zip web-build"
            sh 'mv "/var/lib/jenkins/workspace/QUEUE/web-build/QUEUE_${BUILD_NUMBER}.zip" "/var/jen_artifacts/"'
            sh "mv /var/lib/jenkins/workspace/QUEUE/report_${env.BUILD_NUMBER}.html /var/trivydocs/queue/report_${env.BUILD_NUMBER}.html"
            // Cleaning Project Files
            cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: false, cleanWhenSuccess: true, cleanWhenUnstable: false, deleteDirs: true)
        }
        success {
            echo 'Build successful...'
        }
        
        failure {
            echo 'Build failed... :('
        }
    }
}
