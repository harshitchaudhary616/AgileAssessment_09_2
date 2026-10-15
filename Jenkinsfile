
pipeline {
    agent any

    environment {
        IMAGE = 'perkeylynx/food-ordering'
        APP = 'food-ordering-service'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    docker build -t $IMAGE:$BUILD_NUMBER .
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
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
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

                        # Remember which colour currently serves traffic.
                        ACTIVE=$(kubectl get service $APP \
                          -o jsonpath='{.spec.selector.version}' 2>/dev/null || true)

                        if [ "$ACTIVE" != "blue" ] && [ "$ACTIVE" != "green" ]; then
                          ACTIVE=blue
                        fi

                        # Apply resources, then restore the active selector.
                        kubectl apply -f deployment.yaml
                        kubectl patch service $APP --type=merge \
                          -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$ACTIVE\\"}}}"

                        if [ "$ACTIVE" = "blue" ]; then
                          TARGET=green
                        else
                          TARGET=blue
                        fi

                        DEPLOY=foodie-$TARGET
                        echo "Stable version: $ACTIVE"
                        echo "Deploying new version: $TARGET"

                        # Update only the inactive deployment.
                        kubectl set image deployment/$DEPLOY \
                          food-ordering=$IMAGE:$BUILD_NUMBER

                        # Failed rollout exits before traffic is switched.
                        kubectl rollout status deployment/$DEPLOY --timeout=120s

                        # Verify every new pod can serve the page.
                        for i in 1 2 3; do
                          POD=$(kubectl get pods -l app=foodie,version=$TARGET \
                            -o jsonpath='{.items[0].metadata.name}')
                          kubectl exec "$POD" -- wget -qO- http://127.0.0.1/ >/dev/null
                        done

                        # Switch traffic to the healthy release.
                        kubectl patch service $APP --type=merge \
                          -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$TARGET\\"}}}"

                        # Confirm the service has endpoints.
                        sleep 3
                        ENDPOINTS=$(kubectl get endpoints $APP \
                          -o jsonpath='{.subsets[*].addresses[*].ip}')
                        if [ -z "$ENDPOINTS" ]; then
                          echo "No endpoints: restoring stable version $ACTIVE"
                          kubectl patch service $APP --type=merge \
                            -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"version\\":\\"$ACTIVE\\"}}}"
                          exit 1
                        fi

                        echo "Traffic switched to $TARGET"
                        kubectl get deployments
                        kubectl get pods
                        kubectl get service $APP
                    '''
                }
            }
        }
    }

    post {
        failure {
            echo 'Release failed. The previous stable colour should continue serving traffic.'
        }
        success {
            echo 'Blue-Green deployment completed successfully.'
        }
    }
}
