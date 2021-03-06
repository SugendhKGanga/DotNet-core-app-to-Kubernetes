/*
    This is an example pipeline that implement full CI/CD for .Net core application packed in a Docker image.

    The pipeline is made up of 6 main steps
    1. Git clone and setup
    2. Build and local tests
    3. Publish Docker and Helm
    4. Deploy to dev and test
    5. Deploy to staging and test
    6. Optionally deploy to production and test
 */

/*
    Create the kubernetes namespace
 */
def createNamespace (namespace) {
    echo "Creating namespace ${namespace} if needed"

    sh "[ ! -z \"\$(kubectl get ns ${namespace} -o name 2>/dev/null)\" ] || kubectl create ns ${namespace}"
}

/*
    Helm install
 */
def helmInstall (namespace, release) {
    echo "Installing ${release} in ${namespace}"

    script {
        release = "${release}-${namespace}"
        sh "helm repo add helm ${HELM_REPO}; helm repo update"
        sh """
            helm upgrade --install --namespace ${namespace} ${release} \
                --set imagePullSecrets=${IMG_PULL_SECRET} \
                --set image.repository=${DOCKER_REG_HUB}/${IMAGE_NAME},image.tag=${DOCKER_TAG} helm/dotnetcore
        """
        sh "sleep 5"
    }
}

/*
    Helm delete (if exists)
 */
def helmDelete (namespace, release) {
    echo "Deleting ${release} in ${namespace} if deployed"

    script {
        release = "${release}-${namespace}"
        sh "[ -z \"\$(helm ls --short ${release} 2>/dev/null)\" ] || helm delete --purge ${release}"
    }
}

/*
    Run a curl against a given url
 */
def curlRun (url, out) {
    echo "Running curl on ${url}"

    script {
        if (out.equals('')) {
            out = 'http_code'
        }
        echo "Getting ${out}"
            def result = sh (
                returnStdout: true,
                script: "curl --output /dev/null --silent --connect-timeout 5 --max-time 5 --retry 5 --retry-delay 5 --retry-max-time 30 --write-out \"%{${out}}\" ${url}"
        )
        echo "Result (${out}): ${result}"
    }
}

/*
    Test with a simple curl and check we get 200 back
 */
def curlTest (namespace, out) {
    echo "Running tests in ${namespace}"

    script {
        if (out.equals('')) {
            out = 'http_code'
        }

        // Get deployment's service IP
        def svc_ip = sh (
                returnStdout: true,
                script: "kubectl get svc -n ${namespace} | grep ${ID} | awk '{print \$3}'"
        )

        if (svc_ip.equals('')) {
            echo "ERROR: Getting service IP failed"
            sh 'exit 1'
        }

        echo "svc_ip is ${svc_ip}"
        url = 'http://' + svc_ip

        curlRun (url, out)
    }
}

/*
    This is the main pipeline section with the stages of the CI/CD
 */
pipeline {

    options {
        // Build auto timeout
        timeout(time: 60, unit: 'MINUTES')
    }

    // Some global default variables
    environment {
        IMAGE_NAME = 'dotnetcore'
        TEST_LOCAL_PORT = 8817
        DEPLOY_PROD = false
        PARAMETERS_FILE = "${JENKINS_HOME}/parameters.groovy"
    }

    parameters {
        string (name: 'GIT_BRANCH',           defaultValue: 'master',  description: 'Git branch to build')
        booleanParam (name: 'DEPLOY_TO_PROD', defaultValue: false,     description: 'If build and tests are good, proceed and deploy to production without manual approval')


        // The commented out parameters are for optionally using them in the pipeline.
        // In this example, the parameters are loaded from file ${JENKINS_HOME}/parameters.groovy later in the pipeline.
        // The ${JENKINS_HOME}/parameters.groovy can be a mounted secrets file in your Jenkins container.

        string (name: 'DOCKER_REG',       defaultValue: 'Docker-Hub',                   description: 'Docker registry')
        string (name: 'DOCKER_REG_HUB',       defaultValue: 'sugendh',                   description: 'Docker registry_hub')
        
        string (name: 'DOCKER_TAG',       defaultValue: 'latest',                                     description: 'Docker tag')
        string (name: 'DOCKER_USR',       defaultValue: 'sugendh',                                   description: 'Your helm repository user')
        string (name: 'DOCKER_PSW',       defaultValue: 'Password123',                                description: 'Your helm repository password')
        string (name: 'IMG_PULL_SECRET',  defaultValue: 'docker-reg-secret',                       description: 'The Kubernetes secret for the Docker registry (imagePullSecrets)')
        string (name: 'HELM_REPO',        defaultValue: 'http://35.224.229.155/artifactory/helm-local', description: 'Your helm repository')
        string (name: 'HELM_USR',         defaultValue: 'admin',                                   description: 'Your helm repository user')
        string (name: 'HELM_PSW',         defaultValue: 'AP8j5ZZuUam1u3zX',                                description: 'Your helm repository password')

    }

    // In this example, all is built and run from the master
    agent { node { label 'master' } }

    // Pipeline stages
    stages {

        ////////// Step 1 //////////
        stage('Git clone and setup') {
            steps {
                echo "Check out the code!"
                
                // Validate kubectl
                sh "kubectl cluster-info"

                // Init helm client
                sh "helm init"

                // Load Docker registry and Helm repository configurations from file

                echo "DOCKER_REG is ${DOCKER_REG}"
                echo "HELM_REPO  is ${HELM_REPO}"

                // Define a unique name for the tests container and helm release
                script {
                    branch = GIT_BRANCH.replaceAll('/', '-').replaceAll('\\*', '-')
                    ID = "${IMAGE_NAME}-${DOCKER_TAG}-${branch}"

                    echo "Global ID set to ${ID}"
                }
            }
        }

        ////////// Step 2 //////////
        stage('Build and tests') {
            steps {
                echo "Building application and Docker image"
                sh "chmod 777 ${WORKSPACE}/build.sh"
                //sh "chmod 777 ${WORKSPACE}/dotnet_sdk_install.sh"
                
                //sh "${WORKSPACE}/dotnet_sdk_install.sh"
                //sh "dotnet publish -c Release"
                sh "${WORKSPACE}/build.sh --build --registry ${DOCKER_REG} --tag ${DOCKER_TAG} --docker_usr ${DOCKER_USR} --docker_psw ${DOCKER_PSW}"

                echo "Running tests"

                // Kill container in case there is a leftover
                sh "[ -z \"\$(docker ps -a | grep ${ID} 2>/dev/null)\" ] || docker rm -f ${ID}"

                echo "Starting ${IMAGE_NAME} container"
                sh "docker run --detach --name ${ID} --rm --publish ${TEST_LOCAL_PORT}:8080 ${DOCKER_REG_HUB}/${IMAGE_NAME}:${DOCKER_TAG}"

                script {
                    host_ip = sh(returnStdout: true, script: '/sbin/ip route | awk \'/default/ { print $3 ":${TEST_LOCAL_PORT}" }\'')
                }
            }
        }

        // Run the 3 tests on the currently running ACME Docker container
        stage('Local tests') {
            parallel {
                stage('Curl http_code') {
                    steps {
                        //curlRun ("http://localhost:${TEST_LOCAL_PORT}", 'http_code')
                        echo "to do!"
                    }
                }
                stage('Curl total_time') {
                    steps {
                        //curlRun ("http://localhost:${TEST_LOCAL_PORT}", 'total_time')
                        echo "to do"
                    }
                }
                stage('Curl size_download') {
                    steps {
                        //curlRun ("http://localhost:${TEST_LOCAL_PORT}", 'size_download')
                        echo "to do"
                    }
                }
            }
        }

        ////////// Step 3 //////////
        stage('Publish Docker and Helm') {
            steps {
                echo "Stop and remove container"
                sh "docker stop ${ID}"

                echo "Pushing ${DOCKER_REG}/${IMAGE_NAME}:${DOCKER_TAG} image to registry"
                sh "${WORKSPACE}/build.sh --push --registry ${DOCKER_REG} --tag ${DOCKER_TAG} --docker_usr ${DOCKER_USR} --docker_psw ${DOCKER_PSW}"

                //echo "Packing helm chart"
                //sh "${WORKSPACE}/build.sh --pack_helm --push_helm --helm_repo ${HELM_REPO} --helm_usr ${HELM_USR} --helm_psw ${HELM_PSW}"
            }
        }

        ////////// Step 4 //////////
        stage('Deploy to dev') {
            steps {
                script {
                    namespace = 'development'

                    echo "Deploying application ${ID} to ${namespace} namespace"
                    createNamespace (namespace)



                    
                }
                
                    sh "kubectl run hello-dotnet --image=${DOCKER_REG_HUB}/${IMAGE_NAME}:${DOCKER_TAG} --port=8080 -n development"
                    sleep 60
                    sh "kubectl expose deployment hello-dotnet --type=LoadBalancer --port=8080 -n development"
            }
        }

        // Run the 3 tests on the deployed Kubernetes pod and service
        stage('Dev tests') {
            parallel {
                stage('Curl http_code') {
                    steps {
                        //curlTest (namespace, 'http_code')
                        echo "to do"
                    }
                }
                stage('Curl total_time') {
                    steps {
                        //curlTest (namespace, 'time_total')
                        echo "to do"
                    }
                }
                stage('Curl size_download') {
                    steps {
                       // curlTest (namespace, 'size_download')
                        echo "to do"
                    }
                }
            }
        }
        
        ////////// Step 5 //////////
        stage('Deploy to staging') {
            steps {
                script {
                    namespace = 'staging'

                    echo "Deploying application ${ID} to ${namespace} namespace"
                    createNamespace (namespace)



                    
                }
                
                    sh "kubectl run hello-dotnet --image=${DOCKER_REG_HUB}/${IMAGE_NAME}:${DOCKER_TAG} --port=8080 -n staging"
                    sleep 60
                    sh "kubectl expose deployment hello-dotnet --type=LoadBalancer --port=8080 -n staging"
            }
        }

        // Run the 3 tests on the deployed Kubernetes pod and service
        stage('Staging tests') {
            parallel {
                stage('Curl http_code') {
                    steps {
                        //curlTest (namespace, 'http_code')
                        echo "to do"
                    }
                }
                stage('Curl total_time') {
                    steps {
                        //curlTest (namespace, 'time_total')
                        echo "to do"
                    }
                }
                stage('Curl size_download') {
                    steps {
                       // curlTest (namespace, 'size_download')
                        echo "to do"
                    }
                }
            }
        }
        
        ////////// Step 6 //////////
        // Waif for user manual approval, or proceed automatically if DEPLOY_TO_PROD is true
        stage('Go for Production?') {
            when {
                allOf {
                    environment name: 'GIT_BRANCH', value: 'master'
                    environment name: 'DEPLOY_TO_PROD', value: 'false'
                }
            }
            steps {
                // Prevent any older builds from deploying to production
                milestone(1)
                input 'Proceed and deploy to Production?'
                milestone(2)
                script {
                    DEPLOY_PROD = true
                }
            }
        }
        stage('Deploy to Production') {
            when {
                anyOf {
                    expression { DEPLOY_PROD == true }
                    environment name: 'DEPLOY_TO_PROD', value: 'true'
                }
            }
            steps {
                script {
                    DEPLOY_PROD = true
                    namespace = 'production'
                    echo "Deploying application ${IMAGE_NAME}:${DOCKER_TAG} to ${namespace} namespace"
                    createNamespace (namespace)
                    // Deploy with helm
                    echo "Deploying"
                    //helmInstall (namespace, "${ID}")
                }
					sh "kubectl run hello-dotnet --image=${DOCKER_REG_HUB}/${IMAGE_NAME}:${DOCKER_TAG} --port=8080 -n production"
                    sleep 60
                    sh "kubectl expose deployment hello-dotnet --type=LoadBalancer --port=8080 -n production"
            }
        }
        // Run the 3 tests on the deployed Kubernetes pod and service
        stage('Production tests') {
            when {
                expression { DEPLOY_PROD == true }
            }
            parallel {
                stage('Curl http_code') {
                    steps {
                        //curlTest (namespace, 'http_code')
						echo "to do"
                    }
                }
                stage('Curl total_time') {
                    steps {
                        //curlTest (namespace, 'time_total')
						echo "to do"
                    }
                }
                stage('Curl size_download') {
                    steps {
                        //curlTest (namespace, 'size_download')
						echo "to do"
                    }
                }
            }
        }

    }

}
