# k8n-firsttry
First try with k8n cluster

Mögliche Optionen:  
* [Minikube](https://minikube.sigs.k8s.io/docs/)
* [KIND - Kubernetes in Docker](https://kind.sigs.k8s.io/)
* [containerd](https://containerd.io/)


## Vorbereitung Nodes:

### Swap deaktivieren
Warum deaktivieren? 
Kubernetes verwaltet Ressourcen (RAM) sehr präzise. Es sagt einem Pod z.B. „du bekommst 512MB RAM". Wenn Swap aktiv ist, weiß Kubernetes nicht mehr genau wieviel echter RAM verfügbar ist – das bricht die Ressourcenplanung.
```
sudo swapoff -a
sudo sed -i '/swap/s/^/#/' /etc/fstab

cat /proc/swaps
Filename                                Type            Size            Used            Priority

```


### Kernel-Module laden
**overlay**: Container-Dateisystem (mehrere Schichten übereinander – das macht Docker/containerd)  
**br_netfilter**: Netzwerkfilterung über Bridges – damit iptables auch den Container-Traffic sieht  

```
sudo modprobe overlay
sudo modprobe br_netfilter
```

### Module dauerhaft aktivieren
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

### Netzwerk-Einstellungen
**bridge-nf-call-iptables = 1**: Traffic der über eine Netzwerk-Bridge läuft (Container!) wird durch iptables gefiltert  
**bridge-nf-call-ip6tables = 1**: Dasselbe für IPv6  
**ip_forward = 1**: Der Server darf Pakete weiterleiten (von Container A zu Container B, oder nach außen)  

```
sudo cat <<EOF | sudo tee /etc/sysctl.d/99-k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

## Containerd installieren:
```
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
```

### Standardkonfiguration erstellen
```
sudo mkdir -p /etc/containerd
sudo  containerd config default | sudo tee /etc/containerd/config.toml
```

#### SystemdCgroup aktivieren (wichtig!)
```
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/'  /etc/containerd/config.toml
```

```
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## kubeadm, kubelet, kubectl installieren :

```
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo mkdir -p -m 755 /etc/apt/keyrings

# Kubernetes GPG-Key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg

# Repository hinzufügen
echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Installieren
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Version einfrieren (kein automatisches Update)
Kubernetes hat eine sehr strenge Versionskompatibilitätsregel: der Control Plane (API-Server) und die Nodes (kubelet) dürfen maximal eine Minor-Version auseinanderliegen.
Wenn also apt upgrade einfach kubelet von 1.29 auf 1.31 zieht, während der API-Server noch auf 1.29 läuft, ist der Cluster kaputt — der Node meldet sich nicht mehr an.

sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable kubelet
```


## Cluster initialisieren (Control Node)
```
#ggfs. Port 6443 öffnen

sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<IP-des-Control-Plane> \
  --cri-socket=unix:///run/containerd/containerd.sock
```

## kubectl einrichten (Control Node)
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


kubectl get nodes
# NAME      STATUS     ROLES           AGE
# XYZ       NotReady   control-plane   1m
```


## Netzwerk-Plugins in Kubernetes
Das Plugin läuft als Pod auf jedem Node und kümmert sich um:  
* IP-Vergabe an Pods  
* Routing zwischen Nodes  
* Netzwerkregeln (NetworkPolicies)

Die wichtigsten Plugins  
Flannel – simpel und bewährt    
* Einfachstes Plugin, leicht zu verstehen
* Baut ein flaches Netzwerk über alle Nodes (Overlay-Netzwerk)
* Keine NetworkPolicies unterstützt
* Gut für Testumgebungen
* Standard-CIDR: 10.244.0.0/16

Calico – mächtig und weit verbreitet  
* Unterstützt NetworkPolicies (Firewall-Regeln zwischen Pods)
* Kein Overlay nötig – nutzt echtes IP-Routing
* Auch in Produktion sehr beliebt
* Standard-CIDR: 192.168.0.0/16

Cilium – modern und sehr leistungsfähig  
* Nutzt eBPF (sehr effizienter Linux-Kernel-Mechanismus)
* Sehr gutes Monitoring und Observability
* Komplexer aber zukunftssicher
* Wird immer populärer

```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

#Wenn CIDR ändern:
# Calico-Manifest herunterladen
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# CIDR anpassen
sed -i 's|192.168.0.0/16|10.244.0.0/16|' calico.yaml

# Installieren
kubectl apply -f calico.yaml

kubectl get nodes
NAME          STATUS   ROLES           AGE   VERSION
XYZ           Ready    control-plane   36m   v1.29.15
```

## Worker nodes hinzufügen
```
sudo kubeadm join 192.168.0.51:6443 --token ...         --discovery-token-ca-cert-hash sha256:b......
```

