pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 90, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

///////////////////////////////////////////////////////////
// PARAMETERS
///////////////////////////////////////////////////////////
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
        name: 'PRIMARY_REGION',
        choices: ['westeurope', 'centralindia', 'eastus', 'westus2'],
        description: 'Preferred Azure region'
    )

    string(
        name: 'VM_NAME',
        defaultValue: 'winvm',
        description: 'Base VM name'
    )
}

///////////////////////////////////////////////////////////
// ENVIRONMENT
///////////////////////////////////////////////////////////
environment {
    COMPANY_PREFIX = "corp"
    RESOURCE_GROUP = "rg-${params.ENVIRONMENT}"
    VM_SIZE = "Standard_B2s"
    ADMIN_USER = "azureuser"
    ADMIN_PASS = credentials('vm-admin-password')   // Jenkins secret
}

///////////////////////////////////////////////////////////
// STAGES
///////////////////////////////////////////////////////////
stages {

///////////////////////////////////////////////////////////
stage('Validate Inputs') {
    steps {
        echo "Environment = ${params.ENVIRONMENT}"
        echo "Primary region = ${params.PRIMARY_REGION}"
    }
}

///////////////////////////////////////////////////////////
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

///////////////////////////////////////////////////////////
stage('Select Best Azure Region') {
    steps {
        script {

            def regionPriority = [
                params.PRIMARY_REGION,
                "centralindia",
                "eastus",
                "westus2"
            ].unique()

            env.SELECTED_REGION = regionPriority[0]
            echo "Selected Azure region → ${env.SELECTED_REGION}"
        }
    }
}

///////////////////////////////////////////////////////////
stage('Production Approval') {
    when { expression { params.ENVIRONMENT == 'prod' } }
    steps {
        input message: "Approve PRODUCTION deployment?", ok: "Deploy"
    }
}

///////////////////////////////////////////////////////////
stage('Create Resource Group') {
    steps {
        bat """
        az group create ^
          --name %RESOURCE_GROUP% ^
          --location %SELECTED_REGION% ^
          --tags environment=${params.ENVIRONMENT} owner=jenkins
        """
    }
}

///////////////////////////////////////////////////////////
stage('Generate Enterprise Resource Names') {
    steps {
        script {

            def maxRetries = 20
            def found = false

            for (int i = 0; i < maxRetries; i++) {

                def suffix = powershell(
                    script: "[guid]::NewGuid().ToString('N').Substring(0,10).ToLower()",
                    returnStdout: true
                ).trim()

                def storageName = "${COMPANY_PREFIX}${params.ENVIRONMENT}st${suffix}".toLowerCase()

                if (storageName.length() > 24) continue

                def available = bat(
                    script: """
                    az storage account check-name ^
                      --name ${storageName} ^
                      --query nameAvailable ^
                      -o tsv
                    """,
                    returnStdout: true
                ).trim()

                if (available == "true") {
                    env.STORAGE_ACCOUNT_NAME = storageName
                    echo "Storage account selected → ${storageName}"
                    found = true
                    break
                } else {
                    echo "Name not available → retrying"
                }
            }

            if (!found) {
                error "Unable to generate unique storage account name"
            }

            def vmSuffix = powershell(
                script: "[guid]::NewGuid().ToString('N').Substring(0,5).ToLower()",
                returnStdout: true
            ).trim()

            env.FINAL_VM_NAME = "${params.VM_NAME}-${vmSuffix}"
            echo "VM name → ${env.FINAL_VM_NAME}"
        }
    }
}

///////////////////////////////////////////////////////////
stage('Create Storage Account') {
    steps {
        bat """
        az storage account create ^
          --name %STORAGE_ACCOUNT_NAME% ^
          --resource-group %RESOURCE_GROUP% ^
          --location %SELECTED_REGION% ^
          --sku Standard_LRS ^
          --https-only true ^
          --min-tls-version TLS1_2
        """
    }
}

///////////////////////////////////////////////////////////
stage('Create Windows VM') {
    steps {
        bat """
        az vm create ^
          --resource-group %RESOURCE_GROUP% ^
          --name %FINAL_VM_NAME% ^
          --image Win2019Datacenter ^
          --admin-username %ADMIN_USER% ^
          --admin-password %ADMIN_PASS% ^
          --size %VM_SIZE% ^
          --public-ip-sku Standard
        """
    }
}

///////////////////////////////////////////////////////////
stage('Open RDP Port') {
    steps {
        bat """
        az vm open-port ^
          --resource-group %RESOURCE_GROUP% ^
          --name %FINAL_VM_NAME% ^
          --port 3389
        """
    }
}

///////////////////////////////////////////////////////////
stage('Get VM Public IP') {
    steps {
        bat """
        az vm show ^
          --resource-group %RESOURCE_GROUP% ^
          --name %FINAL_VM_NAME% ^
          -d ^
          --query publicIps ^
          -o table
        """
    }
}

}

///////////////////////////////////////////////////////////
// POST
///////////////////////////////////////////////////////////
post {
    always {
        bat "az logout || exit 0"
    }

    success {
        echo "Infrastructure deployed successfully"
    }

    failure {
        echo "Deployment failed"
    }
}
}
