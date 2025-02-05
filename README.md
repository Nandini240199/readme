properties([
    parameters([
        string(
            defaultValue: 'dev',
            name: 'Environment'
        ),
        choice(
            choices: ['plan', 'apply', 'destroy'], 
            name: 'Terraform_Action'
        )])
])

pipeline {
    agent any
    stages {
        stage('Preparing') {
            steps {
                sh 'echo Preparing'
            }
        }
        
        stage('Git Pulling') {
            steps {
                git branch: 'master', url: 'https://github.com/nandinipatil24/EKS-Terraform-GitHub-Actions.git'
            }
        }

        stage('Init') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    // Debug: List the files in the eks directory to check if it's accessible
                    sh 'ls -R eks/'  
                    
                    // Debug: Print the current directory to ensure we're in the correct path
                    sh 'pwd'  
                    
                    // Initialize terraform in eks directory
                    sh 'terraform -chdir=eks init -migrate-state'   
                }
            }
        }

        stage('Validate') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    sh 'terraform -chdir=eks validate'
                }
            }
        }

        stage('Action') {
            steps {
                withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                    script {
                        if (params.Terraform_Action == 'plan') {
                            sh "terraform -chdir=eks plan -var-file=${params.Environment}.tfvars -lock=false"
                        } else if (params.Terraform_Action == 'apply') {
                            sh "terraform -chdir=eks apply -var-file=${params.Environment}.tfvars -auto-approve -lock=false"
                        } else if (params.Terraform_Action == 'destroy') {
                            sh "terraform -chdir=eks destroy -var-file=${params.Environment}.tfvars -auto-approve -lock=false"
                        } else {
                            error "Invalid value for Terraform_Action: ${params.Terraform_Action}"
                        }
                    }
                }
            }
        }

        stage('Clean Workspace') {
            steps {
                deleteDir() // Deletes all files in the workspace after Terraform operations
            }
        }
    }
}
