# MIG testing with Rancher

## Setup and configuration

### NVIDIA MIG DRIVER
1. Download and install the mig driver
1. Create GPU instances
    * https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html#create-gi
1. Configure a profile 
    * list profiles `nvidia-smi mig -lgip`
    * slice into 7 for example `nvidia-smi mig -cgi 19,19,19,19,19,19,19 -C`
    * https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html#device-names

    *NOTE*
    
    ```
    Without creating GPU instances (and corresponding compute instances), CUDA workloads cannot be run on the GPU. In other words, simply enabling MIG mode on the GPU is not sufficient. Also note that, the created MIG devices are not persistent across system reboots. Thus, the user or system administrator needs to recreate the desired MIG configurations if the GPU or system is reset. For automated tooling support for this purpose, refer to the NVIDIA MIG Partition Editor (or mig-parted) tool.
    ```

### CONTAINER RUNTIME
Currently nvidia-docker is your best option. Other CRI will be available in the future

1. Install Docker

    ```
    #Add containers repository
    zypper addrepo https://download.opensuse.org/repositories/Virtualization:containers/openSUSE_Leap_15.3/Virtualization:containers.repo \
    && zypper refresh

    #Install Docker
    zypper install docker

    #Enable docker in the supervisor
    systemctl --now enable docker

    #Test docker
    sudo docker run --rm hello-world
    ```
    
1. Setup the nvidia container toolkit

    ```
    zypper ar https://nvidia.github.io/nvidia-docker/opensuse-leap15.1/nvidia-docker.repo \
    && zypper refresh

    #Install nvidia docker runtime
    zypper install -y nvidia-docker2

    #Restart the docker daemon to enable the changes
    systemctl restart docker

    #Test with a cuda container
    docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi
    ```
https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#installing-on-suse-15

### Container Orchestration

1. Install K3s and use the docker runtime

	`curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--docker" sh -`

1. Install Helm

	```
	curl -L -O https://get.helm.sh/helm-v3.6.1-linux-amd64.tar.gz
	tar -xzvf helm-v3.6.1-linux-amd64.tar.gz 
	mv linux-amd64/helm /usr/local/bin/helm
	```
	
1. Move the kubeconfig to a location where helm can use it

	`cp /etc/rancher/k3s/k3s.yaml ~/.kube/config`

1. Deploy and run nvidia-gpu-plugin and gpu-feature-discovery

	```
	helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
	helm repo add nvgfd https://nvidia.github.io/gpu-feature-discovery
	helm repo update
	```
	
1. Verify nvdp and nvgfd versions

	`helm search repo nvdp --devel`
	
	```
	NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
nvdp/nvidia-device-plugin       0.9.0           0.9.0           A Helm chart for the nvidia-device-plugin on Ku...
	```
	
	`helm search repo nvgfd --devel`
	
	```
	NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                       
nvgfd/gpu-feature-discovery     0.4.1           0.4.1           A Helm chart for gpu-feature-discovery on Kuber...
	```
	
1. Assign a MIG strategy and install nvdp and nvgfd

	```
	export MIG_STRATEGY=<none | single | mixed>`

	helm install \
   --version=0.9.0 \
   --generate-name \
   --set migStrategy=${MIG_STRATEGY} \
   nvdp/nvidia-device-plugin

	helm install \
   --version=0.4.0 \
   --generate-name \
   --set migStrategy=${MIG_STRATEGY} \
   nvgfd/gpu-feature-discovery
   ```
	
