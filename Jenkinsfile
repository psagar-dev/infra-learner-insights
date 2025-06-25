pipeline { 
    agent any

    parameters {
        string(name: 'SERVICE_NAME', defaultValue: 'frontend', description: 'The name of the service/chart to deploy')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'The Docker image tag to deploy')
    }

    stages {
        stage('Lint Helm Chart') {
            steps {
                script {
                    def serviceName = params.SERVICE_NAME
                    def chartPath = "./helm/${serviceName}"
                    echo "Linting Helm chart for ${serviceName}..."
                    sh "helm lint ${chartPath} -f ${chartPath}/values.yaml"
                }
            }
        }
        stage('Deploy with Helm') {
            steps {
                script {
                    def serviceName = params.SERVICE_NAME
                    def imageTag = params.IMAGE_TAG
                    def chartPath = "./helm/${serviceName}"

                    echo "Deploying ${serviceName} with image tag ${imageTag} using Helm..."

                    sh """
                        helm upgrade --install learner-insights-${serviceName} ${chartPath} \
                        -f ${chartPath}/values.yaml \
                        --set image.repository=securelooper/learner-insights-${serviceName} \
                        --set image.tag=${imageTag} \
                        --namespace learner-insights \
                        --create-namespace
                    """
                }
            }
        }
    }
}