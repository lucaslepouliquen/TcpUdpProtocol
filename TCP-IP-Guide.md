# TCP/IP pour l'Ingénieur Infrastructure

> Guide théorique et pratique pour DevOps/Kubernetes

[![Made for DevOps](https://img.shields.io/badge/Made%20for-DevOps-blue)](https://github.com)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Ready-326CE5?logo=kubernetes)](https://kubernetes.io)
[![Azure](https://img.shields.io/badge/Azure-AKS-0078D4?logo=microsoft-azure)](https://azure.microsoft.com)

---

## Table des matières

- [Introduction](#-introduction)
- [PARTIE 1 - Fondamentaux Théoriques](#partie-1---fondamentaux-théoriques)
- [1. Architecture en Couches](#1-architecture-en-couches-modèle-ositcp-ip)
- [2. Adressage IP et Subnetting](#2-adressage-ip-et-subnetting)
- [3. TCP vs UDP](#3-tcp-vs-udp--protocoles-de-transport)
- [4. Routage IP](#4-routage-ip-et-tables-de-routage)
- [5. MTU et Fragmentation](#5-mtu-maximum-transmission-unit-et-fragmentation)
- [6. NAT](#6-nat-network-address-translation)
- [7. ARP](#7-arp-address-resolution-protocol)
- [8. DNS](#8-dns-domain-name-system)
- [9. Timeouts et Keep-Alive](#9-timeouts-keep-alive-et-connection-pooling)
- [PARTIE 2 - Applications Pratiques](#partie-2---applications-pratiques-contexte-devopskubernetes)
- [10. TCP/IP dans Kubernetes](#10-tcpip-dans-kubernetes--architecture-réseau)
- [11. Services Kubernetes](#11-services-kubernetes-et-kube-proxy)
- [12. Application Gateway et Ingress](#12-azure-application-gateway-et-ingress)
- [13. Network Policies](#13-network-policies--firewall-l3l4-dans-kubernetes)
- [14. Troubleshooting](#14-troubleshooting-réseau--outils-et-méthodologie)
- [15. Cas Pratiques](#15-cas-pratiques--contexte-promodaks)
- [16. Best Practices](#16-best-practices-tcpip-pour-infrastructure-kubernetes)
- [Ressources](#-ressources-recommandées)

---

## Introduction

Ce guide couvre les aspects fondamentaux du protocole TCP/IP pour un ingénieur infrastructure, avec des exemples concrets d'application dans un contexte DevOps et Kubernetes.

**Pourquoi ce guide ?**
- Comprendre les fondamentaux théoriques
- Appliquer les concepts à Kubernetes/Azure
- Débugger efficacement les problèmes réseau
- Optimiser les performances réseau en production

---

# PARTIE 1 - Fondamentaux Théoriques

## 1. Architecture en Couches (Modèle OSI/TCP-IP)

Le modèle TCP/IP est organisé en **4 couches principales**, chacune ayant des responsabilités spécifiques :

```
┌─────────────────────────────────────┐
│ Couche 4 - APPLICATION              │ HTTP, DNS, SSH, FTP
│ (Interface utilisateur)             │
├─────────────────────────────────────┤
│ Couche 3 - TRANSPORT                │ TCP (fiable) / UDP (rapide)
│ (Segmentation, contrôle de flux)    │
├─────────────────────────────────────┤
│ Couche 2 - INTERNET (IP)            │ IPv4, IPv6, ICMP, ARP
│ (Routage entre réseaux)             │
├─────────────────────────────────────┤
│ Couche 1 - LIAISON/PHYSIQUE         │ Ethernet, WiFi, MAC
│ (Transmission physique)             │
└─────────────────────────────────────┘
```

### Encapsulation

Chaque couche encapsule les données de la couche supérieure en ajoutant son propre en-tête :

```
Application → Segment (TCP/UDP) → Paquet (IP) → Trame (Ethernet)
```

---

## 2. Adressage IP et Subnetting

### 2.1 IPv4

- **Format** : 32 bits (4 octets), notation décimale pointée
- **Exemple** : `192.168.1.10`
- **Classes historiques** : A (/8), B (/16), C (/24)
- **Aujourd'hui** : CIDR (Classless Inter-Domain Routing)

**Adresses privées (RFC 1918)** :
```
10.0.0.0/8 → 10.0.0.0 - 10.255.255.255
172.16.0.0/12 → 172.16.0.0 - 172.31.255.255
192.168.0.0/16 → 192.168.0.0 - 192.168.255.255
```

### 2.2 Notation CIDR et Calcul

**Formule rapide** : `2^(32-masque) - 2` (network + broadcast)

| CIDR | Masque | IPs totales | IPs utilisables |
|------|--------|-------------|-----------------|
| /24 | 255.255.255.0 | 256 | 254 |
| /20 | 255.255.240.0 | 4,096 | 4,094 |
| /16 | 255.255.0.0 | 65,536 | 65,534 |
| /12 | 255.240.0.0 | 1,048,576 | 1,048,574 |

### Exemple Subnetting

Diviser `10.0.0.0/16` en sous-réseaux `/24` :

```
10.0.0.0/24 → 10.0.0.1 - 10.0.0.254
10.0.1.0/24 → 10.0.1.1 - 10.0.1.254
10.0.2.0/24 → 10.0.2.1 - 10.0.2.254
...
10.0.255.0/24 → 10.0.255.1 - 10.0.255.254
```

---

## 3. TCP vs UDP : Protocoles de Transport

### 3.1 TCP (Transmission Control Protocol)

**Caractéristiques** :
- Orienté connexion
- Fiable (garantit livraison et ordre)
- Contrôle de flux et de congestion
- Overhead plus élevé

#### Three-Way Handshake

```
Client Server
| |
|-------- SYN --------> | (1) Demande connexion
| |
|<----- SYN-ACK ------- | (2) Acceptation
| |
|-------- ACK --------> | (3) Confirmation
| |
|=== ESTABLISHED ========>| Connexion établie
```

#### États TCP Importants

| État | Description |
|------|-------------|
| `LISTEN` | En attente de connexion entrante |
| `SYN_SENT` | SYN envoyé, attente SYN-ACK |
| `ESTABLISHED` | Connexion active |
| `TIME_WAIT` | Connexion fermée, attente sécurité |
| `CLOSE_WAIT` | En attente de fermeture locale |
| `FIN_WAIT` | FIN envoyé, attente confirmation |

#### Flags TCP

| Flag | Signification |
|------|---------------|
| `SYN` | Synchronisation (établir connexion) |
| `ACK` | Acknowledgement (confirmer réception) |
| `FIN` | Finish (terminer connexion) |
| `RST` | Reset (fermeture abrupte) |
| `PSH` | Push (envoyer immédiatement) |
| `URG` | Urgent (données prioritaires) |

#### Retransmission TCP

- TCP retransmet automatiquement les paquets perdus
- **RTO** (Retransmission TimeOut) : ajusté dynamiquement
- **Fenêtre TCP** : contrôle de flux via fenêtre glissante

### 3.2 UDP (User Datagram Protocol)

**Caractéristiques** :
- Sans connexion
- Latence minimale
- Overhead minimal
- Pas de garantie de livraison
- Pas de contrôle de flux

**Use cases** :
- Gaming (latence critique)
- VoIP (temps réel)
- Streaming vidéo
- DNS (requêtes courtes)
- Métriques (tolérance perte)

---

## 4. Routage IP et Tables de Routage

### Table de Routage

Dictionnaire indiquant où envoyer les paquets selon leur IP de destination.

**Structure d'une entrée** :
```
Destination Gateway Netmask Interface
10.0.1.0 0.0.0.0 255.255.255.0 eth0
192.168.1.0 0.0.0.0 255.255.255.0 eth1
0.0.0.0 10.0.1.1 0.0.0.0 eth0 ← Route par défaut
```

### Commandes Utiles

```bash
# Voir la table de routage
ip route show
route -n

# Vérifier quelle route est utilisée
ip route get 8.8.8.8

# Ajouter une route statique
ip route add 192.168.2.0/24 via 10.0.1.1
```

### Protocoles de Routage

| Protocole | Type | Usage |
|-----------|------|-------|
| **RIP** | Distance Vector | Petits réseaux, max 15 hops |
| **OSPF** | Link State | Réseaux entreprise, scalable |
| **BGP** | Path Vector | Internet, routage inter-AS |

---

## 5. MTU (Maximum Transmission Unit) et Fragmentation

### MTU Standard

- **Ethernet standard** : 1500 bytes
- **Overlay networks (VXLAN)** : 1450 bytes (overhead 50 bytes)
- **Jumbo frames** : 9000 bytes (data centers)

### Impact Fragmentation

```
Paquet 2000 bytes avec MTU 1500
↓
Fragment 1: 1500 bytes
Fragment 2: 500 bytes
↓
Si un fragment perdu → TOUT retransmis
```

### Path MTU Discovery

Mécanisme pour déterminer le MTU minimum sur le chemin réseau :

1. Envoyer paquet avec flag **"Don't Fragment"**
2. Si MTU dépassé → routeur retourne **ICMP "Fragmentation Needed"**
3. Source ajuste la taille et réessaye

### Tester MTU

```bash
# Test avec ping (1472 + 28 headers = 1500)
ping -M do -s 1472 8.8.8.8

# Si ça passe, tester 1473
ping -M do -s 1473 8.8.8.8
# Erreur "Frag needed" → MTU < 1501
```

---

## 6. NAT (Network Address Translation)

### Types de NAT

#### SNAT (Source NAT)

Modifie l'IP source sortante. Utilise **PAT** (Port Address Translation).

```
Réseau privé NAT Internet
10.0.1.5:54321 → 203.0.113.5:12345 → 8.8.8.8:53
```

**Limite** : ~65k connexions simultanées par IP publique (limitation ports)

#### DNAT (Destination NAT)

Modifie l'IP destination entrante (port forwarding).

```
Internet NAT Réseau privé
8.8.8.8 → 203.0.113.5:80 → 10.0.1.10:8080
```

### Problèmes NAT

- **Exhaustion ports SNAT** : Plus de ports disponibles
- **Latence accrue** : Traitement supplémentaire
- **Debugging complexe** : Perte traçabilité IP source
- **Incompatibilité protocols** : FTP, SIP, IPSec

---

## 7. ARP (Address Resolution Protocol)

### Fonction

Résout une **adresse IP** en **adresse MAC** sur un réseau local (Layer 2).

### Fonctionnement

```
1. Machine A veut communiquer avec 192.168.1.10
2. A broadcast : "Qui a 192.168.1.10 ?" (ARP Request)
3. Machine avec 192.168.1.10 répond : "C'est moi, MAC: aa:bb:cc:dd:ee:ff"
4. A stocke l'info dans son cache ARP
```

### Commandes

```bash
# Voir le cache ARP
arp -n
ip neigh show

# Vider le cache ARP
ip neigh flush all

# ARP ping
arping -I eth0 192.168.1.10
```

---

## 8. DNS (Domain Name System)

### Hiérarchie DNS

```
. (root)
|
┌───────────┼───────────┐
com org fr
| | |
google wikipedia gouv
|
www
```

### Types d'Enregistrements

| Type | Description | Exemple |
|------|-------------|---------|
| **A** | IPv4 | `example.com → 93.184.216.34` |
| **AAAA** | IPv6 | `example.com → 2606:2800:220:1:...` |
| **CNAME** | Alias | `www.example.com → example.com` |
| **MX** | Mail exchange | `example.com → mail.example.com` |
| **NS** | Nameserver | `example.com → ns1.example.com` |
| **TXT** | Texte arbitraire | SPF, DKIM, vérifications |

### Requête DNS

```bash
# Résolution simple
nslookup google.com

# Résolution détaillée
dig google.com

# Spécifier serveur DNS
dig @8.8.8.8 google.com

# Requête inverse (IP → nom)
dig -x 93.184.216.34
```

### Cache DNS et TTL

**TTL (Time To Live)** : durée de mise en cache (en secondes)

```
example.com. 300 IN A 93.184.216.34
↑
TTL = 5 minutes
```

---

## 9. Timeouts, Keep-Alive et Connection Pooling

### Types de Timeouts

| Timeout | Description |
|---------|-------------|
| **Connection timeout** | Durée max pour établir connexion (handshake TCP) |
| **Read timeout** | Durée max pour recevoir données |
| **Write timeout** | Durée max pour envoyer données |
| **Idle timeout** | Durée max d'inactivité avant fermeture |

### TCP Keep-Alive

Mécanisme pour maintenir connexion active et détecter connexions mortes.

**Paramètres Linux** :
```bash
# Délai avant premier keep-alive (défaut: 7200s = 2h)
sysctl net.ipv4.tcp_keepalive_time

# Intervalle entre keep-alives (défaut: 75s)
sysctl net.ipv4.tcp_keepalive_intvl

# Nombre de keep-alives avant abandon (défaut: 9)
sysctl net.ipv4.tcp_keepalive_probes
```

### Connection Pooling

Réutilise des connexions TCP établies au lieu d'en créer de nouvelles.

**Avantages** :
- Réduit latence (pas de handshake)
- Réduit charge serveur
- Économise ressources (ports, file descriptors)

---

# PARTIE 2 - Applications Pratiques (Contexte DevOps/Kubernetes)

## 10. TCP/IP dans Kubernetes : Architecture Réseau

### 10.1 Modèle Réseau Kubernetes

**Principes fondamentaux** :
1. Chaque **Pod** obtient une IP unique
2. Les Pods communiquent sans NAT entre eux
3. Communication directe peu importe le nœud

### Trois Réseaux Distincts

```
┌──────────────────────────────────────────┐
│ POD NETWORK (Pod CIDR) │
│ Exemple: 10.244.0.0/16 │
│ → IPs des Pods │
├──────────────────────────────────────────┤
│ SERVICE NETWORK (Service CIDR) │
│ Exemple: 10.96.0.0/12 │
│ → IPs virtuelles des Services │
├──────────────────────────────────────────┤
│ NODE NETWORK │
│ Exemple: 172.16.0.0/24 │
│ → IPs des nœuds Kubernetes │
└──────────────────────────────────────────┘
```

### 10.2 CNI (Container Network Interface)

**Rôle** : Plugin qui configure le réseau pour chaque Pod.

#### Workflow CNI

```
1. Kubelet crée Pod
2. Kubelet appelle CNI plugin
3. CNI crée network namespace
4. CNI alloue IP (via IPAM)
5. CNI configure interfaces (veth pair)
6. CNI configure routes
7. Pod prêt avec réseau configuré
```

### 10.3 Overlay vs Underlay Networks

#### Overlay Networks (Encapsulation)

**Technique** : Encapsulation paquets Pod dans UDP/IP (VXLAN)

```
Original Pod Packet
↓
[Outer IP Header][UDP Header][VXLAN Header][Original Packet]
↓
Tunnel via UDP port 8472
```

| Avantages | Inconvénients |
|-----------|---------------|
| Isolation L2 | Overhead encapsulation |
| Pas de config routeurs | MTU réduit (1450 vs 1500) |
| Portable multi-cloud | Performance légèrement réduite |

**Exemples CNI** : Flannel, Weave, Canal (Calico + Flannel)

#### Underlay Networks (Routage L3)

**Technique** : Routage natif via BGP

```
Pod CIDR routes propagées aux routeurs physiques via BGP
Pas d'encapsulation → Routage IP natif
```

| Avantages | Inconvénients |
|-----------|---------------|
| Pas d'overhead | Config routeurs nécessaire |
| Performance maximale | Plus complexe |
| MTU standard (1500) | Dépendance infrastructure |

**Exemples CNI** : Calico (mode BGP), Cilium

### 10.4 Azure CNI (Contexte AKS)

**Architecture** :
- Chaque Pod = IP du subnet Azure VNet directement
- Pas d'overlay (sauf Azure CNI Overlay mode)
- Routage via Azure SDN

**Avantages** :
- Performance maximale
- Intégration native Azure (NSG, UDR, Firewall)
- Pods accessibles depuis VNet

**Inconvénient majeur** :
- **Consommation IP élevée** : 1 Pod = 1 IP subnet
- Risque d'épuisement IP si mal dimensionné

#### Calcul IP Azure CNI

Pour un cluster avec :
- 5 nœuds
- Max 110 Pods par nœud (défaut AKS)

**IPs nécessaires** :
```
5 nœuds × 110 Pods = 550 IPs
+ Marge pour scaling = 800-1000 IPs minimum
→ Subnet /22 (1024 IPs) recommandé
```

---

## 11. Services Kubernetes et kube-proxy

### Types de Services

| Type | Description | Use Case |
|------|-------------|----------|
| **ClusterIP** | IP interne cluster | Communication interne |
| **NodePort** | Expose sur port nœud (30000-32767) | Dev/test externe |
| **LoadBalancer** | Crée LB cloud (Azure LB, AWS ELB) | Production externe |
| **ExternalName** | Map à nom DNS externe | Intégration services externes |

### kube-proxy : Le Chef d'Orchestre

**Rôle** : Implémente les règles de forwarding pour les Services

#### Modes kube-proxy

##### iptables (défaut)

```bash
# Règles iptables créées automatiquement
iptables -t nat -L -n | grep <service-name>

# Exemple de règle
-A KUBE-SERVICES -d 10.96.0.10/32 -p tcp -m tcp --dport 80 \
-j KUBE-SVC-XXXXX
-A KUBE-SVC-XXXXX -m statistic --mode random --probability 0.5 \
-j KUBE-SEP-POD1
-A KUBE-SVC-XXXXX -j KUBE-SEP-POD2
```

**Caractéristiques** :
- Load balancing aléatoire
- Scalabilité limitée (>1000 services)
- Debugging difficile

##### IPVS (IP Virtual Server)

```bash
# Voir les services IPVS
ipvsadm -Ln

# Exemple output
TCP 10.96.0.10:80 rr
-> 10.244.1.5:8080 Masq 1 0 0
-> 10.244.2.8:8080 Masq 1 0 0
```

**Caractéristiques** :
- Meilleure performance (grands clusters)
- Algorithmes LB avancés :
- `rr` : Round Robin
- `lc` : Least Connection
- `wrr` : Weighted Round Robin
- `sh` : Source Hashing

### Flux Connexion Pod → Service

```
1. DNS Resolution
nginx.default.svc.cluster.local → ClusterIP (10.96.0.10)

2. Paquet TCP
SRC: 10.244.1.5:54321
DST: 10.96.0.10:80

3. kube-proxy (iptables/IPVS)
DNAT: 10.96.0.10:80 → 10.244.2.8:8080 (Pod backend)

4. Routage CNI
Paquet acheminé vers Pod cible (même nœud ou autre)

5. Réponse
Pod répond, SNAT inverse appliqué
```

---

## 12. Azure Application Gateway et Ingress

### Architecture

```
Internet
↓
Azure Public IP
↓
Application Gateway (L7)
↓
Backend Pool (Pod IPs)
↓
AKS Pods
```

### AGIC (Application Gateway Ingress Controller)

Controller Kubernetes qui synchronise les ressources **Ingress** k8s avec la config App Gateway.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
name: example-ingress
annotations:
kubernetes.io/ingress.class: azure/application-gateway
spec:
rules:
- host: example.com
http:
paths:
- path: /
pathType: Prefix
backend:
service:
name: example-service
port:
number: 80
```

### Flux Réseau Détaillé

```
1. Client → App Gateway IP publique
TCP handshake + TLS termination

2. App Gateway → Routing L7
URL path matching, header inspection

3. App Gateway → DNAT vers Pod IP
Backend pool selection (load balancing)

4. Paquet acheminé via Azure VNet
Azure CNI route vers Pod

5. Pod répond
SNAT (IP source = App Gateway subnet IP)
```

### Points d'Attention TCP/IP

#### MTU
```bash
# Vérifier MTU cohérent
ip link show | grep mtu

# Tester fragmentation
ping -M do -s 1472 <pod-ip>
```

#### SNAT Exhaustion

**Limite** : ~64k connexions simultanées par IP backend pool

**Solutions** :
- Augmenter nombre d'instances App Gateway
- Configurer **connection draining**
- Activer **HTTP keep-alive**

```yaml
# Dans le Pod
server {
keepalive_timeout 65;
keepalive_requests 100;
}
```

#### Health Probes

App Gateway fait des health checks vers Pods.

**Configuration** :
- Protocol : HTTP ou HTTPS
- Path : `/health` ou `/ready`
- Interval : 30s (défaut)
- Timeout : 30s (défaut)
- Unhealthy threshold : 3 (défaut)

```bash
# Vérifier health depuis App Gateway subnet
curl -v http://<pod-ip>:8080/health
```

---

## 13. Network Policies : Firewall L3/L4 dans Kubernetes

### Principe

**Deny-by-default** dès qu'une policy sélectionne un Pod.

### Exemple : Isoler un namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: deny-all
namespace: production
spec:
podSelector: {} # Sélectionne tous les pods
policyTypes:
- Ingress
- Egress
# Pas de règles = tout bloqué
```

### Exemple : Autoriser trafic spécifique

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: allow-frontend-to-backend
namespace: production
spec:
podSelector:
matchLabels:
app: backend
policyTypes:
- Ingress
ingress:
- from:
- podSelector:
matchLabels:
app: frontend
ports:
- protocol: TCP
port: 8080
```

### Exemple : Autoriser egress vers Internet

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: allow-external-api
spec:
podSelector:
matchLabels:
app: api-client
policyTypes:
- Egress
egress:
- to:
- ipBlock:
cidr: 0.0.0.0/0
except:
- 169.254.169.254/32 # Bloquer metadata service
ports:
- protocol: TCP
port: 443
```

### Implémentation CNI

**Azure CNI standard** : Ne supporte PAS Network Policies

**Solutions** :
- **Azure Network Policy Manager**
- **Calico** (installer en addon)
- **Cilium**

```bash
# Installer Calico sur AKS
az aks create \
--network-plugin azure \
--network-policy calico
```

---

## 14. Troubleshooting Réseau : Outils et Méthodologie

### 14.1 Outils Essentiels

#### tcpdump

Capture paquets réseau au niveau interface.

```bash
# Capturer tout le trafic sur eth0
tcpdump -i eth0 -n

# Filtrer par port
tcpdump -i eth0 port 80

# Filtrer par host
tcpdump -i eth0 host 10.244.1.5

# Voir flags TCP
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'

# Sauvegarder pour analyse Wireshark
tcpdump -i eth0 -w capture.pcap
```

**Filtres utiles** :
```bash
# Voir uniquement SYN packets
tcpdump 'tcp[tcpflags] & tcp-syn != 0'

# Voir RST (connexion refusée)
tcpdump 'tcp[tcpflags] & tcp-rst != 0'

# Exclure SSH (pour pas polluer)
tcpdump 'not port 22'
```

#### ss (socket statistics)

```bash
# Toutes connexions TCP/UDP
ss -tunap

# Connexions établies
ss -tun state established

# Connexions en TIME_WAIT
ss -tun state time-wait

# Statistiques globales
ss -s

# Écoute sur quel port
ss -tlnp | grep :8080
```

#### netstat (legacy)

```bash
# Connexions actives
netstat -tunap

# Table de routage
netstat -rn

# Statistiques interfaces
netstat -i
```

#### ping / traceroute

```bash
# Test connectivité ICMP
ping -c 4 8.8.8.8

# Traceroute
traceroute google.com
# ou
tracepath google.com

# Traceroute TCP (contourne blocage ICMP)
tcptraceroute google.com 443
```

#### curl / wget

```bash
# Test HTTP avec détails
curl -v https://example.com

# Voir timing détaillé
curl -w "@curl-format.txt" -o /dev/null -s https://example.com

# curl-format.txt :
time_namelookup: %{time_namelookup}s
time_connect: %{time_connect}s
time_appconnect: %{time_appconnect}s
time_pretransfer: %{time_pretransfer}s
time_starttransfer: %{time_starttransfer}s
time_total: %{time_total}s
```

#### nslookup / dig

```bash
# Résolution DNS simple
nslookup google.com

# Résolution détaillée
dig google.com

# Spécifier serveur DNS
dig @8.8.8.8 google.com

# Reverse DNS
dig -x 8.8.8.8

# Voir TTL
dig google.com +noall +answer
```

#### iptables / nft

```bash
# Lister toutes les règles
iptables -L -n -v

# Table NAT (SNAT/DNAT)
iptables -t nat -L -n -v

# Compter paquets par règle
iptables -L -n -v -x

# Tracer un paquet (debug)
iptables -t raw -A PREROUTING -p tcp --dport 80 -j TRACE
```

### 14.2 Méthodologie de Debugging

#### Étape 1 : Identifier la Couche

```
L7 (Application) ?
→ Erreurs HTTP 4xx/5xx, timeouts applicatifs

L4 (Transport) ?
→ Connection refused, timeout TCP, handshake fail

L3 (IP) ?
→ Routage incorrect, paquet perdu, MTU issues

L2 (Liaison) ?
→ Problème ARP, switch, VLAN
```

#### Étape 2 : Vérifier Connectivité de Base

```bash
# Test ICMP
ping <destination-ip>

# Test port TCP
telnet <ip> <port>
# ou
nc -zv <ip> <port>

# Test DNS
nslookup <hostname>
dig <hostname>
```

#### Étape 3 : Capturer et Analyser Trafic

```bash
# Sur source ET destination
tcpdump -i any -n host <ip>

# Analyser flags TCP
# SYN envoyé mais pas de SYN-ACK ?
# → Firewall ou service down
# RST reçu ?
# → Port fermé
# Retransmissions (duplicate ACK) ?
# → Problème réseau (latence, perte)
```

#### Étape 4 : Vérifier Routage

```bash
# Quelle route utilisée ?
ip route get <destination-ip>

# Table de routage complète
ip route show

# Vérifier si paquet transite par bonne interface
tcpdump -i eth0 host <destination-ip>
```

#### Étape 5 : Analyser États Connexions

```bash
# Stats globales
ss -s

# Beaucoup de TIME_WAIT ?
ss state time-wait | wc -l
# → Check tcp_tw_reuse, timeouts TCP

# CLOSE_WAIT élevé ?
ss state close-wait | wc -l
# → App ne ferme pas connexions
```

### 14.3 Debug Kubernetes Spécifique

#### Test Pod → Pod

```bash
# Depuis un Pod source
kubectl exec -it pod-source -- ping <pod-destination-ip>
kubectl exec -it pod-source -- curl http://<pod-destination-ip>:8080

# Vérifier CNI
kubectl logs -n kube-system -l k8s-app=calico-node
```

#### Test Service DNS

```bash
# Résolution interne
kubectl exec -it pod -- nslookup service.namespace.svc.cluster.local

# Vérifier CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Config CoreDNS
kubectl get configmap -n kube-system coredns -o yaml
```

#### Test Connectivité Externe

```bash
# Depuis Pod
kubectl exec -it pod -- curl -v https://google.com

# Vérifier egress (NSG, Firewall, Network Policy)
kubectl get networkpolicies -A
```

#### Debug Service

```bash
# Voir endpoints du Service
kubectl get endpoints <service-name>

# Si endpoints vide → Problème selector ou Pods pas ready
kubectl get pods -l <selector>
kubectl describe pod <pod-name>

# Vérifier kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

---

## 15. Cas Pratiques : Contexte Promod/AKS

### 15.1 Épuisement du Service CIDR

#### Symptôme

```
Error creating service: no IPs available in range
```

#### Diagnostic

```bash
# Compter les Services existants
kubectl get services -A | wc -l

# Vérifier Service CIDR configuré
kubectl cluster-info dump | grep service-cluster-ip-range

# Calculer IPs disponibles
# Ex: /20 = 2^12 = 4096 IPs
# Mais attention aux Services headless qui consomment aussi !
```

#### Solution

**Court terme** :
```bash
# Nettoyer Services inutilisés
kubectl get svc -A | grep <pattern>
kubectl delete svc <service-name> -n <namespace>
```

**Long terme** :
```bash
# Recréer cluster avec Service CIDR plus large
az aks create \
--service-cidr 10.96.0.0/12 \ # Au lieu de /20
--dns-service-ip 10.96.0.10
```

#### ️ Prévention

**Calcul dimensionnement** :
```
Services max prévus : 500
Services headless : 100 (multiplient par nombre de Pods)
Marge sécurité : 2x

→ Service CIDR : /16 minimum (65k IPs)
Idéalement : /12 (1M IPs)
```

### 15.2 Problèmes Application Gateway

#### Symptômes

- Timeouts HTTP aléatoires
- 502 Bad Gateway
- Latence élevée

#### Causes Possibles

##### 1. Exhaustion SNAT

```bash
# Vérifier métriques Azure
az monitor metrics list \
--resource <app-gateway-id> \
--metric "SNAT Port Utilization"

# Si >80% → Problème SNAT
```

**Solution** :
- Augmenter instances App Gateway
- Activer connection pooling côté backend
- Configurer keep-alive

```nginx
# Dans le Pod nginx
http {
upstream backend {
server backend:8080;
keepalive 32; # Connection pool
}
}
```

##### 2. Health Probes Fails

```bash
# Tester health check depuis App Gateway subnet
curl -v http://<pod-ip>:8080/health

# Vérifier logs backend
kubectl logs <pod-name> | grep health

# Ajuster probe settings
```

```yaml
# Dans l'Ingress
metadata:
annotations:
appgw.ingress.kubernetes.io/health-probe-interval: "30"
appgw.ingress.kubernetes.io/health-probe-timeout: "30"
appgw.ingress.kubernetes.io/health-probe-unhealthy-threshold: "3"
```

##### 3. NSG/Firewall Blocking

```bash
# Vérifier NSG rules
az network nsg rule list \
--resource-group <rg> \
--nsg-name <nsg-name> \
--output table

# Règles nécessaires :
# App Gateway subnet → AKS subnet : ports backend (80, 443, custom)
# App Gateway subnet → Internet : port 443 (health checks externe)
```

##### 4. MTU Mismatch

```bash
# Tester MTU
ping -M do -s 1472 <pod-ip> # 1472 + 28 = 1500

# Si erreur "Frag needed" → Ajuster MTU
ip link set dev eth0 mtu 1450
```

### 15.3 Migration Workload Identity

#### Contexte

**Avant** : Azure AD Pod Identity (inject proxy sidecar)
**Après** : Workload Identity (OIDC native)

#### Impact Réseau

**Pod Identity** :
```
Pod → Proxy sidecar (NMI) → IMDS (169.254.169.254) → Azure AD
↑ Overhead réseau
```

**Workload Identity** :
```
Pod → Service Account Token → Azure AD (direct OIDC)
↑ Pas de proxy
```

#### Points d'Attention TCP/IP

```bash
# Vérifier connectivité Azure AD endpoints
kubectl exec -it pod -- curl -v https://login.microsoftonline.com

# Vérifier egress autorisé (NSG/Firewall)
# Azure AD endpoints :
# - login.microsoftonline.com (443)
# - *.graph.microsoft.com (443)
```

### 15.4 Debug Production : Checklist

#### Pod ne démarre pas

```bash
# 1. Vérifier events
kubectl describe pod <pod-name>

# 2. Vérifier image pull
kubectl get events --field-selector involvedObject.name=<pod-name>

# 3. Vérifier réseau
kubectl exec -it <pod-name> -- ip addr
kubectl exec -it <pod-name> -- ip route
```

#### Service inaccessible

```bash
# 1. Vérifier endpoints
kubectl get endpoints <service-name>

# 2. Si vide, vérifier selector
kubectl get pods -l <selector> -o wide

# 3. Tester depuis un autre Pod
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
wget -O- http://<service-name>.<namespace>.svc.cluster.local

# 4. Vérifier Network Policies
kubectl get netpol -n <namespace>
```

#### Latence élevée

```bash
# 1. Mesurer latence réseau
kubectl exec -it pod-source -- ping <pod-destination-ip>

# 2. Vérifier retransmissions TCP
kubectl exec -it pod-source -- netstat -s | grep retrans

# 3. Analyser avec tcpdump
kubectl exec -it pod -- tcpdump -i eth0 -w /tmp/capture.pcap
kubectl cp pod:/tmp/capture.pcap ./capture.pcap
# Analyser avec Wireshark

# 4. Vérifier CPU/Memory du Pod
kubectl top pod <pod-name>
```

---

## 16. Best Practices TCP/IP pour Infrastructure Kubernetes

### Dimensionnement Réseau

#### Pod CIDR

```
Calcul :
- Nœuds max : 50
- Pods max par nœud : 110 (défaut AKS)
- IPs nécessaires : 50 × 110 = 5,500

→ Pod CIDR : /16 (65k IPs)
Alternative : /17 (32k IPs) si budget IP limité
```

#### Service CIDR

```
Calcul :
- Services max : 1,000
- Marge sécurité : 3x
- IPs nécessaires : 3,000

→ Service CIDR : /16 (65k IPs)
Idéalement : /12 (1M IPs) pour éviter tout risque
```

#### Node Subnet (Azure CNI)

```
Calcul :
- Nœuds actuels : 20
- Nœuds max (avec scaling) : 50
- Marge : 2x

→ Node subnet : /25 (128 IPs) minimum
Recommandé : /24 (256 IPs)
```

### MTU Optimization

```bash
# Overlay networks (VXLAN)
ip link set dev eth0 mtu 1450

# Underlay / Azure CNI
ip link set dev eth0 mtu 1500

# Jumbo frames (on-prem datacenter)
ip link set dev eth0 mtu 9000
```

### Connection Pooling

#### Application Level

```python
# Python example
import requests
from requests.adapters import HTTPAdapter

session = requests.Session()
adapter = HTTPAdapter(
pool_connections=100,
pool_maxsize=100,
max_retries=3
)
session.mount('http://', adapter)
session.mount('https://', adapter)
```

```go
// Go example
client := &http.Client{
Transport: &http.Transport{
MaxIdleConns: 100,
MaxIdleConnsPerHost: 100,
IdleConnTimeout: 90 * time.Second,
},
}
```

#### Database Connections

```yaml
# PostgreSQL example
env:
- name: DB_POOL_SIZE
value: "20"
- name: DB_POOL_TIMEOUT
value: "30"
- name: DB_MAX_OVERFLOW
value: "10"
```

### Kernel Tuning (Linux)

```bash
# Augmenter limites connexions
sysctl -w net.core.somaxconn=4096
sysctl -w net.ipv4.tcp_max_syn_backlog=4096

# Réutiliser ports TIME_WAIT plus rapidement
sysctl -w net.ipv4.tcp_tw_reuse=1

# Ajuster timeouts TCP
sysctl -w net.ipv4.tcp_keepalive_time=600
sysctl -w net.ipv4.tcp_keepalive_intvl=60
sysctl -w net.ipv4.tcp_keepalive_probes=3

# Augmenter buffer sizes
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
```

### Monitoring Réseau

#### Métriques Clés

| Métrique | Seuil Alert | Action |
|----------|-------------|--------|
| **Latency (RTT)** | >100ms | Investiguer routage/congestion |
| **Packet loss** | >1% | Check infrastructure réseau |
| **TCP retransmissions** | >1% | Analyser qualité réseau |
| **Connexions TIME_WAIT** | >10k | Ajuster timeouts, activer tw_reuse |
| **SNAT port usage** | >80% | Scale out, optimize connection pooling |

#### Prometheus Queries

```promql
# Latency moyenne
rate(http_request_duration_seconds_sum[5m])
/ rate(http_request_duration_seconds_count[5m])

# Taux d'erreur réseau
rate(network_errors_total[5m])

# Connexions actives
node_netstat_Tcp_CurrEstab

# TIME_WAIT connexions
node_sockstat_TCP_tw
```

### Sécurité Réseau

#### Network Policies : Stratégie Deny-All

```yaml
# 1. Deny everything par défaut
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: default-deny-all
namespace: production
spec:
podSelector: {}
policyTypes:
- Ingress
- Egress

---
# 2. Autoriser uniquement le nécessaire
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: allow-frontend-backend
namespace: production
spec:
podSelector:
matchLabels:
tier: backend
policyTypes:
- Ingress
ingress:
- from:
- podSelector:
matchLabels:
tier: frontend
ports:
- protocol: TCP
port: 8080
```

#### TLS/mTLS

```yaml
# Avec Istio
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
name: default
namespace: production
spec:
mtls:
mode: STRICT # Force mTLS
```

#### Egress Control

```yaml
# Limiter egress Internet
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: allow-external-apis
spec:
podSelector:
matchLabels:
app: api-client
policyTypes:
- Egress
egress:
- to:
- ipBlock:
cidr: 0.0.0.0/0
except:
- 10.0.0.0/8 # RFC 1918
- 172.16.0.0/12 # RFC 1918
- 192.168.0.0/16 # RFC 1918
- 169.254.169.254/32 # Metadata service
ports:
- protocol: TCP
port: 443
```

---

## Ressources Recommandées

### RFCs Officielles

| RFC | Sujet | Lien |
|-----|-------|------|
| **RFC 791** | Internet Protocol (IP) | [rfc-editor.org/rfc/rfc791](https://www.rfc-editor.org/rfc/rfc791) |
| **RFC 793** | Transmission Control Protocol (TCP) | [rfc-editor.org/rfc/rfc793](https://www.rfc-editor.org/rfc/rfc793) |
| **RFC 768** | User Datagram Protocol (UDP) | [rfc-editor.org/rfc/rfc768](https://www.rfc-editor.org/rfc/rfc768) |
| **RFC 1180** | TCP/IP Tutorial | [rfc-editor.org/rfc/rfc1180](https://www.rfc-editor.org/rfc/rfc1180) |
| **RFC 1918** | Private Address Space | [rfc-editor.org/rfc/rfc1918](https://www.rfc-editor.org/rfc/rfc1918) |
| **RFC 826** | Address Resolution Protocol (ARP) | [rfc-editor.org/rfc/rfc826](https://www.rfc-editor.org/rfc/rfc826) |

### Livres

- **TCP/IP Illustrated, Volume 1** - W. Richard Stevens
*La référence absolue sur TCP/IP*

- **IBM Redbook: TCP/IP Tutorial and Technical Overview**
*Gratuit, ~900 pages, très complet*
[ibm.com/redbooks](https://www.redbooks.ibm.com/abstracts/gg243376.html)

### Kubernetes Networking

- **CNI Specification**
[github.com/containernetworking/cni](https://github.com/containernetworking/cni)

- **The Kubernetes Networking Guide**
[tkng.io](https://www.tkng.io)

- **Azure CNI Documentation**
[docs.microsoft.com/azure/aks/configure-azure-cni](https://docs.microsoft.com/azure/aks/configure-azure-cni)

### Outils Pratiques

| Outil | Description | Installation |
|-------|-------------|--------------|
| **Wireshark** | Analyse graphique captures réseau | [wireshark.org](https://www.wireshark.org) |
| **mtr** | Traceroute + ping combinés | `apt install mtr` |
| **netcat (nc)** | Swiss army knife réseau | `apt install netcat` |
| **kubectl-debug** | Debug pods réseau dans k8s | [github.com/aylei/kubectl-debug](https://github.com/aylei/kubectl-debug) |
| **termshark** | Wireshark en CLI | `apt install termshark` |
| **tcptrace** | Analyse avancée captures TCP | `apt install tcptrace` |

### Formations

- **Azure AZ-104** (certification)
Section networking très complète

- **Kubernetes CKA/CKAD**
Networking troubleshooting

- **Linux Foundation: Kubernetes Networking**
[training.linuxfoundation.org](https://training.linuxfoundation.org)

---

## Conclusion

La maîtrise de TCP/IP est **fondamentale** pour un ingénieur infrastructure moderne, particulièrement dans un contexte cloud et Kubernetes.

### Points Clés à Retenir

1. **Comprendre les couches** : OSI/TCP-IP aide à identifier où se situe un problème
2. **Maîtriser le subnetting** : Crucial pour dimensionner correctement (Pod/Service/Node CIDRs)
3. **Connaître TCP vs UDP** : Choisir le bon protocole selon les besoins (fiabilité vs latence)
4. **Débugger méthodiquement** : tcpdump, ss, et analyse des flags TCP sont tes meilleurs amis
5. **Anticiper les limites** : SNAT exhaustion, MTU, Service CIDR

### Prochaines Étapes

- [ ] Pratiquer sur tes 12 clusters Promod/AKS
- [ ] Documenter chaque incident réseau rencontré
- [ ] Automatiser les checks réseau (scripts, monitoring)
- [ ] Approfondir CNI spécifique à ton environnement (Azure CNI)
- [ ] Préparer AZ-104 section networking

### Philosophie

> "Chaque incident réseau est une opportunité d'apprentissage concret des concepts TCP/IP."

Continue à pratiquer, à expérimenter, et à documenter. La maîtrise vient avec l'expérience terrain.

---

<div align="center">

**Document créé pour Lucas Le Pouliquen**
*DevOps Engineer @ Promod (via Experis)*

Janvier 2026

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?logo=linkedin)](https://linkedin.com)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-181717?logo=github)](https://github.com)

</div>

---

## 17. Cycle de Vie d'un Paquet TCP/IP : De l'Application au Câble

Cette section détaille le parcours complet d'un paquet réseau, de l'application qui l'émet jusqu'à sa réception par la machine distante, en passant par toutes les transformations qu'il subit.

### 17.1 Vue d'Ensemble : Les 7 Étapes

```
┌─────────────────────────────────────────────────────────────┐
│ APPLICATION (Browser, curl, app) │
│ └─> Génère données : "GET /index.html HTTP/1.1" │
└────────────────────┬────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ COUCHE TRANSPORT (TCP/UDP) │
│ └─> Segmentation + ajout header TCP (ports, seq#, flags) │
└────────────────────┬────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ COUCHE RÉSEAU (IP) │
│ └─> Ajout header IP (IPs source/dest, TTL, protocole) │
└────────────────────┬────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ ROUTAGE │
│ └─> Table de routage : quelle interface ? quel gateway ? │
└────────────────────┬────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ ARP (si nécessaire) │
│ └─> Résolution IP → MAC de la machine ou gateway │
└────────────────────┬────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ COUCHE LIAISON (Ethernet) │
│ └─> Encapsulation dans trame : MAC src/dest, Type, FCS │
└────────────────────┬────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ COUCHE PHYSIQUE (Câble, WiFi) │
│ └─> Conversion en signaux électriques/lumineux/radio │
└─────────────────────────────────────────────────────────────┘
```

---

### 17.2 Étape par Étape : Exemple Concret

**Scénario** : Un Pod Kubernetes (10.244.1.5) envoie une requête HTTP vers google.com (142.250.185.46).

#### **Étape 1 : Couche Application**

L'application génère les données à envoyer.

```python
# Application Python dans le Pod
import requests
response = requests.get("https://google.com")
```

**Données générées** :
```
GET / HTTP/1.1
Host: google.com
User-Agent: Python-requests/2.31.0
```

**Taille** : ~150 bytes de données HTTP

---

#### **Étape 2 : Couche Transport (TCP)**

Le kernel prend les données et ajoute un **header TCP**.

##### Structure du Header TCP (20 bytes minimum)

```
0 1 2 3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Source Port | Destination Port |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Sequence Number |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Acknowledgment Number |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Offset|Res|Flags| Window Size |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Checksum | Urgent Pointer |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Exemple de notre paquet** :
```
Source Port: 54321 (port éphémère du Pod)
Dest Port: 443 (HTTPS)
Sequence Number: 1000000 (numéro de séquence actuel)
ACK Number: 500000 (dernier byte reçu de l'autre côté)
Flags: ACK + PSH (0x18)
└─> ACK = acquitte les données reçues
└─> PSH = envoie immédiatement au destinataire
Window Size: 64240 (bytes que je peux recevoir)
Checksum: 0x3a4f (calculé sur header + données)
```

**Résultat** : Segment TCP = **Header TCP (20 bytes) + Données (150 bytes) = 170 bytes**

---

#### **Étape 3 : Couche Réseau (IP)**

Le kernel ajoute un **header IP** au segment TCP.

##### Structure du Header IPv4 (20 bytes minimum)

```
0 1 2 3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| IHL |Type of Service| Total Length |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Identification |Flags| Fragment Offset |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Time to Live | Protocol | Header Checksum |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Source IP Address |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Destination IP Address |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Exemple de notre paquet** :
```
Version: 4 (IPv4)
IHL: 5 (header length = 5 × 4 = 20 bytes)
Type of Service: 0x00 (pas de QoS particulière)
Total Length: 190 (20 header IP + 170 segment TCP)
Identification: 54321 (ID unique pour ce datagram)
Flags: Don't Fragment (0x4000)
Fragment Offset: 0 (pas de fragmentation)
TTL: 64 (nombre max de hops/routeurs)
Protocol: 6 (TCP)
Header Checksum: 0x5f2a (vérifie intégrité header IP)
Source IP: 10.244.1.5 (IP du Pod)
Dest IP: 142.250.185.46 (IP de google.com)
```

**Résultat** : Paquet IP = **Header IP (20) + Header TCP (20) + Données (150) = 190 bytes**

---

#### **Étape 4 : Décision de Routage**

Le kernel consulte sa **table de routage** pour savoir où envoyer le paquet.

```bash
# Table de routage du Pod
ip route show

default via 10.244.1.1 dev eth0 # Route par défaut
10.244.1.0/24 dev eth0 scope link # Réseau local du Pod
```

**Décision** :
- Destination `142.250.185.46` ne matche aucune route spécifique
- → Utiliser la **route par défaut** : `via 10.244.1.1` (gateway = nœud Kubernetes)
- → Interface : `eth0`

**Prochaine étape** : Envoyer le paquet à `10.244.1.1` (le gateway)

---

#### **Étape 5 : Résolution ARP**

Avant d'envoyer la trame Ethernet, le kernel doit connaître l'**adresse MAC** du gateway (`10.244.1.1`).

##### 5a. Vérifier le cache ARP

```bash
# Consulter le cache ARP du Pod
ip neigh show

10.244.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

**Scénario 1** : MAC trouvée dans le cache → Passer directement à l'étape 6

**Scénario 2** : MAC pas dans le cache → Effectuer une requête ARP

##### 5b. Requête ARP (si nécessaire)

```
┌─────────────────────────────────────────────────────────────┐
│ ARP REQUEST (Broadcast) │
├─────────────────────────────────────────────────────────────┤
│ Sender MAC: 11:22:33:44:55:66 (MAC du Pod) │
│ Sender IP: 10.244.1.5 │
│ Target MAC: 00:00:00:00:00:00 (cherche cette info) │
│ Target IP: 10.244.1.1 │
│ Broadcast à: ff:ff:ff:ff:ff:ff (toutes les machines) │
└─────────────────────────────────────────────────────────────┘
```

**Réponse du gateway** :
```
┌─────────────────────────────────────────────────────────────┐
│ ARP REPLY (Unicast) │
├─────────────────────────────────────────────────────────────┤
│ Sender MAC: aa:bb:cc:dd:ee:ff (MAC du gateway) │
│ Sender IP: 10.244.1.1 │
│ Target MAC: 11:22:33:44:55:66 (MAC du Pod) │
│ Target IP: 10.244.1.5 │
└─────────────────────────────────────────────────────────────┘
```

**Résultat** : Le Pod connaît maintenant la MAC du gateway : `aa:bb:cc:dd:ee:ff`

---

#### **Étape 6 : Couche Liaison (Ethernet)**

Le kernel encapsule le paquet IP dans une **trame Ethernet**.

##### Structure de la Trame Ethernet

```
┌──────────────┬──────────────┬──────┬────────────┬──────┐
│ MAC Dest │ MAC Source │ Type │ Payload │ FCS │
│ (6 bytes) │ (6 bytes) │(2 B) │ (IP packet)│(4 B) │
└──────────────┴──────────────┴──────┴────────────┴──────┘
```

**Exemple de notre paquet** :
```
MAC Destination: aa:bb:cc:dd:ee:ff (MAC du gateway)
MAC Source: 11:22:33:44:55:66 (MAC du Pod)
EtherType: 0x0800 (IPv4)
Payload: [Paquet IP de 190 bytes]
FCS: 0x8a3f2c1d (checksum pour détecter erreurs)
```

**Taille totale** :
```
14 (header Ethernet) + 190 (paquet IP) + 4 (FCS) = 208 bytes
```

**Important** : Taille minimum d'une trame Ethernet = **64 bytes**. Si payload < 46 bytes, on ajoute du **padding**.

---

#### **Étape 7 : Couche Physique**

La carte réseau (NIC) convertit la trame en **signaux électriques/optiques/radio**.

##### Avec Câble Ethernet (RJ45)

```
Trame Ethernet (bits)
↓
Encodage Manchester (pour Ethernet 10/100 Mbps)
ou
Encodage 8B/10B (pour Gigabit Ethernet)
↓
Modulation en tensions électriques
↓
+3.3V → bit 1
-3.3V → bit 0
↓
Transmission sur les paires de câbles (TX+ TX- RX+ RX-)
```

##### Avec Fibre Optique

```
Bits → Impulsions lumineuses
1 = LED/Laser ON
0 = LED/Laser OFF
↓
Transmission via fibre optique (monomode ou multimode)
```

##### Avec WiFi

```
Bits → Modulation radio (OFDM pour WiFi 802.11ac/ax)
↓
Transmission sur ondes radio (2.4 GHz ou 5 GHz)
```

---

### 17.3 Le Chemin Retour : Réception du Paquet

Le processus inverse se produit sur la machine distante (google.com).

```
┌─────────────────────────────────────────────────────────────┐
│ 1. COUCHE PHYSIQUE │
│ Signaux électriques → Bits │
└────────────────────┬────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ 2. COUCHE LIAISON (Ethernet) │
│ - Vérifie FCS (checksum) │
│ - Vérifie MAC destination = MAC de cette interface │
│ - Extrait paquet IP du payload │
└────────────────────┬────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ 3. COUCHE RÉSEAU (IP) │
│ - Vérifie checksum IP │
│ - Vérifie IP destination = IP de cette machine │
│ - Décrémente TTL (évite boucles infinies) │
│ - Extrait segment TCP │
└────────────────────┬────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ 4. COUCHE TRANSPORT (TCP) │
│ - Vérifie checksum TCP │
│ - Vérifie port destination = 443 (HTTPS) │
│ - Vérifie sequence number (ordre des segments) │
│ - Réassemble segments si fragmentés │
│ - Met à jour fenêtre TCP (flow control) │
│ - Envoie ACK pour confirmer réception │
└────────────────────┬────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ 5. COUCHE APPLICATION │
│ - Le serveur web (nginx, Apache) reçoit la requête │
│ - Traite "GET / HTTP/1.1" │
│ - Génère réponse HTTP │
└─────────────────────────────────────────────────────────────┘
```

---

### 17.4 Cas Spéciaux : Particularités

#### Fragmentation IP

Si le paquet IP dépasse le **MTU** de l'interface (ex: 1500 bytes), il est fragmenté.

**Exemple** :
```
Paquet original : 3000 bytes
MTU : 1500 bytes

Fragment 1:
- Header IP (20 bytes) + 1480 bytes de données
- Flags: More Fragments = 1
- Fragment Offset = 0

Fragment 2:
- Header IP (20 bytes) + 1480 bytes de données
- Flags: More Fragments = 1
- Fragment Offset = 1480

Fragment 3:
- Header IP (20 bytes) + 40 bytes de données
- Flags: More Fragments = 0 (dernier)
- Fragment Offset = 2960
```

**Problème** : Si un seul fragment est perdu → TOUT le paquet doit être retransmis.

**Solution** : Path MTU Discovery (Don't Fragment + ajuster taille)

---

#### VXLAN (Overlay Network Kubernetes)

Dans un overlay network comme VXLAN, le paquet original est **encapsulé** dans un nouveau paquet UDP/IP.

```
Paquet Pod original:
┌─────────┬─────────┬─────────┬──────────┐
│ MAC Hdr │ IP Hdr │ TCP Hdr │ Data │
└─────────┴─────────┴─────────┴──────────┘

Après encapsulation VXLAN:
┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬─────────┬──────────┐
│ MAC Hdr │ IP Hdr │ UDP Hdr │ VXLAN │ MAC Hdr │ IP Hdr │ TCP Hdr │ Data │
│ (Node) │ (Node) │ (8472) │ (8 B) │ (Pod) │ (Pod) │ │ │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴─────────┴──────────┘
↑ ↑
Outer headers Inner headers
(entre nœuds k8s) (entre Pods)
```

**Overhead VXLAN** : 50 bytes
- Outer Ethernet header: 14 bytes
- Outer IP header: 20 bytes
- UDP header: 8 bytes
- VXLAN header: 8 bytes

**Conséquence** : MTU effectif = 1500 - 50 = **1450 bytes**

---

#### NAT (Network Address Translation)

Quand le paquet traverse un **routeur NAT**, les headers IP et TCP sont modifiés.

**SNAT (Sortie du réseau privé)** :

```
Paquet original (dans le réseau privé):
Source IP: 10.0.1.5
Source Port: 54321
Dest IP: 8.8.8.8
Dest Port: 443

Après SNAT par le routeur:
Source IP: 203.0.113.5 (IP publique du routeur)
Source Port: 12345 (port traduit par PAT)
Dest IP: 8.8.8.8
Dest Port: 443
```

Le routeur NAT garde une **table de translation** :

```
| IP Privée | Port Privé | IP Publique | Port Public |
|------------|------------|-------------|-------------|
| 10.0.1.5 | 54321 | 203.0.113.5 | 12345 |
| 10.0.1.8 | 42000 | 203.0.113.5 | 12346 |
```

**Réponse** : Le routeur fait la translation inverse (DNAT) pour renvoyer au bon client privé.

---

### 17.5 Timeline Complète d'une Connexion TCP

Voici la séquence temporelle complète pour établir une connexion, envoyer des données, et fermer.

```
Client (10.244.1.5) Serveur (142.250.185.46)
| |
|--- SYN (seq=1000, win=64240) ----------> | Three-way handshake
| | (établir connexion)
|<-- SYN-ACK (seq=5000, ack=1001, win=32768) |
| |
|--- ACK (seq=1001, ack=5001) ------------> |
| |
|=== ESTABLISHED ============================= |
| |
|--- PSH-ACK (seq=1001, data="GET /") ----> | Envoi requête HTTP
| |
|<-- ACK (ack=1051) ------------------------ | Acquitte réception
| |
|<-- PSH-ACK (seq=5001, data=HTTP response) | Réponse serveur
| |
|--- ACK (ack=5501) ------------------------> | Acquitte réception
| |
|--- FIN-ACK (seq=1051) --------------------> | Client ferme connexion
| |
|<-- FIN-ACK (seq=5501) ---------------------- | Serveur ferme aussi
| |
|--- ACK (ack=5502) ------------------------> |
| |
|=== CLOSED =================================== |
```

**Durée typique** :
- Handshake (3 étapes) : ~30-100 ms (dépend de la latence réseau)
- Transfert données : dépend de la taille et du débit
- Fermeture (4 étapes) : ~30-100 ms

---

### 17.6 Voir les Paquets en Action

#### Avec tcpdump

```bash
# Capturer et afficher le contenu des paquets
tcpdump -i eth0 -XX -n host 8.8.8.8

# Exemple output:
15:30:42.123456 IP 10.244.1.5.54321 > 8.8.8.8.443: Flags [S], seq 1000, win 64240
0x0000: aabb ccdd eeff 1122 3344 5566 0800 4500 ........."3DUf..E.
0x0010: 003c 0001 4000 4006 f2a1 0af4 0105 0808 .<..@.@.........
0x0020: 0808 d431 01bb 0000 03e8 0000 0000 a002 ...1............
0x0030: faf0 5f2a 0000 0204 05b4 0402 080a 1234 .._*...........4
```

**Décodage** :
- `0x0000-0x000d` : Header Ethernet (MAC dest, MAC src, Type)
- `0x000e-0x0021` : Header IP (version, TTL, proto, IPs src/dest)
- `0x0022-0x0035` : Header TCP (ports, seq, flags, window)

#### Avec Wireshark

Interface graphique pour analyser les paquets avec décodage automatique de toutes les couches.

```bash
# Capturer dans un fichier
tcpdump -i eth0 -w capture.pcap

# Ouvrir avec Wireshark
wireshark capture.pcap
```

---

### 17.7 Optimisations Réseau

#### Jumbo Frames

Augmente le MTU au-delà de 1500 bytes pour réduire l'overhead.

```bash
# MTU standard
1500 bytes → ~94% efficiency (1460 données / 1500 total)

# Jumbo frames
9000 bytes → ~98.9% efficiency (8960 données / 9000 total)
```

**Avantage** : Moins de paquets à traiter par le CPU
**Contrainte** : Tout le chemin réseau doit supporter 9000 MTU

#### TCP Window Scaling

Augmente la fenêtre TCP au-delà de 64KB (limite du champ 16 bits).

```
Sans scaling: Window max = 65535 bytes
Avec scaling: Window max = 65535 × 2^14 = 1 GB
```

**Formule débit max** :
```
Throughput = Window Size / RTT

Exemple:
Window = 64 KB, RTT = 100 ms
→ Throughput max = 64 KB / 0.1s = 640 KB/s = 5 Mbps

Avec window scaling (1 MB):
→ Throughput max = 1 MB / 0.1s = 10 MB/s = 80 Mbps
```

#### TCP Fast Open

Envoie des données dès le SYN (économise un round-trip).

```
Normal TCP:
SYN → SYN-ACK → ACK → [DATA]
(3 RTT avant les données)

TCP Fast Open:
SYN + [DATA] → SYN-ACK + [DATA] → ACK
(1 RTT avant les données)
```

---

### 17.8 Debugging : Où le Paquet est-il Bloqué ?

Checklist systématique pour débugger un paquet qui n'arrive pas.

#### Le paquet quitte-t-il l'application ?

```bash
# Vérifier que l'app écoute sur le bon port
ss -tlnp | grep :8080

# Vérifier qu'il y a des connexions sortantes
ss -tn | grep ESTABLISHED
```

#### Le paquet est-il créé au niveau TCP ?

```bash
# Capturer sur loopback (communication locale)
tcpdump -i lo port 8080

# Vérifier stats TCP
netstat -s | grep -i retrans
# Beaucoup de retransmissions ? → Problème réseau
```

#### 3️⃣ Le paquet est-il routé correctement ?

```bash
# Vérifier la route prise
ip route get 8.8.8.8

# Capturer sur l'interface de sortie
tcpdump -i eth0 host 8.8.8.8

# Si le paquet n'apparaît pas → Problème routage local
```

#### 4️⃣ Le paquet traverse-t-il les firewalls ?

```bash
# Vérifier iptables
iptables -L -n -v | grep DROP

# Tracer un paquet (debug iptables)
iptables -t raw -A PREROUTING -p tcp --dport 443 -j TRACE
tail -f /var/log/kern.log
```

#### 5️⃣ Le paquet arrive-t-il sur l'interface destination ?

```bash
# Sur le serveur distant, capturer
tcpdump -i eth0 host 10.244.1.5

# Si pas de paquet → Bloqué en route (firewall réseau, routeur)
```

#### 6️⃣ Le paquet est-il traité par l'application ?

```bash
# Vérifier logs application
kubectl logs pod-name

# Vérifier connexions entrantes
ss -tn | grep :443
```

---

### 17.9 Points Clés à Retenir

1. **Encapsulation progressive** : Chaque couche ajoute son header sans toucher aux données des couches supérieures

2. **Headers fixes** :
- Ethernet : 14 bytes
- IP : 20 bytes (minimum)
- TCP : 20 bytes (minimum)
- **Total overhead minimum : 54 bytes**

3. **MTU est crucial** :
- Ethernet standard : 1500 bytes
- Overlay (VXLAN) : 1450 bytes
- Fragmentation = performance dégradée

4. **ARP résout IP → MAC** : Uniquement sur le réseau local (même subnet)

5. **Routage se fait au niveau IP** : La table de routage décide de l'interface et du next-hop

6. **TCP garantit fiabilité** :
- Séquence numbers pour l'ordre
- ACKs pour confirmer réception
- Retransmissions automatiques

7. **Chaque hop décrémente TTL** : Évite les boucles infinies (paquet meurt après 64 hops par défaut)

---

### 17.10 Exercice Pratique

**Objectif** : Tracer un paquet de bout en bout

```bash
# 1. Sur la machine source
# Capturer les paquets sortants
tcpdump -i eth0 -w source.pcap host 8.8.8.8 &

# Envoyer requête
curl -v https://8.8.8.8

# Arrêter capture
killall tcpdump

# 2. Analyser avec Wireshark
wireshark source.pcap

# 3. Identifier dans Wireshark:
# - Le SYN (flags S)
# - Le SYN-ACK (flags SA)
# - Le ACK (flags A)
# - Les données HTTP (flags PA)
# - Les FIN pour fermer (flags F)

# 4. Examiner les headers:
# - MAC addresses dans Ethernet header
# - IP addresses dans IP header
# - Ports dans TCP header
# - Sequence numbers et ACK numbers
# - TTL qui décrémente à chaque hop
```

---

