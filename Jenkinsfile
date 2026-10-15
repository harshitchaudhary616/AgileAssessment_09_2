
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

                        # Apply both deployments and the Service.
                        kubectl apply -f deployment.yaml

                        # Determine the active, stable colour.
                        ACTIVE=$(kubectl get service $APP \
                            -o jsonpath='{.spec.selector.version}')

                        if [ "$ACTIVE" != "blue" ] && [ "$ACTIVE" != "green" ]; then
                            echo "ERROR: Service has no valid active colour."
                            exit 1
                        fi

                        if [ "$ACTIVE" = "blue" ]; then
                            TARGET=green
                        else
                            TARGET=blue
                        fi

                        DEPLOY=foodie-$TARGET

                        echo "Stable colour: $ACTIVE"
                        echo "Candidate colour: $TARGET"

                        # Deploy the new image to the inactive colour.
                        # To test a failed rollout, temporarily use:
                        # IMAGE_TAG=does-not-exist
                        # For normal releases, use the build number.
                        IMAGE_TAG=$BUILD_NUMBER

                        kubectl set image deployment/$DEPLOY \
                            food-ordering=$IMAGE:$IMAGE_TAG

                        # Wait for the candidate to become ready.
                        if ! kubectl rollout status deployment/$DEPLOY \
                            --timeout=120s; then

                            echo "Candidate failed. Keeping $ACTIVE active."

                            kubectl patch service $APP --type=merge \
                                -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$ACTIVE\\"}}}"

                            exit 1
                        fi

                        # Test the candidate before switching traffic.
                        POD=$(kubectl get pods \
                            -l app=foodie,version=$TARGET \
                            -o jsonpath='{.items[0].metadata.name}')

                        if ! kubectl exec "$POD" -- \
                            wget -qO- http://127.0.0.1/ >/dev/null; then

                            echo "Candidate health check failed. Keeping $ACTIVE active."

                            kubectl patch service $APP --type=merge \
                                -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$ACTIVE\\"}}}"

                            exit 1
                        fi

                        # Switch traffic only after candidate tests pass.
                        kubectl patch service $APP --type=merge \
                            -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$TARGET\\"}}}"

                        sleep 3

                        # Validate Service endpoints after the switch.
                        ENDPOINTS=$(kubectl get endpoints $APP \
                            -o jsonpath='{.subsets[*].addresses[*].ip}')

                        if [ -z "$ENDPOINTS" ]; then
                            echo "Post-switch check failed. Restoring $ACTIVE."

                            kubectl patch service $APP --type=merge \
                                -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$ACTIVE\\"}}}"

                            exit 1
                        fi

                        echo "Blue-Green deployment succeeded."
                        echo "Active colour: $TARGET"

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
            echo 'Build, test, image push and Blue-Green deployment succeeded.'
        }

        failure {
            echo 'Release failed. Verify that the previous stable colour remains active.'
        }
    }
}
