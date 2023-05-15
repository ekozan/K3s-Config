



```Yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: PeerA
  namespace: metallb-system
spec:
  myASN: 64500
  peerASN: 64501
  peerAddress: 10.10.0.1
   bfdProfile: defbfdprofile
```

```Yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: Pool1
  namespace: metallb-system
spec:
  addresses:
  - 10.99.0.2-10.99.0.2
 ```
 
 ```Yaml
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgp
  namespace: metallb-system
 ```
 
  
 ```Yaml
apiVersion: metallb.io/v1beta1
kind: BFDProfile
metadata:
  name: defbfdprofile
  namespace: metallb-system
spec:
  receiveInterval: 300
  transmitInterval: 300
 ```
 
 
