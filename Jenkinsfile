pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
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
            name: 'VM_NAME',
            defaultValue: 'jenkins-win-vm',
            description: 'Windows VM name'
        )

        string(
            name: 'STORAGE_ACCOUNT',
            defaultValue: 'jenkinsstorage12345',
            description: 'Storage account name (must be globally unique)'
        )
    }

    ///////////////////////////////////////////////////////
    // ENVIRONMENT VARIABLES
    ///////////////////////////////////////////////////////
    environment {
        RESOURCE_GROUP = "rg-${params.ENVIRONMENT}"
        VM_SIZE = "Standard_B2s"
        ADMIN_USER = "azureuser"
        ADMIN_PASS = "Password@12345!"   // change in real projects
    }

    ///////////////////////////////////////////////////////
    // STAGES
    ///////////////////////////////////////////////////////
    stages {

        ///////////////////////////////////////////////////
        // CHECKOUT CODE
        ///////////////////////////////////////////////////
        stage('Checkout') {
            steps {
                git branch: "${params.GIT_BRANCH}",
                    url: 'https://github.com/your-repo.git'
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
                  --location ${params.AZURE_REGION}
                """
            }
        }

        ///////////////////////////////////////////////////
        // CREATE STORAGE ACCOUNT
        ///////////////////////////////////////////////////
        stage('Create Storage Account') {
            steps {
                bat """
                az storage account create ^
                  --name ${params.STORAGE_ACCOUNT} ^
                  --resource-group %RESOURCE_GROUP% ^
                  --location ${params.AZURE_REGION} ^
                  --sku Standard_LRS
                """
            }
        }

        ///////////////////////////////////////////////////
        // CREATE WINDOWS VM
        ///////////////////////////////////////////////////
        stage('Create Windows VM') {
            steps {
                bat """
                az vm create ^
                  --resource-group %RESOURCE_GROUP% ^
                  --name ${params.VM_NAME} ^
                  --image Win2019Datacenter ^
                  --admin-username %ADMIN_USER% ^
                  --admin-password %ADMIN_PASS% ^
                  --size %VM_SIZE% ^
                  --public-ip-sku Standard
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
                  --name ${params.VM_NAME} ^
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
            echo "Infrastructure created successfully!"
        }
        failure {
            echo "Deployment failed!"
        }
    }
}
