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
        
        
        string(name: 'AZURE_LOGICAPP_NAME', defaultValue:'dev-jenkins-logic-app', description: '''The name of LogicApp to deploy
            dev-jenkins-logic-app
            jenkins-logic-app''' )

        string(name: 'AZURE_LOGIC_ASP_NAME', defaultValue:'dev-logic-app-ASP', description: '''The name of App service Plan for FunctionApp to deploy
            dev-logic-app-ASP
            logic-ASP
            ''' )
        
        
        string(name: 'LOGIC_STORAGE_ACCOUNT_NAME', defaultValue:'v2funcappstg569650', description: '''select the existing Storage account name for Func App or create new .
            v2funcappstg569650
            ccadevfunctionappstgacc 
            ''' )

        string(name: 'AZURE_APP_INSIGHTS_NAME', defaultValue:'v2-func-app-insight', description: '''The name of Application insight for FunctionApp to deploy
            v2-func-app-insight
            ''' )
        
        // string(name: 'APP_INSIGHTS_INSTRUMENTATION_KEY', description: '''select the existing Application insight Instrumentation Key .
        //     9b3a9c7a-fec6-4f67-b669-a149294fbeee 
        //     ''' )

        string(name: 'REGION', defaultValue:'Central India', description: '''Region to Deploy to. 
            Central India
            East Us''')


        // string(name: 'PRIVATE_ENDPOINT_NAME', description: ''' Private endpoint name.
        //     jenkins-private-endpoint
        //     ''')

        // string(name: 'PRIVATE_CONNECTION_NAME', description: ''' Private endpoint Connection name.
        //     jenkins-logic-connection
        //     jenkins-privateend-connection
        //     ''')

        string(name: 'VNET_NAME', defaultValue:'jenkins-vm-vnet', description: ''' Vnet name for Private endpoint connection & Vnet integration.
            jenkins-vm-vnet
            ''')

        string(name: 'INBOUND_SNET_NAME', defaultValue:'jenkins-logicapp-inboud-subnet', description: ''' Inbound Subnet name for Private endpoint connection.
            jenkins-logicapp-inboud-subnet
            jenkins-inbound-subnet
            jenkins-subnet
            ''')

        // string(name: 'OUTBOUND_VNET_NAME', description: ''' Outbound Vnet name for Vnet integration.
        //     jenkins-vm-vnet
        //     ''')

        string(name: 'OUTBOUND_SNET_NAME', defaultValue:'jenkins-logicapp-outboud-subnet', description: ''' Outbound Subnet name for Private endpoint connection.
            jenkins-logicapp-outboud-subnet
            jenkins-outbound-subnet
            jenkins-subnet-01
            ''')


        // choice(name: 'SUBSCRIPTION', choices:[
        //     '48986b2e-5349-4fab-a6e8-d5f02072a4b8',
        //     '34b1c36e-d8e8-4bd5-a6f3-2f92a1c0626e',
        //     '70c3af66-8434-419b-b808-0b3c0c4b1a04'
        //     ],
        //     description: 'Subscription to deploy to .')

        string(name: 'SUBSCRIPTION', defaultValue:'48986b2e-5349-4fab-a6e8-d5f02072a4b8', description: ''' select subscription as:
            48986b2e-5349-4fab-a6e8-d5f02072a4b8
            34b1c36e-d8e8-4bd5-a6f3-2f92a1c0626e
            70c3af66-8434-419b-b808-0b3c0c4b1a04''')

        string(name: 'RESOURCE_GROUP_NAME', defaultValue:'jenkins-247-rg', description: ''' Azure Resource Group in which the FunctionApp need to deploy.
            jenkins-247-rg
            ''')

        choice(name: 'SKU', choices:[
            'WS2'], 
            description: 'ASP SKU.')

    }

    environment {
        AZURE_CLIENT_ID = credentials('azurerm_client_id')
        AZURE_CLIENT_SECRET = credentials('azurerm_client_secret')
        AZURE_TENANT_ID = credentials('azurerm_tenant_id')
        ZIP_FILE_NAME = "${params.AZURE_LOGICAPP_NAME}"
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

        // stage('Package Code') {
        //     steps {
        //         sh "zip -r ${ZIP_FILE_NAME} ."
        //     }
        // }


        /*
        stage('Static Code Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-9.9') {
                    script {
                        sh """
                            echo "SonarQube Analysis"
                            cd src/test-workflow1/
                            ${SONARQUBE_SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectKey=My-Logic-App \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=http://4.240.69.23:9000 \
                                -Dsonar.login=sqp_66cc4a7f4a8a07d5671cb5d42f241200b5297eba
                            cd $WORKSPACE
                        """
                    }
                }
            }
        } */


        stage('SonarQube Analysis') {
            steps {
                echo "SonarQube Analysis !!"
                withSonarQubeEnv('sonarqube-9.9') {
                    sh '/opt/sonarscanner/bin/sonar-scanner'
                }
            }
        }
        
        stage('Create APP Service Plan') {
            steps {
                sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET --tenant $AZURE_TENANT_ID'
                sh "az account set --subscription ${params.SUBSCRIPTION}"

                // Create App service Plan
                sh "az appservice plan create -g ${params.RESOURCE_GROUP_NAME} -n ${params.AZURE_LOGIC_ASP_NAME} --sku ${params.SKU} --location '${params.REGION}'"
            }

        }

        stage('Create LogicApp') {
            steps {
                // create Logicapp
                sh "az logicapp create -g ${params.RESOURCE_GROUP_NAME} --subscription ${params.SUBSCRIPTION} -p ${params.AZURE_LOGIC_ASP_NAME} -n ${params.AZURE_LOGICAPP_NAME} -s ${params.LOGIC_STORAGE_ACCOUNT_NAME} --app-insights ${params.AZURE_APP_INSIGHTS_NAME}"
                
                // Vnet integration
                sh "az webapp vnet-integration add --resource-group ${params.RESOURCE_GROUP_NAME}  --name ${params.AZURE_LOGICAPP_NAME} --vnet ${params.VNET_NAME} --subnet ${params.OUTBOUND_SNET_NAME}"
                
                // private end point
                sh "az network private-endpoint create -g ${params.RESOURCE_GROUP_NAME} -n ${params.AZURE_LOGICAPP_NAME}-private-endpoint --vnet-name ${params.VNET_NAME} --subnet ${params.INBOUND_SNET_NAME} --private-connection-resource-id $logicAppResourceId --connection-name ${params.AZURE_LOGICAPP_NAME}-private-connection -l '${params.REGION}' --group-id sites"

                // Create PrivateDNS Zone
                // sh "az network private-dns zone create -g ${params.RESOURCE_GROUP_NAME} -n privatelink.azurewebsites.net"

                // Link Vnet to Private DNS Zone
                // sh "az network private-dns link vnet create --name ${params.AZURE_LOGICAPP_NAME}-dns-link --registration-enabled true --resource-group ${params.RESOURCE_GROUP_NAME} --virtual-network ${params.VNET_NAME} --zone-name privatelink.azurewebsites.net"

                // Link Private Endpoint to Private DNS Zone
                // az network private-endpoint dns-zone-group add --endpoint-name MyPE -g saurav -n functionapp --zone-name "privatelink.azurewebsites.net"
                sh "az network private-endpoint dns-zone-group create --endpoint-name ${params.AZURE_LOGICAPP_NAME}-private-endpoint -g ${params.RESOURCE_GROUP_NAME} -n ${params.AZURE_LOGICAPP_NAME}-dns-config --zone-name default --private-dns-zone privatelink.azurewebsites.net" 
            }
        }

        // stage('Deploy code to LogicApp'){
        //     steps {
        //         sh "az webapp deployment source config-zip -g ${params.RESOURCE_GROUP_NAME} -n ${params.AZURE_LOGICAPP_NAME} --subscription ${params.SUBSCRIPTION} --src $ZIP_FILE_NAME-\${artifact_version}.zip"
        //         // sh "az logicapp deployment source config-zip -g ${params.RESOURCE_GROUP_NAME} -n ${params.AZURE_LOGICAPP_NAME} --subscription ${params.SUBSCRIPTION} --src $ZIP_FILE_NAME-\${artifact_version}.zip"
        //     }
        // }

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
                            cd src
                            zip -r "../$ZIP_FILE_NAME-\${artifact_version}.zip" *
                            cd $WORKSPACE
                            echo "CREATED [$ZIP_FILE_NAME-\${artifact_version}.zip]"
                            curl -v -u nexus-user:nexus@123 --upload-file \
                                "$ZIP_FILE_NAME-\${artifact_version}.zip" \
                                "http://20.40.49.121:8081/repository/ci-config-service/$ZIP_FILE_NAME-\${artifact_version}.zip"
                        else
                            artifact_version=$ver
                            echo "Downloading specified artifact version from Nexus..."
                            curl -v -u nexus-user:nexus@123 -O "http://20.40.49.121:8081/repository/ci-config-service/ZIP_FILE_NAME-\${artifact_version}.zip"
                        fi
                        rm -rf "$ZIP_FILE_NAME-\${artifact_version}"
                        unzip "$ZIP_FILE_NAME-\${artifact_version}.zip" -d "$ZIP_FILE_NAME-\${artifact_version}"

                        ls -ltr
                        az logicapp deployment source config-zip -g ${params.RESOURCE_GROUP_NAME} -n ${params.AZURE_LOGICAPP_NAME} --subscription ${params.SUBSCRIPTION} --src $ZIP_FILE_NAME-\${artifact_version}.zip
                    """
                }
            }
        }

    }
}

