---
linkTitle: "MPS sharing of GPUs with DRA"
title: "MPS sharing of GPUs with DRA"
description: "This tutorial guides you through how to do device sharing of NVIDIA GPUs with Dynamic Resource Allocation on Google Kubernetes Engine (GKE) with the MPS mode"
weight: 35
owner:
  - name: "Morten Torkildsen"
    link: "https://github.com/mortent"
type: docs
tags:
 - GPU
 - Device Sharing
 - DRA
 - MPS
draft: true
cloudShell: 
    enabled: true
    folder: site/content/docs/tutorials/dynamic-resource-allocation/gpu-mps
    editorFile: index.md
---

## **Background**

This tutorial guides you through how to do device sharing of NVIDIA GPUs with Dynamic Resource Allocation on Google Kubernetes Engine (GKE). This guide covers Multi-Process Service (MPS), with separate guides covering the other device sharing modes, Time Slicing and Multi-Instance GPU (MIG).

> [!NOTE]
> MPS through the GPU DRA driver does not currently work on GKE: https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu/issues/469

### **GPU Device Sharing Modes**

When sharing a single physical GPU among multiple containers or pods, you typically choose between three primary mechanisms. Here is a quick summary of how they work and their tradeoffs:

1.  **Time Slicing (Time Sharing)**:
    *   **How it works**: The GPU's compute resources are shared in time. The GPU context switches between different workloads.
    *   **Pros**: Simple to configure, works on almost all modern GPUs, and has no memory overhead.
    *   **Cons**: No memory isolation (one workload can consume all memory and OOM the other) and potential latency impact if one workload hogs the GPU.
    *   **Best for**: Development, testing, or workloads with low and bursty utilization where hard isolation is not required.

2.  **Multi-Process Service (MPS)**:
    *   **How it works**: Multiple processes share the GPU compute and memory simultaneously (spatial sharing).
    *   **Pros**: Higher utilization and lower overhead than time slicing. It allows concurrent execution of kernels from different processes.
    *   **Cons**: Limited isolation compared to MIG. Memory limits can be set but are not as strictly enforced at the hardware level as MIG.
    *   **Best for**: Cooperative workloads, like running multiple inference servers that trust each other and benefit from high throughput.

3.  **Multi-Instance GPU (MIG)**:
    *   **How it works**: The GPU is physically partitioned into separate "instances," each with its own dedicated compute and memory resources.
    *   **Pros**: Hard isolation, guaranteed QoS (Quality of Service), and complete memory separation.
    *   **Cons**: Rigid partition sizes and requires specific high-end GPUs (Ampere architecture or newer, e.g., A100, H100).
    *   **Best for**: Production workloads requiring strict isolation, predictable performance, and multi-tenancy security.

Let’s get started and explore how to share GPUs with MPS.

## **Prepare the Environment**

To set up your environment with Cloud Shell, follow these steps:

1. In the Google Cloud console, launch a Cloud Shell session by clicking the **Activate Cloud Shell** icon in the Google Cloud console. This launches a session in the bottom pane of Google Cloud console.  
2. Set the default environment variables:

```bash
export PROJECT_ID=$(gcloud config get project)
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)")
export CLUSTER_NAME=gpu-vllm-mps
export LOCATION=us-central1 # Choose a region that has NVIDIA A100 GPUs available
export ZONE=us-central1-c # Choose a zone within the region that has A100 GPUs available
export HF_TOKEN=HUGGING_FACE_TOKEN
export CLUSTER_VERSION="1.35.2-gke.1269001" # Must be 1.34 or later
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
--labels=created-by=ai-on-gke,guide=gpu-mps
```

### Create a node pool with A100 GPUs

This creates a node pool with just a single machine with a single A100 GPU. We disable installation of the GPU Device Plugin since we will be installing
the NVIDIA GPU DRA driver instead. We request spot capacity here to increase the chance of getting a node quickly.

```bash
gcloud container node-pools create a100-pool \
    --cluster=${CLUSTER_NAME} \
    --location=${LOCATION} \
    --node-locations=${ZONE} \
    --machine-type="a2-highgpu-1g" \
    --accelerator="type=nvidia-tesla-a100,count=1,gpu-driver-version=disabled" \
    --num-nodes=1 \
    --node-labels=gke-no-default-nvidia-gpu-device-plugin=true,nvidia.com/gpu.present=true,cloud.google.com/gke-nvidia-gpu-dra-driver=true \
    --image-type=UBUNTU_CONTAINERD \
    --spot
```

## **Configure Kubectl to communicate with your cluster**
To configure kubectl to communicate with your cluster, run the following command:

```bash
gcloud container clusters get-credentials ${CLUSTER_NAME} --location=${LOCATION}
```

## **Create Kubernetes Secret for Hugging Face credentials**

> [!NOTE]
> Make sure you have accepted the model license terms on Hugging Face for the `google/gemma-3-1b-it` model before proceeding. Your Hugging Face token must have access to this model.

To create a Kubernetes Secret that contains the Hugging Face token, run the following command:

```bash
kubectl create secret generic hf-secret --from-literal=hf_api_token=${HF_TOKEN} --namespace=${NAMESPACE}
```

## **Install the NVIDIA GPU driver**

Since we disabled the installation of the GPU Device Plugin at node pool creation time, we need to install the NVIDIA GPU driver manually.

```bash
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/ubuntu/daemonset-preloaded.yaml
```

## **Install the NVIDIA GPU DRA driver**

We install the NVIDIA GPU DRA driver using a Helm chart. Make sure that you have Helm installed, if not,
you can follow the [Helm documentation](https://helm.sh/docs/intro/install/) to install it. MPS support
is enabled by setting `featureGates.MPSSupport=true`.

```bash
helm repo add nvidia https://nvidia.github.io/gpu-operator
helm repo update

helm install nvidia-dra-driver-gpu nvidia/nvidia-dra-driver-gpu \
    --version="25.12.0" --create-namespace --namespace=nvidia-dra-driver-gpu \
    --set nvidiaDriverRoot="/opt/nvidia" \
    --set gpuResourcesEnabledOverride=true \
    --set resources.computeDomains.enabled=false \
    --set featureGates.MPSSupport=true \
    --set kubeletPlugin.priorityClassName="" \
    --set 'kubeletPlugin.tolerations[0].operator=Exists'
```

### Verify that the NVIDIA GPU DRA driver is working

Check that the NVIDIA GPU DRA driver is installed and working by inspecting the driver pod:

```bash
kubectl -n nvidia-dra-driver-gpu get pods
```

The pod should be in a Running state. If not, you can inspect the logs with:

```bash
kubectl -n nvidia-dra-driver-gpu logs -l app.kubernetes.io/name=nvidia-dra-driver-gpu -c gpus
```

Verify that the driver has published a ResourceSlice object that lists the GPU on the node:

```bash
kubectl get resourceslices -o yaml
```

## **Create the DRA ResourceClaim**

We will be using a single GPU that will be shared between replicas. Therefore we create a ResourceClaim
that will be referenced from the Pod spec in the Deployment.

Inspect the following `claim.yaml`. We use `strategy: MPS` and configure limits for active thread percentage and pinned device memory.

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
              strategy: MPS
              mpsConfig:
                defaultActiveThreadPercentage: 50
                defaultPinnedDeviceMemoryLimit: 8Gi
```

Apply the manifest

```bash
kubectl apply -f claim.yaml
```

## **Deploy the vllm workload**

We are using the Gemma 3 models as they are smaller and therefore let us run the tutorial using hardware (i.e., GPUs) that are more easily available.

We create a Deployment that runs replicas of vllm. We reference the ResourceClaim `gpu-claim` in
the pod spec, so both pods will reference the same claim.

Inspect the following `vllm.yaml`. Note that with MPS, the hard memory limit is set in the `ResourceClaim` parameters we applied earlier.

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
        - --gpu-memory-utilization=0.20
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

View the logs from the running model servers:

```bash
kubectl logs vllm-gpu-<suffix> -f
```

## **Generate traffic to the model**

We will send requests to the model servers and then use the logs to verify that we are getting
responses from both replicas.

First we get the external IP of the service

```bash
export vllm_service=$(kubectl get service vllm-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}' -n ${NAMESPACE})
```

Send a bunch of requests to the model servers.

```bash
curl http://$vllm_service:8000/v1/completions \
-H "Content-Type: application/json" \
-d '{
    "model": "google/gemma-3-1b-it",
    "prompt": "Write a story about san francisco",
    "max_tokens": 100,
    "temperature": 0
}'
```

## **Clean up** 

### Delete the deployed resources:

To avoid incurring charges to your Google Cloud account for the resources that you created in this guide, run the following commands:

Delete the cluster:

```bash
gcloud container clusters delete ${CLUSTER_NAME} \
  --location=${LOCATION}
```
