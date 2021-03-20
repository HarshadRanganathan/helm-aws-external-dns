# helm-aws-external-dns
Helm chart for setting up External DNS in your EKS cluster to update public and private Route53 hosted zones.

Chart Reference - https://github.com/helm/charts/tree/master/stable/external-dns

We need to install two External DNS services, one for updating Route53 public hosted zones and the other for updating private hosted zones.

Table of Contents
=================

   * [helm-aws-external-dns](#helm-aws-external-dns)
      * [Usage](#usage)
      * [Pre-requisites](#pre-requisites)
         * [Namespace](#namespace)
         * [IAM](#iam)
         * [Config Updates](#config-updates)
      * [Install/Upgrade Chart](#install/upgrade-chart)

## Usage

To create DNS in Route53 public hosted zone, use below annotations in your ingress/service:

```
external-dns.alpha.kubernetes.io/hostname:
```


To create DNS in Route53 private hosted zone, use below annotations in your ingress/service:

```
external-dns.alpha.kubernetes.io/hostname:
external-dns/zone: private
```

## Pre-requisites

### Namespace

Create a new namespace `platform` where we will install two external DNS services.

```bash
kubectl create namespace platform
```

### IAM

We will be using IRSA (IAM Roles for Service Accounts) to give the required permissions to the ExternalDNS pods to update Route53.

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

2. Create new IAM roles `k8s-route53-public-zone-rol` and `k8s-route53-private-zone-rol`. Attach the IAM policies which we had created earlier.

3. Update the trust relationship of the IAM roles as below replacing the `account_id`, `eks_cluster_id` and `region` with the appropriate values.

For example, this trust relationship allows pods with serviceaccount `external-dns-private-zone` in `platform` namespace to assume the role.

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
          "oidc.eks.<region>.amazonaws.com/id/<eks_cluster_id>:sub": "system:serviceaccount:platform:external-dns-private-zone"
        }
      }
    }
  ]
}
```

### Config Updates

In `prod-public-zone-values.yaml` file available inside `stages/prod` folder, add values for below settings:

|||
|--|--|
|domainFilters |Public domain suffixes |
|txtOwnerId |Text Registry Identifiers  |
|eks.amazonaws.com/role-arn |ARN of the IAM role `k8s-route53-public-zone-rol`. This will be added as an annotation to the service account `external-dns-public-zone`  |


In `prod-private-zone-values.yaml` file available inside `stages/prod` folder, add values for below settings:

|||
|--|--|
|domainFilters |Private domain suffixes |
|txtOwnerId |Text Registry Identifiers  |
|eks.amazonaws.com/role-arn |ARN of the IAM role `k8s-route53-private-zone-rol`. This will be added as an annotation to the service account `external-dns-private-zone`  |

## Install/Upgrade Chart

Run below commands to install/upgrade the public and private external dns charts.

```bash
helm upgrade -i external-dns-public-zone . -n platform --values stages/prod/prod-public-zone-values.yaml

helm upgrade -i external-dns-private-zone . -n platform --values stages/prod/prod-private-zone-values.yaml
```
