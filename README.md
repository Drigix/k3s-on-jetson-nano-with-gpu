# K3S cluster installation using GPU on Jetson nano devices

## Introduction
Together with a couple of other students, we carried out a project in which we operated on a K3S cluster built from Jetson nano edge devices. Each of the aforementioned devices has an NVIDIA graphics card so that work can be greatly accelerated. But first you need to do the configuration so that the system, Docker and eventually the K3S cluster can access the GPU.
It turned out that the instructions and documentation from the various sites are not well written and there are a lot of discrepancies especially in the versions of Ubuntu, CUDY and NVIDI tools.

Therefore, we decided to write down the instructions, which took us no small amount of time.

## System installation

## Update and install the necessary libraries
```bash
sudo apt-get dist-upgrade
sudo apt-get install curl
sudo apt-get install nano
```

## Add CUDA PATH
Jetson nano device should have installed CUDA with system installation, however PATH not setting up so you have to do it for your own:
```bash
nano ~/.bashrc

export PATH=/usr/local/cuda/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}

source ~/.bashrc
```
After setting PATH check CUDA version:
```bash
nvcc --version
```

## Setting the NVIDIA runtime for Docker
Since the K3S cluster will use docker containers, it is necessary to install the tools for cooperation with NVIDIA. Otherwise, using GPUs at the container and cluster level will not be possible.
```bash

curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

```
There is a configuration in the daemon.json file that needs to be added or changed (if the file already exists) to the one shown below, so that the default runtime will be the NVIDIA runtime.
```bash
sudo nano /etc/docker/daemon.json

{
 "default-runtime": "nvidia",
  "runtimes": {
     "nvidia": {
         "path": "nvidia-container-runtime",
         "runtimeArgs": []
     }
 }
}
```
Restart the docker service
```bash
sudo systemctl restart docker
```
Check if the configuration has been changed correctly: 
```bash
sudo docker info | grep Runtime
```

![Runtime result](/images/nvidia-runtime.png)

## K3S

### MASTER
For the cluster to function properly, it is necessary to store files in specific locations. To prevent conflicts, first, just in case, uninstall k3s and clean the given locations.
```bash
/usr/local/bin/k3s-uninstall.sh
rm -rf /var/lib/rancher
rm -rf /etc/rancher
rm -rf /opt/cni/bin
rm -rf /var/lib/calico
rm -rf /var/run/calico
```
The installation starts by creating new locations that will be needed for calico to work (the network interface in the cluster).
```bash
mkdir -p /opt/cni/bin
mkdir /var/lib/calico
mkdir -p /var/run/calico
```
Once the locations for the network interface exist, you can run the command to create a master.
```bash
curl -sfL https://get.k3s.io/ | sh -sv - --flannel-backend=none --disable-network-policy --write-kubeconfig-mode 644 --node-name master --cluster-cidr=10.42.0.0/16 --docker
```
To check if everything started correctly, you can execute the following commands:
```bash
sudo systemctl status k3s
```
![K3S running](/images/k3s-running.png)

or 

```bash
sudo kubectl get nodes
```
![Control plane ready](/images/control-plane-ready.png)

#### Calico
Then you can install calico to make the cluster communication possible.
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```
To check if calico containers have started use the command below:
```bash
watch kubectl get pods --all-namespaces
```

### NODES
The workers installation should be started similarly to the master, uninstalling the k3s cluster, deleting the network interface file locations and creating them again.
```bash
/usr/local/bin/k3s-agent-uninstall.sh
rm -rf /var/lib/rancher
rm -rf /etc/rancher
```

```bash
mkdir -p /opt/cni/bin
mkdir /var/lib/calico
mkdir -p /var/run/calico
```
Starting workers requires providing a master token, which can be read using the command:
```bash
cat /var/lib/rancher/k3s/server/node-token
```
Then, substitute the read value for ***K3S_MASTER_TOKEN*** and the master's IP address for ***YOUR_MASTER_ADDRESS***:
```bash
curl -sfL https://get.k3s.io/ | K3S_URL=YOUR_MASTER_ADDRESS  K3S_TOKEN=$ K3S_MASTER_TOKEN INSTALL_K3S_EXEC="--docker" sh -
```

### Testing container with GPU
After all settings and configuration you can test it creating demployment with pytorch. First you need start container with NVIDIA device plugin:
```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.13.0/nvidia-device-plugin.yml
```

Now you can create yaml file with deployment or just download from this github page.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jetson-pytorch
spec:
  replicas: 3
  selector:
    matchLabels:
      app: jetson-pytorch
  template:
    metadata:
      labels:
        app: jetson-pytorch
    spec:
      containers:
      - name: jetson-pytorch
        image: alex480/jetson-pytorch:latest
        resources:
          limits:
            nvidia.com/gpu: 1 # Request one GPU
        ports:
        - containerPort: 8888
```
```bash
kubectl apply -f jetson-nano-pytorch.yaml
```
