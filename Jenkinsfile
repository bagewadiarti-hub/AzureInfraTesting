pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 90, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'test', 'prod'],
            description: 'Deployment environment'
        )

        string(
            name: 'AZURE_REGION',
            defaultValue: 'westeurope',
            description: 'Azure region (example: westeurope, centralindia, westus)'
        )

        string(
            name: 'GIT_BRANCH',
            defaultValue: 'main',
            description: 'Git branch or release version'
        )

        booleanParam(
            name: 'AUTO_CLEANUP',
            defaultValue: false,
            description: 'Auto delete resource group after deployment (non-prod only)'
        )
    }

    environment {
        RESOURCE_GROUP = "rg-${params.ENVIRONMENT}"
        VM_ADMIN_USER = "azureuser"
    }

    stages {

        stage('Validate Inputs') {
            steps {
                script {
                    echo "Environment = ${params.ENVIRONMENT}"
                    echo "Region = ${params.AZURE_REGION}"
                }
            }
        }

        stage('Production Approval') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                input message: 'Approve production infrastructure deployment?', ok: 'Deploy'
            }
        }

        stage('Azure Login') {
            steps {
                withCredentials([
                    string(credentialsId: 'azure-client-id', variable: 'AZ_CLIENT'),
                    string(credentialsId: 'azure-client-secret', variable: 'AZ_SECRET'),
                    string(credentialsId: 'azure-tenant-id', variable: 'AZ_TENANT'),
                    string(credentialsId: 'azure-subscription-id', variable: 'AZ_SUB'),
                    string(credentialsId: 'azure-vm-admin-password', variable: 'ADMIN_PASS')
                ]) {

                    script {
                        if (!env.ADMIN_PASS?.trim()) {
                            error "VM admin password is empty. Check Jenkins credential mapping."
                        }
                    }

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

        stage('Create or Validate Resource Group') {
            steps {
                echo "Creating resource group in ${params.AZURE_REGION}"
                bat """
                az group create ^
                  --name ${env.RESOURCE_GROUP} ^
                  --location ${params.AZURE_REGION} ^
                  --tags environment=${params.ENVIRONMENT} owner=jenkins
                """
            }
        }

        stage('Generate Unique Names') {
            steps {
                script {
                    env.STORAGE_NAME = "jenkinsstorage${UUID.randomUUID().toString().take(8)}".toLowerCase()
                    env.VM_NAME = "jenkins-${params.ENVIRONMENT}-win-${env.BUILD_NUMBER}"

                    echo "Storage account = ${env.STORAGE_NAME}"
                    echo "VM name = ${env.VM_NAME}"
                }
            }
        }

        stage('Create Storage Account') {
            steps {
                retry(2) {
                    bat """
                    az storage account create ^
                      --name ${env.STORAGE_NAME} ^
                      --resource-group ${env.RESOURCE_GROUP} ^
                      --location ${params.AZURE_REGION} ^
                      --sku Standard_LRS ^
                      --min-tls-version TLS1_2
                    """
                }
            }
        }

        stage('Create Windows VM') {
    steps {
        script {

            def vmSizes = [
                "Standard_B2s",
                "Standard_DS1_v2",
                "Standard_D2s_v5",
                "Standard_B2ms"
            ]

            def vmCreated = false

            for (size in vmSizes) {

                if (vmCreated) break

                echo "Trying VM size: ${size}"

                try {

                    withCredentials([
                        string(credentialsId: 'azure-vm-admin-password', variable: 'ADMIN_PASS')
                    ]) {

                        bat """
                        az vm create ^
                          --resource-group rg-${params.ENVIRONMENT} ^
                          --name ${env.VM_NAME} ^
                          --image Win2019Datacenter ^
                          --admin-username azureuser ^
                          --admin-password "%ADMIN_PASS%" ^
                          --size ${size} ^
                          --location ${params.AZURE_REGION} ^
                          --public-ip-sku Standard ^
                          --tags environment=${params.ENVIRONMENT}
                        """
                    }

                    echo "VM successfully created with size ${size}"
                    vmCreated = true

                } catch (err) {
                    echo "VM size ${size} not available. Trying next..."
                }
            }

            if (!vmCreated) {
                error("All VM sizes unavailable in this region. Deployment failed.")
            }
        }
    }
}

        stage('Open RDP Port') {
            steps {
                bat """
                az vm open-port ^
                  --resource-group ${env.RESOURCE_GROUP} ^
                  --name ${env.VM_NAME} ^
                  --port 3389
                """
            }
        }

        stage('Get VM Public IP') {
            steps {
                script {
                    def ip = bat(
                        script: """
                        az vm show ^
                          --resource-group ${env.RESOURCE_GROUP} ^
                          --name ${env.VM_NAME} ^
                          -d ^
                          --query publicIps ^
                          -o tsv
                        """,
                        returnStdout: true
                    ).trim()

                    echo "VM PUBLIC IP = ${ip}"
                    echo "RDP → mstsc /v:${ip}"
                }
            }
        }

        stage('Auto Cleanup (Non-Prod)') {
            when {
                expression {
                    params.AUTO_CLEANUP && params.ENVIRONMENT != 'prod'
                }
            }
            steps {
                echo "Auto deleting resource group ${env.RESOURCE_GROUP}"
                bat """
                az group delete ^
                  --name ${env.RESOURCE_GROUP} ^
                  --yes --no-wait
                """
            }
        }
    }

    post {
        always {
            script {
                bat 'az logout || exit 0'
            }
        }

        success {
            echo 'Infrastructure deployment completed successfully'
        }

        failure {
            echo 'Infrastructure deployment failed'
        }
    }
}
