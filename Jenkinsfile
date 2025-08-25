pipeline {
    agent any

    environment {
        AWS_REGION    = 'us-east-1'
        VPC_ID        = 'vpc-00856bf74da11cc87'
        SUBNET_ID     = 'subnet-03e723a43edad34d9'
        KEY_NAME      = 'SonarQube.pem'
        INSTANCE_TYPE = 't3.medium'
        SG_NAME       = "sonarqube-sg-${env.BUILD_NUMBER}"
        TAG_NAME      = "sonarqube-${env.BUILD_NUMBER}"
        AWS_CRED_ID   = 'aws-cred'
    }

    options {
        timestamps()
    }

    stages {
        stage('Prepare AWS CLI & User Data') {
            steps {
                sh '''#!/bin/bash


# Prepare userdata script
cat > userdata.sh <<'EOF'
#!/bin/bash
set -eux

if [ "$(id -u)" -ne 0 ]; then
  exec sudo -H bash "$0" "$@"
fi

sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' > /etc/sysctl.d/99-sonarqube.conf

if [ -f /etc/amazon-linux-release ] || grep -qi "Amazon Linux" /etc/os-release; then
  yum update -y
  amazon-linux-extras install docker -y || yum install -y docker
  systemctl enable docker
  systemctl start docker
else
  apt-get update -y
  apt-get install -y docker.io
  systemctl enable docker
  systemctl start docker
fi

mkdir -p /opt/sonarqube/{data,extensions,logs}

docker run -d --name sonarqube \
  --restart always \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  -v /opt/sonarqube/data:/opt/sonarqube/data \
  -v /opt/sonarqube/extensions:/opt/sonarqube/extensions \
  -v /opt/sonarqube/logs:/opt/sonarqube/logs \
  sonarqube:lts-community
EOF

chmod +x userdata.sh
'''
            }
        }

        stage('Provision Security Group') {
            steps {
                withAWS(credentials: "${env.AWS_CRED_ID}", region: "${env.AWS_REGION}") {
                    sh '''#!/bin/bash
set -euo pipefail

MY_IP="$(curl -s https://checkip.amazonaws.com || true)"
if [ -n "$MY_IP" ]; then
  CIDR="${MY_IP}/32"
else
  CIDR="0.0.0.0/0"
fi

SG_ID=$(aws ec2 create-security-group \
  --group-name "${SG_NAME}" \
  --description "SonarQube SG for build ${BUILD_NUMBER}" \
  --vpc-id "${VPC_ID}" \
  --query 'GroupId' --output text)

echo "SG_ID=${SG_ID}" > sg.env

aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --ip-permissions "IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges=[{CidrIp=${CIDR},Description=\\"SSH from Jenkins\\"}]"

aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --ip-permissions "IpProtocol=tcp,FromPort=9000,ToPort=9000,IpRanges=[{CidrIp=${CIDR},Description=\\"SonarQube UI\\"}]"

echo "Created Security Group: $SG_ID with ingress from $CIDR"
'''
                }
            }
        }

        stage('Launch EC2 with SonarQube') {
            steps {
                withAWS(credentials: "${env.AWS_CRED_ID}", region: "${env.AWS_REGION}") {
                    sh '''#!/bin/bash
set -euo pipefail
source sg.env

AMI_ID=$(aws ssm get-parameters \
  --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 \
  --query 'Parameters[0].Value' --output text)
echo "Using AMI: $AMI_ID"

RUN_JSON=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type "${INSTANCE_TYPE}" \
  --subnet-id "${SUBNET_ID}" \
  --security-group-ids "$SG_ID" \
  --key-name "${KEY_NAME}" \
  --user-data file://userdata.sh \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${TAG_NAME}}]" \
  --count 1)

INSTANCE_ID=$(echo "$RUN_JSON" | jq -r '.Instances[0].InstanceId')
echo "INSTANCE_ID=${INSTANCE_ID}" > ec2.env
echo "Launched instance: $INSTANCE_ID"

aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

echo "PUBLIC_IP=${PUBLIC_IP}" >> ec2.env
echo "SonarQube will be available at: http://${PUBLIC_IP}:9000"
'''
                }
            }
        }

        stage('Output') {
            steps {
                script {
                    def envFile = readFile 'ec2.env'
                    echo envFile
                    def ip = sh(returnStdout: true, script: "awk -F= '/PUBLIC_IP/{print \$2}' ec2.env | tr -d '\\n'").trim()
                    echo "Open SonarQube: http://${ip}:9000"
                    echo "Default login: admin / admin (you will be prompted to change password)"
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*.env', onlyIfSuccessful: false
        }
    }
}