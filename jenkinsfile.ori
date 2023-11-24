pipeline {
    agent any
    
    environment {
        AWS_ACCESS_KEY_ID = credentials('MASTER_AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('MASTER_AWS_SECRET_ACCESS_KEY')
        AWS_DEFAULT_REGION = "us-east-1"
    }

    stages {
        stage('Check Account') {
            steps {
                script {
                    sh '''
                        awsaccount=$(echo $account | cut -d" " -f1)
                        
                        rolearn="arn:aws:iam::$awsaccount:role/Terraform"
                        temp_role=$(aws sts assume-role --role-arn $rolearn --role-session-name AWSCLI-Session)
                        
                        export AWS_ACCESS_KEY_ID=$(echo $temp_role | jq -r .Credentials.AccessKeyId)
                        export AWS_SECRET_ACCESS_KEY=$(echo $temp_role | jq -r .Credentials.SecretAccessKey)
                        export AWS_SESSION_TOKEN=$(echo $temp_role | jq -r .Credentials.SessionToken)
                        
                        get_id=$(aws sts get-caller-identity)
                        check_account=$(echo $get_id | jq -r .Account)
                        
                        if [ "$check_account" = "$awsaccount" ]
                        then
                            echo "OK!"
                        else
                            echo "Could NOT connect to the account $awsaccount"
                            exit 1
                        fi
                        
                        aws rds describe-db-instances --query 'DBInstances[*].[DBInstanceIdentifier]' \
                        --region $region --output text
                    '''
                }
            }
        }
        
        stage('Take RDS Snapshot') {
            steps {
                script {
                    sh '''
                        awsaccount=$(echo $account | cut -d" " -f1)
                        
                        rolearn="arn:aws:iam::$awsaccount:role/Terraform"
                        temp_role=$(aws sts assume-role --role-arn $rolearn --role-session-name AWSCLI-Session)
                        
                        export AWS_ACCESS_KEY_ID=$(echo $temp_role | jq -r .Credentials.AccessKeyId)
                        export AWS_SECRET_ACCESS_KEY=$(echo $temp_role | jq -r .Credentials.SecretAccessKey)
                        export AWS_SESSION_TOKEN=$(echo $temp_role | jq -r .Credentials.SessionToken)
                        
                        get_id=$(aws sts get-caller-identity)
                        check_account=$(echo $get_id | jq -r .Account)
                        
                        ## Check if RDS instance exist
                        existingrdsinstance=$(aws rds describe-db-instances --query 'DBInstances[*].[DBInstanceIdentifier]' \
                        --region $region --filters Name=db-instance-id,Values=$rdsinstanceidentifier --output text)
                            
                        if [ ! -z $existingrdsinstance ]
                        then
                            echo "OK!"
                        else
                            echo "RDS INSTANCE $rdsinstanceidentifier DOES NOT EXIST!"
                            exit 1
                        fi
                        
                        ## Check if RDS status is available
                        rdsinstanceavailable=$(aws rds describe-db-instances --query 'DBInstances[].DBInstanceStatus' \
                        --region $region --filters Name=db-instance-id,Values=$rdsinstanceidentifier --output text)
                        
                        if [ $rdsinstanceavailable = 'available' ]
                        then
                            echo "OK!"
                        else
                            echo "RDS instance $rdsinstanceidentifier status NOT available!"
                            exit 1
                        fi
                        
                        ## Check snapshot name exist
                        snapshotname=$(aws rds describe-db-snapshots --query 'DBSnapshots[].DBSnapshotIdentifier' \
                        --region $region --filters Name=db-snapshot-id,Values=$rdssnapshotname --output text)
                        
                        if [ -z $snapshotname ]
                        then
                            echo "OK!"
                        else
                            echo "Snapshot already created. Choose another name!"
                            exit 1
                        fi
                        
                        ## Start creating snapshot
                        aws rds create-db-snapshot \
                            --db-instance-identifier $rdsinstanceidentifier --db-snapshot-identifier $rdssnapshotname \
                            --region $region
                        i=1
                        while [ $i -lt 100 ]; do i=$(aws rds describe-db-snapshots --snapshot-type manual \
                            --query 'DBSnapshots[].[PercentProgress]' --region $region \
                            --filters Name=db-snapshot-id,Values=$rdssnapshotname --output text);
                            echo "SNAPSHOT CREATION PROGRESS:" $i"%"; sleep 15;
                        done
                    '''
                }
            }
        }
    }
}
