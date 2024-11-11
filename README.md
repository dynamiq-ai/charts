## How to install Dynamiq AI

```bash
export K8S_VERSION="1.31"

export AWS_PARTITION="aws" 
export CLUSTER_NAME="dynamiq-test"
export AWS_DEFAULT_REGION="us-east-2"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
#export TEMPOUT="$(mktemp)"
export TEMPOUT="/Users/andrey/sites/dynamiq/infra/cloudformation/dynamiq-v2.yaml"

export AMD_AMI_ID="$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2/recommended/image_id --query Parameter.Value --output text)"
export GPU_AMI_ID="$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2-gpu/recommended/image_id --query Parameter.Value --output text)"
```

### Create Required Roles
```bash
#curl -fsSL https://raw.githubusercontent.com/aws/karpenter-provider-aws/v"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml  > "${TEMPOUT}" \
aws cloudformation deploy \
  --stack-name "${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"
```

### Create EKS Cluster
```bash
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
export EXTERNALSECRETS_IAM_ROLE_ARN="arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-external-secrets"

echo "${CLUSTER_ENDPOINT} ${KARPENTER_IAM_ROLE_ARN}"
```

### Create RDS

```bash
export RDS_PASSWORD="d$(date | md5sum  |cut -f1 -d' ')"

aws cloudformation deploy \
  --stack-name "${CLUSTER_NAME}-rds" \
  --template-file "/Users/andrey/sites/dynamiq/infra/cloudformation/dynamiq-rds.yaml" \
  --parameter-overrides ClusterName=${CLUSTER_NAME} DBMasterUserPassword=${RDS_PASSWORD}  
```

## Setup Karpenter
```bash
helm registry logout public.ecr.aws

export KARPENTER_VERSION="1.0.6"

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace "kube-system" \
  --create-namespace \
  --set replicas=1 \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```

```bash
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
```


#### Create Node Pools

##### Platform Node Pool
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

##### Create GPU Node Pools
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

### Setup External Secrets

```bash
helm upgrade --install external-secrets external-secrets \
    --repo https://charts.external-secrets.io \
    --namespace external-secrets \
    --create-namespace \
    --wait
```

#### Create Secret Store
```bash
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: dynamiq
spec:
  provider:
    aws:
      region: "${AWS_DEFAULT_REGION}"
      service: SecretsManager
EOF
```
```bash
kubectl create -k "github.com/fission/fission/crds/v1?ref=v1.20.5" && \
kubectl create namespace apps && \
helm upgrade --install fission fission-all \
  --repo https://fission.github.io/fission-charts/ \
  --namespace fission \
  --create-namespace \
  --set routerServiceType=ClusterIP \
  --set defaultNamespace=apps \
  --set analyticsNonHelmInstall=false \
  --set analytics=false \
  --wait
```

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.ingressClassResource.default=true \
  --wait
```

```bash
export STRING_DATA=$(cat<<EOF
{
  "AUTH_ACCESS_TOKEN_KEY": "change-me-please-i-am-no-secure_abcdefghjklmnopqrstvwxyz12345678",
  "AUTH_REFRESH_TOKEN_KEY": "change-me-please-i-am-no-secure_abcdefghjklmnopqrstvwxyz12345678",
  "AUTH_VERIFICATION_TOKEN_KEY": "change-me-please-i-am-no-secure_abcdefghjklmnopqrstvwxyz12345678",
  "HUGGING_FACE_TOKEN": "token",
  "SENTRY_ENABLED": "false",
  "SENTRY_DSN": "",
  "SMTP_DEFAULT_FROM": "noreply@getdynamiq.app",
  "SMTP_HOST": "127.0.0.1",
  "SMTP_PORT": "587",
  "SMTP_PASSWORD": "password",
  "SMTP_USERNAME": "username"
 }
EOF
)

aws secretsmanager create-secret \
    --no-cli-pager \
    --output table \
    --name DYNAMIQ-NEXUS \
    --description "Dynamiq Application secrets" \
    --secret-string "${STRING_DATA}"

```

```bash
kubectl create namespace dynamiq
            
eksctl create iamserviceaccount \
    --name aws-dynamiq \
    --namespace dynamiq \
    --cluster ${CLUSTER_NAME} \
    --attach-policy-arn arn:aws:iam::aws:policy/AWSMarketplaceMeteringFullAccess \
    --attach-policy-arn arn:aws:iam::aws:policy/AWSMarketplaceMeteringRegisterUsage \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AWSLicenseManagerConsumptionPolicy \
    --approve \
    --override-existing-serviceaccounts
```


```bash
export STORAGE_S3_BUCKET="${CLUSTER_NAME}-data-${RANDOM}"
export BASE_DOMAIN="chart-example.lc"

aws s3api create-bucket \
  --bucket ${STORAGE_S3_BUCKET} \
  --region ${AWS_DEFAULT_REGION} \
  --create-bucket-configuration LocationConstraint=${AWS_DEFAULT_REGION} \
  --output table \
  --no-cli-pager
  

cat <<EOF >local.values.yaml
nexus:
  ingress:
    enabled: true
    hosts:
      - host: nexus.${BASE_DOMAIN}
        paths:
          - path: /
            pathType: Prefix
  configMapData:
    DOMAIN: nexus.${BASE_DOMAIN}
    UI_URL: 'https://ui.${BASE_DOMAIN}'
    STORAGE_S3_BUCKET: dynamiqai-prod

synapse:
  ingress:
    enabled: true
    hosts:
      - host: synapse.${BASE_DOMAIN}
        paths:
          - path: /
            pathType: Prefix
  configMapData:
    STORAGE_S3_BUCKET: dynamiqai-prod

ui:
  ingress:
    enabled: true
    hosts:
      - host: ui.${BASE_DOMAIN}
        paths:
          - path: /
            pathType: Prefix
  configMapData:
    UI_APP_API_URL: 'https://nexus.${BASE_DOMAIN}'
EOF
```

```bash
aws ecr get-login-password \
    --region us-east-1 | helm registry login \
    --username AWS \
    --password-stdin 709825985650.dkr.ecr.us-east-1.amazonaws.com
```

```bash
helm upgrade --install dynamiq oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/dynamiq/dynamiq \
  --namespace dynamiq \
  --values local.values.yaml
```

```bash
kubectl get ingress --namespace dynamiq
```

# Cleanup 

```bash
helm uninstall karpenter --namespace kube-system

kubectl delete nodeclaims --all
```

```bash

aws cloudformation delete-stack \
  --stack-name "${CLUSTER_NAME}-rds" \
  --output table \
  --no-cli-pager

eksctl delete cluster -n ${CLUSTER_NAME}

aws cloudformation delete-stack --stack-name "${CLUSTER_NAME}" \
  --output table \
  --no-cli-pager
  
aws s3api delete-bucket --bucket ${STORAGE_S3_BUCKET}
```

```bash  

```