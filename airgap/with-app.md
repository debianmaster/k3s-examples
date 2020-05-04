> Start with a fresh machine


## pre-req
```
# setenforce 0  if centos

apt install docker.io
mkdir -p /etc/rancher/k3s/

# Get primary interface address of machine, manually assign if empty
ip=$(ip route get 8.8.8.8 | grep -Eo 'src \S+' | awk '{print $2}')

alias k=kubectl
```

# set priv registry 
```
mkdir -p /registry


docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v /registry:/var/lib/registry \
  registry:2
```

> any request to docker.io/org/imagename will be now made as http://${ip}:5000/org/imagename   
> containerd first checks if an image is there in http://${ip}:5000/org/imagename, else image will be pulled and cached in intermediate registry
```
cat > /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  docker.io:
    endpoint:
      - "http://${ip}:5000"
  ${ip}:5000:
    endpoint:
      - "http://${ip}:5000"
EOF
```
## Download k3s binary and k3s system images

```
wget https://github.com/rancher/k3s/releases/download/v1.18.2-rc2%2Bk3s1/k3s-airgap-images-amd64.tar
wget https://github.com/rancher/k3s/releases/download/v1.18.2-rc2%2Bk3s1/k3s
curl -sfL https://get.k3s.io > install.sh
chmod +x install.sh
chmod +x k3s 
```


## Install k3s
```
mkdir -p /var/lib/rancher/k3s/agent/images/
cp k3s-airgap-images-amd64.tar /var/lib/rancher/k3s/agent/images/
cp k3s /usr/local/bin/k3s
INSTALL_K3S_SKIP_DOWNLOAD=true K3S_TOKEN=mynodetoken ./install.sh
systemctl restart k3s
kubectl  get nodes
```

## Install required cli tools [helm,skopeo]
```
. /etc/os-release
sudo sh -c "echo 'deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/x${NAME}_${VERSION_ID}/ /' > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"
wget -nv https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/x${NAME}_${VERSION_ID}/Release.key -O- | sudo apt-key add -
sudo apt-get update -qq
sudo apt-get install skopeo
sudo apt install net-tools 

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

```

## generate image list of helm package
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm fetch jetstack/cert-manager --version v0.12.0
helm template ./cert-manager-v0.12.0.tgz | grep -oP "(?<=image: ).*(?=)" | tr -d \' | tr -d \" >> ./image-list.txt
```

## sync images from pubic repo to private reg

```
ex:- 
skopeo copy docker://quay.io/jetstack/cert-manager-cainjector:v0.12.0 docker://localhost:5000/jetstack/cert-manager-cainjector:v0.12.0 --dest-tls-verify=false
```

```
input="./image-list.txt"
while IFS= read -r line
do
  tarfile=$(echo ${line} | tr \/ _)
  skopeo copy docker://${line} dir:${tarfile}
done < "$input"


input="./image-list.txt"
while IFS= read -r line
do
  tarfile=$(echo ${line} | tr \/ _)
  retag=$(echo ${line} | tr _ \/ | sed 's/quay.io/localhost:5000/g')
  echo ${tarfile} ${retag}
  skopeo copy dir:${tarfile} docker://${retag} --dest-tls-verify=false
done < "$input"
```

## to test disconnect registry pull
```
iptables -A OUTPUT -p tcp -m string --string "docker.io" --algo kmp -j REJECT
```


## deploy application
```
helm template ./cert-manager-v0.12.0.tgz | k apply -f-
```


## add more machines
```
sudo su
setenforce 0
wget https://github.com/rancher/k3s/releases/download/v1.18.2-rc2%2Bk3s1/k3s-airgap-images-amd64.tar
wget https://github.com/rancher/k3s/releases/download/v1.18.2-rc2%2Bk3s1/k3s
curl -sfL https://get.k3s.io > install.sh
chmod +x install.sh
chmod +x k3s 

export ip=<machine_with_priv_reg>
cat > /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  docker.io:
    endpoint:
      - "http://${ip}:5000"
  ${ip}:5000:
    endpoint:
      - "http://${ip}:5000"
EOF

export MASTER=172.31.28.23. #replace with master ip

mkdir -p /var/lib/rancher/k3s/agent/images/
cp k3s-airgap-images-amd64.tar /var/lib/rancher/k3s/agent/images/
mv k3s /usr/local/bin/k3s
K3S_URL=https://${MASTER}:6443 INSTALL_K3S_SKIP_DOWNLOAD=true K3S_TOKEN=mynodetoken ./install.sh

```
