## on Master
```
 sudo su
 setenforce 0

 > W/ Internet access
 wget https://github.com/rancher/k3s/releases/download/v1.18.2-rc2%2Bk3s1/k3s-airgap-images-amd64.tar
 wget https://github.com/rancher/k3s/releases/download/v1.18.2-rc2%2Bk3s1/k3s
 curl -sfL https://get.k3s.io > install.sh
 chmod +x install.sh
 chmod +x k3s 

 > W/o Internet access
 mkdir -p /var/lib/rancher/k3s/agent/images/
 cp k3s-airgap-images-amd64.tar /var/lib/rancher/k3s/agent/images/
 cp k3s /usr/local/bin/k3s
 INSTALL_K3S_SKIP_DOWNLOAD=true K3S_TOKEN=mynodetoken ./install.sh
 systemctl restart k3s
 kubectl  get nodes
```

## On node 
```
 > W/o Internet access
 sudo su
 setenforce 0
 export MASTER=172.31.28.23
 mkdir -p /var/lib/rancher/k3s/agent/images/
 cp k3s-airgap-images-amd64.tar /var/lib/rancher/k3s/agent/images/
 mv k3s /usr/local/bin/k3s
 K3S_URL=https://${MASTER}:6443 INSTALL_K3S_SKIP_DOWNLOAD=true K3S_TOKEN=mynodetoken ./install.sh


``` 


