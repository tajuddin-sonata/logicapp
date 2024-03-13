pipeline {
    agent {
        label 'jenkins-slave'
    }

    parameters{
        choice(name: 'ENVIRONMENT', choices:[
            'dev',
            'staging',
            'prod'],
            description: 'Choose which environment to deploy to.')
        
        string(name: 'VERSION', description: 'Explicit version to deploy (i.e., "v0.1-51-g87b72a"). Leave blank to build latest commit')


        string(name: 'SUBSCRIPTION', defaultValue:'48986b2e-5349-4fab-a6e8-d5f02072a4b8', description: ''' select subscription as:
            48986b2e-5349-4fab-a6e8-d5f02072a4b8
            34b1c36e-d8e8-4bd5-a6f3-2f92a1c0626e
            70c3af66-8434-419b-b808-0b3c0c4b1a04''')

        string(name: 'REGION', defaultValue:'eastus',  description: '''Region to Deploy to.
        eastus, eastus2, westus, westus2, 
        southindia, centralindia, westindia''')

        string(name: 'RESOURCE_GROUP_NAME', defaultValue:'ssna-rg-cca-dev-eus', description: ''' Azure Resource Group in which the FunctionApp need to deploy.
            ssna-rg-cca-dev-eus
            ssna-rg-cca-stg-eus
            ssna-rg-cca-prd-eus
            ''')

        string(name: 'CONFIGURE_FUNCTIONAPP_NAME', defaultValue: 'ssna-func-cca-dev-eastus-wfconfigure', description: '''The name of FunctionApp 
            ssna-func-cca-dev-eastus-wfconfigure
            ssna-func-cca-stg-eastus-wfconfigure
            ssna-func-cca-prd-eastus-wfconfigure
            ''' )

        string(name: 'TRANSCODE_FUNCTIONAPP_NAME', defaultValue: 'ssna-func-cca-dev-eastus-wftranscode', description: '''The name of FunctionApp 
            ssna-func-cca-dev-eastus-wftranscode
            ssna-func-cca-stg-eastus-wftranscode
            ssna-func-cca-prd-eastus-wftranscode
            ''' )

        string(name: 'TRANSCRIBE_FUNCTIONAPP_NAME', defaultValue: 'ssna-func-cca-dev-eastus-wftranscribe', description: '''The name of FunctionApp 
            ssna-func-cca-dev-eastus-wftranscribe
            ssna-func-cca-stg-eastus-wftranscribe
            ssna-func-cca-prd-eastus-wftranscribe
            ''' )

        string(name: 'ANALYSE_FUNCTIONAPP_NAME', defaultValue: 'ssna-func-cca-dev-eastus-wfanalyse', description: '''The name of FunctionApp 
            ssna-func-cca-dev-eastus-wfanalyse
            ssna-func-cca-stg-eastus-wfanalyse
            ssna-func-cca-prd-eastus-wfanalyse
            ''' )

        string(name: 'REDACT_FUNCTIONAPP_NAME', defaultValue: 'ssna-func-cca-dev-eastus-wfredact', description: '''The name of FunctionApp 
            ssna-func-cca-dev-eastus-wfredact
            ssna-func-cca-stg-eastus-wfredact
            ssna-func-cca-prd-eastus-wfredact
            ''' )
        
        string(name: 'AZURE_LOGICAPP_NAME', defaultValue:'ssna-logicapp-cca-dev-eastus', description: '''The name of LogicApp to deploy
            ssna-logicapp-cca-dev-eastus
            ssna-logicapp-cca-stg-eastus
            ssna-logicapp-cca-prd-eastus
            ''' )


    }

    environment {
        AZURE_CLIENT_ID = credentials('azurerm_client_id')
        AZURE_CLIENT_SECRET = credentials('azurerm_client_secret')
        AZURE_TENANT_ID = credentials('azurerm_tenant_id')
        SONARQUBE_SCANNER_HOME = tool 'sonarscanner-5'
        logicAppResourceId="/subscriptions/${params.SUBSCRIPTION}/resourceGroups/${params.RESOURCE_GROUP_NAME}/providers/Microsoft.Web/sites/${params.AZURE_LOGICAPP_NAME}"
    }

    stages {
        stage('Checkout') {
            steps {
                // checkout scm
                git branch: 'main', url: 'https://github.com/tajuddin-sonata/logicapp.git'

            }
        }

        stage('Check/install Azure Tools') {
            steps {
                script {

                    // check if Azure CLI is installed, if not installed then install it.
                    def azcliVersion = sh(script: 'az -v', returnStatus: true)
                    if (azcliVersion != 0) {
                        echo "Azure CLI is not installed, installing now..."
                        // Install Azure function core tool
                        sh 'sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc'
                        sh 'sudo dnf install -y https://packages.microsoft.com/config/rhel/9.0/packages-microsoft-prod.rpm'
                        sh 'sudo dnf install azure-cli -y'
                    } else {
                        def az_installedVersion = sh(script: 'az -v', returnStdout: true).trim()
                        echo "func core tool is already installed (Version: ${az_installedVersion})"
                    }

                    // check if nodejs is installed, if not installed then install it.
                    def nodeVersion = sh(script: 'node -v', returnStatus: true)
                    if (nodeVersion != 0) {
                        echo "Node.js is not installed, installing now..."
                        // Install Node.js
                        // sh 'curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -'
                        sh 'sudo yum install -y nodejs'
                    } else {
                        def node_installedVersion = sh(script: 'node -v', returnStdout: true).trim()
                        echo "Node.js is already installed (Version: ${node_installedVersion})"
                    }

                    // check if Azure function core tool is installed, if not installed then install it.
                    def funcVersion = sh(script: 'func -v', returnStatus: true)
                    if (funcVersion != 0) {
                        echo "Azure function core tool is not installed, installing now..."
                        // Install Azure function core tool
                        sh 'sudo npm i -g azure-functions-core-tools@4 --unsafe-perm true'
                    } else {
                        def func_installedVersion = sh(script: 'func -v', returnStdout: true).trim()
                        echo "Azure function core tool is already installed (Version: ${func_installedVersion})"
                    }
                }
            }
        }

        stage ('replace variable in code') {
            steps {
                script {
                    sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant $AZURE_TENANT_ID'
                    sh "az account set --subscription ${params.SUBSCRIPTION}"
                    
                    def configure_func_url = sh(script: "func azure functionapp list-functions ${params.CONFIGURE_FUNCTIONAPP_NAME} --show-keys | awk '/Invoke url:/ {print \$3}'", returnStdout: true).trim()
                    echo "configure_func_url: ${configure_func_url}"

                    def transcode_func_url = sh(script: "func azure functionapp list-functions ${params.TRANSCODE_FUNCTIONAPP_NAME} --show-keys | awk '/Invoke url:/ {print \$3}'", returnStdout: true).trim()
                    echo "transcode_func_url: ${transcode_func_url}"
                    
                    def transcribe_func_url = sh(script: "func azure functionapp list-functions ${params.TRANSCRIBE_FUNCTIONAPP_NAME} --show-keys | awk '/Invoke url:/ {print \$3}'", returnStdout: true).trim()
                    echo "transcribe_func_url: ${transcribe_func_url}"
                    
                    def analyse_func_url = sh(script: "func azure functionapp list-functions ${params.ANALYSE_FUNCTIONAPP_NAME} --show-keys | awk '/Invoke url:/ {print \$3}'", returnStdout: true).trim()
                    echo "analyse_func_url: ${analyse_func_url}"
                    
                    def redact_func_url = sh(script: "func azure functionapp list-functions ${params.REDACT_FUNCTIONAPP_NAME} --show-keys | awk '/Invoke url:/ {print \$3}'", returnStdout: true).trim()
                    echo "redact_func_url: ${redact_func_url}"
                    
                    sh """
                        cd src/workflow1/
                        sed -i 's|\$CONFIGURE_FUNC_URL|${configure_func_url}|g' workflow.json
                        sed -i 's|\$TRANSCODE_FUNC_URL|${transcode_func_url}|g' workflow.json
                        sed -i 's|\$TRANSCRIBE_FUNC_URL|${transcribe_func_url}|g' workflow.json
                        sed -i 's|\$ANALYSE_FUNC_URL|${analyse_func_url}|g' workflow.json
                        sed -i 's|\$REDACT_FUNC_URL|${redact_func_url}|g' workflow.json

                    """
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo "SonarQube Analysis !!"
                withSonarQubeEnv('sonarqube-9.9') {
                    sh '/opt/sonarscanner/bin/sonar-scanner'
                }
            }
        }
        
        stage('Deploy artifact to Nexus & LogicApp') {
            steps {
                script {
                    echo "Deploy artifact to Nexus"
                    def ver = params.VERSION
                    sh """
                        #!/bin/bash
                
                        if [ -z "$ver" ]; then
                            artifact_version=\$(git describe --tags)
                            echo "\${artifact_version}" > src/version.txt
                            cd src/
                            zip -r "../az-ci-workflow-orchestrator-\${artifact_version}.zip" *
                            cd $WORKSPACE
                            echo "CREATED [az-ci-workflow-orchestrator-\${artifact_version}.zip]"
                            curl -v -u deployment:deployment123 --upload-file \
                                "az-ci-workflow-orchestrator-\${artifact_version}.zip" \
                                "http://74.225.187.237:8081/repository/packages/cca/az-ci-workflow-orchestrator-\${artifact_version}.zip"
                        else
                            artifact_version=$ver
                            echo "Downloading specified artifact version from Nexus..."
                            curl -v -u nexus-user:nexus@123 -O "http://74.225.187.237:8081/repository/packages/cca/az-ci-workflow-orchestrator-\${artifact_version}.zip"
                        fi
                        rm -rf "az-ci-workflow-orchestrator-\${artifact_version}"
                        unzip "az-ci-workflow-orchestrator-\${artifact_version}.zip" -d "az-ci-workflow-orchestrator-\${artifact_version}"

                        ls -ltr
                        az logicapp deployment source config-zip -g ${params.RESOURCE_GROUP_NAME} -n ${params.AZURE_LOGICAPP_NAME} --subscription ${params.SUBSCRIPTION} --src az-ci-workflow-orchestrator-\${artifact_version}.zip
                    """
                }
            }
        }

    }
}

