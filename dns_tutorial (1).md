# Tutoriel : DNS Recursor + Cluster Kubernetes V2

Résolution DNS complète pour VMs + Pods Kubernetes sur Scaleway.

---

## 1️⃣ VM PowerDNS Recursor

### Installation

```bash
ssh root@<IP_RECURSOR>

apt update && apt install -y pdns-recursor
```

### Configuration

```bash
cat > /etc/powerdns/recursor.conf << 'EOF'
local-address=172.16.4.4
local-port=53
allow-from=172.16.4.0/22,172.16.12.0/22,10.0.0.0/8

# Zones privées
forward-zones=toto-mint.private=172.16.4.2,internal=169.254.169.254

# Public + Scaleway
forward-zones-recurse=.=169.254.169.254;1.1.1.1;9.9.9.9

# Cache
max-cache-ttl=300
max-negative-ttl=60
loglevel=6
EOF
```

### Démarrage

```bash
systemctl restart pdns-recursor
systemctl status pdns-recursor
```

### Test

```bash
dig @172.16.4.4 google.com +short
# Doit retourner une IP
```

---

## 2️⃣ VM PowerDNS Authoritative

### Installation

```bash
ssh root@<IP_AUTH>

apt update && apt install -y pdns-backend-sqlite3 pdns-tools
```

### Configuration

```bash
cat > /etc/powerdns/pdns.conf << 'EOF'
launch=gsqlite3
gsqlite3-database=/var/lib/powerdns/pdns.sqlite3

local-address=172.16.4.2
local-port=53
EOF
```

### Initialisation DB

```bash
# Créer la base vierge
sqlite3 /var/lib/powerdns/pdns.sqlite3 << 'EOF'
CREATE TABLE domains (
  id INTEGER PRIMARY KEY,
  name TEXT UNIQUE,
  master TEXT,
  type TEXT
);

CREATE TABLE records (
  id INTEGER PRIMARY KEY,
  domain_id INTEGER,
  name TEXT,
  type TEXT,
  content TEXT,
  ttl INTEGER DEFAULT 3600
);
EOF

chown pdns:pdns /var/lib/powerdns/pdns.sqlite3
chmod 640 /var/lib/powerdns/pdns.sqlite3
```

### Démarrage

```bash
systemctl restart pdns
systemctl status pdns
```

### Créer la zone privée

```bash
sudo pdnsutil create-zone toto-mint.private ns1.toto-mint.private
```

### Ajouter des entrées DNS

```bash
sudo pdnsutil add-record toto-mint.private ns1 A 172.16.4.2
sudo pdnsutil add-record toto-mint.private webapp A 172.16.4.20
sudo pdnsutil add-record toto-mint.private db A 172.16.4.21
```

### Vérifier les entrées

```bash
sudo pdnsutil list-zone toto-mint.private
```

### Test

```bash
dig @172.16.4.2 db.toto-mint.private +short
# Doit retourner 172.16.4.21
```

---

## 3️⃣ VM Client

### Configuration manuelle

```bash
ssh root@<IP_CLIENT>

cat > /etc/netplan/90-dns.yaml << 'EOF'
network:
  version: 2
  ethernets:
    ens6:
      dhcp4: true
      nameservers:
        addresses: [172.16.4.4]
EOF

chmod 600 /etc/netplan/90-dns.yaml
netplan apply
```

### Configuration Cloud-init

Utiliser ce **cloud-init** lors de la création de l'instance Scaleway :

```yaml
#cloud-config
package_update: true

write_files:
  - path: /etc/netplan/90-dns.yaml
    owner: root:root
    permissions: '0600'
    content: |
      network:
        version: 2
        ethernets:
          ens6:
            dhcp4: true
            nameservers:
              addresses: [172.16.4.4]

runcmd:
  - netplan apply
```

### Test

```bash
nslookup db.toto-mint.private
nslookup google.com
```

---

## 4️⃣ Cluster Kubernetes Kapsule

### Configuration CoreDNS

```bash
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        
        forward . 172.16.4.4 169.254.169.254:53 {
            policy sequential
        }
        
        cache 300
        loop
        reload
        loadbalance
        import custom/*.override
    }
    
    internal:53 {
        forward . 169.254.169.254:53
        cache 300
    }
    
    toto-mint.private:53 {
        forward . 172.16.4.4
        cache 300
    }

    import custom/*.server
  empty: |
    # empty
EOF
```

### Redémarrage CoreDNS

```bash
kubectl -n kube-system rollout restart deployment/coredns
```

### Test

```bash
kubectl run dns-test --image=busybox:1.36 --restart=Never -it -- sh

/ # nslookup db.toto-mint.private
/ # nslookup google.com
/ # exit
```

---

## 5️⃣ Synchronisation automatique des Services Kubernetes

### Déployer le Controller de synchronisation DNS

Créer le Controller qui sync automatiquement les Services Kubernetes vers PowerDNS :

```bash
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dns-sync
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dns-sync
rules:
  - apiGroups: [""]
    resources: ["services", "pods"]
    verbs: ["get", "watch", "list"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dns-sync
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dns-sync
subjects:
  - kind: ServiceAccount
    name: dns-sync
    namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: dns-sync-script
  namespace: kube-system
data:
  sync.py: |
    #!/usr/bin/env python3
    import os
    import sqlite3
    import subprocess
    from kubernetes import client, config, watch
    from threading import Thread
    import time
    
    config.load_incluster_config()
    v1 = client.CoreV1Api()
    
    PDNS_DB = "/mnt/pdns-db/pdns.sqlite3"
    ZONE = "toto-mint.private"
    DOMAIN_ID = 1
    TTL = 300
    
    def sync_dns(name, ip_address, action="add"):
        """Sync à PowerDNS via la DB SQLite"""
        try:
            conn = sqlite3.connect(PDNS_DB)
            c = conn.cursor()
            
            fqdn = f"{name}.{ZONE}"
            
            if action == "add":
                c.execute("""
                    INSERT OR REPLACE INTO records 
                    (domain_id, name, type, content, ttl) 
                    VALUES (?, ?, 'A', ?, ?)
                """, (DOMAIN_ID, fqdn, ip_address, TTL))
                print(f"[DNS] Ajouté : {fqdn} -> {ip_address}")
            
            elif action == "delete":
                c.execute("""
                    DELETE FROM records 
                    WHERE domain_id = ? AND name = ? AND type = 'A'
                """, (DOMAIN_ID, fqdn))
                print(f"[DNS] Supprimé : {fqdn}")
            
            conn.commit()
            conn.close()
            
            # Recharger PowerDNS
            subprocess.run(["pdns_control", "reload"], capture_output=True)
            
        except Exception as e:
            print(f"[ERROR] {e}")
    
    def watch_services():
        """Monitore les Services Kubernetes"""
        w = watch.Watch()
        for event in w.stream(v1.list_service_for_all_namespaces):
            svc = event['object']
            name = svc.metadata.name
            namespace = svc.metadata.namespace
            
            if event['type'] == 'ADDED':
                if svc.spec.cluster_ip and svc.spec.cluster_ip != "None":
                    sync_dns(name, svc.spec.cluster_ip, "add")
            
            elif event['type'] == 'DELETED':
                sync_dns(name, "", "delete")
    
    def watch_pods():
        """Monitore les Pods annotés avec dns.internal"""
        w = watch.Watch()
        for event in w.stream(v1.list_pod_for_all_namespaces):
            pod = event['object']
            
            if pod.metadata.annotations and "dns.internal" in pod.metadata.annotations:
                name = pod.metadata.annotations["dns.internal"]
                
                if event['type'] == 'ADDED':
                    if pod.status.pod_ip:
                        sync_dns(name, pod.status.pod_ip, "add")
                
                elif event['type'] == 'DELETED':
                    sync_dns(name, "", "delete")
    
    if __name__ == '__main__':
        print("[DNS-SYNC] Démarrage du contrôleur...")
        
        t1 = Thread(target=watch_services, daemon=True)
        t2 = Thread(target=watch_pods, daemon=True)
        
        t1.start()
        t2.start()
        
        try:
            while True:
                time.sleep(1)
        except KeyboardInterrupt:
            print("\n[DNS-SYNC] Arrêt...")
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dns-sync
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dns-sync
  template:
    metadata:
      labels:
        app: dns-sync
    spec:
      serviceAccountName: dns-sync
      containers:
        - name: dns-sync
          image: python:3.11-slim
          command: ["/bin/bash", "-c"]
          args:
            - |
              pip install kubernetes -q
              python3 /scripts/sync.py
          volumeMounts:
            - name: script
              mountPath: /scripts
            - name: pdns-db
              mountPath: /mnt/pdns-db
          env:
            - name: PDNS_HOST
              value: "172.16.4.2"
      volumes:
        - name: script
          configMap:
            name: dns-sync-script
            defaultMode: 0755
        - name: pdns-db
          nfs:
            server: 172.16.4.2
            path: /var/lib/powerdns
EOF
```

### Utilisation

**Pour synchroniser un Service :**
```bash
kubectl expose deployment my-app --name=my-app --port=80
# Automatiquement ajouté à DNS : my-app.toto-mint.private
```

**Pour synchroniser un Pod :**
```bash
kubectl run my-pod --image=nginx --annotations="dns.internal=my-pod"
# Automatiquement ajouté à DNS : my-pod.toto-mint.private
```

### Vérifier la synchronisation

```bash
# Depuis le cluster
kubectl run dns-test --image=busybox:1.36 --restart=Never -it -- sh
/ # nslookup my-app.toto-mint.private
/ # nslookup my-pod.toto-mint.private
```

---

## ✅ Résumé de la stack

| Composant | IP | Rôle |
|-----------|----|----|
| **Recursor** | 172.16.4.4:53 | Proxy DNS central |
| **Authoritative** | 172.16.4.2:53 | Zone `toto-mint.private` |
| **CoreDNS** | 10.32.0.10:53 | DNS cluster K8s |
| **DNS-Sync** | K8s Pod | Synchronisation automatique |
| **Scaleway Metadata** | 169.254.169.254:53 | DNS `.internal` |

---

## 📋 Checklist finale

- [x] Recursor accessible sur 172.16.4.4:53
- [x] Authoritative accessible sur 172.16.4.2:53
- [x] Zone `toto-mint.private` créée
- [x] Entrées DNS manuelles ajoutées
- [x] CoreDNS forward vers Recursor
- [x] Pods résolvent `*.toto-mint.private` ✅
- [x] Pods résolvent `*.internal` ✅
- [x] Pods résolvent DNS public ✅
- [x] Services Kubernetes syncés automatiquement ✅
- [x] Pods annotés syncés automatiquement ✅