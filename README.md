# basic-k8s-istio-on-raspberry-pi


I used below stack to setup my k8s luster environment.

## My Hardware and OS:

* Hardware: Raspberry Pi 4 Model B Rev 1.4 
* Arch: ARM 64
* OS: Ubuntu 20.03.3 LTS


## Software:

* [kind](https://kind.sigs.k8s.io/)
* [docker engine](https://docs.docker.com/engine/)
* [Kubectl](https://kubernetes.io/docs/reference/kubectl/)
* [Helm](https://helm.sh)


## Installations :


### kind :

```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-arm64
chmod +x ./kind
sudo chown root:root ./kind
sudo mv ./kind /usr/local/bin/kind
```

### docker engine: (right from linux installation steps on [docker website](https://docs.docker.com/engine/install/ubuntu/))

```
 sudo apt-get remove docker docker-engine docker.io containerd runc
```

```
 sudo apt-get update

 sudo apt-get install ca-certificates curl gnupg lsb-release
```

```
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

```
sudo apt-get update
```

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

[Post Docker Installation](https://docs.docker.com/engine/install/linux-postinstall/):

```
 sudo groupadd docker
 sudo usermod -aG docker $USER
```

```
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```


### kubectl

https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

```
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo apt-get install -y apt-transport-https


sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg



echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list


sudo apt-get update
sudo apt-get install -y kubectl
```

### helm

used script here : https://helm.sh/docs/intro/install/
  
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

---
Also had to make sure the correct c-groups are enabled otherwise `kind` had issues with starting up the cluster.[reference](https://github.com/kubernetes-sigs/kind/issues/3044).
[solution](https://askubuntu.com/questions/1189480/raspberry-pi-4-ubuntu-19-10-cannot-enable-cgroup-memory-at-boostrap).

```
vi /boot/firmware/cmdline.txt
```


Append below lines and reboot:
```
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```


---

Setup Nginx Ingress and Metallb :

Used the setup via Kind website for linux, reference [here.](https://kind.sigs.k8s.io/docs/user/loadbalancer/)

Below copied for quick guide :


Apply Metallb
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```

Check docker network
```
docker network inspect -f '{{.IPAM.Config}}' kind
```

IPAddressPool and L2Advertisement
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.200-172.18.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system

```

---

Using the loadbalancer :

```
kind: Pod
apiVersion: v1
metadata:
  name: foo-app
  labels:
    app: http-echo
spec:
  containers:
  - name: bar-app
    image: ealen/echo-server:latest
    ports:
    - containerPort: 80
    env:
    - name: PORT
      value: "80"
---
kind: Service
apiVersion: v1
metadata:
  name: foo-service
spec:
  type: LoadBalancer
  selector:
    app: http-echo
  ports:
  - port: 80

```

---