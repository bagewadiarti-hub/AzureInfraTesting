pipeline {
    agent any

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        APP_NAME = "azure-infra-app"
        RESOURCE_GROUP = "rg-dev"
        LOCATION = "westeurope"
        STORAGE_NAME_PREFIX = "corpdevst"
        VM_NAME_PREFIX = "winvm"
        VM_SIZE_LIST = "Standard_B2s,Standard_B1ms,Standard_D2s_v5"
        VM_ADMIN_USER = "azureuser"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/bagewadiarti-hub/AzureInfraTesting.git',
                        credentialsId: 'github-creds'
                    ]]
                ])
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
                bat """
                az group create ^
                  --name ${RESOURCE_GROUP} ^
                  --location ${LOCATION}
                """
            }
        }

        stage('Generate Names') {
            steps {
                script {
                    STORAGE_NAME = "${STORAGE_NAME_PREFIX}${UUID.randomUUID().toString().replaceAll('-', '').take(10)}"
                    VM_NAME = "${VM_NAME_PREFIX}${UUID.randomUUID().toString().replaceAll('-', '').take(6)}"
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
                  --location ${LOCATION} ^
                  --sku Standard_LRS ^
                  --https-only true
                """
            }
        }

        stage('Create Windows VM') {
            steps {
                withCredentials([string(credentialsId: 'azure-vm-admin-password', variable: 'ADMIN_PASS')]) {
                    script {
                        def vmSizes = VM_SIZE_LIST.split(',')
                        def created = false
                        for (size in vmSizes) {
                            try {
                                bat """
                                az vm create ^
                                  --resource-group ${RESOURCE_GROUP} ^
                                  --name ${VM_NAME} ^
                                  --image Win2019Datacenter ^
                                  --size ${size} ^
                                  --admin-username ${VM_ADMIN_USER} ^
                                  --admin-password ${ADMIN_PASS}
                                """
                                echo "VM created with size ${size}"
                                created = true
                                break
                            } catch (err) {
                                echo "VM creation failed for size ${size}, trying next..."
                            }
                        }
                        if (!created) {
                            error "All VM sizes failed. Cannot create VM."
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
                    def publicIP = bat(
                        script: "az vm list-ip-addresses --name ${VM_NAME} --resource-group ${RESOURCE_GROUP} --query '[0].virtualMachine.network.publicIpAddresses[0].ipAddress' -o tsv",
                        returnStdout: true
                    ).trim()
                    echo "Public IP → ${publicIP}"
                }
            }
        }
    }

    post {
        always {
            echo "Logging out from Azure"
            bat 'az logout || exit 0'
        }
        failure {
            echo "Deployment failed!"
        }
        success {
            echo "Deployment succeeded!"
        }
    }
}
