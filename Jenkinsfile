
pipeline {
    agent any

    environment {
        PATH = "/Applications/Docker.app/Contents/Resources/bin:/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"
        IMAGE = 'perkeylynx/food-ordering'
        APP = 'food-ordering-service'
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
                    docker --version
                    kubectl version --client
                    git --version
                '''
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    docker build -t $IMAGE:$BUILD_NUMBER .

                    docker rm -f food-test-$BUILD_NUMBER 2>/dev/null || true

                    docker run -d --name food-test-$BUILD_NUMBER \
                        -p 18082:80 $IMAGE:$BUILD_NUMBER

                    sleep 3
                    curl --fail http://localhost:18082/

                    docker rm -f food-test-$BUILD_NUMBER
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh '''
                        echo "$DH_PASS" | docker login \
                            -u "$DH_USER" --password-stdin
                        docker push $IMAGE:$BUILD_NUMBER
                        docker logout
                    '''
                }
            }
        }

        stage('Blue-Green Deploy') {
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig',
                    variable: 'KCFG'
                )]) {
                    sh '''
                        export KUBECONFIG="$KCFG"
                        kubectl config use-context docker-desktop

                        kubectl apply -f deployment.yaml

                        ACTIVE=$(kubectl get service $APP \
                            -o jsonpath='{.spec.selector.version}' 2>/dev/null || true)

                        if [ "$ACTIVE" != "blue" ] && [ "$ACTIVE" != "green" ]; then
                            echo "ERROR: Service must select blue or green."
                            exit 1
                        fi

                        if [ "$ACTIVE" = "blue" ]; then
                            TARGET=green
                        else
                            TARGET=blue
                        fi

                        DEPLOY=foodie-$TARGET

                        echo "Stable version: $ACTIVE"
                        echo "Candidate version: $TARGET"

                        # Use the image created and pushed by this build.
                        IMAGE_TAG=$BUILD_NUMBER

                        kubectl set image deployment/$DEPLOY \
                            food-ordering=$IMAGE:$IMAGE_TAG

                        # Wait for the candidate deployment to finish.
                        if ! kubectl rollout status deployment/$DEPLOY \
                            --timeout=120s; then

                            echo "Rollout failed. Keeping $ACTIVE active."

                            kubectl patch service $APP --type=merge \
                                -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$ACTIVE\\"}}}"

                            exit 1
                        fi

                        # Select only a Running candidate pod.
                        POD=""

                        for i in $(seq 1 30); do
                            POD=$(kubectl get pods \
                                -l app=foodie,version=$TARGET \
                                --field-selector=status.phase=Running \
                                -o jsonpath='{.items[0].metadata.name}')

                            if [ -n "$POD" ]; then
                                break
                            fi

                            sleep 2
                        done

                        if [ -z "$POD" ]; then
                            echo "No running candidate pod. Keeping $ACTIVE active."

                            kubectl patch service $APP --type=merge \
                                -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$ACTIVE\\"}}}"

                            exit 1
                        fi

                        # Test the candidate before switching traffic.
                        if ! kubectl exec "$POD" -- \
                            wget -qO- http://127.0.0.1/ >/dev/null; then

                            echo "Candidate test failed. Keeping $ACTIVE active."

                            kubectl patch service $APP --type=merge \
                                -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$ACTIVE\\"}}}"

                            exit 1
                        fi

                        # Switch traffic after the candidate test passes.
                        kubectl patch service $APP --type=merge \
                            -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$TARGET\\"}}}"

                        sleep 3

                        ENDPOINTS=$(kubectl get endpoints $APP \
                            -o jsonpath='{.subsets[*].addresses[*].ip}')

                        if [ -z "$ENDPOINTS" ]; then
                            echo "No endpoints. Restoring $ACTIVE."

                            kubectl patch service $APP --type=merge \
                                -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$ACTIVE\\"}}}"

                            exit 1
                        fi

                        echo "Blue-Green deployment successful."
                        echo "Active version: $TARGET"

                        kubectl get deployments
                        kubectl get pods
                        kubectl get service $APP
                        kubectl get endpoints $APP
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f food-test-$BUILD_NUMBER 2>/dev/null || true
            '''
        }

        success {
            echo 'Build, test, push and Blue-Green deployment succeeded.'
        }

        failure {
            echo 'Release failed. Verify the active Service selector and website.'
        }
    }
}
