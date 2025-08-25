pipeline {
  agent any

  environment {
    AWS_REGION    = 'us-east-1'
    VPC_ID        = 'vpc-00856bf74da11cc87'
    SUBNET_ID     = 'subnet-03e723a43edad34d9'
    //KEY_NAME      = 'SonarQube.pem'
    INSTANCE_TYPE = 't2.medium'
    SG_NAME       = "sonarqube-sg-${env.BUILD_NUMBER}"
    TAG_NAME      = "sonarqube-${env.BUILD_NUMBER}"

    // Jenkins credential ID for AWS (Username = AWS_ACCESS_KEY_ID, Password = AWS_SECRET_ACCESS_KEY)
    AWS_CRED_ID   = 'AWS'
  }

  options {
    timestamps()
  }

  stages {
    stage('Prepare AWS CLI & User Data') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${AWS_CRED_ID}",
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        )]) {
          sh '''bash -euxo pipefail << 'EOF'
# Generate userdata.sh
cat > userdata.sh << 'EOD'
#!/bin/bash
set -eux

# Ensure we run as root
if [ "$(id -u)" -ne 0 ]; then
  exec sudo -H bash "$0" "$@"
fi

# Elasticsearch tuning
sysctl -w vm.max_map_count=262144
echo 'vm.max_map_count=262144' > /etc/sysctl.d/99-sonarqube.conf

# Install Docker
if grep -qi amazon-linux /etc/os-release; then
  yum update -y
  amazon-linux-extras install docker -y || yum install -y docker
  systemctl enable docker && systemctl start docker
else
  apt-get update -y
  apt-get install -y docker.io
  systemctl enable docker && systemctl start docker
fi

# Prepare SonarQube data dirs
mkdir -p /opt/sonarqube/{data,extensions,logs}

# Launch SonarQube container
docker run -d --name sonarqube \\
  --restart always \\
  -p 9000:9000 \\
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \\
  -v /opt/sonarqube/data:/opt/sonarqube/data \\
  -v /opt/sonarqube/extensions:/opt/sonarqube/extensions \\
  -v /opt/sonarqube/logs:/opt/sonarqube/logs \\
  sonarqube:lts-community
EOD

chmod +x userdata.sh
EOF'''
        }
      }
    }

    stage('Provision Security Group') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${AWS_CRED_ID}",
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        )]) {
          sh '''bash -euxo pipefail << 'EOF'
# Determine your IP for restricted access
MY_IP=$(curl -s https://checkip.amazonaws.com || echo "")
if [ -n "$MY_IP" ]; then
  CIDR="${MY_IP}/32"
else
  CIDR="0.0.0.0/0"
fi

# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name "${SG_NAME}" \
  --description "SonarQube SG for build ${BUILD_NUMBER}" \
  --vpc-id "${VPC_ID}" \
  --query GroupId --output text)
echo "SG_ID=${SG_ID}" > sg.env

# Open SSH + SonarQube ports
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --ip-permissions "IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges=[{CidrIp=${CIDR}}]"
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" \
  --ip-permissions "IpProtocol=tcp,FromPort=9000,ToPort=9000,IpRanges=[{CidrIp=${CIDR}}]"

echo "Created SG $SG_ID with ingress from $CIDR"
EOF'''
        }
      }
    }

    stage('Launch EC2 with SonarQube') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: "${AWS_CRED_ID}",
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        )]) {
          sh '''bash -euxo pipefail << 'EOF'
# Load security group ID
source sg.env

# Lookup latest Amazon Linux 2 AMI (SSM Parameter)
AMI_ID=$(aws ssm get-parameters \
  --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 \
  --query 'Parameters[0].Value' --output text)

# Launch EC2 instance with our userdata
RUN_JSON=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type "${INSTANCE_TYPE}" \
  --subnet-id "${SUBNET_ID}" \
  --security-group-ids "$SG_ID" \
  --user-data file://userdata.sh \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${TAG_NAME}}]" \
  --count 1)

RUN_JSON=$(aws ec2 run-instances …)
INSTANCE_ID=$(echo "$RUN_JSON" | jq -r '.Instances[0].InstanceId')
echo "INSTANCE_ID=${INSTANCE_ID}" > ec2.env

# Wait until running, then fetch the public IP
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)
echo "PUBLIC_IP=${PUBLIC_IP}" >> ec2.env

echo "SonarQube available at http://${PUBLIC_IP}:9000"
EOF'''
        }
      }
    }

    stage('Output') {
      steps {
        script {
          // Print out and parse ec2.env
          echo readFile('ec2.env')
          def ip = sh(
            script: "awk -F= '/PUBLIC_IP/{print \$2}' ec2.env",
            returnStdout: true
          ).trim()
          echo "→ Access SonarQube UI: http://${ip}:9000"
          echo "   Default credentials: admin / admin"
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