# helm-aws-external-dns
Helm chart for setting up External DNS in your EKS cluster to update Route53 records in your public and private hosted zones.


## Pre-requisites

### Namespace

Create a new namespace `platform` where we will install the `aws-load-balancer-controller` service.

```bash
kubectl create namespace platform
```

### IAM

We will be using IRSA (IAM Roles for Service Accounts) to give the required permissions to the AWS Load Balancer Controller pod to provision load balancers.

`Note: You need to create an OIDC provider for your cluster to make use of IRSA. Refer - https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html`

1. Create two new IAM policies named `k8s-route53-public-zone-pol` and `k8s-route53-private-zone-pol` respectively with below policy document.

Replace `hosted_zone_id` with the zone id of your public and private hosted zones in AWS.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowRoute531",
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets",
                "route53:ListResourceRecordSets"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/${hosted_zone_id}"
            ]
        },
        {
            "Sid": "AllowRoute532",
            "Effect": "Allow",
            "Action": [
                "route53:GetChange"
            ],
            "Resource": [
                "arn:aws:route53:::change/*"
            ]
        },
        {
            "Sid": "AllowRoute533",
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

2. Create a new IAM role `aws-load-balancer-controller-rol` and attach the IAM policy `aws-load-balancer-controller-pol`

3. Update the trust relationship of the IAM role `aws-load-balancer-controller-rol` as below replacing the `account_id`, `eks_cluster_id` and `region` with the appropriate values.

This trust relationship allows pods with serviceaccount `aws-load-balancer-controller` in `platform` namespace to assume the role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account_id>:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<eks_cluster_id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<region>.amazonaws.com/id/<eks_cluster_id>:sub": "system:serviceaccount:platform:aws-load-balancer-controller"
        }
      }
    }
  ]
}
```

### Service Account

Create a new service account in the `platform` namespace and associate it with the IAM role which we had created earlier.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: platform
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/aws-load-balancer-controller-rol
EOF
```

We specify the service account to be used by the pods in the file `stages/shared-values.yaml`
