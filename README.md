# Cluster Kubernetes WSL2

## Prérequis

* Une distribution linux avec cgroup v2 ( obligatoire )
* 2 VCpu - 4Go de RAM
* VSCode / Codium ou un éditeur de texte
* helm, kubectl, kind, clusterctl, docker

## Préparatifs

( Le tuto est réalisé avec ubuntu mais pour l'être avec une autre distribution, debian, fedora, alma, rocky etc ... )

* il faut de manière impérative utiliser les cgroup v2.

### cgroupv2

C'est une fonctionnalité qui permet la compartimentation des différentes appels au noyau de la machine

* cpu/ram/disque ( notamment les limites et la segmentation des process dans des lsns )
* droits d'accès
* utilisation du réseau

La version 2 permet une gestion plus fine et avancée, il est recommandé en tout temps désormais pour kubernetes et docker, notamment pour la gestion cpu et ram

Lancez la commande suivante dans votre wsl :

```bash
grep cgroup /proc/filesystems
```

si cgroupv2 est activé vous aurez le retour suivant :

```bash
grep cgroup /proc/filesystems
nodev cgroup
nodev cgroup2
```

dans le cas contraire :

```bash
nodev cgroup
```

pour l'activer il suffit de modifier la configuration de la machine wsl

```cmd
notepad %UserProfile%\.wslconfig

```

ajouter les lignes suivante :

```bash
[wsl2]
kernelCommandLine = cgroup_no_v1=all
```

fermez et redémarrer la machine wsl :

```cmd
wsl --shutdown
```

---

Pour monter cgroupv2 il faut modifier le fstab :

```bash
sudo echo -e "cgroup2 /sys/fs/cgroup cgroup2 rw,nosuid,nodev,noexec,relatime,nsdelegate 0 0" | sudo tee -a /etc/fstab
sudo mount -a
grep cgroup /proc/filesystems
```

Le cgroup2 doit apparaitre une fois cette étape passée et docker pourra se lancer.

## installation des binaires requis

* Docker :

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
 "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
 "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
 sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

* Kubectl :

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
### Autocomplete pour bash
sudo apt install bash-completion
echo 'source <(kubectl completion bash)' >>~/.bashrc
source ~/.bashrc
rm kubectl
```

* Kind :

```bash
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

* Helm :

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
## Autocomplete pour bash
echo 'source <(helm completion bash)' >>~/.bashrc
source ~/.bashrc
```

* Clusterctl :

```bash
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.5.1/clusterctl-linux-amd64 -o clusterctl
sudo install -o root -g root -m 0755 clusterctl /usr/local/bin/clusterctl
clusterctl version
## Autocomplete pour bash
echo 'source <(clusterctl completion bash)' >>~/.bashrc
source ~/.bashrc
rm clusterctl
```
## Lancer et intialiser CAPI

```bash
# Enable the experimental Cluster topology feature.
kind create cluster --config kind-cluster-with-extramounts.yaml
export CLUSTER_TOPOLOGY=true
clusterctl init --infrastructure docker --addon helm
```
Attendre que tout soit en running

## Cluster CAPI avec Cilium
```bash
kubectl apply -f capi-docker-helm.yaml
clusterctl get kubeconfig capi-docker > capi-docker.kubeconfig
```
utiliser kubecm pour merge le kubeconfig ou directement le faire depuis vscode
```bash
brew install kubecm
kubecm add --file capi-docker-helm.yaml
kubectx kind-capi-docker
```
et voila

## Documents de référence
* https://cluster-api.sigs.k8s.io/introduction
* https://cluster-api.sigs.k8s.io/user/quick-start
* https://github.com/kubernetes-sigs/cluster-api-addon-provider-helm
