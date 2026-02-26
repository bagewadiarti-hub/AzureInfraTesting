pipeline {
    agent any

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 90, unit: 'MINUTES')
    }

    environment {
        RESOURCE_GROUP = ''
        AZ_REGION = 'westeurope'
        VM_SIZE = 'Standard_B2s'
        STORAGE_SKU = 'Standard_LRS'
        ADMIN_USER = 'azureuser'
        ENVIRONMENT = 'dev'
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Azure Login') {
            steps {
                withCredentials([
                    string(credentialsId: 'azure-client-id', variable: 'AZ_CLIENT'),
                    string(credentialsId: 'azure-client-secret', variable: 'AZ_SECRET'),
                    string(credentialsId: 'azure-tenant-id', variable: 'AZ_TENANT'),
                    string(credentialsId: 'azure-subscription-id', variable: 'AZ_SUB')
                ]) {
                    bat """
                    az login --service-principal ^
                      --username %AZ_CLIENT% ^
                      --password %AZ_SECRET% ^
                      --tenant %AZ_TENANT%
                    az account set --subscription %AZ_SUB%
                    """
                }
            }
        }

        stage('Create Resource Group') {
            steps {
                script {
                    RESOURCE_GROUP = "rg-${ENVIRONMENT}"
                }
                bat """
                az group create ^
                  --name ${RESOURCE_GROUP} ^
                  --location ${AZ_REGION}
                """
            }
        }

        stage('Generate Names') {
            steps {
                script {
                    def rand = UUID.randomUUID().toString().replaceAll('-', '').take(8)
                    env.STORAGE_NAME = "corp${ENVIRONMENT}st${rand}"
                    env.VM_NAME = "winvm-${rand}"
                    echo "Storage → ${STORAGE_NAME}"
                    echo "VM → ${VM_NAME}"
                }
            }
        }

        stage('Create Storage') {
            steps {
                bat """
                az storage account create ^
                  --name ${STORAGE_NAME} ^
                  --resource-group ${RESOURCE_GROUP} ^
                  --location ${AZ_REGION} ^
                  --sku ${STORAGE_SKU} ^
                  --https-only true
                """
            }
        }

        stage('Create Windows VM') {
            steps {
                withCredentials([string(credentialsId: 'azure-vm-admin-password', variable: 'ADMIN_PASS')]) {
                    bat """
                    az vm create ^
                      --resource-group ${RESOURCE_GROUP} ^
                      --name ${VM_NAME} ^
                      --image Win2019Datacenter ^
                      --size ${VM_SIZE} ^
                      --admin-username ${ADMIN_USER} ^
                      --admin-password ${ADMIN_PASS}
                    """
                }
            }
        }

        stage('Open RDP Port') {
            steps {
                bat """
                az vm open-port ^
                  --resource-group ${RESOURCE_GROUP} ^
                  --name ${VM_NAME} ^
                  --port 3389
                """
            }
        }

        stage('Get Public IP') {
            steps {
                bat """
                az vm show ^
                  --resource-group ${RESOURCE_GROUP} ^
                  --name ${VM_NAME} ^
                  --show-details ^
                  --query publicIps ^
                  --output tsv
                """
            }
        }
    }

    post {
        always {
            echo "Logging out from Azure"
            bat 'az logout || exit 0'
        }
        success {
            echo "Deployment completed successfully!"
        }
        failure {
            echo "Deployment failed!"
        }
    }
}
