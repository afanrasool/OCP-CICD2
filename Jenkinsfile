 pipeline {
          agent {
            label 'jenkins-slave-image-mgmt'
          }
          stages {
            stage('Create Build') {
                        when {
                          expression {
                            openshift.withCluster() {
                              openshift.withProject(dev) {
                                return !openshift.selector("bc", "weather").exists();
                              }
                            }
                          }
                        }
                        steps {
                          script {
                            openshift.withCluster() {
                              openshift.withProject(dev) {
                                openshift.newBuild("openshift/nodejs~https://github.com/ArctiqTeam/weather-app.git", "--name=weather")
                              }
                            }
                          }
                        }
                      }
                      stage('Start Build') {
                        steps {
                          script {
                            openshift.withCluster() {
                              openshift.withProject(dev) {
                                openshift.selector("bc", "weather").startBuild("--wait=true")
                              }
                            }
                          }
                        }
                      }
                      stage('Deploy Image in Dev') {
                        when {
                          expression {
                            openshift.withCluster() {
                              openshift.withProject(dev) {
                                return !openshift.selector('dc', 'weather').exists()
                              }
                            }
                          }
                        }
                        steps {
                          script {
                            openshift.withCluster() {
                              openshift.withProject(dev) {
                                def app = openshift.newApp("weather:latest", "--name=weather")
                                app.narrow("svc").expose();
                                def dc = openshift.selector("dc", "weather")
                                while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                                    sleep 10
                                }
                                openshift.set("triggers", "dc/weather", "--manual")
                              }
                            }
                          }
                        }
                      }
                      stage('Deploy DEV') {
                        steps {
                          script {
                            openshift.withCluster() {
                              openshift.withProject(env.DEV_PROJECT) {
                                openshift.selector("dc", "weather").rollout().latest();
                              }
                            }
                          }
                        }
                      }
                      stage('Tag for QA') {
                        steps {
                          script {
                            openshift.withCluster() {
                              openshift.tag("${env.DEV_PROJECT}/weather:latest", "${env.QA_PROJECT}/weather:${QA_TAG}")
                            }
                          }
                        }
                      }
                      stage('Create QA') {
                        when {
                          expression {
                            openshift.withCluster() {
                              openshift.withProject(env.QA_PROJECT) {
                                return !openshift.selector('dc', 'weather').exists()
                              }
                            }
                          }
                        }
                        steps {
                          script {
                            openshift.withCluster() {
                              openshift.withProject(env.QA_PROJECT) {
                                def app = openshift.newApp("weather:${QA_TAG}", "--name=weather")
                                app.narrow("svc").expose();
                                def dc = openshift.selector("dc", "weather")
                                while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                                    sleep 10
                                }
                                openshift.set("triggers", "dc/weather", "--manual")
                              }
                            }
                          }
                        }
                      }
                      stage('Deploy QA') {
                        steps {
                          script {
                            openshift.withCluster() {
                              openshift.withProject(env.QA_PROJECT) {
                                openshift.selector("dc", "weather").rollout().latest();
                              }
                            }
                          }
                        }
                      }
                stage('Promote to PROD?') {
              steps {
                timeout(time:15, unit:'MINUTES') {
                    input message: "Promote to PROD?", ok: "Promote"
                }
              }
            }
           stage('Tag for Prod') {
               steps {
                   script {
                     openshift.withCluster() {
                      openshift.tag("${env.QA_PROJECT}/weather:${QA_TAG}", "${env.QA_PROJECT}/weather:${PRODA_TAG}")
                       openshift.tag("${env.QA_PROJECT}/weather:${QA_TAG}", "${env.QA_PROJECT}/weather:${PRODB_TAG}")
                            }
                          }
                        }
                      }
             stage('Copy Image to PROD A') {
              steps {
                  script {
                      openshift.withCluster() {
                          openshift.withProject("${QA_PROJECT}") {
                              withCredentials([string(credentialsId: 'QA_CREDS', variable: 'QA_CREDS'),
                                               string(credentialsId: 'PRODA_CREDS', variable: 'PRODA_CREDS')]) {
                                  // call like this so we dont print credentials to logs
                                  sh '''
                                        set -x
                                        skopeo --debug copy --src-tls-verify=false --dest-tls-verify=false --src-creds="${QA_CREDS}" --dest-creds="${PRODA_CREDS}" "${QA_REGISTRY}/${QA_PROJECT}/weather:${PRODA_TAG}" "${PRODA_REGISTRY}/${PRODA_PROJECT}/weather:${PRODA_TAG}"
                                      '''
                              }
                          }
                      }
                  }
              }
            }
           stage('Deploy Image in Prod A') {
                        steps {
                          script {
                            sh 'oc login https://console.ocp.lab.arctiq.ca:8443 --token=INKjqz61A4qzfBgZEv6q1RXN0xL_5UH62l4OAQCoc2A --insecure-skip-tls-verify -n weather && oc new-app weather/weather:promoteToProdA && oc expose svc/weather --hostname=weather.apps.ocp.lab.arctiq.ca'
                            slackSend (color: '#0eaf2b', message: "Success: Job - App is deployed in Prod A - check the weather here: http://weather.apps.ocp.lab.arctiq.ca")
                      }
                      }
          }
             stage('Copy Image to PROD B') {
              steps {
                  script {
                      openshift.withCluster() {
                          openshift.withProject("${QA_PROJECT}") {
                              withCredentials([string(credentialsId: 'QA_CREDS', variable: 'QA_CREDS'),
                                               string(credentialsId: 'PRODB_CREDS', variable: 'PRODB_CREDS')]) {
                                  // call like this so we dont print credentials to logs
                                  sh '''
                                        set -x
                                        skopeo --debug copy --src-tls-verify=false --dest-tls-verify=false --src-creds="${QA_CREDS}" --dest-creds="${PRODB_CREDS}" "${QA_REGISTRY}/${QA_PROJECT}/weather:${PRODB_TAG}" "${PRODB_REGISTRY}/${PRODB_PROJECT}/weather:${PRODB_TAG}"
                                      '''
                              }
                          }
                      }
                  }
              }
            }
           stage('Deploy Image in Prod B') {
                        steps {
                          script {
                            sh 'oc login https://console.gcp.event.openshift.ca:8443 --token=FH6uhELsjYScjsY1Xc4TXPMcuw9Kq8hVaSQ0w22RTdg --insecure-skip-tls-verify -n weather && oc new-app weather/weather:promoteToProdB && oc expose svc/weather --hostname=weather.apps.gcp.event.openshift.ca'
                            slackSend (color: '#0d3baf', message: "Success: Job - App is deployed in Prod B - check the weather here: http://weather.apps.gcp.event.openshift.ca")
                      }
                      }
          }
          }
        }
  output:
  resources:
  postCommit:
