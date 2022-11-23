Entitle-agent
===========

An Entitle agent Helm chart for Kubernetes
## Pre-Install
```shell
helm dependency update charts/entitle-agent
helm repo add entitle https://anycred.github.io/entitle-charts/
```

### GCP installation
#### A. Declare Variables
Define your cluster and namespace names:
```shell
export CLUSTER_NAME=<your-cluster-name>
export NAMESPACE=entitle
```

#### B. Workload Identity
**Notice:** If you installed our IaC then you may now skip to the [chart installation part](#gcp-chart-installation).

Follow the following GCP (GKE) guides:
- [Google Kubernetes Engine (GKE) > Documentation > Guides > About Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity)
- [Google Kubernetes Engine (GKE) > Documentation > Guides > Use Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)

In the step "**Configure applications to use Workload Identity**", use the following roles for the gcp service account:
- `roles/secretmanager.admin`
- `roles/iam.securityAdmin`
- `roles/container.developer`
- `roles/iam.workloadIdentityUser`

#### C. Update `kubeconfig`

* If you have installed Entitle's Terraform IaC you simple run the following command:
    ```shell
    gcloud container clusters get-credentials $(terraform output -raw kubernetes_cluster_name) --region $(terraform output -raw region)
    ```
* Otherwise, simply replace `<CLUSTER_NAME>` and `<REGION>` and run the following command:
    ```shell
    gcloud container clusters get-credentials <CLUSTER_NAME> --region <REGION>
    ```

#### D. [GCP Chart Installation](https://helm.sh/docs/helm/helm_upgrade/)
- `imageCredentials` and `agent.kafka.token` are given to you by Entitle
- Replace `<YOUR_ORG_NAME>` in `datadog.tags` to your company name
```shell
helm upgrade --install entitle-agent entitle/entitle-agent \
  --set imageCredentials="<IMAGE_CREDENTIALS_FROM_ENTITLE>" \
  --set datadog.datadog.apiKey="<DATADOG_API_KEY>" \
  --set datadog.datadog.tags={company:<YOUR_ORG_NAME>} \
  --set platform.gke.serviceAccount="<ENTITLE_AGENT_GKE_SERVICE_ACCOUNT_NAME>" \
  --set platform.gke.projectId="<GCP_PROJECT_ID>" \
  --set agent.kafka.token="<TOKEN_FROM_ENTITLE>" \
  -n "${NAMESPACE}" --create-namespace
```
## AWS installation

### First things first:
#### A. Declare Variables

1. Define bash variable for `CLUSTER_NAME`:
   `CLUSTER_NAME=<your-cluster-name>`
1. Define your cluser's name:
   ```shell
    export CLUSTER_NAME=<your-cluster-name>
   ```

2. Update kubeconfig:
   `aws eks update-kubeconfig --name $CLUSTER_NAME --region us-east-2 # Or any other region`
   ```shell
    aws eks update-kubeconfig --name $CLUSTER_NAME --region us-east-2   # (or any other region)
   ```

**Notice:** If you installed our IAC then you may now skip to the [chart installation part](#chart-installation)
3. **Notice:** If you installed our IaC then you may skip to the [chart installation part](#chart-installation).

### [Create OIDC Provider](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)
#### B. [Create OIDC Provider](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)

You can check if you already have the identity provider for your cluster using one of the following:
You can check if you already have the Identity Provider for your cluster using one of the following:

- Run this command:
  `aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text`
- Or [here](https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/identity_providers).
- Run the following command:
  ```shell
    aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text
  ```
- Alternatively, refer to [IAM Identity Providers](https://console.aws.amazon.com/iamv2/home#/identity_providers) page in AWS Console.

If you don't have an OIDC provider, please create new one:
`eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve`
If you don't have an OIDC provider, create new one:
```shell
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve
```

### [Create IAM Policy and Role](https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html)
#### C. [Create IAM Policy and Role](https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html)

<details>
  <summary>Create policy</summary>

  ```shell
  ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
  echo $ACCOUNT_ID

  cat > entitle-entitle-agent-chart-policy.json <<ENDOF
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

  aws iam create-policy --policy-name entitle-entitle-agent-policy --policy-document file://entitle-entitle-agent-policy.json
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

aws iam create-role --role-name entitle-entitle-agent-chart-role --assume-role-policy-document file://trust.json --description "entitle entitle-agent access aws"
aws iam attach-role-policy --role-name entitle-entitle-agent-chart-role --policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/entitle-entitle-agent-chart-policy
```

</details>


**Eventually you can helm install our chart:**

### [Chart Installation](https://helm.sh/docs/helm/helm_upgrade/)
Eventually, you can install our Helm chart:
- `imageCredentials` and `agent.kafka.token` are given to you by Entitle
- Replace `platform.aws.iamRole` with Entitle's AWS IAM Role you've created
- Replace `<YOUR_ORG_NAME>` in `datadog.tags` to your company name

```shell
helm upgrade --install entitle-agent entitle/entitle-agent \
    --set imageCredentials="<IMAGE_CREDENTIALS_FROM_ENTITLE>" \
    --set datadog.datadog.apiKey="<DATADOG_API_KEY>" \
    --set datadog.datadog.tags={company:<YOUR_ORG_NAME>} \
    --set platform.aws.iamRole="arn:aws:iam::<ACCOUNT_ID>:role/entitle-agent-role" \
    --set agent.kafka.token="<TOKEN_FROM_ENTITLE>" \
    -n entitle --create-namespace
```
<br /><br />
You are ready to go!

## Configuration

The following table lists the configurable parameters of the Entitle-agent chart and their default values.

| Parameter                         | Description                                                                                                      | Default        | Required input by user          |
|-----------------------------------|------------------------------------------------------------------------------------------------------------------| -------------- |---------------------------------|
| `imageCredentials`                | Credentials you've received upon agent installation (Contact us for more info)                                   | `null` | `true`                          |
| `platform.mode`                   |                                                                                                                  | `"gcp"` | `true`                          |
| `platform.aws.iamRole`            | IAM role for agent's service account annotations                                                                 | `null` | `true` if `platform.mode="aws"` |
| `platform.gke.serviceAccount`     | GKE service account for agent's service account annotations                                                      | `null` | `true` if `mode="platform.gcp"` |
| `platform.gke.projectId`          | GCP project ID for agent's service account annotations                                                           | `null` | `true` if `mode="platform.gcp"` |
| `podAnnotations`                  | https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/                                   | `{}` | `false`                         |
| `nodeSelector`                    | https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector                            | `{}` | `false`                         |
| `global.environment`              | Used for metadata of deployment                                                                                  | `"onprem"` | `false`                         |
| `agent.image.repository`          | Docker image repository                                                                                          | `"ghcr.io/anycred/entitle-agent"` | `false`                         |
| `agent.image.tag`                 | Tag for docker image of agent                                                                                    | `"master-kafka"` | `false`                         |
| `agent.mode`                      | Take values from: [kafka, websocket]                                                                             | `"kafka"` | `false`                         |
| `agent.replicas`                  | Number of pods to run                                                                                            | `1` | `false`                         |
| `agent.resources.requests.cpu`    | CPU request for agent pod                                                                                        | `"500m"` | `false`                         |
| `agent.resources.requests.memory` | Memory request for agent pod                                                                                     | `"1Gi"` | `false`                         |
| `agent.resources.limits.cpu`      | CPU limit for agent pod                                                                                          | `"1000m"` | `false`                         |
| `agent.resources.limits.memory`   | Memory limit for agent pod                                                                                       | `"3Gi"` | `false`                         |
| `agent.websocket.token`           | **Deprecated** [backward compatibility] Token you've received upon agent installation (Contact us for more info) | `null` | `false`                         |
| `agent.kafka.token`               | Credentials you've received upon agent installation (Contact us for more info)                                   | `null` | `true`                          |
| `datadog.providers.gke.autopilot` | Whether to enable autopilot or not                                                                               | `false` | `false`                         |
| `datadog.datadog.apiKey`          | Datadog API key                                                                                                  | `null` | `true`                          |
| `datadog.datadog.tags`            | Datadog Tag - Put your company name (https://docs.datadoghq.com/tagging/)                                        | `null` | `true`                          |
