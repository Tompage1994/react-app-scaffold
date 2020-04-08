pipeline {
    options {
        timeout(time: 60, unit: 'MINUTES')
    }
    agent {
      node {
          label 'nodejs'
      }
    }

    environment {
        DEV_PROJECT = "tpage-dev"
        STAGE_PROJECT = "tpage-stage"
        APP_GIT_URL = "https://github.com/Tompage1994/react-app-scaffold.git"
        APP_NAME = "react-app-scaffold"
    }


    stages {

        stage('NPM Install') {
            steps {
                echo '### Installing NPM dependencies ###'
                sh '''
                        npm install
                   '''
            }
        }

        stage('Run Unit Tests') {
            steps {
                echo '### Running unit tests ###'
                sh 'npm run test:ci'
            }
        }

        stage('Run Linting Tools') {
            steps {
                echo '### Running eslint on code ###'
                sh 'npm run lint'
            }
        }

        stage('Launch new app in DEV env') {
            steps {
                echo '### Creating a new app in DEV env ###'
                sh '''
                        oc project ${DEV_PROJECT}

                        oc process -f .openshift-apply/dev_template.yml \
                        -n tpage-dev -p APP_NAME=${APP_NAME} \
                        -p SOURCE_REPOSITORY_URL="${APP_GIT_URL}" \
                        | oc apply -f -

                        oc start-build ${APP_NAME}
                '''
            }
        }

        stage('Wait for S2I build to complete') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject( "${DEV_PROJECT}" ) {
                            def bc = openshift.selector("bc", "${APP_NAME}")
                            bc.logs('-f')
                            def builds = bc.related('builds')
                            builds.untilEach(1) {
                                return (it.object().status.phase == "Complete")
                            }
                        }
                    }
                }
            }
        }

        stage('Wait for deployment in DEV env') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject( "${DEV_PROJECT}" ) {
                            def deployment = openshift.selector("dc", "${APP_NAME}").rollout()
                            openshift.selector("dc", "${APP_NAME}").related('pods').untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }
                    }
                }
            }
        }

        stage('Promote to Staging Env') {
            steps {
                timeout(time: 60, unit: 'MINUTES') {
                    input message: "Promote to Staging?"
                }
                script {
                    openshift.withCluster() {
                        openshift.tag("${DEV_PROJECT}/${APP_NAME}:latest", "${STAGE_PROJECT}/${APP_NAME}:stage")
                    }
                }
            }
        }

        stage('Deploy to Staging Env') {
            steps {
                echo '### Cleaning existing resources in Staging ###'
                sh '''
                        oc project ${STAGE_PROJECT}

                        oc process -f .openshift-apply/stage_template.yml \
                        -n tpage-stage -p APP_NAME=${APP_NAME} \
                        -p SOURCE_REPOSITORY_URL="${APP_GIT_URL}" \
                        | oc apply -f -

                        oc rollout latest dc/${APP_NAME}
                '''
            }
        }

        stage('Wait for deployment in Staging') {
            steps {
                sh "oc get route ${APP_NAME} -n ${STAGE_PROJECT} -o jsonpath='{ .spec.host }' --loglevel=4 > routehost"

                script {
                    routeHost = readFile('routehost').trim()

                    openshift.withCluster() {
                        openshift.withProject( "${STAGE_PROJECT}" ) {
                            def deployment = openshift.selector("dc", "${APP_NAME}").rollout()
                            openshift.selector("dc", "${APP_NAME}").related('pods').untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }

                        echo "Deployment to Staging env is complete. Access the app at the URL http://${routeHost}."
                    }
                }
            }
        }
    }
}