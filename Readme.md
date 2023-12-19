```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_VERSION=v1.25.9+k3s1 sh -s - --disable traefik,servicelb --cluster-init --cluster-cidr=10.42.0.0/16,2001:cafe:42:0::/56 --service-cidr=10.43.0.0/16,2001:cafe:42:1::/112
```

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
```

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: home
  namespace: metallb-system
spec:
  addresses:
  - 10.99.0.1/24
  avoidBuggyIPs: true
---
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: kubemaster1
  namespace: metallb-system
spec:
  myASN: 65002
  peerASN: 65000
  peerAddress: 10.10.0.1

---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: local
  namespace: metallb-system
spec:
  ipAddressPools:
  - home
  peers:
  - kubemaster1
```

cp max@kube1.home:/etc/rancher/k3s/k3s.yaml ~/kubeconfig

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.11.1 \
  --create-namespace \
  --set installCRDs=true
  
  or 
  kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
  
```

```bash
helm repo add traefik https://helm.traefik.io/traefik 
helm repo update
helm install --namespace=traefik-v2  --create-namespace    traefik traefik/traefik
```


```
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: traefik.padok.fr
  namespace: traefik-system
spec:
  dnsNames:
    - traefik.padok.fr
  secretName: traefik.padok.fr
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
    
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik-system
spec:
  entryPoints:
    - web
    - websecure
  routes:
    - match: Host(`traefik.padok.fr`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
      middlewares:
        - name: auth
  # Use the secret generated by cert-manager
  tls:
    secretName: traefik.padok.fr
```


```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
kubectl create namespace cattle-system
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.server \
  --set bootstrapPassword=admin \
  --values rancher.value
  
```


```yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ffd-link
  namespace: certmanager
spec:
  secretName: ffd-link-tls
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  commonName: "*.ffd.link"
  dnsNames:
  - "ffd.link"
  - "*.ffd.link"
  secretTemplate:
      annotations:
        replicator.v1.mittwald.de/replicate-to: "*" #"my-ns-1,namespace-[0-9]*"
```


```bash
 helm repo add mittwald https://helm.mittwald.de
 helm repo update
 helm upgrade --install kubernetes-replicator mittwald/kubernetes-replicator
```

```yaml

apiVersion: v1
kind: Secret
metadata:
  name: ffd-link-tls
  namespace: aaaaaa
  annotations:
    replicator.v1.mittwald.de/replicate-from: certmanager/ffd-link-tls
data: {}
```



