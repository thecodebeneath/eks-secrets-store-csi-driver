# Objective

Experiment to use the AWS Secrets Store CSI Driver Provider to fetch external secrets as part of Kustomization postBuild. I want to use the syncSecret.enabled option to create the Secret objects without having to mount a volume to a pod.

Ref: https://github.com/aws/secrets-store-csi-driver-provider-aws

## Provison EKS Cluster
From the AWS console, use the Create Cluster option: *"Quick configuration (with EKS Auto Mode) - new"*

## Install Secrets Store Driver and Provider
```
export REGION=us-east-2
export CLUSTERNAME=scrumptious-jazz-goose

aws eks update-kubeconfig --region "$REGION" --name "$CLUSTERNAME"

helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

helm install -n kube-system csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --set syncSecret.enabled=true

helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws

helm install -n kube-system secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws
```

## Create the ServiceAccount and Secrets Manager Policy
```
source ./policy-arn.sh

eksctl create iamserviceaccount --name nginx-deployment-sa --region="$REGION" --cluster "$CLUSTERNAME" --attach-policy-arn "$POLICY_ARN" --approve --override-existing-serviceaccounts
```

## Create the EKS OIDC Provider
```
eksctl utils associate-iam-oidc-provider --region="$REGION" --cluster="$CLUSTERNAME" --approve
```

## Deploy and Apply Kustomization
```
kubectl apply -f ks/gotk.yaml
```

## Confirm Kustomization Uses Secrets Manager for ENVs and PVs
```
kubectl exec -it $(kubectl get pods | awk '/nginx-deployment/{print $1}' | head -1) -- bash
```

## Teardown
```
kubectl delete ks nginx-eks-secrets-store

kubectl delete deployment nginx-deployment
kubectl delete svc nginx-deployment
kubectl delete secret existing-secret
kubectl delete secretproviderclasses nginx-deployment-aws-secrets
kubectl delete sa nginx-deployment-sa
```