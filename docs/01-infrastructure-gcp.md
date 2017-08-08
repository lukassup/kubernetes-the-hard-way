# Cloud Infrastructure Provisioning - Google Cloud Platform

This lab will walk you through provisioning the compute instances required for running a H/A Kubernetes cluster. A total of 6 virtual machines will be created.

After completing this guide you should have the following compute instances:

```
gcloud compute instances list
```

````
NAME         ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP      STATUS
controller0  us-central1-f  n1-standard-1               10.240.0.10  XXX.XXX.XXX.XXX  RUNNING
controller1  us-central1-f  n1-standard-1               10.240.0.11  XXX.XXX.XXX.XXX  RUNNING
controller2  us-central1-f  n1-standard-1               10.240.0.12  XXX.XXX.XXX.XXX  RUNNING
worker0      us-central1-f  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XXX  RUNNING
worker1      us-central1-f  n1-standard-1               10.240.0.21  XXX.XXX.XXX.XXX  RUNNING
worker2      us-central1-f  n1-standard-1               10.240.0.22  XXX.XXX.XXX.XXX  RUNNING
````

> All machines will be provisioned with fixed private IP addresses to simplify the bootstrap process.

To make our Kubernetes control plane remotely accessible, a public IP address will be provisioned and assigned to a Load Balancer that will sit in front of the 3 Kubernetes controllers.

## Prerequisites

Set the compute region and zone to us-central1:

```
gcloud config set compute/region europe-west1
```

```
gcloud config set compute/zone europe-west1-b
```

## Setup Networking


Create a custom network:

```
gcloud compute networks create k8s-net --mode custom
```

Create a subnet for the Kubernetes cluster:

```
gcloud compute networks subnets create k8s \
  --network k8s-net \
  --range 10.240.0.0/24 \
  --region europe-west1
```

### Create Firewall Rules

```
gcloud compute firewall-rules create allow-internal \
  --allow tcp,udp,icmp \
  --network k8s-net \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```

```
gcloud compute firewall-rules create allow-external \
  --allow tcp:22,tcp:3389,tcp:6443,icmp \
  --network k8s-net \
  --source-ranges 0.0.0.0/0
```

```
gcloud compute firewall-rules create allow-healthz \
  --allow tcp:8080 \
  --network k8s-net \
  --source-ranges 35.195.0.0/16,104.199.100.0/22
```


```
gcloud compute firewall-rules list --filter "network=k8s-net"
```

```
NAME            NETWORK                  SRC_RANGES                     RULES                          SRC_TAGS  TARGET_TAGS
allow-external  k8s-net                  0.0.0.0/0                      tcp:22,tcp:3389,tcp:6443,icmp
allow-healthz   k8s-net                  35.195.0.0/16,104.199.100.0/22 tcp:8080
allow-internal  k8s-net                  10.240.0.0/24,10.200.0.0/16    tcp,udp,icmp
```

### Create the Kubernetes Public Address

Create a public IP address that will be used by remote clients to connect to the Kubernetes control plane:

```
gcloud compute addresses create k8s --region=europe-west1
```

```
gcloud compute addresses list k8s
```

```
NAME                     REGION        ADDRESS          STATUS
k8s-net                  europe-west1  XXX.XXX.XXX.XXX  RESERVED
```

## Provision Virtual Machines

All the VMs in this lab will be provisioned using Debian 9 mainly because it runs a newish Linux kernel with good support for Docker.

### Virtual Machines

#### Kubernetes Controllers

```
gcloud compute instances create controller0 \
 --preemptible \
 --boot-disk-size 20G \
 --can-ip-forward \
 --image-project debian-cloud \
 --image debian-9-stretch-v20170717 \
 --machine-type n1-standard-1 \
 --private-network-ip 10.240.0.10 \
 --subnet k8s
```

```
gcloud compute instances create controller1 \
 --preemptible \
 --boot-disk-size 20G \
 --can-ip-forward \
 --image-project debian-cloud \
 --image debian-9-stretch-v20170717 \
 --machine-type n1-standard-1 \
 --private-network-ip 10.240.0.11 \
 --subnet k8s
```

```
gcloud compute instances create controller2 \
 --preemptible \
 --boot-disk-size 20G \
 --can-ip-forward \
 --image-project debian-cloud \
 --image debian-9-stretch-v20170717 \
 --machine-type n1-standard-1 \
 --private-network-ip 10.240.0.12 \
 --subnet k8s
```

#### Kubernetes Workers

Include socat depedency on worker VMs to enable kubelet's portfw functionality.

```
gcloud compute instances create worker0 \
 --preemptible \
 --boot-disk-size 20G \
 --can-ip-forward \
 --image-project debian-cloud \
 --image debian-9-stretch-v20170717 \
 --machine-type n1-standard-1 \
 --private-network-ip 10.240.0.20 \
 --subnet k8s \
 --metadata startup-script='#! /bin/bash
apt-get update
apt-get install -y socat
EOF'
```

```
gcloud compute instances create worker1 \
 --preemptible \
 --boot-disk-size 20G \
 --can-ip-forward \
 --image-project debian-cloud \
 --image debian-9-stretch-v20170717 \
 --machine-type n1-standard-1 \
 --private-network-ip 10.240.0.21 \
 --subnet k8s \
 --metadata startup-script='#! /bin/bash
apt-get update
apt-get install -y socat
EOF'
```

```
gcloud compute instances create worker2 \
 --preemptible \
 --boot-disk-size 20G \
 --can-ip-forward \
 --image-project debian-cloud \
 --image debian-9-stretch-v20170717 \
 --machine-type n1-standard-1 \
 --private-network-ip 10.240.0.22 \
 --subnet k8s \
 --metadata startup-script='#! /bin/bash
apt-get update
apt-get install -y socat
EOF'
```
