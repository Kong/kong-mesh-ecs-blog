### ECS Deployment Requirements


* Cloudformation scripts to create vpc and ecs cluster

In this step we will create following resources with cloudformation:
   * VPC
   * NLB
   * ECS cluster 

Command to run:
```
aws cloudformation deploy \
    --capabilities CAPABILITY_IAM \
    --stack-name ecs-demo-vpc \
    --template-file deploy/ecs/1-vpc.yaml
```

Create the CP certs:
```
CP_ADDR=$(aws cloudformation describe-stacks --stack-name ecs-demo-vpc \
  | jq -r '.Stacks[0].Outputs[] | select(.OutputKey == "ExternalCPAddress") | .OutputValue')

kumactl generate tls-certificate --type=server --hostname ${CP_ADDR} --hostname controlplane.kongmesh

TLS_KEY=$(
  aws secretsmanager create-secret \
  --name ecs-demo/CPTLSKey \
  --description "Secret containing TLS private key for serving control plane traffic" \
  --secret-string file://key.pem \
  | jq -r .ARN)

TLS_CERT=$(
  aws secretsmanager create-secret \
  --name ecs-demo/CPTLSCert \
  --description "Secret containing TLS certificate for serving control plane traffic" \
  --secret-string file://cert.pem \
  | jq -r .ARN)
```

* Cloudformation script for Zone CP deployment in ECS
Run this command: 
```
aws cloudformation deploy \
  --capabilities CAPABILITY_IAM \
  --stack-name ecs-demo-kong-mesh-cp \
  --parameter-overrides VPCStackName=ecs-demo-vpc \
    ServerKeySecret=${TLS_KEY} \
    ServerCertSecret=${TLS_CERT} \
    KonnectCPID=<CP_ID> \
    KonnectSPAT=<SPAT> \
  --template-file deploy/ecs/2-controlplane.yaml
```

* Deploy the Zone Ingress to ECS
```
aws cloudformation deploy \
  --capabilities CAPABILITY_IAM \
  --stack-name ecs-demo-ingress \
  --parameter-overrides VPCStackName=ecs-demo-vpc CPStackName=ecs-demo-kong-mesh-cp \
  --template-file deploy/ecs/3-ingress.yaml
```


* Now you can run deployment script as following:
```
aws cloudformation deploy \
    --capabilities CAPABILITY_IAM \
    --stack-name ecs-demo-redis \
    --parameter-overrides VPCStackName=ecs-demo-vpc CPStackName=ecs-demo-kong-mesh-cp \
    --template-file deploy/ecs/4-redis.yaml
```
* Same for demo-app:
```
aws cloudformation deploy \
    --capabilities CAPABILITY_IAM \
    --stack-name ecs-demo-demo-app \
    --parameter-overrides VPCStackName=ecs-demo-vpc CPStackName=ecs-demo-kong-mesh-cp \
    --template-file deploy/ecs/5-demo-app.yaml
```

* the demo-app is configured to be accessible via the NLB created in VPC step over port 80 so you can use following to address it:


1- Get NLB address: 

```
aws cloudformation describe-stacks --stack-name ecs-demo-vpc \
    | jq -r '.Stacks[0].Outputs[] | select(.OutputKey == "ExternalCPAddress") | .OutputValue'
```

By this point you should see followings
![KonnectOverview](images/konnect-overview.png)
![KonnectZones](images/konnect-zones.png)
![KonnectMeshServices](images/konnect-mesh-services.png)

2- In browser use the address with port 80, it should be like ecs-d-LoadB-1VXB673P2MABU-06bda31926631307.elb.ap-southeast-2.amazonaws.com:80


### Cleanup Steps

Run following commands in order:

```
aws cloudformation delete-stack --stack-name ecs-demo-demo-app
aws cloudformation delete-stack --stack-name ecs-demo-demo-redis
aws cloudformation delete-stack --stack-name ecs-demo-kong-mesh-cp
aws cloudformation delete-stack --stack-name ecs-demo-vpc
```