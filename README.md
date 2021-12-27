# Install From Scratch For Gitpod (v0.10.0) WITH K3S - Cloudflare

###Requirements
- Vmware Workstation with Ubuntu Server 20.04.3 LTS (tested) installed

I used one virtual machine. You can use as many workers as you want.

###Installation

*You need to replace _$DOMAIN_ with your own domain name.*

Let's make the necessary settings on the server.
```
sudo sed -i "s/ubuntu/master.$DOMAIN/" /etc/hostname

sudo hostname -F /etc/hostname

sudo sed -i "s/127.0.1.1 ubuntu/127.0.1.1 master.$DOMAIN master/" /etc/hosts

sudo dpkg-reconfigure tzdata
# Europe -> Istanbul (Write according to your own location.)

sudo apt -y update && sudo apt -y upgrade

sudo sed -i.bak '/swap/ s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
```
Need to reboot. Installing k3s, helm, docker and required packages

```
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release -y

curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Installing k3s
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644 --disable traefik --disable servicelb
```
```
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER /home/$USER/.kube/config
sed -i "s/127.0.0.1/master.$DOMAIN/" ~/.kube/config
```
```
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io helm

sudo usermod -aG docker $USER
chmod go-r ~/.kube/config
```
Install cert-manager
```
helm upgrade \
	--atomic \
	--cleanup-on-fail \
	--create-namespace \
	--install \
	--namespace cert-manager \
	--repo https://charts.jetstack.io \
	--reset-values \
	--set installCRDs=true \
	--wait \
	cert-manager \
	cert-manager
```
You need to create a api-token in Cloudflare
```
First, let’s create a token to use the CloudFlare API:

1. Profile → API Tokens → Create Token.

2. Set access rights as follows:

Permissions:
Zone — DNS — Edit
Zone — Zone — Read
Zone Resources:
Include — All Zones

Done
```
You need to write the api token in the following line of code.
```
kubectl create secret generic cloudflare-api-token -n cert-manager --from-literal=api-token="$CLOUDFLARE_API_TOKEN" --dry-run=client -o yaml | kubectl replace --force -f -
```
Make the necessary edits in certificate.yaml with cloudflare.yaml.Then run the following codes sequentially.

```
kubectl create namespace gitpod
kubectl apply -f cloudflare.yaml
kubectl apply -f certificate.yaml
```
Let's get started with Gitpod Installation.
```
docker create -ti --name installer eu.gcr.io/gitpod-core-dev/build/installer:main.2090
docker cp installer:/app/installer ./installer
docker rm -f installer


kubectl label node master.$DOMAIN gitpod.io/workload_meta=true gitpod.io/workload_ide=true gitpod.io/workload_workspace_services=true gitpod.io/workload_workspace_regular=true gitpod.io/workload_workspace_headless=true


./installer init > gitpod.config.yaml

```
You need to make changes to domain, containerdRuntimeDir and containerdSocket in gitpod.config.yaml file.
```
containerdRuntimeDir="/run/k3s/containerd/io.containerd.runtime.v2.task/k8s.io"
containerdSocket="/run/k3s/containerd/containerd.sock"
```

```
./installer validate config --config gitpod.config.yaml

./installer validate cluster --kubeconfig ~/.kube/config --config gitpod.config.yaml

./installer render --config gitpod.config.yaml --namespace gitpod > gitpod.yaml

kubectl apply -f gitpod.yaml

watch -n1 kubectl get pods -n gitpod

kubectl patch svc proxy -p '{"spec":{"externalIPs":["192.168.1.100"]}}' -n gitpod
```
Don't forget to replace it with your own ip address.

Finish :)