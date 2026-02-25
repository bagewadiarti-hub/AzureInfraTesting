pipeline {
    agent any

    ///////////////////////////////////////////////////////
    // PIPELINE OPTIONS
    ///////////////////////////////////////////////////////
    options {
        timestamps()
        timeout(time: 90, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '25'))
    }

    ///////////////////////////////////////////////////////
    // BUILD PARAMETERS
    ///////////////////////////////////////////////////////
    parameters {

        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'test', 'prod'],
            description: 'Deployment environment'
        )

        string(
            name: 'GIT_BRANCH',
            defaultValue: 'main',
            description: 'Git branch / release version'
        )

        choice(
            name: 'AZURE_REGION',
            choices: ['westeurope', 'westus', 'centralindia'],
            description: 'Azure region'
        )

        string(
            name: 'VM_BASE_NAME',
            defaultValue: 'jenkins-win-vm',
            description: 'Base name for Windows VM'
        )

        string(
            name: 'VM_SIZE',
            defaultValue: 'Standard_B2s',
            description: 'Azure VM size'
        )

        booleanParam(
            name: 'AUTO_DESTROY',
            defaultValue: false,
            description: 'Delete resource group after build (recommended for DEV/TEST)'
        )
    }

    ///////////////////////////////////////////////////////
    // ENVIRONMENT VARIABLES
    ///////////////////////////////////////////////////////
    environment {
        RESOURCE_GROUP = "rg-${params.ENVIRONMENT}"
        ADMIN_USER = "azureuser"
        BUILD_TAG_SAFE = "${env.BUILD_NUMBER}"
    }

    ///////////////////////////////////////////////////////
    // STAGES
    ///////////////////////////////////////////////////////
    stages {

        ///////////////////////////////////////////////////
        // VALIDATION
        ///////////////////////////////////////////////////
        stage('Validate Inputs') {
            steps {
                script {
                    echo "Environment = ${params.ENVIRONMENT}"
                    echo "Region = ${params.AZURE_REGION}"

                    if (params.ENVIRONMENT == 'prod') {
                        input message: "CONFIRM production infrastructure deployment"
                    }
                }
            }
        }

        ///////////////////////////////////////////////////
        // AZURE LOGIN
        ///////////////////////////////////////////////////
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
                    az account show
                    """
                }
            }
        }

        ///////////////////////////////////////////////////
        // CREATE RESOURCE GROUP
        ///////////////////////////////////////////////////
        stage('Create Resource Group') {
            steps {
                bat """
                az group create ^
                  --name %RESOURCE_GROUP% ^
                  --location ${params.AZURE_REGION} ^
                  --tags environment=${params.ENVIRONMENT} owner=jenkins
                """
            }
        }

        ///////////////////////////////////////////////////
        // GENERATE UNIQUE NAMES
        ///////////////////////////////////////////////////
        stage('Generate Unique Names') {
            steps {
                script {
                    def unique = new Date().format("HHmmss")

                    env.STORAGE_NAME = "st${params.ENVIRONMENT}${unique}".toLowerCase()
                    env.VM_NAME = "${params.VM_BASE_NAME}-${params.ENVIRONMENT}-${unique}".toLowerCase()

                    echo "Storage Account = ${env.STORAGE_NAME}"
                    echo "VM Name = ${env.VM_NAME}"
                }
            }
        }

        ///////////////////////////////////////////////////
        // CREATE STORAGE ACCOUNT
        ///////////////////////////////////////////////////
        stage('Create Storage Account') {
            steps {
                retry(2) {
                    bat """
                    az storage account create ^
                      --name %STORAGE_NAME% ^
                      --resource-group %RESOURCE_GROUP% ^
                      --location ${params.AZURE_REGION} ^
                      --sku Standard_LRS ^
                      --min-tls-version TLS1_2 ^
                      --allow-blob-public-access false
                    """
                }
            }
        }

        ///////////////////////////////////////////////////
        // CREATE WINDOWS VM
        ///////////////////////////////////////////////////
        stage('Create Windows VM') {
            steps {
                withCredentials([
                    string(credentialsId: 'azure-vm-admin-password', variable: 'ADMIN_PASS')
                ]) {

                    retry(2) {
                        bat """
                        az vm create ^
                          --resource-group %RESOURCE_GROUP% ^
                          --name %VM_NAME% ^
                          --image Win2019Datacenter ^
                          --admin-username %ADMIN_USER% ^
                          --admin-password %ADMIN_PASS% ^
                          --size ${params.VM_SIZE} ^
                          --public-ip-sku Standard ^
                          --tags environment=${params.ENVIRONMENT} deployedBy=jenkins
                        """
                    }
                }
            }
        }

        ///////////////////////////////////////////////////
        // OPEN RDP PORT
        ///////////////////////////////////////////////////
        stage('Open RDP Port') {
            steps {
                bat """
                az vm open-port ^
                  --resource-group %RESOURCE_GROUP% ^
                  --name %VM_NAME% ^
                  --port 3389
                """
            }
        }

        ///////////////////////////////////////////////////
        // GET PUBLIC IP
        ///////////////////////////////////////////////////
        stage('Get VM Public IP') {
            steps {
                bat """
                az vm show ^
                  --resource-group %RESOURCE_GROUP% ^
                  --name %VM_NAME% ^
                  -d ^
                  --query publicIps ^
                  -o table
                """
            }
        }
    }

    ///////////////////////////////////////////////////////
    // POST ACTIONS
    ///////////////////////////////////////////////////////
    post {
        success {
            echo "Infrastructure deployment completed successfully"
        }

        failure {
            echo "Infrastructure deployment failed"
        }

        always {
            script {
                if (params.AUTO_DESTROY && params.ENVIRONMENT != 'prod') {
                    echo "Auto-destroy enabled — deleting resource group"

                    bat """
                    az group delete ^
                      --name %RESOURCE_GROUP% ^
                      --yes ^
                      --no-wait
                    """
                }
            }

            bat "az logout"
        }
    }
}
