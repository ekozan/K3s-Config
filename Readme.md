```bash
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_VERSION=v1.25.9+k3s1 sh -s - --disable traefik,servicelb
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

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.2.0 \
  --create-namespace \
  --set installCRDs=true
```

```bash
helm repo add traefik https://helm.traefik.io/traefik 
helm repo update
kubectl create ns traefik-v2
helm install --namespace=traefik-v2     traefik traefik/traefik
```



