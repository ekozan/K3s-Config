# Ansible — hardening des nœuds k3s

Partie **OS** du plan de durcissement (`../ubuntu-hardening.md`). ArgoCD gère le
in-cluster ; Ansible gère ce qui vit hors de Kubernetes : SSH, sysctl, nftables,
paquets apt, permissions, modules noyau, auditd, et l'agent CrowdSec + le
firewall-bouncer des hôtes (rattachés à la LAPI du cluster).

## Principe

- **Idempotent** : rejouable sans effet de bord.
- **Sûr par défaut** : les tâches qui peuvent couper l'accès (SSH, nftables) sont
  **désactivées par défaut** et s'activent explicitement par variable, une fois
  testées. On applique d'abord sur `kube2`, on valide le cluster, puis `kube1`.
- **Aligné k3s** : on ne touche jamais à `ip_forward`, au FORWARD, ni aux modules
  réseau dont k3s dépend (cf. tableau des pièges dans `ubuntu-hardening.md`).

## Prérequis

- Ansible 2.14+ sur la machine de contrôle.
- Accès SSH par clé aux nœuds (compte sudo).
- Collection : `ansible-galaxy collection install ansible.posix` (module `sysctl`).

## Récupérer les identifiants CrowdSec générés par ArgoCD

Le Job `crowdsec-host-register` (agrocd-home) crée un Secret. On en extrait le mot
de passe machine et la clé bouncer de chaque hôte, à passer en variables Ansible
(idéalement via `ansible-vault`) :

```bash
NS=crowdsec
kubectl -n $NS get secret crowdsec-host-credentials \
  -o jsonpath='{.data.kube1-host\.password}' | base64 -d ; echo
kubectl -n $NS get secret crowdsec-host-credentials \
  -o jsonpath='{.data.kube1-fw\.key}' | base64 -d ; echo
```

Renseigner ensuite par hôte (cf. `inventory.example.ini`) :
`crowdsec_machine_password`, `crowdsec_bouncer_key`.

## Utilisation

```bash
cp inventory.example.ini inventory.ini   # puis adapter IP / creds

# 1. Run "à blanc" sur kube2 (rien d'appliqué, on lit le diff)
ansible-playbook -i inventory.ini site.yml --limit kube2 --check --diff

# 2. Application réelle sur kube2 (tâches sûres uniquement par défaut)
ansible-playbook -i inventory.ini site.yml --limit kube2

# 3. Une fois validé, activer SSH + nftables (variables) puis kube1
ansible-playbook -i inventory.ini site.yml --limit kube1
```

## Ordre conseillé d'activation des flags risqués

1. `harden_ssh: true` — **garder une session SSH ouverte** pendant le reload.
2. `harden_nftables: true` — valider le peering BGP et l'accès après application.
3. `harden_nodeport_guard: true` — bloque la fuite NodePort (cf. §4bis).

Voir `roles/hardening/defaults/main.yml` pour toutes les variables.
