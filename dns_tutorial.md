# Tutoriel : DNS Recursor + Cluster Kubernetes

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

apt update && apt install -y pdns-backend-sqlite3
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

INSERT INTO domains (name, type) VALUES ('toto-mint.private', 'Native');

INSERT INTO records (domain_id, name, type, content, ttl) VALUES
  (1, 'db.toto-mint.private', 'A', '172.16.4.21', 300),
  (1, 'webapp.toto-mint.private', 'A', '172.16.4.20', 300);
EOF

chown pdns:pdns /var/lib/powerdns/pdns.sqlite3
chmod 640 /var/lib/powerdns/pdns.sqlite3
```

### Démarrage

```bash
systemctl restart pdns
systemctl status pdns
```

### Test

```bash
dig @172.16.4.2 db.toto-mint.private +short
# Doit retourner 172.16.4.21
```

---

## 3️⃣ VM Client (optionnelle)

### Configuration DNS

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

netplan apply
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
# Récupérer le kubeconfig Scaleway
# Puis :

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
# Doit retourner 172.16.4.21

/ # nslookup google.com
# Doit retourner une IP

/ # nslookup scw-client-dns.rpvn-new-minint.internal
# Doit retourner l'IP de la VM Scaleway

/ # exit
```

---

## ✅ Résumé de la stack

| Composant | IP | Rôle |
|-----------|----|----|
| **Recursor** | 172.16.4.4:53 | Proxy DNS central |
| **Authoritative** | 172.16.4.2:53 | Zone `toto-mint.private` |
| **CoreDNS** | 10.32.0.10:53 | DNS cluster K8s |
| **Scaleway Metadata** | 169.254.169.254:53 | DNS `.internal` |

---

## 🔧 Ajouter une nouvelle entrée DNS

### Via VM Scaleway

```bash
sqlite3 /var/lib/powerdns/pdns.sqlite3 \
  "INSERT INTO records (domain_id, name, type, content, ttl) VALUES (1, 'new-host.toto-mint.private', 'A', '172.16.4.30', 300);"

pdns_control reload
```

### Depuis Kubernetes

```bash
kubectl run dns-test --image=busybox:1.36 --restart=Never -it -- sh
/ # nslookup new-host.toto-mint.private
```

---

## 📋 Checklist finale

- [x] Recursor accessible sur 172.16.4.4:53
- [x] Authoritative accessible sur 172.16.4.2:53
- [x] CoreDNS forward vers Recursor
- [x] Pods résolvent `*.toto-mint.private` ✅
- [x] Pods résolvent `*.internal` ✅
- [x] Pods résolvent DNS public ✅