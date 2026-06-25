# Plan de hardening Ubuntu — nœuds k3s

> Plan de durcissement des deux nœuds Ubuntu du cluster k3s.
> Rédigé pour le contexte spécifique de ce homelab : il **ne faut pas** appliquer
> un benchmark CIS « brut » sans tenir compte de k3s (IP forwarding, MetalLB/BGP,
> AppArmor, modules noyau réseau). Chaque section signale les points qui
> casseraient le cluster si on les durcit aveuglément.

## Contexte & périmètre

| Élément | Valeur |
|---------|--------|
| Nœud 1 | `kube1.home` — `10.10.0.130` (control-plane, `--cluster-init`) |
| Nœud 2 | `kube2.server` — `10.10.0.131` (control-plane, HA) |
| OS | Ubuntu Server (LTS) |
| k3s | `v1.30.7+k3s1`, Traefik & servicelb désactivés |
| Réseau | dual-stack IPv4/IPv6, pfSense (FRR/BGP AS 65000), MetalLB BGP (AS 65002) |
| LB | MetalLB en mode BGP, pools `10.99.0.0/24` + `FC00:99::/32` |
| Accès admin | SSH `max@kubeX` |
| Sécurité applicative | CrowdSec (bouncer Traefik), Zitadel OIDC, Vault/OpenBao |

**Objectif** : réduire la surface d'attaque au niveau **OS** (l'applicatif est déjà
couvert par Traefik/CrowdSec/Zitadel), sans dégrader la disponibilité du cluster.

### Principe de déploiement

- **Tester sur `kube2` d'abord**, valider que le cluster reste sain
  (`kubectl get nodes`, peering BGP `UP`, pods `Running`), puis appliquer sur `kube1`.
- Ne jamais durcir les deux nœuds simultanément : on garde toujours un nœud sain
  comme filet de sécurité (control-plane HA).
- Idéalement, tout passe par **Ansible** (idempotent, versionné dans ce repo) plutôt
  que par des commandes manuelles. Voir §11.

---

## ⚠️ Constats prioritaires (à corriger en premier)

1. **`K3S_KUBECONFIG_MODE="644"`** — le kubeconfig admin (`/etc/rancher/k3s/k3s.yaml`)
   est **lisible par tous les utilisateurs locaux**. Il contient un certificat
   client `cluster-admin`. ➜ Passer à `600` (voir §7). C'est la faille la plus
   directe : tout process/utilisateur local = admin du cluster.
2. **Pas de pare-feu hôte documenté** — on s'appuie uniquement sur pfSense. Un
   pare-feu local (nftables) limite le mouvement latéral entre nœuds et expose
   uniquement les ports nécessaires (voir §4).
3. **SSH** — vérifier qu'on est bien en auth par clé uniquement, root login
   désactivé (voir §3).
4. **Mises à jour automatiques** — confirmer `unattended-upgrades` actif pour les
   correctifs de sécurité (voir §2).

---

## Phase 1 — Socle (faible risque, gains immédiats)

### §2. Mises à jour & gestion des correctifs

```bash
sudo apt update && sudo apt -y full-upgrade
sudo apt -y install unattended-upgrades apt-listchanges
sudo dpkg-reconfigure -plow unattended-upgrades
```

`/etc/apt/apt.conf.d/50unattended-upgrades` :
- Activer uniquement le canal `-security` (laisser les MAJ de features manuelles
  pour éviter une montée de version surprise sur un nœud k3s).
- `Unattended-Upgrade::Automatic-Reboot "false";` — **ne jamais** rebooter
  automatiquement un nœud k3s. Planifier les reboots manuellement, un nœud à la
  fois, avec `kubectl drain`.

> ⚠️ k3s lui-même n'est **pas** mis à jour par apt. Le binaire est figé
> (`INSTALL_K3S_VERSION`). Gérer les MAJ de k3s séparément (drain → upgrade → uncordon).

### §3. Durcissement SSH

`/etc/ssh/sshd_config.d/99-hardening.conf` (drop-in, ne pas éditer le fichier principal) :

```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
X11Forwarding no
AllowTcpForwarding no          # ⚠️ mettre 'yes' si tu fais du kubectl port-forward via SSH
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
AllowUsers max                 # restreindre aux comptes admin réels
Protocol 2
```

- Restreindre l'écoute SSH au VLAN d'admin (`ListenAddress 10.10.0.130`) si l'IPv6
  publique n'a pas à exposer SSH.
- Côté pfSense : n'autoriser le 22/tcp que depuis le réseau d'administration.
- `sudo sshd -t` puis `sudo systemctl reload ssh` (toujours garder une session
  ouverte en secours pendant le test).

### §4. Pare-feu hôte (nftables) — **section critique pour k3s**

> ⚠️ **Ne pas utiliser `ufw enable` tel quel** : il pose une politique
> `FORWARD drop` qui casse le routage des pods et MetalLB. k3s gère déjà ses
> propres chaînes iptables/nftables. La bonne approche est un jeu de règles
> **INPUT** restrictif qui laisse k3s gérer le FORWARD.

Ports à **autoriser en INPUT** (d'après la doc k3s + ce setup) :

| Port | Proto | Usage | Source autorisée |
|------|-------|-------|------------------|
| 22 | tcp | SSH | VLAN admin `10.10.0.0/24` |
| 6443 | tcp | API server k3s | nœuds + VLAN admin |
| 2379-2380 | tcp | etcd (embedded HA) | **uniquement** les nœuds `10.10.0.130/131` |
| 10250 | tcp | kubelet | nœuds |
| 8472 | udp | Flannel VXLAN (si flannel) | nœuds |
| 51820-51821 | udp | Flannel WireGuard (si activé) | nœuds |
| 179 | tcp | BGP MetalLB ↔ pfSense | `10.10.0.1` + nœuds |
| 3784 | udp | BFD (MetalLB) | `10.10.0.1` |
| (ICMP/ICMPv6) | — | indispensable IPv6 ND | LAN |

- **Tout le trafic inter-nœuds** sur le VLAN 10 doit rester permis (etcd, kubelet,
  VXLAN). Le pare-feu sert surtout à fermer le reste (IPv6 publique, autres VLAN).
- Laisser passer `lo`, `cni0`/`flannel.1`, et les CIDR pods/services
  (`10.42.0.0/16`, `10.43.0.0/16`, `2001:cafe:42::/56`).
- **Ne pas filtrer la chaîne FORWARD** — la déléguer à k3s.
- Tester avec `nft list ruleset` et vérifier le peering : sur pfSense
  `vtysh -c "show bgp summary"` doit montrer les voisins `Established`.

#### §4bis. Fuite des NodePorts (`30000-32767`) — **point critique dual-stack**

Les services `NodePort` se bindent sur **toutes** les interfaces du nœud, IPv4
**et** IPv6 (`0.0.0.0:<port>` et `[::]:<port>`). Comme l'IPv6 du cluster est
routée publiquement via pfSense (BGP), **un NodePort peut être joignable depuis
Internet sur l'IPv6 globale du nœud** si rien ne le bloque. Même sans NodePort
explicite aujourd'hui, la plage reste « ouverte par conception ». Trois couches,
de la plus propre à la défense en profondeur :

**1. (Recommandé) Restreindre le bind de kube-proxy — corrige à la source.**
Faire en sorte que les NodePorts n'écoutent **que** sur le VLAN d'admin, jamais
sur l'IPv6 publique. Au réinstall/MAJ k3s, ajouter :

```
--kube-proxy-arg=nodeport-addresses=10.10.0.0/24,<CIDR_IPv6_admin>
```

Les NodePorts ne se bindent alors plus que sur ces CIDR → plus aucune écoute sur
l'IPv6 globale. C'est le correctif le plus net (rien à filtrer ensuite).

**2. pfSense — périmètre.** S'assurer que le WAN n'autorise **aucun** flux entrant
vers la plage `30000-32767` (TCP/UDP) à destination des nœuds, **y compris en
IPv6** vers leurs GUA. Idéalement default-deny inbound vers les IP des nœuds, et
n'exposer que les VIP MetalLB voulues.

**3. nftables hôte — défense en profondeur.**
> ⚠️ Piège : un `DROP` en chaîne **INPUT ne marche pas** pour les NodePorts.
> kube-proxy fait le DNAT en `nat/PREROUTING` (priorité `-100`) **avant** INPUT,
> puis le paquet part en FORWARD vers le pod. Il faut donc dropper **avant le
> DNAT**, dans un hook `prerouting` de priorité plus basse (ex. `raw`, `-300`),
> et uniquement pour les sources non-LAN :

```nft
table inet nodeport_guard {
  chain prerouting {
    type filter hook prerouting priority -300; policy accept;
    # autoriser le VLAN admin/nœuds
    ip  saddr 10.10.0.0/24 tcp dport 30000-32767 accept
    ip  saddr 10.10.0.0/24 udp dport 30000-32767 accept
    # tout le reste (IPv4 et IPv6 publique) → drop avant le DNAT kube-proxy
    tcp dport 30000-32767 drop
    udp dport 30000-32767 drop
  }
}
```

> Si la couche 1 (`nodeport-addresses`) est en place, cette table devient
> redondante mais reste un bon filet. **Ne jamais** dropper la plage NodePort
> entre nœuds (`10.10.0.0/24`) ni sur `lo` — certains health-checks l'utilisent.

Vérifier : depuis une IP hors-LAN (ou via l'IPv6 publique), `nmap -p 30000-32767`
sur un nœud ne doit renvoyer **aucun** port ouvert ; `ss -tlnp | grep -E ':3[0-2][0-9]{3}'`
sur l'hôte ne doit montrer des binds que sur l'IP du VLAN admin.

### §5. Comptes, sudo & mots de passe

- Un compte admin nominatif par personne, pas de compte partagé.
- `sudo` via groupe, exiger un mot de passe (`Defaults timestamp_timeout=5`),
  pas de `NOPASSWD` global.
- Politique mots de passe : `libpam-pwquality` (longueur ≥ 14, complexité),
  verrouillage après échecs (`faillock`).
- Verrouiller les comptes système inutiles, désactiver `root` (`passwd -l root`).
- Auditer : `awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd`.

---

## Phase 2 — Durcissement noyau & services

### §6. Paramètres noyau (sysctl) — **attention au forwarding**

`/etc/sysctl.d/99-hardening.conf` :

```
# --- Réseau : durcissement SANS casser k3s ---
net.ipv4.ip_forward = 1                 # ⚠️ OBLIGATOIRE pour k3s (NE PAS mettre 0)
net.ipv6.conf.all.forwarding = 1        # ⚠️ OBLIGATOIRE (dual-stack)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.tcp_syncookies = 1
# ⚠️ NE PAS désactiver les redirects/RA de façon globale sans vérifier
#    l'autoconf IPv6 et le ND sur le VLAN MetalLB.

# --- Mémoire / divers ---
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.yama.ptrace_scope = 1
kernel.unprivileged_bpf_disabled = 1
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.suid_dumpable = 0
```

> ⚠️ Le piège classique : un guide CIS met `net.ipv4.ip_forward = 0`. Sur un nœud
> k3s ça **coupe le réseau des pods**. Toujours laisser le forwarding à `1`.
> Idem, ne pas activer `protect-kernel-defaults` côté k3s sans avoir posé
> les sysctl correspondants au préalable (sinon kubelet refuse de démarrer).

Appliquer : `sudo sysctl --system` puis valider le réseau pods/LB.

### §7. Système de fichiers & permissions

```bash
# Kubeconfig admin — corriger le 644
sudo chmod 600 /etc/rancher/k3s/k3s.yaml
sudo chown root:root /etc/rancher/k3s/k3s.yaml
```
- Réinstaller k3s sans `K3S_KUBECONFIG_MODE="644"` (le défaut `600` est correct),
  ou retirer cette variable du script d'install dans `Readme.md`.
- Distribuer le kubeconfig admin via un canal sûr (copie ponctuelle, pas un fichier
  world-readable laissé en place).
- Options de montage durcies sur `/tmp`, `/var/tmp`, `/dev/shm` :
  `nodev,nosuid,noexec` (⚠️ vérifier que rien dans les workloads n'exécute depuis
  `/tmp` de l'hôte). Sur k3s, garder `/var/lib/rancher` et `/var/lib/kubelet`
  **sans** `noexec`.
- Auditer les binaires SUID/SGID : `find / -perm -4000 -type f 2>/dev/null`.

### §8. Réduction de la surface — services & modules

- Désinstaller/désactiver ce qui n'est pas utilisé : `snapd` (si non requis),
  serveurs d'impression, `avahi`, `whoopsie`, etc.
- `systemctl list-unit-files --state=enabled` → ne garder que k3s, ssh, ntp,
  unattended-upgrades, monitoring.
- Blacklister les modules noyau inutiles et à risque :
  `/etc/modprobe.d/blacklist-hardening.conf` →
  `usb-storage` (si pas de besoin), `firewire-core`, `cramfs`, `freevxfs`,
  `jffs2`, `hfs`, `hfsplus`, `udf`, protocoles réseau exotiques (`dccp`, `sctp`,
  `rds`, `tipc`).
  > ⚠️ **Ne pas blacklister** `vxlan`, `wireguard`, `br_netfilter`, `overlay`,
  > `nf_conntrack`, `ip_tables`/`nf_tables` — k3s/flannel en dépendent.

### §9. AppArmor

- Vérifier `aa-status` : AppArmor doit être **enabled** (k3s s'appuie dessus pour
  les profils de conteneurs).
- Mettre les profils en `enforce` pour les démons hôte (ssh, etc.) une fois testés.
- Ne pas désactiver AppArmor globalement.

---

## Phase 3 — Détection, audit & traçabilité

### §10. Audit & journalisation

- **auditd** + règles CIS (`/etc/audit/rules.d/`) : suivi des modifs sur
  `/etc/passwd`, `/etc/ssh/`, `/etc/rancher/`, appels `execve`, modules noyau.
- **journald** persistant (`Storage=persistent`), rotation/limite de taille.
- **Protection SSH** : plutôt que fail2ban (silo séparé), on **unifie avec la
  CrowdSec déjà déployée dans le cluster** — l'agent hôte parse `auth.log`, remonte
  ses alertes à la **LAPI du cluster**, et un `cs-firewall-bouncer` local applique
  les décisions au pare-feu nftables. Décisions et bannissements centralisés,
  partagés entre le HTTP (Traefik) et le SSH (hôtes). ➜ Voir l'**Annexe A**.
- Exporter les logs hors-nœud (TrueNAS/syslog/Loki) pour préserver les preuves en
  cas de compromission d'un nœud.

### §11. Industrialisation & vérification

- **Ansible** : le plan OS est implémenté en rôle idempotent versionné dans
  `K3s-Config` → [`ansible/`](./ansible/) (rôle `hardening`). Les tâches
  bloquantes (SSH, nftables, garde NodePort, CrowdSec hôte) sont **désactivées par
  défaut** et s'activent par flag une fois testées. `serial: 1` → un nœud à la fois.
  > **Répartition hybride** : ArgoCD déploie le *in-cluster* (Service LoadBalancer
  > LAPI + Job d'enregistrement des hôtes, cf. `agrocd-home/init/04-*` et `05-*`) ;
  > Ansible configure l'*OS* des nœuds (y compris l'agent CrowdSec et le
  > firewall-bouncer, qui consomment les identifiants générés par le Job).
- **Audit automatisé** :
  - `lynis audit system` (baseline + score, rapide à mettre en place).
  - **OpenSCAP** / `ssg-debian` ou le profil CIS Ubuntu (`oscap xccdf eval`).
  - **kube-bench** (Aqua) avec le profil **CIS k3s** pour la partie cluster :
    complète le hardening OS côté Kubernetes.
- Rejouer ces scans **après** chaque phase pour mesurer le delta et détecter les
  régressions.

---

## Récapitulatif des pièges spécifiques k3s (à NE PAS faire)

| Durcissement « CIS » classique | Effet sur ce cluster |
|--------------------------------|----------------------|
| `net.ipv4.ip_forward = 0` | ❌ casse le réseau des pods |
| `ufw enable` (FORWARD drop) | ❌ casse routage pods + MetalLB |
| Blacklist `vxlan`/`nf_tables`/`overlay` | ❌ flannel/k3s ne démarrent plus |
| `noexec` sur `/var/lib/...` | ❌ kubelet/conteneurs cassés |
| Reboot auto (unattended-upgrades) | ❌ nœud drainé sans préavis |
| Fermer 6443/2379-2380/10250 entre nœuds | ❌ casse l'API HA + etcd |
| Bloquer ICMPv6 (ND) | ❌ casse l'IPv6 sur le VLAN MetalLB |
| Dropper les NodePorts en chaîne INPUT | ❌ inopérant : DNAT kube-proxy en PREROUTING (filtrer avant, cf. §4bis) |

---

## Annexe A — CrowdSec hôte unifié avec la LAPI du cluster

> Objectif : ne **pas** monter une seconde instance CrowdSec sur les nœuds, mais
> rattacher la protection SSH de l'hôte à la **LAPI déjà déployée dans le cluster**
> (`init/01-crowdsec.yaml`). Une seule base de décisions, partagée entre le HTTP
> (bouncer plugin Traefik) et le SSH (bouncers firewall des hôtes).
>
> **Implémentation (hybride)** — le **cluster** (A.1, A.2) est livré par ArgoCD :
> `agrocd-home/init/04-crowdsec-lapi-lan.yaml` (Service LoadBalancer) et
> `05-crowdsec-host-register.yaml` (Job d'enregistrement). L'**hôte** (A.3, A.4)
> est livré par Ansible : rôle `hardening`, tâche `crowdsec.yml` (flag
> `harden_crowdsec`). Les commandes manuelles ci-dessous documentent ce que ces
> automatisations font.

### A.0. Architecture cible

```
   ┌─────────────────────── Cluster k3s ───────────────────────┐
   │                                                            │
   │   Agent (DaemonSet) ─┐                                     │
   │   parse logs Traefik │                                     │
   │                      ▼                                     │
   │                 ┌──────────┐   ◄── plugin Traefik (HTTP)   │
   │                 │  LAPI    │       décisions HTTP/AppSec    │
   │                 │ (in-pod) │                               │
   │                 └────┬─────┘                               │
   │   Service LoadBalancer (MetalLB) → IP LAN stable :8080     │
   └────────────────────┬──────────────────────────────────────┘
                         │  (VLAN admin, LAN uniquement)
        ┌────────────────┼────────────────┐
        ▼                                  ▼
  kube1 (hôte)                       kube2 (hôte)
  ├─ crowdsec (agent)  → parse /var/log/auth.log → push alertes vers LAPI
  └─ cs-firewall-bouncer → poll décisions LAPI → DROP nftables
```

- L'agent hôte **n'active pas** sa propre LAPI (`api.server.enable: false`) : il
  pointe vers la LAPI du cluster.
- Le `cs-firewall-bouncer` est ce qui **bloque réellement** au niveau OS (nftables),
  en complément du §4. Il crée sa propre table/chaîne nftables `crowdsec`.

### A.1. Côté cluster — exposer la LAPI aux nœuds (MetalLB)

La LAPI n'est aujourd'hui qu'un `ClusterIP`. Pour que les hôtes la joignent à une
IP stable, on l'expose via un `Service` LoadBalancer sur le pool MetalLB
(`10.99.0.0/24`). À ajouter au dépôt **agrocd-home** (`init/`) :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: crowdsec-lapi-lan
  namespace: crowdsec
  annotations:
    # IP fixe choisie dans le pool MetalLB BGP, annoncée à pfSense
    metallb.universe.tf/loadBalancerIPs: 10.99.0.10
spec:
  type: LoadBalancer
  # LAN uniquement — restreindre les sources autorisées
  loadBalancerSourceRanges:
    - 10.10.0.0/24       # VLAN nœuds/admin
  selector:
    # ⚠️ aligner sur les labels réels du pod LAPI : `kubectl -n crowdsec get pod -l type=lapi --show-labels`
    type: lapi
  ports:
    - name: lapi
      port: 8080
      targetPort: 8080
      protocol: TCP
```

> Alternative sans LoadBalancer : les nœuds k3s atteignent déjà le `ClusterIP`
> (kube-proxy programme l'hôte), mais l'IP n'est pas stable et la résolution
> `*.svc.cluster.local` n'existe pas hors-cluster. Le LoadBalancer MetalLB donne
> une IP fixe et routée par BGP → approche recommandée.

> 🔒 Durcissement : `loadBalancerSourceRanges` limite l'accès au VLAN admin. Pour
> du TLS sur la LAPI, activer `tls` dans les values du chart et utiliser
> `crowdsecLapiScheme: https` côté hôtes (sinon le trafic LAPI est en clair sur le
> LAN — acceptable en LAN de confiance, à arbitrer).

### A.2. Côté cluster — enregistrer machines & bouncers des hôtes

Chaque hôte a besoin de **deux** identités auprès de la LAPI : une **machine**
(pour l'agent qui pousse des alertes) et un **bouncer** (pour le firewall-bouncer
qui lit les décisions). On les crée via `cscli` dans le pod LAPI, comme le fait
déjà `init/02-crowdsec-bouncer.yaml` pour Traefik :

```bash
LAPI_POD=$(kubectl -n crowdsec get pod -l type=lapi -o jsonpath='{.items[0].metadata.name}')

# Machines (agents hôtes)
kubectl -n crowdsec exec "$LAPI_POD" -- cscli machines add kube1-host --password '<MDP1>'
kubectl -n crowdsec exec "$LAPI_POD" -- cscli machines add kube2-host --password '<MDP2>'

# Bouncers firewall (un par hôte)
kubectl -n crowdsec exec "$LAPI_POD" -- cscli bouncers add kube1-fw -k '<CLE_FW1>'
kubectl -n crowdsec exec "$LAPI_POD" -- cscli bouncers add kube2-fw -k '<CLE_FW2>'
```

> Idéalement, industrialiser ça en Job ArgoCD calqué sur `02-crowdsec-bouncer.yaml`
> (clés stockées en Secret stable, idempotent) plutôt qu'en commandes manuelles.

### A.3. Côté hôte — agent CrowdSec rattaché à la LAPI distante

```bash
# Dépôt officiel + paquets
curl -s https://install.crowdsec.net | sudo sh
sudo apt-get install -y crowdsec crowdsec-firewall-bouncer-nftables

# Collections SSH/Linux
sudo cscli collections install crowdsecurity/sshd crowdsecurity/linux
```

`/etc/crowdsec/config.yaml` — **désactiver la LAPI locale** et pointer le cluster :

```yaml
api:
  server:
    enable: false          # pas de LAPI locale : on utilise celle du cluster
  client:
    credentials_path: /etc/crowdsec/local_api_credentials.yaml
```

`/etc/crowdsec/local_api_credentials.yaml` (machine créée en A.2) :

```yaml
url: http://10.99.0.10:8080
login: kube1-host
password: <MDP1>
```

`/etc/crowdsec/acquis.yaml` — sources de logs SSH de l'hôte :

```yaml
filenames:
  - /var/log/auth.log
labels:
  type: syslog
---
source: journalctl
journalctl_filter:
  - "_SYSTEMD_UNIT=ssh.service"
labels:
  type: syslog
```

```bash
sudo systemctl enable --now crowdsec
sudo cscli metrics            # vérifie l'acquisition auth.log + lien LAPI
```

### A.4. Côté hôte — firewall-bouncer (nftables) qui applique les décisions

`/etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml` :

```yaml
mode: nftables
update_frequency: 10s
api_url: http://10.99.0.10:8080
api_key: <CLE_FW1>          # clé bouncer créée en A.2
# nftables : table dédiée, ne touche pas aux chaînes k3s
nftables:
  ipv4: { enabled: true, table: crowdsec, chain: crowdsec-chain }
  ipv6: { enabled: true, table: crowdsec6, chain: crowdsec-chain }
deny_action: DROP
```

```bash
sudo systemctl enable --now crowdsec-firewall-bouncer
sudo nft list table inet crowdsec     # vérifie la table de bannissement
```

> ⚠️ **Cohabitation avec le §4** : le firewall-bouncer crée sa **propre** table
> nftables, indépendante des chaînes de k3s et de tes règles INPUT. Il n'interfère
> pas avec le FORWARD géré par k3s. Vérifier l'ordre d'évaluation (priority des
> hooks) pour que le DROP CrowdSec s'applique bien avant l'ACCEPT de tes règles.

### A.5. Vérification de l'unification

```bash
# Sur le pod LAPI : les hôtes apparaissent comme machines + bouncers
kubectl -n crowdsec exec "$LAPI_POD" -- cscli machines list     # kube1-host, kube2-host validés
kubectl -n crowdsec exec "$LAPI_POD" -- cscli bouncers list     # traefik-bouncer + kubeX-fw

# Test : une décision (HTTP ou SSH) est visible partout
kubectl -n crowdsec exec "$LAPI_POD" -- cscli decisions list

# Bannir manuellement une IP de test et vérifier le DROP sur l'hôte
kubectl -n crowdsec exec "$LAPI_POD" -- cscli decisions add --ip 203.0.113.7 --duration 5m
sudo nft list table inet crowdsec | grep 203.0.113.7
```

Une fois en place : un scan SSH bloque l'IP au pare-feu des **deux** nœuds **et**
la décision est partagée avec le bouncer Traefik (et inversement) — une seule
source de vérité pour toute la stack.

---

## Ordre d'exécution recommandé

1. **Constats prioritaires** : `chmod 600` kubeconfig, vérifier SSH, activer
   unattended-upgrades (canal security, no auto-reboot).
2. **Phase 1** sur `kube2` → valider cluster → `kube1`.
3. **Phase 2** (sysctl + FS + modules) sur `kube2` → valider réseau pods/LB → `kube1`.
4. **Phase 3** (auditd, fail2ban, scans) sur les deux nœuds.
5. Baseliner avec `lynis` + `kube-bench` (CIS k3s), corriger le delta, documenter
   les exceptions justifiées (forwarding, modules réseau).

> À chaque étape : `kubectl get nodes -o wide`, pods `Running`, peering BGP
> `Established` côté pfSense, et un service exposé via MetalLB joignable.
