## How to install Dynamiq AI

```bash
export KARPENTER_VERSION="1.0.6"
export K8S_VERSION="1.31"

export AWS_PARTITION="aws" 
export CLUSTER_NAME="dynamiq-test"
export AWS_DEFAULT_REGION="us-east-1"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
#export TEMPOUT="$(mktemp)"
export TEMPOUT="/Users/andrey/sites/dynamiq/infra/cloudformation/dynamiq-v2.yaml"

export AMD_AMI_ID="$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2/recommended/image_id --query Parameter.Value --output text)"
export GPU_AMI_ID="$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2-gpu/recommended/image_id --query Parameter.Value --output text)"
```

```bash
#curl -fsSL https://raw.githubusercontent.com/aws/karpenter-provider-aws/v"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml  > "${TEMPOUT}" \
aws cloudformation deploy \
  --stack-name "${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"

eksctl create cluster -f - <<EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  version: "${K8S_VERSION}"
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}

iam:
  withOIDC: true
  podIdentityAssociations:
  - serviceAccountName: karpenter 
    namespace: "kube-system"
    roleName: ${CLUSTER_NAME}-karpenter
    permissionPolicyARNs:
    - arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}
  - serviceAccountName: external-secrets
    namespace: "external-secrets"
    roleName: ${CLUSTER_NAME}-external-secrets
    permissionPolicyARNs:
    - arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:policy/ExternalSecretsPolicy-${CLUSTER_NAME}

iamIdentityMappings:
- arn: "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}"
  username: system:node:{{EC2PrivateDNSName}}
  groups:
  - system:bootstrappers
  - system:nodes

managedNodeGroups:
- instanceType: m5.large
  amiFamily: AmazonLinux2
  name: ${CLUSTER_NAME}-ng
  desiredCapacity: 1
  minSize: 1
  maxSize: 2

addons:
- name: eks-pod-identity-agent
EOF

export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name "${CLUSTER_NAME}" --query "cluster.endpoint" --output text)"
export KARPENTER_IAM_ROLE_ARN="arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"
export EXTERNALSECRETS_IAM_ROLE_ARN="arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"

echo "${CLUSTER_ENDPOINT} ${KARPENTER_IAM_ROLE_ARN}"
```

### Create RDS
```bash
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=eksctl-${CLUSTER_NAME}-cluster/VPC" --query 'Vpcs[*].VpcId' --output json | jq -c '.[0]')
export PRIVATE_SUBNETS_ID=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=eksctl-${CLUSTER_NAME}-cluster/SubnetPrivate*" \
    --query 'Subnets[*].SubnetId' \
    --output json | jq -c .)
    
aws rds create-db-subnet-group \
    --db-subnet-group-name rds-${CLUSTER_NAME} \
    --db-subnet-group-description rds-${CLUSTER_NAME} \
    --subnet-ids ${PRIVATE_SUBNETS_ID}

export RDS_PASSWORD="$(date | md5sum  |cut -f1 -d' ')"
echo "\r\nRDS_PASSWORD=${RDS_PASSWORD}\r\n"

aws rds create-db-instance \
    --db-instance-identifier rds-${CLUSTER_NAME} \
    --db-name dynamiq \
    --db-instance-class db.t4g.small \
    --engine postgres \
    --db-subnet-group-name "rds-${CLUSTER_NAME}" \
    --master-username dynamiq \
    --master-user-password ${RDS_PASSWORD} \
    --backup-retention-period 0 \
    --allocated-storage 20

aws rds wait db-instance-available --db-instance-identifier rds-${CLUSTER_NAME}
```



```bash
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
```

```bash
helm registry logout public.ecr.aws

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version "${KARPENTER_VERSION}" --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```



```bash
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: platform
spec:
  role: "KarpenterNodeRole-${CLUSTER_NAME}" # replace with your cluster name
  amiFamily: AL2
  amiSelectorTerms:
    - id: ${AMD_AMI_ID}
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        deleteOnTermination: true
        encrypted: true
        iops: 3000
        throughput: 125
        volumeSize: 100Gi
        volumeType: gp3
  metadataOptions:
    httpEndpoint: enabled
    httpProtocolIPv6: disabled
    httpPutResponseHopLimit: 2
    httpTokens: required
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
---
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: m5
spec:
  disruption:
    budgets:
      - nodes: 10%
    consolidationPolicy: WhenUnderutilized
    expireAfter: 48h
  limits:
    cpu: 128
  template:
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: platform
      requirements:
        - key: getdynamiq.ai/workload
          operator: In
          values: ["application"]
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["m5"]
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ["large", "xlarge", "2xlarge", "4xlarge", "8xlarge"]
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["on-demand"]
EOF
```

```bash
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: gpu
spec:
  role: "KarpenterNodeRole-${CLUSTER_NAME}" # replace with your cluster name
  amiFamily: AL2
  amiSelectorTerms:
    - id: ${GPU_AMI_ID}
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        deleteOnTermination: true
        encrypted: true
        iops: 3000
        throughput: 125
        volumeSize: 300Gi
        volumeType: gp3
  metadataOptions:
    httpEndpoint: enabled
    httpProtocolIPv6: disabled
    httpPutResponseHopLimit: 2
    httpTokens: required
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
---
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: gpu-g5
spec:
  disruption:
    consolidateAfter: 1m0s
    consolidationPolicy: WhenEmpty
    expireAfter: Never
  limits:
    cpu: 256
  template:
    spec:
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: gpu
      requirements:
        - key: nvidia.com/gpu
          operator: In
          values:
            - 'true'
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["g5"]
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ["xlarge", "2xlarge", "4xlarge", "8xlarge", "16xlarge", "12xlarge", "24xlarge", "48xlarge"]
        - key: "karpenter.sh/capacity-type"
          operator: In
          values: ["on-demand"]
      taints:
        - effect: NoSchedule
          key: nvidia.com/gpu
          value: 'true'        
EOF
```


```bash
helm repo add external-secrets https://charts.external-secrets.io

helm upgrade --install external-secrets external-secrets/external-secrets \
    -n external-secrets \
    --create-namespace \
    --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="${EXTERNALSECRETS_IAM_ROLE_ARN}" \
    --wait
```

```bash
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: dynamiq
spec:
  provider:
    aws:
      region: us-east-1
      service: SecretsManager
      role: ${EXTERNALSECRETS_IAM_ROLE_ARN}
#---
#apiVersion: external-secrets.io/v1beta1
#kind: ExternalSecret
#metadata:
#  name: nexus-db-secret
#spec:
#  refreshInterval: 1h
#  secretStoreRef:
#    kind: ClusterSecretStore
#    name: dynamiq
#  target:
#    name: nexus-db-secret
#    template:
#      type: Opaque
#      data:
#        DATABASE_PORT: "5432"
#        DATABASE_SSLMODE: "require"
#        DATABASE_SCHEMA: "public"
#        DATABASE_NAME: "{{ .database }}"
#        DATABASE_HOST: "{{ .server_name }}"
#        DATABASE_USERNAME: "{{ .username }}"
#        DATABASE_PASSWORD: "{{ .password | urlquery }}"
#  dataFrom:
#    - extract:
#        key: DYNAMIQ-DB
EOF
```
