pipeline {
    agent any

    triggers {
        pollSCM('H/1 * * * *')
    }

    environment {
        IMAGE = 'perkeylynx/food-ordering'
        PATH = "/Applications/Docker.app/Contents/Resources/bin:/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Tools') {
            steps {
                sh '''
                    set -e
                    git --version
                    docker --version
                    kubectl version --client
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    set -e
                    docker build -t $IMAGE:$BUILD_NUMBER .
                '''
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                    set -e
                    docker rm -f food-test || true
                    docker run -d --name food-test -p 18082:80 $IMAGE:$BUILD_NUMBER
                    sleep 3
                    curl --fail http://localhost:18082/
                    docker rm -f food-test
                '''
            }

            post {
                always {
                    sh 'docker rm -f food-test || true'
                }
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        set -e
                        echo "$DH_TOKEN" | docker login --username "$DH_USER" --password-stdin
                        docker push $IMAGE:$BUILD_NUMBER
                        docker tag $IMAGE:$BUILD_NUMBER $IMAGE:latest
                        docker push $IMAGE:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy and Rollback') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kuberconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh '''
                        set -e

                        kubectl config current-context
                        kubectl get nodes

                        kubectl apply -f deployment.yaml

                        kubectl set image deployment/food-ordering \
                            food-ordering=$IMAGE:$BUILD_NUMBER

                        if ! kubectl rollout status deployment/food-ordering --timeout=120s; then
                            echo "Rollout failed. Attempting rollback."

                            kubectl rollout undo deployment/food-ordering
                            kubectl rollout status deployment/food-ordering --timeout=120s

                            exit 1
                        fi

                        kubectl get deployment food-ordering
                        kubectl get pods -l app=food-ordering
                        kubectl get service food-ordering-service
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Food Ordering pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
    }
}
