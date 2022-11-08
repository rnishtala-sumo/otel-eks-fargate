# otel-eks-fargate

This repo is a PoC to collect data from EKS clusters on Fargate into Sumo Logic. Sumo Logic leverages [CNCF](https://www.cncf.io) supported technology including [Opentelemetry](https://https://opentelemetry.io/) to collect logs, metrics and traces from Kubernetes clusters. The following diagram provides an overview of the collection process.

![overview](/images/overview.png)

## Deviations with EKS Fargate
- Volumes need to created manually and exposed via EFS access points
- Daemonsets are not supported
- containers cannot be run in privileged mode  
  -  https://github.com/aws/containers-roadmap/issues/1000
- It isn't possible to mount volumes using hostpaths, one could instead use persistent volumes with EFS
  - https://github.com/pachyderm/pachyderm/issues/4522

## Pre-requisites
#### Deploy EKS on Fargate
```
 eksctl create cluster --name $CLUSTER --region $AWS_REGION --fargate
```
#### Create an EFS filesystem
```
aws efs create-file-system --creation-token $NEW_TOKEN \
  --encrypted \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --tags Key=Name,Value=OtelVolume \
  --region $AWS_REGION \
  --output text \
  --query "FileSystemId"
```
#### Create EFS access points
```
aws efs create-access-point \
  --file-system-id $EFS_FS_ID \
  --posix-user Uid=1000,Gid=1000 \
  --root-directory "Path=/path,CreationInfo={OwnerUid=1000,OwnerGid=1000,Permissions=777}" \
  --region $AWS_REGION \
  --query 'AccessPointId' \
  --output text)
```
For the following paths
- /conf
- /etc/otel
- /var/lib/storage/otc
- /var/log/pods
- /var/log/journal
- /var/lib/docker/containers
- /etc/otelcol
#### Create a Security group that allows inbound NFS traffic
```
aws ec2 create-security-group \
  --description Otel-on-Fargate \
  --group-name Otel-on-Fargate \
  --vpc-id $VPC_ID \
  --region $AWS_REGION \
  --query 'GroupId' --output text
```
```
aws ec2 authorize-security-group-ingress \
  --group-id $EFS_SG_ID \
  --protocol tcp \
  --port 2049 \
  --cidr $CIDR_BLOCK
```
#### Create EFS Mount targets for the volume in all subnets used in the Fargate profile
```
aws efs create-mount-target \
  --file-system-id $EFS_FS_ID \
  --subnet-id $subnet \
  --security-group $EFS_SG_ID \
  --region $AWS_REGION
```


## Installation

#### Create the Sumologic secret
```
kubectl create secret generic sumologic --from-literal=accessId=$ACCESS_ID --from-literal=accessKey=$ACCESS_KEY --from-literal=$ENDPOINT_TRACES=$ENDPOINT_URL
--from-literal=$ENDPOINT_METRICS=$ENDPOINT_METRICS_URL
--from-literal=$ENDPOINT_METRICS_API_SERVER=$ENDPOINT_METRICS_API_SERVER_URL
--from-literal=$ENDPOINT_CONTROL_PLANE_METRICS=$ENDPOINT_CONTROL_PLANE_METRICS_URL
--from-literal=$ENDPOINT_METRICS_KUBE_CONTROLLER=$ENDPOINT_METRICS_KUBE_CONTROLLER_URL
--from-literal=$ENDPOINT_METRICS_KUBELET=$ENDPOINT_METRICS_KUBELET_URL
--from-literal=$ENDPOINT_METRICS_NODE_EXPORTER=$ENDPOINT_METRICS_NODE_EXPORTER_URL
--from-literal=$ENDPOINT_METRICS_KUBE_SCHEDULER=$ENDPOINT_METRICS_KUBE_SCHEDULER_URL
--from-literal=$ENDPOINT_METRICS_KUBE_STATE=$ENDPOINT_METRICS_KUBE_STATE_URL
--from-literal=$ENDPOINT_LOGS=$ENDPOINT_LOGS_URL
```
#### Create Persistent volume and persistent volume claims using the EFS file system
```
kubectl apply -f deploy/helm/sumologic/pvc.yaml
```
`Note`: The volumeHandle is defined as
```volumeHandle: $EFS_FILE_SYSTEM_ID::$EFS_ACCESS_POINT```
#### Install the Helm chart
```
helm install -f my_values.yaml sumo .
```
##### Otel pods
![pods](/images/pods.png)

##### Persistent volume claims
![pvc](/images/pvc.png)