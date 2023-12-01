pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID = credentials('MASTER_AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('MASTER_AWS_SECRET_ACCESS_KEY')
        RDS = ""
        SELECTED_RDS = ""
        SELECTED_RDS_STATUS = ""
        SNAPSHOT_NAME = ""
    }

    stages {
        
        stage('Check RDS Status') {
            steps {
                script {
                    RDS = sh(script: '''
                        # Connect to AWS account
                        awsaccount=$(echo $account | cut -d" " -f1)
                        rolearn="arn:aws:iam::$awsaccount:role/Terraform"
                        temp_role=$(aws sts assume-role --role-arn $rolearn --role-session-name AWSCLI-Session)
                        export AWS_ACCESS_KEY_ID=$(echo $temp_role | jq -r .Credentials.AccessKeyId)
                        export AWS_SECRET_ACCESS_KEY=$(echo $temp_role | jq -r .Credentials.SecretAccessKey)
                        export AWS_SESSION_TOKEN=$(echo $temp_role | jq -r .Credentials.SessionToken)
                        
                        # List the RDS instances
                        aws rds describe-db-instances \
                            --query 'DBInstances[*].[DBInstanceIdentifier]' \
                            --region $region --output text
                    ''', returnStdout: true).trim()

                    echo "RDS: ${RDS}"
                    
                    // Select the RDS instance
                    def choicesList = RDS.tokenize()
                    def userInput = input(
                        id: 'userInput',
                        message: 'Select to continue',
                        parameters: [choice(name: 'SELECTED_RDS', choices: choicesList, description: 'Select the RDS from the list ')]
                    )
                    
                    SELECTED_RDS = userInput
                    echo "Selected RDS: ${SELECTED_RDS}"
                    
                    withEnv(["SELECTED_RDS=${SELECTED_RDS}"]) {
                        SELECTED_RDS_STATUS = sh(script: """
                            awsaccount=\$(echo \$account | cut -d" " -f1)
                            rolearn="arn:aws:iam::\$awsaccount:role/Terraform"
                            temp_role=\$(aws sts assume-role --role-arn \$rolearn --role-session-name AWSCLI-Session)
                            export AWS_ACCESS_KEY_ID=\$(echo \$temp_role | jq -r .Credentials.AccessKeyId)
                            export AWS_SECRET_ACCESS_KEY=\$(echo \$temp_role | jq -r .Credentials.SecretAccessKey)
                            export AWS_SESSION_TOKEN=\$(echo \$temp_role | jq -r .Credentials.SessionToken)
                            
                            # Check if selected RDS instance status is available
                            aws rds describe-db-instances \\
                                --query 'DBInstances[].DBInstanceStatus' \\
                                --filters Name=db-instance-id,Values=\$SELECTED_RDS \\
                                --region \$region \\
                                --output text
                        """, returnStdout: true).trim()
                    }
                    
                    // Print an error if RDS status is not available
                    if (SELECTED_RDS_STATUS != "available") {
                        error "RDS instance: ${SELECTED_RDS} status is not available!"
                    }
                }
            }
        }

        stage('Check if Snapshot Exists') {
            steps {
                script {
                    def snapshotExists = sh(script: """
                        awsaccount=\$(echo \$account | cut -d" " -f1)
                        rolearn="arn:aws:iam::\$awsaccount:role/Terraform"
                        temp_role=\$(aws sts assume-role --role-arn \$rolearn --role-session-name AWSCLI-Session)
                        export AWS_ACCESS_KEY_ID=\$(echo \$temp_role | jq -r .Credentials.AccessKeyId)
                        export AWS_SECRET_ACCESS_KEY=\$(echo \$temp_role | jq -r .Credentials.SecretAccessKey)
                        export AWS_SESSION_TOKEN=\$(echo \$temp_role | jq -r .Credentials.SessionToken)
                        
                        # Check if snapshot name exist
                        echo "Executing AWS CLI command..."
                        aws rds describe-db-snapshots \
                            --query 'DBSnapshots[*].[DBSnapshotIdentifier]' \
                            --snapshot-type manual \
                            --region \$region \
                            --output text | grep -w \$snapshotName
                    """, returnStatus: true) == 0
        
                    echo "snapshotExists: ${snapshotExists}"
                    
                    // Print an error if snapshot exists
                    if (snapshotExists) {
                        error "Snapshot: $snapshotName already exists."
                    }
                }
            }
        }

        stage ('Create RDS snapshot') {
            steps {
                script {
                    withEnv(["SELECTED_RDS=${SELECTED_RDS}"]) {
                        createSnapshot = sh(script: """
                            awsaccount=\$(echo \$account | cut -d" " -f1)
                            rolearn="arn:aws:iam::\$awsaccount:role/Terraform"
                            temp_role=\$(aws sts assume-role --role-arn \$rolearn --role-session-name AWSCLI-Session)
                            export AWS_ACCESS_KEY_ID=\$(echo \$temp_role | jq -r .Credentials.AccessKeyId)
                            export AWS_SECRET_ACCESS_KEY=\$(echo \$temp_role | jq -r .Credentials.SecretAccessKey)
                            export AWS_SESSION_TOKEN=\$(echo \$temp_role | jq -r .Credentials.SessionToken)
                            
                            # Create the snapshot
                            echo "Creating snapshot..."
                            aws rds create-db-snapshot \\
                                --db-instance-identifier \$SELECTED_RDS \\
                                --db-snapshot-identifier $snapshotName \\
                                --region $region
                            
                            # Show snapshot creation progress
                            i=1
                            while [ \$i -lt 100 ]; do 
                                i=\$(aws rds describe-db-snapshots --snapshot-type manual \\
                                    --query 'DBSnapshots[].[PercentProgress]' --region $region \\
                                    --filters Name=db-snapshot-id,Values=$snapshotName \\
                                    --output text);
                                    echo "SNAPSHOT CREATION PROGRESS:" \$i"%"; sleep 15;
                            done
                        """, returnStdout: true).trim()
                    }
                }
            }
        }
    }
}
