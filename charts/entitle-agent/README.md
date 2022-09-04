# Entitle Agent Helm Chart

## Introduction
This Helm chart guide will take you through the installation of Entitle agent on your cluster.

##### What will be installed using this Helm chart
* Keys to pull the docker image of our agent from github container registry.
* DataDog helm chart that will help us help you :)
* New namespace.
* Deployment roles and other kubernetes CRD.

## Prerequisites

* Kubernetes cluster is required to run our Helm on - Entitle agent needs its own namespace (we create one in the chart) so we can
  run with other tools in a friendly manner
    * **NOTICE:** If you don't have an existing Kubernetes cluster it is recommended to use our IAC to deploy one
      including the roles/annotations
* ability to read and write to KMSs
    * We have our own guide/IAC for each cloud provider in order to give access to the Entitle agent
## Installation
### Prepare Installation
```shell
helm repo add entitle https://anycred.github.io/entitle-charts/
```
## Amazon installation

#### A. Declare Variables

1. Define your cluser's name:
   ```shell
    export CLUSTER_NAME=<your-cluster-name>
   ```

2. Update kubeconfig:
   ```shell
    aws eks update-kubeconfig --name $CLUSTER_NAME --region us-east-1   # (or any other region)
   ```

3. **Notice:** If you installed our IaC then you may skip to the [chart installation part](#chart-installation).

#### B. [Create OIDC Provider](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)

You can check if you already have the Identity Provider for your cluster using one of the following:

- Run the following command:
  ```shell
  aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text
  ```
- Alternatively, refer to [IAM Identity Providers](https://console.aws.amazon.com/iamv2/home#/identity_providers) page in AWS Console.

If you don't have an OIDC provider, create new one:
```shell
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve
```

#### C. [Create IAM Policy and Role](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)

<details>
  <summary>Create policy</summary>
    
  ```shell
  ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
  echo $ACCOUNT_ID

  cat > entitle-agent-policy.json <<ENDOF
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Sid": "VisualEditor0",
              "Effect": "Allow",
              "Action": [
                "secretsmanager:UpdateSecret",
                "secretsmanager:TagResource",
                "secretsmanager:PutSecretValue",
                "secretsmanager:ListSecretVersionIds",
                "secretsmanager:GetSecretValue",
                "secretsmanager:GetResourcePolicy",
                "secretsmanager:DescribeSecret",
                "secretsmanager:DeleteSecret",
                "secretsmanager:CreateSecret"
              ],
              "Resource": "arn:aws:secretsmanager:*:${ACCOUNT_ID}:secret:Entitle/*"
          },
          {
              "Sid": "VisualEditor1",
              "Effect": "Allow",
              "Action": "secretsmanager:ListSecrets",
              "Resource" : "*"
          }
      ]
  }
  ENDOF

  aws iam create-policy --policy-name entitle-agent-policy --policy-document file://entitle-agent-policy.json --tags Key=CreatedBy,Value=Entitle
  ```

</details>

<details>
<summary>Create IAM role and attach policy</summary>

```shell
ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
echo $ACCOUNT_ID
OIDC_PROVIDER=$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.identity.oidc.issuer" --output text | sed -e "s/^https:\/\///")
echo $OIDC_PROVIDER

cat > trust.json <<ENDOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:aud": "sts.amazonaws.com",
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:entitle:entitle-agent-sa"
        }
      }
    }
  ]
}
ENDOF

aws iam create-role --role-name entitle-agent-role --assume-role-policy-document file://trust.json --description "Entitle.IO: Agent's Role" --tags Key=CreatedBy,Value=Entitle
aws iam attach-role-policy --role-name entitle-agent-role --policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/entitle-agent-policy
```
</details>



#### [Chart Installation](https://helm.sh/docs/helm/helm_upgrade/)
Eventually, you can install our Helm chart:
1. Add _application token_ to your Kubernetes secrets:
    ```shell
    echo -n '{"token":"<YOUR_APP_TOKEN>"}' > entitle-agent-secret                 # This file name is mandatory
    kubectl create secret generic entitle-agent-secret --from-file=./entitle-agent-secret --namespace entitle
    ```

- Replace `serviceAccount.iamrole` with `secretsmanager_role_arn` from the Terraform's output if you installed our IaC
- Replace `<DATADOG_CUSTOMER_ID>` in `datadog.tags` to your company name

```shell
helm upgrade --install entitle-agent-chart ./ \
    --set dockerConfigJson="<BASE64_ENCODED_DOCKER_CONFIG_JSON>" \
    --set datadog.datadog.apiKey="<DATADOG_API_KEY>" \
    --set datadog.clusterAgent.metricsProvider.enabled=true \
    --set serviceAccount.iamrole="arn:aws:iam::<ACCOUNT_ID>:role/entitle--agent-role" \
    --set entitleAgent.env.KMS_TYPE="aws_secret_manager" \
    -n entitle --create-namespace
```
<br /><br />
You are ready to go!

## GCP Installation
#### A. Workload Identity

**Notice:** If you installed our IaC then you may now skip to the [chart installation part](#chart-installation).

Follow the following GCP (GKE) guides:
- [Google Kubernetes Engine (GKE) > Documentation > Guides > About Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity)
- [Google Kubernetes Engine (GKE) > Documentation > Guides > Use Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)

In the step "**Configure applications to use Workload Identity**", use the following roles:
- `roles/secretmanager.admin`
- `roles/iam.securityAdmin`

#### B. Update `kubeconfig`

* If you have installed Entitle's Terraform IaC you simple run the following command:
    ```shell
    gcloud container clusters get-credentials $(terraform output -raw kubernetes_cluster_name) --region $(terraform output -raw region)
    ```
* Otherwise, simply replace `<CLUSTER_NAME>` and `<REGION>` and run the following command:
    ```shell
    gcloud container clusters get-credentials <CLUSTER_NAME> --region <REGION>
    ```

#### C. [Chart Installation](https://helm.sh/docs/helm/helm_upgrade/)

- Replace `<DATADOG_CUSTOMER_ID>` in `datadog.tags` to your company name

```shell
helm upgrade --install entitle-agent-chart ./ \
    --set dockerConfigJson="<BASE64_ENCODED_DOCKER_CONFIG_JSON>" \
    --set datadog.datadog.apiKey="<DATADOG_API_KEY>" \
    --set datadog.clusterAgent.metricsProvider.enabled=true \
    --set entitleAgent.env.KMS_TYPE=gcp_secret_manager \
    --set entitleAgent.serviceAccountName=entitle-agent \
    -n entitle --create-namespace
```
