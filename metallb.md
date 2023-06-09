

My Pfsense FRR config

```
##################### DO NOT EDIT THIS FILE! ######################
###################################################################
# This file was created by an automatic configuration generator.  #
# The contents of this file will be overwritten without warning!  #
###################################################################
!
frr defaults traditional
hostname heimdall.home
password **********
service integrated-vtysh-config
!
ip router-id 10.10.0.1
!
router bgp 65000
 bgp router-id 10.10.0.1
 no bgp network import-check
 no bgp ebgp-requires-policy
 neighbor metallb peer-group
 neighbor metallb remote-as 65002
 neighbor 10.10.0.130 peer-group metallb
 neighbor 10.10.0.130 remote-as 65002
 neighbor 10.10.0.130 description KubeMaster1
 neighbor 10.10.0.130 bfd
 neighbor 10.10.0.131 peer-group metallb
 neighbor 10.10.0.131 remote-as 65002
 neighbor 10.10.0.131 description KubeMaster2
 neighbor 10.10.0.131 bfd
 !
 address-family ipv4 unicast
  network 10.99.0.1/24
  neighbor 10.10.0.130 activate
  neighbor 10.10.0.131 activate
  no neighbor metallb send-community
  no neighbor 10.10.0.130 send-community
  no neighbor 10.10.0.131 send-community
 exit-address-family
 !
 address-family ipv6 unicast
  network FC00:99::/32
  no neighbor metallb send-community
 exit-address-family
 !
!
route-map Allow-All permit 100
 description Permis any route
!
bfd
 profile std
  detect-multiplier 3
  receive-interval 300
  transmit-interval 300
  no shutdown
  minimum-ttl 254
 !
 peer 10.10.0.130 local-address 10.10.0.1 interface igb2.10
  profile std
 !
 peer 10.10.0.131 local-address 10.10.0.1 interface igb2.10
  profile std
 !
!
line vty
!
end

```

My metal lb config 

```Yaml
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: PeerA
  namespace: metallb-system
spec:
  myASN: 64502
  peerASN: 64500
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
 
 
 My try to get debug info  : 
 
 
 ```
 kubectl -n metallb-system exec speaker-q4sdb  -c frr -- vtysh -c "show running-config"
 ```
 
 ```
 Building configuration...

Current configuration:
!
frr version 7.5.1_git
frr defaults traditional
hostname kube1
log file /etc/frr/frr.log informational
log timestamp precision 3
service integrated-vtysh-config
!
router bgp 65002
 bgp router-id 10.10.0.1
 no bgp ebgp-requires-policy
 no bgp default ipv4-unicast
 no bgp network import-check
 neighbor 10.10.0.1 remote-as 65000
 neighbor 10.10.0.1 bfd profile bfdprofiledef
 neighbor 10.10.0.1 timers 30 90
 !
 address-family ipv4 unicast
  network 10.99.0.1/32
  network 10.99.0.2/32
  neighbor 10.10.0.1 activate
  neighbor 10.10.0.1 route-map 10.10.0.1-in in
  neighbor 10.10.0.1 route-map 10.10.0.1-out out
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor 10.10.0.1 activate
  neighbor 10.10.0.1 route-map 10.10.0.1-in in
  neighbor 10.10.0.1 route-map 10.10.0.1-out out
 exit-address-family
!
ip prefix-list 10.10.0.1-pl-ipv4 seq 5 permit 10.99.0.2/32
ip prefix-list 10.10.0.1-pl-ipv4 seq 10 permit 10.99.0.1/32
ip prefix-list 10.10.0.1-pl-ipv4 seq 15 deny any
!
ipv6 prefix-list 10.10.0.1-pl-ipv4 seq 5 deny any
!
route-map 10.10.0.1-in deny 20
!
route-map 10.10.0.1-out permit 1
 match ip address prefix-list 10.10.0.1-pl-ipv4
!
route-map 10.10.0.1-out permit 2
 match ipv6 address prefix-list 10.10.0.1-pl-ipv4
!
ip nht resolve-via-default
!
ipv6 nht resolve-via-default
!
line vty
!
bfd
 profile bfdprofiledef
 !
!
end
 ```
 
 
 ```
  kubectl -n metallb-system exec speaker-q4sdb  -c frr -- vtysh -c "show bfd peers brief"
 ```
 
 ```
 Session count: 0
SessionId  LocalAddress                             PeerAddress                             Status         
=========  ============                             ===========                             ======   
 ```
 
