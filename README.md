# FastChat-GKE

## Prepare LLM models
- [FastChat](https://github.com/lm-sys/FastChat#model-weights)
- [Chinese-LLaMA-Alpaca](https://github.com/ymcui/Chinese-LLaMA-Alpaca)

## Initialize the environment
```
export PROJECT=$(gcloud info --format='value(config.project)')
export GKE_CLUSTER_NAME=<the name of your new cluster>
export REGION=<the desired region for your cluster, such as us-central1>
export ZONE=<the desired zone for your node pool, such as us-central1-a. The zone must be in the same region as the cluster control plane>
export VPC_NETWORK=<the name of your VPC network>
export VPC_SUBNETWORK=<the name of your subnetwork>

# Filestore
export FILESTORE_NAME=<replace with filestore instance name>
export FILESHARE_NAME=<replace with filestore share name>

# Artifact Repo
export BUILD_REGIST=<replace this with your preferred Artifacts repo name>
```

## Enable APIs
```
gcloud services enable compute.googleapis.com artifactregistry.googleapis.com container.googleapis.com file.googleapis.com
```

## Filestore instance

#### Create a Filestore instance
Use the below commands to create a Filestore instance to store llm model .
```
gcloud filestore instances create ${FILESTORE_NAME} --zone=${ZONE} --tier=BASIC_HDD --file-share=name=${FILESHARE_NAME},capacity=1TB --network=name=${VPC_NETWORK}
```
#### Attach Filestore to your VM
```
sudo apt-get -y update &&
sudo apt-get -y install nfs-common
sudo mkdir /data
sudo chown -R $USER /data
sudo mount your_filestore_ip_address:/${FILESHARE_NAME} /data
```

#### Copy LLM models to Filestore


## Create GKE cluster
```
gcloud container clusters create ${GKE_CLUSTER_NAME} \
    --network ${VPC_NETWORK} \
    --release-channel "None" \
    --cluster-version "1.25.8-gke.500" \
    --image-type "COS_CONTAINERD" \
    --num-nodes 1 \
    --enable-autoscaling --total-min-nodes "1" --total-max-nodes "2" --location-policy "BALANCED" \
    --machine-type "n1-standard-4" \
    --accelerator "type=nvidia-tesla-t4,count=2" \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver,GcpFilestoreCsiDriver \
    --zone ${ZONE}
```


## Install NVIDIA GPU device drivers
After creating a GKE cluster with GPU, you need to install NVIDIA's device drivers on the nodes. Google provides a DaemonSet that you can apply to install the drivers. To deploy the installation [DaemonSet](https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/cos/daemonset-preloaded.yaml) and install the default GPU driver version, run the following command:
```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/cos/daemonset-preloaded.yaml
```


## Create Cloud Artifacts as Docker Repo
```
gcloud artifacts repositories create ${BUILD_REGIST} --repository-format=docker --location=${REGION}
```

## Build FastChat Image
```
gcloud auth configure-docker ${REGION}-docker.pkg.dev
docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${BUILD_REGIST}/fastchat:v1 docker/
docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${BUILD_REGIST}/fastchat:v1
```


## Deploy FastChat
#### FastChat include controller, worker, gui and rest api.
```
kubectl apply -f filstore-pv-pvc.yaml
kubectl apply -f controller.yaml
kubectl apply -f model-worker-t4.yaml
```
check worker pod log and after load model success.
```
kubectl apply -f gui.yaml
kubectl apply -f api.yaml
```


#### Check pod status
```
kubectl get pod
```


## Test GUI
1. Get external IP address
```
kubectl get svc gui-svc
```
2. Open browser and input external ip address
![image](https://github.com/hellof20/fastchat-gke/assets/8756642/b8d7ac9a-ec64-4630-bbcf-47d22566f101)


## Test [OpenAI-Compatible RESTful APIs & SDK](https://github.com/lm-sys/FastChat/blob/05b3bcdea6ac5106e8ef4a57f7f27a36ccaca253/docs/openai_api.md)
```
kubectl get svc api
```
