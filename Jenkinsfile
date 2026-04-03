pipeline {
    agent any
    parameters {
        string(name: 'TARGET_ACCOUNT_IDS', defaultValue: '442889374161', description: 'Comma-separated AWS account IDs')
        string(name: 'REGION', defaultValue: 'ap-south-1', description: 'AWS Region')
    }
    environment {
        ROLE_NAME = 'Jenkins-CrossAccount-Admin'
        S3_BUCKET = 'jenkins-log-files-s3-bucket'
    }
    stages {
        stage('Prepare') {
            steps {
                script {
                    if (!params.TARGET_ACCOUNT_IDS?.trim()) {
                        error "TARGET_ACCOUNT_IDS is empty. Provide one or more account IDs separated by commas."
                    }
                }
            }
        }

        stage('EC2 Inventory') {
            steps {
                script {
                    def accounts = params.TARGET_ACCOUNT_IDS.split(',').collect { it.trim() }.findAll { it }
                    def branches = [:]

                    for (int i = 0; i < accounts.size(); i++) {
                        def accountId = accounts[i]
                        branches["Account-${accountId}"] = {
                            node {
                                stage("Inventory for Account-${accountId}") {
                                    withEnv(["REGION=${params.REGION}"]) {
                                        sh """
                                            set -euo pipefail
                                            set -x

                                            echo "========================================"
                                            echo "Processing account: ${accountId}"
                                            echo "========================================"

                                            # Assume role in target account
                                            CREDS=\$(aws sts assume-role \\
                                                --role-arn arn:aws:iam::${accountId}:role/${ROLE_NAME} \\
                                                --role-session-name jenkins-session-${accountId} \\
                                                --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \\
                                                --output text) || { echo "assume-role failed"; exit 2; }

                                            export AWS_ACCESS_KEY_ID=\$(echo \$CREDS | awk '{print \$1}')
                                            export AWS_SECRET_ACCESS_KEY=\$(echo \$CREDS | awk '{print \$2}')
                                            export AWS_SESSION_TOKEN=\$(echo \$CREDS | awk '{print \$3}')

                                            # Verify identity
                                            aws sts get-caller-identity

                                            # Fetch EC2 instances JSON
                                            aws ec2 describe-instances --region $REGION --output json > instances-${accountId}.json

                                            # Temporary file for instance rows (no header)
                                            TMP_ROWS="inventory-${accountId}.tmp"
                                            : > "\$TMP_ROWS"

                                            # Iterate each instance and build CSV rows with VolumeSizes
                                            jq -c '.Reservations[]?.Instances[]?' instances-${accountId}.json | while read -r INST_JSON; do
                                                # Extract fields (use jq to safely get values; replace newlines/commas where needed)
                                                NAME=\$(echo "\$INST_JSON" | jq -r '(.Tags // [])[]? | select(.Key=="Name") | .Value' 2>/dev/null || true)
                                                NAME=\$(echo "\$NAME" | sed 's/\\r//g' | sed 's/,/ /g')
                                                INSTANCE_ID=\$(echo "\$INST_JSON" | jq -r '.InstanceId // ""')
                                                STATE=\$(echo "\$INST_JSON" | jq -r '.State.Name // ""')
                                                INSTANCE_TYPE=\$(echo "\$INST_JSON" | jq -r '.InstanceType // ""')
                                                AZ=\$(echo "\$INST_JSON" | jq -r '.Placement.AvailabilityZone // ""')
                                                PRIVATE_IP=\$(echo "\$INST_JSON" | jq -r '.PrivateIpAddress // ""')
                                                PUBLIC_DNS=\$(echo "\$INST_JSON" | jq -r '.PublicDnsName // ""')
                                                PUBLIC_IP=\$(echo "\$INST_JSON" | jq -r '.PublicIpAddress // ""')
                                                # Elastic IP from network interfaces association (first match)
                                                ELASTIC_IP=\$(echo "\$INST_JSON" | jq -r '(.NetworkInterfaces // [])[]?.Association?.PublicIp' 2>/dev/null | head -n1 || true)
                                                IPV6=\$(echo "\$INST_JSON" | jq -r '(.NetworkInterfaces // [])[]?.Ipv6Addresses[]? // empty' 2>/dev/null | paste -sd ";" - || true)
                                                SECURITY_GROUPS=\$(echo "\$INST_JSON" | jq -r '([.SecurityGroups[]?.GroupName] | join(\";\")) // ""')
                                                KEY_NAME=\$(echo "\$INST_JSON" | jq -r '.KeyName // ""')
                                                LAUNCH_TIME=\$(echo "\$INST_JSON" | jq -r '.LaunchTime // ""')
                                                PLATFORM=\$(echo "\$INST_JSON" | jq -r '.Platform // "Linux/UNIX"')
                                                PLATFORM_DETAILS=\$(echo "\$INST_JSON" | jq -r '.PlatformDetails // ""')
                                                AMI=\$(echo "\$INST_JSON" | jq -r '.ImageId // ""')
                                                VPC_ID=\$(echo "\$INST_JSON" | jq -r '.VpcId // ""')
                                                SUBNET_ID=\$(echo "\$INST_JSON" | jq -r '.SubnetId // ""')
                                                NETWORK_INTERFACES=\$(echo "\$INST_JSON" | jq -r '([.NetworkInterfaces[]?.NetworkInterfaceId] | join(\";\")) // ""')

                                                # Volume IDs for this instance (joined by ;)
                                                VOLUME_IDS=\$(echo "\$INST_JSON" | jq -r '([.BlockDeviceMappings[]?.Ebs?.VolumeId] | join(\";\")) // ""')

                                                VOLUME_SIZES=""
                                                if [ -n "\$VOLUME_IDS" ]; then
                                                    # Convert semicolon-separated to space-separated for aws CLI
                                                    VIDS=\$(echo "\$VOLUME_IDS" | tr ';' ' ')
                                                    # Describe volumes to get sizes (GB) in the same order returned by AWS
                                                    # aws returns volumes in unspecified order; to preserve mapping we will query each volume individually in the same order
                                                    SIZES_LIST=""
                                                    for vid in \$VIDS; do
                                                        size=\$(aws ec2 describe-volumes --volume-ids \$vid --region $REGION --query 'Volumes[0].Size' --output text 2>/dev/null || echo "")
                                                        if [ -z "\$SIZES_LIST" ]; then
                                                            SIZES_LIST="\$size"
                                                        else
                                                            SIZES_LIST="\$SIZES_LIST;\$size"
                                                        fi
                                                    done
                                                    VOLUME_SIZES=\$SIZES_LIST
                                                fi

                                                # Escape any commas in fields by replacing with space (simple approach)
                                                # Build CSV row (fields in same order as header below, excluding S.NO)
                                                echo "\$NAME,\$INSTANCE_ID,\$STATE,\$INSTANCE_TYPE,\$AZ,\$PRIVATE_IP,\$PUBLIC_DNS,\$PUBLIC_IP,\$ELASTIC_IP,\$IPV6,\$SECURITY_GROUPS,\$KEY_NAME,\$LAUNCH_TIME,\$PLATFORM,\$PLATFORM_DETAILS,\$AMI,\$VOLUME_IDS,\$VOLUME_SIZES,\$VPC_ID,\$SUBNET_ID,\$NETWORK_INTERFACES" >> "\$TMP_ROWS"
                                            done

                                            # Unset assumed-role credentials to switch back to Jenkins account
                                            unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

                                            # Prepare final CSV with header and S.NO as first column
                                            HEADER="S.NO,Name,InstanceID,InstanceState,InstanceType,AvailabilityZone,PrivateIP,PublicDNS,PublicIP,ElasticIP,IPv6,SecurityGroups,KeyName,LaunchTime,Platform,PlatformDetails,AMI,VolumeIDs,VolumeSizes,VPCID,SubnetIDs,NetworkInterfaces"
                                            echo "\$HEADER" > inventory-${accountId}.csv

                                            # Add serial numbers to each data row and append to final CSV
                                            if [ -s "\$TMP_ROWS" ]; then
                                                awk 'BEGIN{FS=OFS=","} {print NR\",\"\$0}' "\$TMP_ROWS" >> inventory-${accountId}.csv
                                            fi

                                            # Create IST timestamped filename
                                            TIMESTAMP=\$(TZ="Asia/Kolkata" date +"%d-%m-%Y-%I-%M%p")
                                            OUTPUT_FILE="Ec2-Inventory_${accountId}_\${TIMESTAMP}.csv"
                                            mv inventory-${accountId}.csv "\$OUTPUT_FILE"

                                            # Upload CSV to S3 using Jenkins credentials
                                            aws s3 cp "\$OUTPUT_FILE" s3://${S3_BUCKET}/ec2-inventory/${accountId}/

                                            echo "Inventory for account ${accountId} uploaded to s3://${S3_BUCKET}/ec2-inventory/${accountId}/\${OUTPUT_FILE}"
                                        """
                                    }
                                }
                            }
                        }
                    }

                    // run all account branches in parallel
                    parallel branches
                }
            }
        }
    }

    post {
        always {
            script {
                echo "EC2 inventory pipeline finished."
            }
        }
    }
}
