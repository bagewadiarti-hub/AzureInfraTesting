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

        string(
            name: 'GIT_BRANCH',
            defaultValue: 'main',
            description: 'Git branch or release version'
        )

        booleanParam(
            name: 'AUTO_CLEANUP',
            defaultValue: false,
            description: 'Auto delete resource group (non-prod only)'
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
                    echo "Requested Region = ${params.AZURE_REGION}"
                }
            }
        }

        /*
        =========================================================
        ENTERPRISE REGION SELECTION + FALLBACK + COST OPTIMIZATION
        =========================================================
        */
        stage('Select Best Azure Region') {
            steps {
                script {

                    def regionPriorityMap = [
                        dev :  [params.AZURE_REGION, "centralindia", "eastus", "westus2", "westeurope"],
                        test:  [params.AZURE_REGION, "centralindia", "westeurope", "eastus", "westus2"],
                        prod:  [params.AZURE_REGION, "westeurope", "eastus2", "centralindia", "westus2"]
                    ]

                    def regionPriority = regionPriorityMap[params.ENVIRONMENT].unique()

                    echo "Region priority order → ${regionPriority}"

                    def selectedRegion = null

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
                            echo "Region valid → ${r}"
                            selectedRegion = r
                            break
                        }
                    }

                    if (!selectedRegion) {
                        error("No valid Azure region available.")
                    }

                    env.ACTIVE_REGION = selectedRegion

                    // DR standby region
                    def drMap = [
                        westeurope  : "northeurope",
                        centralindia: "southindia",
                        eastus      : "westus2",
                        eastus2     : "centralus",
                        westus2     : "eastus"
                    ]

                    env.DR_REGION = drMap[selectedRegion] ?: "eastus"

                    echo "PRIMARY REGION → ${env.ACTIVE_REGION}"
                    echo "DR STANDBY REGION → ${env.DR_REGION}"
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
                            error "VM admin password missing"
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

        stage('Create Resource Group') {
            steps {
                echo "Creating RG in ${env.ACTIVE_REGION}"
                bat """
                az group create ^
                  --name ${env.RESOURCE_GROUP} ^
                  --location ${env.ACTIVE_REGION} ^
                  --tags environment=${params.ENVIRONMENT} owner=jenkins
                """
            }
        }

        stage('Generate Names') {
            steps {
                script {
                    env.STORAGE_NAME = "jenkinsstorage${UUID.randomUUID().toString().take(8)}".toLowerCase()
                    env.VM_NAME = "jenkins-${params.ENVIRONMENT}-win-${env.BUILD_NUMBER}"
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
                      --location ${env.ACTIVE_REGION} ^
                      --sku Standard_LRS ^
                      --min-tls-version TLS1_2
                    """
                }
            }
        }

        /*
        =========================================================
        REGION + VM SIZE COMBINED FALLBACK (CAPACITY SAFE)
        =========================================================
        */
        stage('Create Windows VM (Capacity Safe)') {
            steps {
                script {

                    def regionOrder = [
                        env.ACTIVE_REGION,
                        env.DR_REGION
                    ].unique()

                    def vmSizes = [
                        "Standard_B2s",
                        "Standard_DS1_v2",
                        "Standard_D2s_v5",
                        "Standard_B2ms",
                        "Standard_D4s_v5"
                    ]

                    def created = false

                    for (region in regionOrder) {

                        if (created) break

                        echo "Trying region → ${region}"

                        for (size in vmSizes) {

                            if (created) break

                            echo "Trying size ${size} in ${region}"

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
                                      --location ${region} ^
                                      --public-ip-sku Standard ^
                                      --tags environment=${params.ENVIRONMENT}
                                    """
                                }

                                env.ACTIVE_REGION = region
                                echo "VM CREATED → ${size} in ${region}"
                                created = true

                            } catch (err) {
                                echo "FAILED → ${size} in ${region}"
                            }
                        }
                    }

                    if (!created) {
                        error("VM creation failed across all regions and sizes.")
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
            bat 'az logout || exit 0'
        }
        success {
            echo 'Infrastructure deployment completed successfully'
        }
        failure {
            echo 'Infrastructure deployment failed'
        }
    }
}
