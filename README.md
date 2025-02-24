#To set static network ip : https://linuxconfig.org/setting-a-static-ip-address-in-ubuntu-24-04-via-the-command-line
# Kubernetes-Installation-on-Ubuntu-22.04-with-CRI-Dockerd


## Set up the Docker and Kubernetes repositories:
Docker Installation

```bash
  sudo apt update -y
```
```bash
  sudo curl https://get.docker.com | bash
#if above cannot
###please make sure to be same place for all installation
curl -fsSL https://get.docker.com -o install-docker.sh
chmod +x install-docker.sh
./install-docker.sh
####
```
```bash
   sudo groupadd docker
```
```bash
  sudo usermod -aG docker $USER
  #Log out and log back in so that your group membership is re-evaluated
```


Docker Runtime Interface installation

```bash
  wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.12/cri-dockerd-0.3.12.amd64.tgz
  tar xzvf cri-dockerd-0.3.12.amd64.tgz
```
```bash
  sudo mv cri-dockerd/cri-dockerd /usr/local/bin/
```
```bash
  wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
```
```bash
  wget https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
```
```bash
  sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
```
```bash
  sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
```
```bash
  sudo systemctl daemon-reload
```
```bash
  sudo systemctl start cri-docker.service
  sudo systemctl enable cri-docker.service
  sudo systemctl enable --now cri-docker.socket
```
Modify Host File
```
hostname #check your current hostname
hostnamectl set-hostname master # change host name to master
vi /etc/hosts
#add master and worker private ip and name lie
#172.31.41.1 master
#172.31.37.186 worker
```
Disable swap partition 
```bash
  sudo swapoff -a
```
Kubeadm installation
#https://v1-31.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
```bash
   systemctl status ufw
   systemctl stop ufw
   systemctl disable ufw
```
```bash
  #curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  # If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
  # sudo mkdir -p -m 755 /etc/apt/keyrings
  curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
```bash
  #echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
  # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
  echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```bash
  # Update the repositiries
  sudo apt-get update
```
```bash
  # Use the same versions to avoid issues with the installation.
  sudo apt-get install -y kubelet kubeadm kubectl
```
```bash
  sudo apt-mark hold docker-ce kubelet kubeadm kubectl #Not sure two times and need to test again
```
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system



```
Initialize the Master Node
```bash
  sudo apt-mark hold docker-ce kubelet kubeadm kubectl #Mandatory to hold before initialize
  sudo systemctl status kubelet
  #If not run this
  sudo systemctl enable --now kubelet
  sudo kubeadm init --apiserver-advertise-address=<control_plane_ip> --cri-socket unix:///var/run/cri-dockerd.sock  --pod-network-cidr=192.168.0.0/16
  #sudo kubeadm init --apiserver-advertise-address=192.168.1.9 --cri-socket unix:///var/run/cri-dockerd.sock  --pod-network-cidr=192.168.0.0/16
```
```bash
  # Copy your join command and keep it safe.
  kubeadm join 10.0.0.4:6443 --token 7lf2k2.jju8o1n1d04qv5pq \
	--discovery-token-ca-cert-hash sha256:e420f8f8b5e1feb646a5725ad6867fe519e1862c7494d31acbbb9e95caa75ed1

    # Add --cri-socket /var/run/cri-dockerd.sock to the command
kubeadm join <control_plane_ip>:6443 --token 31rvbl.znk703hbelja7qbx --cri-socket unix:///var/run/cri-dockerd.sock --discovery-token-ca-cert-hash sha256:3dd5f401d1c86be4axxxxxxxxxx61ce965f5xxxxxxxxxxf16cb29a89b96c97dd

```
Configure kubectl
```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
To set up the Calico network
```bash
  # Use this if you have initialised the cluster with Calico network add on.
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml -O

# Change the ip to 10.244.0.0/16 if the node network is 192.168.x.x
kubectl create -f custom-resources.yaml

```
Verify the Cluster
```bash
  kubectl get nodes
```
Joining the node to the cluster:
```bash
  sudo kubeadm join $controller_private_ip:6443 --token $token --discovery-token-ca-cert-hash $hash
#Ex:
# kubeadm join <control_plane_ip>:6443 --cri-socket unix:///var/run/cri-dockerd.sock --token 31rvbl.znk703hbelja7qbx --discovery-token-ca-cert-hash sha256:3dd5f401d1c86be4axxxxxxxxxx61ce965f5xxxxxxxxxxf16cb29a89b96c97dd

```
If the joining code is lost, it can retrieve using below command
```bash
  kubeadm token create --print-join-command
```
