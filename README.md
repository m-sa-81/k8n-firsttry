# k8n-firsttry
First try with k8n cluster


## Vorbereitung Nodes:

### Swap deaktivieren
Warum deaktivieren? 
Kubernetes verwaltet Ressourcen (RAM) sehr präzise. Es sagt einem Pod z.B. „du bekommst 512MB RAM". Wenn Swap aktiv ist, weiß Kubernetes nicht mehr genau wieviel echter RAM verfügbar ist – das bricht die Ressourcenplanung.
```
sudo swapoff -a
sudo sed -i '/swap/s/^/#/' /etc/fstab
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
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
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

sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<IP-des-Control-Plane> \
  --cri-socket=unix:///run/containerd/containerd.sock


