pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 90, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    ///////////////////////////////////////////////////////
    // PARAMETERS
    ///////////////////////////////////////////////////////
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'test', 'prod'],
            description: 'Deployment environment'
        )

        string(
            name: 'AZURE_REGION',
            defaultValue: 'westeurope',
            description: 'Preferred Azure region'
        )

        string(
            name: 'GIT_BRANCH',
            defaultValue: 'main',
            description: 'Git branch / release'
        )

        booleanParam(
            name: 'AUTO_CLEANUP',
            defaultValue: false,
            description: 'Delete resource group after deployment (non-prod only)'
        )
    }

    ///////////////////////////////////////////////////////
    // ENVIRONMENT
    ///////////////////////////////////////////////////////
    environment {
        RESOURCE_GROUP = "rg-${params.ENVIRONMENT}"
        VM_ADMIN_USER = "azureuser"
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
                    echo "Requested Region = ${params.AZURE_REGION}"
                }
            }
        }

        ///////////////////////////////////////////////////
        // AZURE LOGIN (REQUIRED FIRST)
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

                    script {
                        if (!env.ADMIN_PASS?.trim()) {
                            error "VM admin password missing in Jenkins credentials"
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

        ///////////////////////////////////////////////////
        // REGION FALLBACK SELECTION
        ///////////////////////////////////////////////////
        stage('Select Best Azure Region') {
            steps {
                script {

                    def regionPriority = [
                        params.AZURE_REGION,
                        "centralindia",
                        "eastus",
                        "westus2"
                    ].unique()

                    echo "Region priority order → ${regionPriority}"

                    def selected = null

                    for (r in regionPriority) {

                        echo "Checking region → ${r}"

                        def exists = bat(
                            script: """
                            az account list-locations ^
                              --query "[?name=='${r}'].name" ^
                              -o tsv
                            """,
                            returnStdout: true
                        ).trim()

                        if (exists) {
                            selected = r
                            break
                        }
                    }

                    if (!selected) {
                        error("No Azure regions available")
                    }

                    env.SELECTED_REGION = selected
                    echo "Selected Azure region → ${env.SELECTED_REGION}"
                }
            }
        }

        ///////////////////////////////////////////////////
        // PROD APPROVAL
        ///////////////////////////////////////////////////
        stage('Production Approval') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                input message: 'Approve PRODUCTION deployment?', ok: 'Deploy'
            }
        }

        ///////////////////////////////////////////////////
        // CREATE RESOURCE GROUP
        ///////////////////////////////////////////////////
        stage('Create Resource Group') {
            steps {
                bat """
                az group create ^
                  --name ${env.RESOURCE_GROUP} ^
                  --location ${env.SELECTED_REGION} ^
                  --tags environment=${params.ENVIRONMENT} owner=jenkins
                """
            }
        }

        ///////////////////////////////////////////////////
        // GENERATE UNIQUE NAMES
        ///////////////////////////////////////////////////
        stage('Generate Names') {
            steps {
                script {
                    env.STORAGE_NAME = "jenkins${UUID.randomUUID().toString().take(10)}".toLowerCase()
                    env.VM_NAME = "jenkins-${params.ENVIRONMENT}-${env.BUILD_NUMBER}"

                    echo "Storage = ${env.STORAGE_NAME}"
                    echo "VM = ${env.VM_NAME}"
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
                      --name ${env.STORAGE_NAME} ^
                      --resource-group ${env.RESOURCE_GROUP} ^
                      --location ${env.SELECTED_REGION} ^
                      --sku Standard_LRS ^
                      --min-tls-version TLS1_2
                    """
                }
            }
        }

        ///////////////////////////////////////////////////
        // CREATE WINDOWS VM (SIZE FALLBACK)
        ///////////////////////////////////////////////////
        stage('Create Windows VM') {
            steps {
                script {

                    def vmSizes = [
                        "Standard_B2s",
                        "Standard_DS1_v2",
                        "Standard_D2s_v5",
                        "Standard_B2ms",
                        "Standard_D4s_v5"
                    ]

                    def created = false

                    for (size in vmSizes) {

                        if (created) break

                        echo "Trying VM size → ${size}"

                        try {
                            withCredentials([
                                string(credentialsId: 'azure-vm-admin-password', variable: 'ADMIN_PASS')
                            ]) {
                                bat """
                                az vm create ^
                                  --resource-group ${env.RESOURCE_GROUP} ^
                                  --name ${env.VM_NAME} ^
                                  --image Win2019Datacenter ^
                                  --admin-username ${env.VM_ADMIN_USER} ^
                                  --admin-password "%ADMIN_PASS%" ^
                                  --size ${size} ^
                                  --location ${env.SELECTED_REGION} ^
                                  --public-ip-sku Standard ^
                                  --tags environment=${params.ENVIRONMENT}
                                """
                            }

                            echo "VM created using ${size}"
                            created = true

                        } catch (err) {
                            echo "Size unavailable → ${size}"
                        }
                    }

                    if (!created) {
                        error("No VM sizes available in ${env.SELECTED_REGION}")
                    }
                }
            }
        }

        ///////////////////////////////////////////////////
        // OPEN RDP
        ///////////////////////////////////////////////////
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

        ///////////////////////////////////////////////////
        // GET PUBLIC IP
        ///////////////////////////////////////////////////
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

                    echo "VM PUBLIC IP → ${ip}"
                    echo "RDP → mstsc /v:${ip}"
                }
            }
        }

        ///////////////////////////////////////////////////
        // AUTO CLEANUP
        ///////////////////////////////////////////////////
        stage('Auto Cleanup') {
            when {
                expression {
                    params.AUTO_CLEANUP && params.ENVIRONMENT != 'prod'
                }
            }
            steps {
                bat """
                az group delete ^
                  --name ${env.RESOURCE_GROUP} ^
                  --yes --no-wait
                """
            }
        }
    }

    ///////////////////////////////////////////////////////
    // POST
    ///////////////////////////////////////////////////////
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
