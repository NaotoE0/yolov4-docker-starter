# What is
- UNOFFICIAL Procedure to use [YOLOv4](https://github.com/AlexeyAB/darknet) docker container
- It achieve inference on GPU using pre-trained YOLOv4 model


# (Optional) Preparing GPU VM in Google Cloud
Run command in Google cloud console:
```bash
export IMAGE_FAMILY="pytorch-latest-gpu-ubuntu-2004-py310"
export ZONE="asia-northeast1-a"
export INSTANCE_NAME="${USER}-dev2"
export INSTANCE_TYPE="g2-standard-4"
gcloud compute instances create $INSTANCE_NAME \
        --provisioning-model=SPOT \
        --zone=$ZONE \
        --image-family=$IMAGE_FAMILY \
        --image-project=deeplearning-platform-release \
        --maintenance-policy=TERMINATE \
        --accelerator="type=nvidia-l4,count=1" \
        --machine-type=$INSTANCE_TYPE \
        --boot-disk-size=200GB \
        --metadata="install-nvidia-driver=True"
```

- Reference: https://cloud.google.com/deep-learning-vm/docs/create-vm-instance-gcloud

## (Memo) How to get INSTANCE_TYPE values
```bash
gcloud compute images list \
    --project deeplearning-platform-release \
    --no-standard-images
```


# Build YOLOv4 container image
```bash
# Check availability of both nvidia docker and gpu
docker run --gpus all nvidia/cuda:11.6.2-cudnn8-devel-ubuntu20.04 nvidia-smi

# Get latest src
git clone https://github.com/AlexeyAB/darknet
cd darknet

# Refer README.md in darknet
sed -ie 's/GPU=0/GPU=1/g' Makefile
sed -ie 's/OPENCV=0/OPENCV=1/g' Makefile
sed -ie 's/CUDNN=0/CUDNN=1/g' Makefile
sed -ie 's/CUDNN_HALF=0/CUDNN_HALF=1/g' Makefile

# Customize
sed -ie 's/nvidia\/cuda:11.6.0-cudnn8-devel-ubuntu20.04/nvidia\/cuda:11.6.2-cudnn8-devel-ubuntu20.04/g' Dockerfile.gpu
sed -ie 's/RUN rm Docker-compose.yml/# RUN rm Docker-compose.yml/g' Dockerfile.gpu
sed -ie 's/RUN apt-get install -y sudo libgomp1/RUN apt-get install -y sudo libgomp1 wget python3-opencv vim/g' Dockerfile.gpu

# Build container image
docker build \
    -t ${USER}/yolo:gpu \
    -f Dockerfile.gpu \
    .
```


# Run container and preference
```bash
# Launch and login container
docker run -it -d --gpus all \
    --name ${USER}-yolo \
    ${USER}/yolo:gpu /bin/bash
docker exec -it ${USER}-yolo /bin/bash

# Prepare weights
cd
mkdir weights
wget https://github.com/AlexeyAB/darknet/releases/download/darknet_yolo_v3_optimal/yolov4.weights \
    -P weights/

# Prepare config
mkdir config
cp ~/darknet/cfg/coco.data config/
sed -ie 's/data\/coco.names/\/home\/yolo\/darknet\/data\/coco.names/g' config/coco.data

# Run inference
~/darknet/darknet detector test \
    config/coco.data \
    ~/darknet/cfg/yolov4.cfg \
    weights/yolov4.weights \
    ~/darknet/data/dog.jpg \
    -dont_show

# You got result file predictions.jpg
```
