---
title: Serving LLMs using Huggingface Text Generation Inference Server on Kubernetes
description: Serving LLMs with Huggingface Text Generation Inference Server robustly in minutes
slug: huggingface-tgi-serving
date: 2024-03-15 00:00:00+0000


categories:
    - LLM
    - Kubernetes
tags:
    - LLM
    - Huggingface
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## Intro

In case you want to host your own LLM instance of popular models like Mistral, Llama-2 or your own fine-tuned version of one of these models, Hugginface Text Generation Inference (TGI) is a great tool to get the job done. Often this requires running your inference on a Kubernetes or Openshift cluster providing the necessary GPU infrastructure.

In this post, I'll show a quick example of how to get your own LLM instance up and running on your Kubernetes cluster.

## Prerequisite

A suitable GPU (A10, A100, H100, L40s) available within your cluster, depending on the model you want to run. Usually this involves the installation of the [NVIDIA Node Feature Discovery (NFD) Operator](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/install-nfd.html) and the [NVIDIA GPU operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/) to make the GPU available to your pod.

### Example: Deploying Mistral using TGI in Kubernetes

Running TGI on Kubernetes is pretty straightforward, once you've figured out the basic setup.
To avoid downloading the model weights every time you restart the pod, create a PersistentVolumeClaim (PVC) to persist the model weights:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hf-cache
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  selector:
    matchLabels:
      pv: local
  storageClassName: "" # your storage class, if omitted, the default storage class will be used
```

Now that you have created a PVC, you can create the deployment using the following deployment yaml. This deployment will download your model weights, then start the text generation inference servcer. In this example, it will run the `mistralai/Mistral-7B-Instruct-v0.2` model:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tgi-mistral
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tgi-mistral
  template:
    metadata:
      labels:
        app: tgi-mistral
    spec:
      volumes:
        - name: hub
          persistentVolumeClaim:
            claimName: hf-cache
        - name: dshm
          emptyDir:
            medium: Memory
      containers:
        - name: model
          image: ghcr.io/huggingface/text-generation-inference:1.4.0
          command: ["bash", "-c"]
          args:
            - text-generation-server download-weights mistralai/Mistral-7B-Instruct-v0.2;
              text-generation-launcher --model-id mistralai/Mistral-7B-Instruct-v0.2 --port 8080
          env:
            - name: PORT
              value: "8080"
            - name: HF_HUB_CACHE
              value: /data
            - name: HF_HOME
              value: /data
          volumeMounts:
            - mountPath: /dev/shm
              name: dshm
            - mountPath: /data
              name: hub
          resources:
            requests:
              cpu: 1.0
              memory: 8Gi
              nvidia.com/gpu: 1
            limits:
              cpu: 5.0
              memory: 32Gi
              nvidia.com/gpu: 1
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: tgi-mistral-service
  labels:
    app: tgi-mistral-service
spec:
  selector:
    app: tgi-mistral
  type: NodePort
  ports:
    - protocol: TCP
      port: 8080

```

Within your cluster, you can now reach the TGI service using the following curl command in another pod. Make sure to replace the `deployment-namespace` with the name of the namespace you deployed TGI to: 

```bash
curl http://tgi-mistral.<deployment-namespace>.svc.cluster.local::8080/generate \
    -X POST \
    -d '{"inputs":"What is Deep Learning?","parameters":{"max_new_tokens":20}}' \
    -H 'Content-Type: application/json'
```

If you want to expose TGI to the public internet, you can create a Route (Openshift) or Ingress (Kubernetes).

Please make sure to adjust the timeout of your Route/Ingress if you have long running inference jobs. Larger models like Mixtral might run for more than 60s - which is the default timeout for Routes/Ingress - for large prompts wiht long contexts.

## Further information

For those interested in diving deeper, please refer to the [Hugginface TGI documentation](https://github.com/huggingface/text-generation-inference) for CLI parameters, Python SDK, etc.
