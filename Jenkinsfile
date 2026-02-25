pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 90, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
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
            description: 'Git branch / release'
        )

        choice(
            name: 'AZURE_REGION',
            choices: ['westeurope', 'westus', 'centralindia'],
            description: 'Azure region'
        )

        string(
            name: 'VM_NAME',
            defaultValue: 'jenkins-win-vm',
            description: 'Base VM name'
        )

        string(
            name: 'STORAGE_PREFIX',
            defaultValue: 'jenkinsstorage',
            description: 'Storage account prefix (lowercase only)'
        )
    }

    ///////////////////////////////////////////////////////
    // ENVIRONMENT VARIABLES
    ///////////////////////////////////////////////////////
    environment {
        RESOURCE_GROUP = "rg-${params.ENVIRONMENT}"
        VM_SIZE = "Standard_B2s"
        ADMIN_USER = "azureuser"
    }

    ///////////////////////////////////////////////////////
    // STAGES
    ///////////////////////////////////////////////////////
    stages {

        ///////////////////////////////////////////////////
        // VALIDATE INPUTS
        ///////////////////////////////////////////////////
        stage('Validate Inputs') {
            steps {
                script {
                    echo "Environment = ${params.ENVIRONMENT}"
                    echo "Region = ${params.AZURE_REGION}"
                }
            }
        }

        ///////////////////////////////////////////////////
        // PRODUCTION APPROVAL
        ///////////////////////////////////////////////////
        stage('Production Approval') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                input message: "Approve PRODUCTION infrastructure deployment?"
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
                    string(credentialsId: 'azure-subscription-id', variable: 'AZ_SUB'),
                    string(credentialsId: 'azure-vm-admin-password', variable: 'ADMIN_PASS')
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

        ///////////////////////////////////////////////////
        // CREATE OR VALIDATE RESOURCE GROUP
        ///////////////////////////////////////////////////
        stage('Create or Validate Resource Group') {
            steps {
                script {

                    def exists = bat(
                        script: "az group exists --name ${env.RESOURCE_GROUP}",
                        returnStdout: true
                    ).trim()

                    if (exists == "true") {

                        echo "Resource group exists. Checking location..."

                        def existingLocation = bat(
                            script: "az group show --name ${env.RESOURCE_GROUP} --query location -o tsv",
                            returnStdout: true
                        ).trim()

                        echo "Using existing location: ${existingLocation}"
                        env.ACTUAL_REGION = existingLocation

                    } else {

                        echo "Creating resource group in ${params.AZURE_REGION}"

                        bat """
                        az group create ^
                          --name ${env.RESOURCE_GROUP} ^
                          --location ${params.AZURE_REGION} ^
                          --tags environment=${params.ENVIRONMENT} owner=jenkins
                        """

                        env.ACTUAL_REGION = params.AZURE_REGION
                    }
                }
            }
        }

        ///////////////////////////////////////////////////
        // GENERATE UNIQUE NAMES
        ///////////////////////////////////////////////////
        stage('Generate Unique Names') {
            steps {
                script {
                    def rand = UUID.randomUUID().toString().replaceAll("-", "").substring(0,8)

                    env.STORAGE_ACCOUNT = "${params.STORAGE_PREFIX}${rand}".toLowerCase()
                    env.FINAL_VM_NAME = "${params.VM_NAME}-${env.BUILD_NUMBER}"

                    echo "Storage account = ${env.STORAGE_ACCOUNT}"
                    echo "VM name = ${env.FINAL_VM_NAME}"
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
                      --name ${env.STORAGE_ACCOUNT} ^
                      --resource-group ${env.RESOURCE_GROUP} ^
                      --location ${env.ACTUAL_REGION} ^
                      --sku Standard_LRS ^
                      --min-tls-version TLS1_2
                    """
                }
            }
        }

        ///////////////////////////////////////////////////
        // CREATE WINDOWS VM
        ///////////////////////////////////////////////////
        stage('Create Windows VM') {
            steps {
                retry(2) {
                    bat """
                    az vm create ^
                      --resource-group ${env.RESOURCE_GROUP} ^
                      --name ${env.FINAL_VM_NAME} ^
                      --image Win2019Datacenter ^
                      --admin-username ${env.ADMIN_USER} ^
                      --admin-password %ADMIN_PASS% ^
                      --size ${env.VM_SIZE} ^
                      --location ${env.ACTUAL_REGION} ^
                      --public-ip-sku Standard ^
                      --tags environment=${params.ENVIRONMENT}
                    """
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
                  --resource-group ${env.RESOURCE_GROUP} ^
                  --name ${env.FINAL_VM_NAME} ^
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
                  --resource-group ${env.RESOURCE_GROUP} ^
                  --name ${env.FINAL_VM_NAME} ^
                  -d ^
                  --query publicIps ^
                  -o table
                """
            }
        }

        ///////////////////////////////////////////////////
        // AUTO CLEANUP NON PROD
        ///////////////////////////////////////////////////
        stage('Auto Cleanup (Non-Prod)') {
            when {
                expression { params.ENVIRONMENT != 'prod' }
            }
            steps {
                echo "Non-production environment — keeping resources for testing"
            }
        }
    }

    ///////////////////////////////////////////////////////
    // POST
    ///////////////////////////////////////////////////////
    post {
        success {
            echo "Infrastructure deployed successfully"
        }
        failure {
            echo "Infrastructure deployment failed"
        }
        always {
            bat "az logout"
        }
    }
}
