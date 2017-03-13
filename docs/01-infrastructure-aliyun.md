# Cloud Infrastructure Provisioning - Aliyun

This lab will walk you through provisioning the compute instances required for running a H/A Kubernetes cluster. A total of 6 virtual machines will be created.

After completing this guide you should have the following compute instances:

![EC2 Console](ec2-instances.png)

> All machines will be provisioned with fixed private IP addresses to simplify the bootstrap process.

To make our Kubernetes control plane remotely accessible, a public IP address will be provisioned and assigned to a Load Balancer that will sit in front of the 3 Kubernetes controllers.

## Settings

You might want to tweak those configs first if necessary:

```
ZONE_ID=cn-beijing-c
IMAGE_ID=ubuntu_16_0402_64_40G_base_20170222.vhd
INSTANCE_TYPE=ecs.n1.small
INTERNET_BANDWIDTH_IN=1
INTERNET_BANDWIDTH_OUT=1
INTERNET_CHARGE_TYPE=PayByTraffic
INSTANCE_CHARGE_TYPE=PostPaid
ROOTPASS_WORD=RootPassworD123
```

## Networking

### VPC

```
VPC_ID=$(aliyuncli ecs CreateVpc \
  --VpcName kubernetes \
  --CidrBlock 10.240.0.0/16 | \
  jq -r '.VpcId')
```

### VSwitch

Create a VSwitch for the Kubernetes cluster. Note that you might need to tweak the availability zone depending on the region you use.

```
VSWITCH_ID=$(aliyuncli ecs CreateVSwitch \
  --VSwitchName kubernetes \
  --ZoneId ${ZONE_ID} \
  --VpcId ${VPC_ID} \
  --CidrBlock 10.240.0.0/24 | \
  jq -r '.VSwitchId')
```

### Firewall Rules

```
SECURITY_GROUP_ID=$(aliyuncli ecs CreateSecurityGroup \
  --SecurityGroupName kubernetes \
  --Description "Kubernetes security group" \
  --VpcId ${VPC_ID} | \
  jq -r '.SecurityGroupId')
```

```
aliyuncli ecs AuthorizeSecurityGroup \
  --SecurityGroupId ${SECURITY_GROUP_ID} \
  --IpProtocol icmp \
  --PortRange -1/-1 \
  --SourceCidrIp 0.0.0.0/0
```

```
aliyuncli ecs AuthorizeSecurityGroup \
  --SecurityGroupId ${SECURITY_GROUP_ID} \
  --IpProtocol all \
  --PortRange -1/-1 \
  --SourceCidrIp 10.240.0.0/24
```

```
aliyuncli ecs AuthorizeSecurityGroup \
  --SecurityGroupId ${SECURITY_GROUP_ID} \
  --IpProtocol tcp \
  --PortRange 3389/3389 \
  --SourceCidrIp 0.0.0.0/0
```

```
aliyuncli ecs AuthorizeSecurityGroup \
  --SecurityGroupId ${SECURITY_GROUP_ID} \
  --IpProtocol tcp \
  --PortRange 22/22 \
  --SourceCidrIp 0.0.0.0/0
```

```
aliyuncli ecs AuthorizeSecurityGroup \
  --SecurityGroupId ${SECURITY_GROUP_ID} \
  --IpProtocol tcp \
  --PortRange 6443/6443 \
  --SourceCidrIp 0.0.0.0/0
```

### Kubernetes Public Address

(NOTE: this is temporarily pending since we need to find a way to work around ICP registration)

## Provision Virtual Machines

All the VMs in this lab will be provisioned using Ubuntu 16.04 mainly because it runs a newish Linux Kernel that has good support for Docker.

### Virtual Machines

#### Kubernetes Controllers

```
CONTROLLER_0_INSTANCE_ID=$(aliyuncli ecs CreateInstance \
  --SecurityGroupId ${SECURITY_GROUP_ID} \
  --InstanceName kubernetes-controller-0 \
  --HostName controller0 \
  --VSwitchId ${VSWITCH_ID} \
  --PrivateIpAddress 10.240.0.10 \
  --ZoneId ${ZONE_ID} \
  --ImageId ${IMAGE_ID} \
  --InstanceType ${INSTANCE_TYPE} \
  --Password ${ROOTPASS_WORD} \
  --InternetMaxBandwidthIn ${INTERNET_BANDWIDTH_IN} \
  --InternetMaxBandwidthOut ${INTERNET_BANDWIDTH_OUT} \
  --InternetChargeType ${INTERNET_CHARGE_TYPE} \
  --InstanceChargeType ${INSTANCE_CHARGE_TYPE} | \
  jq -r '.InstanceId')
```

```
CONTROLLER_0_PUBLIC_IP=$(aliyuncli ecs AllocatePublicIpAddress \
  --InstanceId ${CONTROLLER_0_INSTANCE_ID} | \
  jq -r '.IpAddress')
```

```
aliyuncli ecs StartInstance \
  --InstanceId ${CONTROLLER_0_INSTANCE_ID}
```

```
CONTROLLER_1_INSTANCE_ID=$(aliyuncli ecs CreateInstance \
  --SecurityGroupId ${SECURITY_GROUP_ID} \
  --InstanceName kubernetes-controller-1 \
  --HostName controller1 \
  --VSwitchId ${VSWITCH_ID} \
  --PrivateIpAddress 10.240.0.11 \
  --ZoneId ${ZONE_ID} \
  --ImageId ${IMAGE_ID} \
  --InstanceType ${INSTANCE_TYPE} \
  --Password ${ROOTPASS_WORD} \
  --InternetMaxBandwidthIn ${INTERNET_BANDWIDTH_IN} \
  --InternetMaxBandwidthOut ${INTERNET_BANDWIDTH_OUT} \
  --InternetChargeType ${INTERNET_CHARGE_TYPE} \
  --InstanceChargeType ${INSTANCE_CHARGE_TYPE} | \
  jq -r '.InstanceId')
```

```
CONTROLLER_1_PUBLIC_IP=$(aliyuncli ecs AllocatePublicIpAddress \
  --InstanceId ${CONTROLLER_1_INSTANCE_ID} | \
  jq -r '.IpAddress')
```

```
aliyuncli ecs StartInstance \
  --InstanceId ${CONTROLLER_1_INSTANCE_ID}
```

```
CONTROLLER_2_INSTANCE_ID=$(aliyuncli ecs CreateInstance \
  --SecurityGroupId ${SECURITY_GROUP_ID} \
  --InstanceName kubernetes-controller-2 \
  --HostName controller2 \
  --VSwitchId ${VSWITCH_ID} \
  --PrivateIpAddress 10.240.0.12 \
  --ZoneId ${ZONE_ID} \
  --ImageId ${IMAGE_ID} \
  --InstanceType ${INSTANCE_TYPE} \
  --Password ${ROOTPASS_WORD} \
  --InternetMaxBandwidthIn ${INTERNET_BANDWIDTH_IN} \
  --InternetMaxBandwidthOut ${INTERNET_BANDWIDTH_OUT} \
  --InternetChargeType ${INTERNET_CHARGE_TYPE} \
  --InstanceChargeType ${INSTANCE_CHARGE_TYPE} | \
  jq -r '.InstanceId')
```

```
CONTROLLER_2_PUBLIC_IP=$(aliyuncli ecs AllocatePublicIpAddress \
  --InstanceId ${CONTROLLER_2_INSTANCE_ID} | \
  jq -r '.IpAddress')
```

```
aliyuncli ecs StartInstance \
  --InstanceId ${CONTROLLER_2_INSTANCE_ID}
```

#### Kubernetes Workers

```
WORKER_0_INSTANCE_ID=$(aliyuncli ecs CreateInstance \
  --SecurityGroupId ${SECURITY_GROUP_ID} \
  --InstanceName kubernetes-worker-0 \
  --HostName worker0 \
  --VSwitchId ${VSWITCH_ID} \
  --PrivateIpAddress 10.240.0.20 \
  --ZoneId ${ZONE_ID} \
  --ImageId ${IMAGE_ID} \
  --InstanceType ${INSTANCE_TYPE} \
  --Password ${ROOTPASS_WORD} \
  --InternetMaxBandwidthIn ${INTERNET_BANDWIDTH_IN} \
  --InternetMaxBandwidthOut ${INTERNET_BANDWIDTH_OUT} \
  --InternetChargeType ${INTERNET_CHARGE_TYPE} \
  --InstanceChargeType ${INSTANCE_CHARGE_TYPE} | \
  jq -r '.InstanceId')
```

```
WORKER_0_PUBLIC_IP=$(aliyuncli ecs AllocatePublicIpAddress \
  --InstanceId ${WORKER_0_INSTANCE_ID} | \
  jq -r '.IpAddress')
```

```
aliyuncli ecs StartInstance \
  --InstanceId ${WORKER_0_INSTANCE_ID}
```

```
WORKER_1_INSTANCE_ID=$(aliyuncli ecs CreateInstance \
  --SecurityGroupId ${SECURITY_GROUP_ID} \
  --InstanceName kubernetes-worker-1 \
  --HostName worker1 \
  --VSwitchId ${VSWITCH_ID} \
  --PrivateIpAddress 10.240.0.21 \
  --ZoneId ${ZONE_ID} \
  --ImageId ${IMAGE_ID} \
  --InstanceType ${INSTANCE_TYPE} \
  --Password ${ROOTPASS_WORD} \
  --InternetMaxBandwidthIn ${INTERNET_BANDWIDTH_IN} \
  --InternetMaxBandwidthOut ${INTERNET_BANDWIDTH_OUT} \
  --InternetChargeType ${INTERNET_CHARGE_TYPE} \
  --InstanceChargeType ${INSTANCE_CHARGE_TYPE} | \
  jq -r '.InstanceId')
```

```
WORKER_1_PUBLIC_IP=$(aliyuncli ecs AllocatePublicIpAddress \
  --InstanceId ${WORKER_1_INSTANCE_ID} | \
  jq -r '.IpAddress')
```

```
aliyuncli ecs StartInstance \
  --InstanceId ${WORKER_1_INSTANCE_ID}
```

```
WORKER_2_INSTANCE_ID=$(aliyuncli ecs CreateInstance \
  --SecurityGroupId ${SECURITY_GROUP_ID} \
  --InstanceName kubernetes-worker-2 \
  --HostName worker2 \
  --VSwitchId ${VSWITCH_ID} \
  --PrivateIpAddress 10.240.0.22 \
  --ZoneId ${ZONE_ID} \
  --ImageId ${IMAGE_ID} \
  --InstanceType ${INSTANCE_TYPE} \
  --Password ${ROOTPASS_WORD} \
  --InternetMaxBandwidthIn ${INTERNET_BANDWIDTH_IN} \
  --InternetMaxBandwidthOut ${INTERNET_BANDWIDTH_OUT} \
  --InternetChargeType ${INTERNET_CHARGE_TYPE} \
  --InstanceChargeType ${INSTANCE_CHARGE_TYPE} | \
  jq -r '.InstanceId')
```

```
WORKER_2_PUBLIC_IP=$(aliyuncli ecs AllocatePublicIpAddress \
  --InstanceId ${WORKER_2_INSTANCE_ID} | \
  jq -r '.IpAddress')
```

```
aliyuncli ecs StartInstance \
  --InstanceId ${WORKER_2_INSTANCE_ID}
```

## Verify

```
aliyuncli ecs DescribeInstances \
  --ZoneId=${ZONE_ID} \
  --PageSize=20 | \
  jq -j '.Instances.Instance[] | .InstanceId, "  ", .ZoneId, "  ", .VpcAttributes.PrivateIpAddress.IpAddress[0], "  ", .PublicIpAddress.IpAddress[0], "\n"'
```
