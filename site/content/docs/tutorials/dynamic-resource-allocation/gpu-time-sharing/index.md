---
linkTitle: "Time sharing of GPUs with DRA"
title: "Time sharing of GPUs with DRA"
description: "This tutorial guides you through how to do device sharing of NVIDIA GPUs with Dynamic Resource Allocation on Google Kubernetes Engine (GKE) with the time sharing mode"
weight: 30
owner:
  - name: "Morten Torkildsen"
    link: "https://github.com/mortent"
type: docs
tags:
 - GPU
 - Device Sharing
 - DRA
 - Time Sharing
draft: true
cloudShell: 
    enabled: true
    folder: site/content/docs/tutorials/dynamic-resource-allocation/gpu-time-sharing
    editorFile: index.md
---

## **Background**

This tutorial guides you through how to do device sharing of NVIDIA GPUs with Dynamic Resource Allocation on Google Kubernetes Engine (GKE). We will cover three different device sharing modes: time sharing, Multi-Process Service (MPS), and Multi-Instance GPU (MIG).

Let’s get started and explore how to share GPU resources effectively using DRA.

## **Prepare the Environment**

To set up your environment with Cloud Shell, follow these steps:

1. In the Google Cloud console, launch a Cloud Shell session by clicking Cloud Shell activation icon Activate Cloud Shell in the Google Cloud console. This launches a session in the bottom pane of Google Cloud console.  
2. Set the default environment variables:

```bash
export PROJECT_ID=$(gcloud config get project)
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)")
export CLUSTER_NAME=gpu-vllm-timeslicing
export LOCATION=us-central1 # Choose a region that has NVIDIA L4 GPUs available
export HF_TOKEN=HUGGING_FACE_TOKEN
export CLUSTER_VERSION=<GKE version 1.36 or later>
export NAMESPACE=default
```

## **Create and configure Google Cloud Resources**

### Create a GKE Cluster

```bash
gcloud container clusters create $CLUSTER_NAME \
--location=$LOCATION \
--cluster-version=$CLUSTER_VERSION \
--project=$PROJECT_ID \
--num-nodes=1 \
--labels=created-by=ai-on-gke,guide=gpu-device-sharing
```

### Create v6e TPU, L4 Preemptible, and L4 on demand node pools

All node pools will have autoscaling enabled in order to demonstrate that [custom compute class (CCC)](https://cloud.google.com/kubernetes-engine/docs/concepts/about-custom-compute-classes) is able to autoscale any type of node pool. We will also add a label and taint with the CCC name so that it can be used in the priority list.

Create a TPU v6e-1 [Spot](https://cloud.google.com/kubernetes-engine/docs/concepts/spot-vms) node pool:

```bash
gcloud container node-pools create a100-pool \
    --cluster=${CLUSTER_NAME} \
    --location=${LOCATION} \
    --node-locations=us-central1-c \
    --machine-type="a2-highgpu-1g" \
    --accelerator="type=nvidia-tesla-a100,count=1,gpu-driver-version=disabled" \
    --num-nodes=1 \
    --node-labels=gke-no-default-nvidia-gpu-device-plugin=true,nvidia.com/gpu.present=true,cloud.google.com/gke-nvidia-gpu-dra-driver=true \
    --spot
```

## **Configure Kubectl to communicate with your cluster**
To configure kubectl to communicate with your cluster, run the following command:

```bash
  gcloud container clusters get-credentials ${CLUSTER_NAME} --region=${REGION}
```

## **Create Kubernetes Secret for Hugging Face credentials**
To create a Kubernetes Secret that contains the Hugging Face token, run the following command:

```bash
kubectl create secret generic hf-secret --from-literal=hf_api_token=${HF_TOKEN}
```

## **Install the NVIDIA GPU driver**

```bash
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/cos/daemonset-preloaded.yaml
```

## **Install the NVIDIA GPU DRA driver**

```bash
helm install nvidia-dra-driver-gpu nvidia/nvidia-dra-driver-gpu \
    --version="25.12.0" --create-namespace --namespace=nvidia-dra-driver-gpu \
    --set nvidiaDriverRoot="/home/kubernetes/bin/nvidia/" \
    --set gpuResourcesEnabledOverride=true \
    --set resources.computeDomains.enabled=false \
    --set featureGates.TimeSlicingSettings=true \
    --set kubeletPlugin.priorityClassName="" \
    --set 'kubeletPlugin.tolerations[0].operator=Exists'
```

## **Create the DRA ResourceClaim**

Inspect the following `claim.yaml`, where we request a GPU with time slicing enabled.

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-claim
spec:
  devices:
    requests:
    - name: gpu
      exactly:
        deviceClassName: gpu.nvidia.com
    config:
      - requests: ["gpu"]
        opaque:
          driver: gpu.nvidia.com
          parameters:
            apiVersion: resource.nvidia.com/v1beta1
            kind: GpuConfig
            sharing:
              strategy: TimeSlicing
              timeSlicingConfig:
                interval: Long
```

Apply the manifest

```bash
kubectl apply -f claim.yaml
```

## **Deploy the vllm workload*

Inspect the following `vllm.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-gpu
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vllm-gpu
  template:
    metadata:
      labels:
        app: vllm-gpu
    spec:
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
      resourceClaims:
      - name: gpu
        resourceClaimName: gpu-claim
      containers:
      - name: vllm-gpu
        image: vllm/vllm-openai:latest
        command: ["python3", "-m", "vllm.entrypoints.openai.api_server"]
        args:
        - --host=0.0.0.0
        - --port=8000
        - --model=google/gemma-3-1b-it
        - --gpu-memory-utilization=0.42
        env: 
        - name: HUGGING_FACE_HUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-secret
              key: hf_api_token
        ports:
        - containerPort: 8000
        resources:
          claims:
          - name: gpu
        readinessProbe:
          tcpSocket:
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 10
        volumeMounts:
        - name: dshm
          mountPath: /dev/shm
      volumes:
      - name: dshm
        emptyDir:
          medium: Memory

---

apiVersion: v1
kind: Service
metadata:
  name: vllm-service
spec:
  selector:
    app: vllm-gpu
  type: LoadBalancer	
  ports:
    - name: http
      protocol: TCP
      port: 8000  
      targetPort: 8000
```

Apply the manifest

```bash
kubectl apply -f vllm.yaml
```

## **Clean up** 

### Delete the deployed resources:

To avoid incurring charges to your Google Cloud account for the resources that you created in this guide, run the following commands:

Stop the bash script that simulates load:

```bash
ps -ef | grep load.sh | awk '{print $2}' | xargs -n1 kill -9
```

Delete the cluster:

```bash
gcloud container clusters delete ${CLUSTER_NAME} \
  --location=${ZONE}



