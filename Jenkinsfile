pipeline {
    agent {
        label 'AGENT-1'
    }

    options{
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds()
        //retry(1)
    }

    parameters{
        
        choices( name: 'ENVIRONMENT', choices: ['dev', 'qa', 'uat', 'pre-prod', 'prod'], description: 'please select the environment')
        string( name: 'VERSION', description: 'enter your application version')
        string(name: 'jira-id',  description: 'Enter your jira id')
    }

    enviroment{
        version = ''
        enviroment = ''
        account_id = ''
        region = 'us-east-1'
        project = 'roboshop'
        componenet = 'user'
        
    }

    stages{
        stage('set the environment'){
            steps{
            enviroment = params.ENVIRONMENT
            version = params.VERSION
            account_id = pipelineGlobals.getAccountID(enviroment)
        }
        }
        stage('run integration tests'){

            when {
                expression {params.ENVIRONMENT == 'qa'}
            }
        steps{

        }
        }

        stage('check jira'){
            when{
                 expression {params.ENVIRONMENT == 'prod'}
            }
            steps{
                sh """
                        echo "check jira status"
                        echo "check jira deployment window"
                        echo "fail pipeline if above two are not true"
                    """
            }
        }

        stage('deploy'){

        steps{
             
              
               withAWS(region: 'us-east-1', credentials: "aws-creds-${{environment}}") {

                 sh """
                   
                    aws eks update-kubeconfig --region ${region} --name ${project}-${environment}
                    cd helm
                    sed -i 's/IMAGE_VERSION/${appVersion}' values-${environment}.yaml
                    helm upgrade --install ${component} -n ${project} -f values-${environment}.yaml
                    if [ $? != 0 ]
                    then
                        echo "helm install/upgrade is failed...rolling back to previous version"
                        helm rollback ${component} -n ${project} -f values-${environment}.yaml

                        if [ $? != 0 ]
                        then
                           echo "helm rollback is failed...aborting the pipeline"
                           exit 1
                        else
                            echo "helm rollback is success"
                        fi
                    else
                        echo "helm install/upgradation is success"
                    fi

                    

                 """
               }

        }
    }
        stage('Deploy with Helm') {
            steps {
                script {
                    // Update Kubeconfig
                    def kubeConfigStatus = sh(script: "aws eks update-kubeconfig --region ${region} --name ${project}-${environment}", returnStatus: true)

                    if (kubeConfigStatus != 0) {
                        error "Failed to update kubeconfig for EKS. Aborting pipeline."
                    }
                    
                    // Navigate to helm directory and modify values file
                    echo "Navigating to helm directory and updating values-${environment}.yaml"
                    def sedStatus = sh(script: """
                        cd helm
                        sed -i 's/IMAGE_VERSION/${appVersion}/' values-${environment}.yaml
                    """, returnStatus: true)
                    if (sedStatus != 0) {
                        error "Failed to update values-${environment}.yaml. Aborting pipeline."
                    }

                    // Run Helm upgrade/install
                    def helmUpgradeStatus = sh(script: "helm upgrade --install ${component} -n ${project} -f values-${environment}.yaml", returnStatus: true)

                    if (helmUpgradeStatus != 0) {
                        echo "Helm upgrade/install failed. Attempting rollback..."
                        
                        // Attempt to rollback
                        def rollbackStatus = sh(script: "helm rollback ${component} -n ${project}", returnStatus: true)

                        if (rollbackStatus != 0) {
                            error "Helm rollback failed. Aborting pipeline."
                        } else {
                            echo "Helm rollback succeeded."
                        }
                    }

                    else {
                        echo "Helm upgrade/install succeeded."
                    }
                }
            }
        }


    }

}
