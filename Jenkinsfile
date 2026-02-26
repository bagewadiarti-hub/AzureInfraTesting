pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 90, unit: 'MINUTES')
    }

////////////////////////////////////////////////////////////
parameters {
    choice(name: 'ENVIRONMENT', choices: ['dev','test','prod'])
    choice(name: 'PRIMARY_REGION', choices: ['westeurope','centralindia','eastus','westus2'])
    string(name: 'VM_NAME', defaultValue: 'winvm')
}

////////////////////////////////////////////////////////////
environment {
    COMPANY_PREFIX = "corp"
    RESOURCE_GROUP = "rg-${params.ENVIRONMENT}"
    VM_SIZE = "Standard_B2s"
    ADMIN_USER = "azureuser"
    ADMIN_PASS = credentials('azure-vm-admin-password')
}

////////////////////////////////////////////////////////////
stages {

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
          --name %RESOURCE_GROUP% ^
          --location ${params.PRIMARY_REGION}
        """
    }
}

stage('Generate Names') {
    steps {
        script {
            def suffix = powershell(
              script: "[guid]::NewGuid().ToString('N').Substring(0,10)",
              returnStdout: true
            ).trim().toLowerCase()

            env.STORAGE_ACCOUNT_NAME = "corp${params.ENVIRONMENT}st${suffix}"
            env.FINAL_VM_NAME = "${params.VM_NAME}-${suffix.substring(0,5)}"

            echo "Storage → ${env.STORAGE_ACCOUNT_NAME}"
            echo "VM → ${env.FINAL_VM_NAME}"
        }
    }
}

stage('Create Storage') {
    steps {
        bat """
        az storage account create ^
          --name %STORAGE_ACCOUNT_NAME% ^
          --resource-group %RESOURCE_GROUP% ^
          --location ${params.PRIMARY_REGION} ^
          --sku Standard_LRS
        """
    }
}

stage('Create Windows VM') {
    steps {
        bat """
        az vm create ^
          --resource-group %RESOURCE_GROUP% ^
          --name %FINAL_VM_NAME% ^
          --image Win2019Datacenter ^
          --admin-username %ADMIN_USER% ^
          --admin-password %ADMIN_PASS% ^
          --size %VM_SIZE%
        """
    }
}

stage('Open RDP') {
    steps {
        bat """
        az vm open-port ^
          --resource-group %RESOURCE_GROUP% ^
          --name %FINAL_VM_NAME% ^
          --port 3389
        """
    }
}

stage('Get Public IP') {
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

////////////////////////////////////////////////////////////
post {

    always {
        script {
            if (env.WORKSPACE) {
                echo "Logging out from Azure"
                bat "az logout || exit 0"
            } else {
                echo "Workspace not available — skipping logout"
            }
        }
    }

    success {
        echo "Infrastructure deployed successfully"
    }

    failure {
        echo "Deployment failed"
    }
}
}
