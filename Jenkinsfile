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
            description: 'Preferred Azure region'
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
                echo "Environment = ${params.ENVIRONMENT}"
                echo "Requested Region = ${params.AZURE_REGION}"
            }
        }

        // =========================
        // AZURE LOGIN FIRST
        // =========================
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
                            error "VM admin password is empty"
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

        // =========================
        // REGION FALLBACK LOGIC
        // =========================
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
                        error "No valid Azure region found"
                    }

                    env.SELECTED_REGION = selected
                    echo "Selected Azure region → ${env.SELECTED_REGION}"
                }
            }
        }

        stage('Production Approval') {
            when { expression { params.ENVIRONMENT == 'prod' } }
            steps {
                input message: 'Approve production deployment?', ok: 'Deploy'
            }
        }

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

        // =========================
        // ENTERPRISE STORAGE NAME GENERATOR
        // =========================
        stage('Generate Enterprise Resource Names') {
            steps {
                script {

                    def maxAttempts = 10
                    def created = false

                    for (int i=0; i<maxAttempts; i++) {

                        def random = UUID.randomUUID()
                            .toString()
                            .replaceAll("-", "")
                            .take(12)
                            .toLowerCase()

                        def envCode = params.ENVIRONMENT.take(3)
                        def candidate = "st${envCode}${random}".take(24)

                        def available = bat(
                            script: """
                            az storage account check-name ^
                              --name ${candidate} ^
                              --query nameAvailable ^
                              -o tsv
                            """,
                            returnStdout: true
                        ).trim()

                        if (available == "true") {
                            env.STORAGE_NAME = candidate
                            created = true
                            break
                        }

                        echo "Storage name ${candidate} not available → retrying"
                    }

                    if (!created) {
                        error "Failed to generate unique storage account name"
                    }

                    env.VM_NAME = "vm-${params.ENVIRONMENT}-${env.BUILD_NUMBER}"

                    echo "Final Storage Name = ${env.STORAGE_NAME}"
                    echo "VM Name = ${env.VM_NAME}"
                }
            }
        }

        stage('Create Storage Account') {
            steps {
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

        // =========================
        // VM CAPACITY SAFE CREATION
        // =========================
        stage('Create Windows VM (Capacity Safe)') {
            steps {
                script {

                    def sizes = [
                        "Standard_B2s",
                        "Standard_DS1_v2",
                        "Standard_D2s_v5",
                        "Standard_B2ms",
                        "Standard_D4s_v5"
                    ]

                    def created = false

                    for (s in sizes) {
                        if (created) break

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
                                  --size ${s} ^
                                  --location ${env.SELECTED_REGION} ^
                                  --public-ip-sku Standard
                                """
                            }

                            echo "VM created with size ${s}"
                            created = true

                        } catch (e) {
                            echo "Size ${s} unavailable → trying next"
                        }
                    }

                    if (!created) {
                        error "No VM sizes available in region"
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
