pipeline {
    agent any

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        APP_NAME = "sample-app"
        RESOURCE_GROUP = "rg-dev"
        STORAGE_PREFIX = "corpdevst"
        VM_PREFIX = "winvm"
        VM_ADMIN_USER = "azureuser"

        // Fallback VM sizes
        VM_SIZES = "Standard_D2a_v4,Standard_D2as_v4,Standard_E2as_v4,Standard_D2ads_v5"
        // Fallback regions
        AZURE_REGIONS = "eastus,westeurope,uksouth,northeurope,northcentralus,francecentral"
    }

    stages {

        stage('Checkout SCM') {
            steps { checkout scm }
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
                bat """
                az group create ^
                  --name ${RESOURCE_GROUP} ^
                  --location westeurope
                """
            }
        }

        stage('Generate Names') {
            steps {
                script {
                    // Safe last 8 digits timestamp
                    def timestamp = (System.currentTimeMillis() % 100_000_000).toString()
                    STORAGE_NAME = "${STORAGE_PREFIX}${timestamp}"
                    VM_NAME = "${VM_PREFIX}${timestamp}"
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
                  --location westeurope ^
                  --sku Standard_LRS ^
                  --https-only true
                """
            }
        }

        stage('Create Windows VM') {
            steps {
                withCredentials([string(credentialsId: 'azure-vm-admin-password', variable: 'ADMIN_PASS')]) {
                    script {
                        def vmCreated = false
                        def sizes = VM_SIZES.tokenize(',')
                        def regions = AZURE_REGIONS.tokenize(',')

                        for (region in regions) {
                            if (vmCreated) { break }
                            for (size in sizes) {
                                if (vmCreated) { break }
                                try {
                                    echo "Trying VM: ${VM_NAME} in ${region} with size ${size}"
                                    bat """
                                    az vm create ^
                                      --resource-group ${RESOURCE_GROUP} ^
                                      --name ${VM_NAME} ^
                                      --image Win2019Datacenter ^
                                      --size ${size} ^
                                      --location ${region} ^
                                      --admin-username ${VM_ADMIN_USER} ^
                                      --admin-password %ADMIN_PASS%
                                    """
                                    echo "VM created successfully in ${region} with size ${size}"
                                    vmCreated = true
                                } catch (err) {
                                    echo "Failed to create size ${size} in ${region}, trying next..."
                                }
                            }
                        }

                        if (!vmCreated) {
                            error "All VM sizes and regions failed. Cannot create VM."
                        }
                    }
                }
            }
        }

        stage('Open RDP Port') {
            steps {
                bat """
                az vm open-port ^
                  --port 3389 ^
                  --resource-group ${RESOURCE_GROUP} ^
                  --name ${VM_NAME}
                """
            }
        }

        stage('Get Public IP') {
            steps {
                script {
                    def ip = bat(
                        script: "az vm list-ip-addresses --name ${VM_NAME} --resource-group ${RESOURCE_GROUP} --query [0].virtualMachine.network.publicIpAddresses[0].ipAddress -o tsv",
                        returnStdout: true
                    ).trim()
                    echo "VM Public IP → ${ip}"
                }
            }
        }

    }

    post {
        always {
            echo "Logging out from Azure"
            bat "az logout || exit 0"
        }
        failure { echo "Deployment failed!" }
        success { echo "Deployment completed successfully!" }
    }
}
