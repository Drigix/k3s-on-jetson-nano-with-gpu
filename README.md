# K3S cluster installation using GPU on Jetson nano devices

## Introduction
Together with a couple of other students, we carried out a project in which we operated on a K3S cluster built from Jetson nano edge devices. Each of the aforementioned devices has an NVIDIA graphics card so that work can be greatly accelerated. But first you need to do the configuration so that the system, Docker and eventually the K3S cluster can access the GPU.
It turned out that the instructions and documentation from the various sites are not well written and there are a lot of discrepancies especially in the versions of Ubuntu, CUDY and NVIDI tools.

Therefore, we decided to write down the instructions, which took us no small amount of time.

## Update and install the necessary libraries
```bash
sudo apt-get dist-upgrade
sudo apt-get install curl
sudo apt-get install nano
```

## Add CUDA PATH
```bash
nano ~/.bashrc

export PATH=/usr/local/cuda/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}

source ~/.bashrc
```

```bash
nvcc --version
```

## Setting the NVIDIA runtime for Docker
```bash

curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

```

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

```bash
sudo systemctl restart docker
```

```bash
sudo docker info | grep Runtime
```

![Runtime result](/images/nvidia-runtime.png)

## K3S

### MASTER
```bash
/usr/local/bin/k3s-uninstall.sh
rm -rf /var/lib/rancher
rm -rf /etc/rancher
rm -rf /opt/cni/bin
rm -rf /var/lib/calico
rm -rf /var/run/calico
```

```bash
mkdir -p /opt/cni/bin
mkdir /var/lib/calico
mkdir -p /var/run/calico
```

```bash
curl -sfL https://get.k3s.io/ | sh -sv - --flannel-backend=none --disable-network-policy --write-kubeconfig-mode 644 --node-name master --cluster-cidr=10.42.0.0/16 --docker
```

```bash
sudo systemctl status k3s
```
![K3S running](/images/k3s-running.png)

```bash
sudo kubectl get nodes
```
![Control plane ready](/images/control-plane-ready.png)

#### Calico
```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
```

```bash
watch kubectl get pods --all-namespaces
```

### NODES

```bash
/usr/local/bin/k3s-agent-uninstall.sh
rm -rf /var/lib/rancher
rm -rf /etc/rancher
mkdir -p /opt/cni/bin
```

```bash
mkdir /var/lib/calico
mkdir -p /var/run/calico
```

```bash
cat /var/lib/rancher/k3s/server/node-token
```

```bash
curl -sfL https://get.k3s.io/ | K3S_URL=YOUR_MASTER_ADDRESS  K3S_TOKEN=$ K3S_MASTER_TOKEN INSTALL_K3S_EXEC="--docker" sh -
```