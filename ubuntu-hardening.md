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
- **fail2ban** sur `sshd` (en plus de CrowdSec qui couvre le HTTP via Traefik) ;
  ou déployer le scénario SSH de CrowdSec directement sur l'hôte pour unifier.
- Exporter les logs hors-nœud (TrueNAS/syslog/Loki) pour préserver les preuves en
  cas de compromission d'un nœud.

### §11. Industrialisation & vérification

- **Ansible** : transformer ce plan en rôle idempotent versionné dans `K3s-Config`
  (`roles/hardening/`). Avantages : reproductible, revue par diff, rollback facile.
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
