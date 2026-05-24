# k8n-firsttry
First try with k8n cluster


## Vorbereitung Nodes:

### Swap deaktivieren
Warum deaktivieren? 
Kubernetes verwaltet Ressourcen (RAM) sehr präzise. Es sagt einem Pod z.B. „du bekommst 512MB RAM". Wenn Swap aktiv ist, weiß Kubernetes nicht mehr genau wieviel echter RAM verfügbar ist – das bricht die Ressourcenplanung.
```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
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
