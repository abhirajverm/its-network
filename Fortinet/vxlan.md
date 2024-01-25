





Configuration.

HUB.

Ipsec configuration.
# config vpn ipsec phase1-interface
    edit "VXLAN-IPSEC"
        set type dynamic
        set interface "port3"
        set mode aggressive
        set peertype any
        set net-device disable
        set proposal des-md5 des-sha1
        set dpd on-idle
        set xauthtype auto
        set authusrgrp "Spokes"
        set psksecret secretpassword
        set dpd-retryinterval 60
    next
end
# config vpn ipsec phase2-interface
    edit "vxlan-phase2"
        set phase1name "VXLAN-IPSEC"
        set proposal null-md5
    next
end
# config system interface
    edit "VXLAN-IPSEC"
        set vdom "root"
        set ip 169.254.10.1 255.255.255.255
        set type tunnel
        set remote-ip 169.254.10.254 255.255.255.0
        set snmp-index 13
        set interface "port3"
    next
end
It is possible to use only PSK authentication if needed.
Net-device on the HUB needs to be disabled.
 
Firewall policy is needed to active IPsec tunnel:
# config firewall policy
    edit 1
        set name "ActiveIpsec”
        set srcintf "VXLAN-IPSEC"
        set dstintf "VXLAN-IPSEC"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
VXLAN configuration.
# config system vxlan
    edit "Vxlan2"
        set interface "VXLAN-IPSEC"
        set vni 2
        set remote-ip "169.254.10.3"
    next
    edit "Vxlan1"
        set interface "VXLAN-IPSEC"
        set vni 1
        set remote-ip "169.254.10.2"
    next
end
# config system switch-interface
    edit "VXLAN_SWITCH"
        set vdom "root"
        set member "port5" "Vxlan1" "Vxlan2"
    next
end
For each pair HUB-SpokeX it is necessary to separate VNI.
This VNI needs to be the same.
o between HUB and firewall1 VNI is 1, between HUB and spoke2 VNI is 2.
On HUB, only option how to bound all interfaces together is software switch.
Recommending keep intra-switch-policy implicit.
Port2 is LAN interface behind HUB.
If needed, it is possible to configure IP address on software switch.

firewall1.

IPsec configuration.
# config vpn ipsec phase1-interface
    edit "VXLAN-IPSEC"
        set interface "port3"
        set mode aggressive
        set peertype any
        set net-device disable
        set proposal des-md5 des-sha1
        set localid "Spokes"
        set xauthtype client
        set authusr "fgta"
        set authpasswd secretuserpassowrd
        set remote-gw 172.16.0.1
        set psksecret secretpassowrd
    next
end
# config vpn ipsec phase2-interface
    edit "phase2Vxlan"
        set phase1name "VXLAN-IPSEC"
        set proposal null-md5
        set src-subnet 169.254.10.2 255.255.255.255 <<< Tunnel IP address
    next
end
# config system interface
    edit "VXLAN-IPSEC"
        set vdom "root"
        set ip 169.254.10.2 255.255.255.255
        set type tunnel
        set remote-ip 169.254.10.1 255.255.255.0
        set snmp-index 13
        set interface "port3"
    next   
end
Important is to make sure, the Tunnel has configured correct IP address.
Again, similar to HUB, firewall policy is needed to activate the IPsec tunnel.
 
VXLAN configuration.
# config system vxlan
    edit "Vxlan1"
        set interface "VXLAN-IPSEC"
        set vni 1
        set remote-ip "169.254.10.1”
    next
end
# config system switch-interface
    edit "VXLAN_SWITCH"
        set vdom "root"
        set member "port5" "Vxlan1"
    next
end
On spoke, there is flexibility to use software switch or Virtual-wire pair.
Based on this, if  Virtual-wire pair is chosen, it is necessary to have virtual-wire pair policy between LAN interface and VXLAN interface.
If software switch is chosen, if intra-switch-policy is implicit, no firewall policy is needed.
If intra-switch-policy is explicit, additional firewall policy between LAN interface and VXLAN interface is needed.

firewall2.

Configuration for firewall2 is the same as firewall1, except couple of things.
Tunnel IP address needs to be correct, in our case spoke2 has 169.154.10.3. Based on this adjust src-subnet in phase2.
 
For VXLAN part, VNI will be 2 to be matching HUB’s configuration.
# config system vxlan
    edit "Vxlan1"
        set interface "VXLAN-IPSEC"
        set vni 2
        set remote-ip "169.254.10.1"
    next
end
# config system switch-interface
    edit "VXLAN_SWITCH"
        set vdom "root"
        set member "port5" "Vxlan1"
    next
end
Verification.

Client1 behind spoke1 has IP address 10.43.0.8.
Client2 behind spoke2 has IP address 10.43.0.18.
As a result of the configuration, if everything is configured correctly, Client1 should be able to reach Client2.
The traffic will go from firewall1 to HUB and to firewall2.
server@client1:~# ping 10.43.0.18                                                                                                                                                                                 
PING 10.43.0.18 (10.43.0.18) 56(84) bytes of data.                                                                                                                                                           
64 bytes from 10.43.0.18: icmp_seq=1 ttl=64 time=4.84 ms                                                                                                                                                        
64 bytes from 10.43.0.18: icmp_seq=2 ttl=64 time=3.89 ms                                                                                                                                                        
64 bytes from 10.43.0.18: icmp_seq=3 ttl=64 time=2.35 ms                                                                                                                                                        
^C                                                                                                                                                                                                                 
--- 10.43.0.18 ping statistics ---                                                                                                                                                                              
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
                                                                                                                                                 
Sniffer from each unit:
firewall1# diag sniffer packet any "host 10.43.0.18 and icmp" 4 0 l
Using Original Sniffing Mode
interfaces=[any]
filters=[host 10.43.0.18 and icmp]
2021-05-12 04:49:52.326749 port5 in 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:49:52.326906 Vxlan1 out 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:49:52.329074 Vxlan1 in 10.43.0.18 -> 10.43.0.8: icmp: echo reply
2021-05-12 04:49:52.329088 port5 out 10.43.0.18 -> 10.43.0.8: icmp: echo reply
2021-05-12 04:49:59.526139 port5 in 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:49:59.526247 Vxlan1 out 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:49:59.527629 Vxlan1 in 10.43.0.18 -> 10.43.0.8: icmp: echo reply
2021-05-12 04:49:59.527651 port5 out 10.43.0.18 -> 10.43.0.8: icmp: echo reply

HUB #  diag sniffer packet any "host 10.43.0.18 and icmp" 4 0 l
Using Original Sniffing Mode.
interfaces=[any]
filters=[host 10.43.0.18 and icmp]
2021-05-12 04:49:52.918545 Vxlan1 in 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:49:52.918569 Vxlan2 out 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:49:52.919803 Vxlan2 in 10.43.0.18 -> 10.43.0.8: icmp: echo reply
2021-05-12 04:49:52.919808 Vxlan1 out 10.43.0.18 -> 10.43.0.8: icmp: echo reply
2021-05-12 04:50:00.117684 Vxlan1 in 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:50:00.117697 Vxlan2 out 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:50:00.118503 Vxlan2 in 10.43.0.18 -> 10.43.0.8: icmp: echo reply
2021-05-12 04:50:00.118507 Vxlan1 out 10.43.0.18 -> 10.43.0.8: icmp: echo reply

firewall2 #  diag sniffer packet any "host 10.43.0.18 and icmp" 4 0 l
Using Original Sniffing Mode
interfaces=[any]
filters=[host 10.43.0.18 and icmp]
2021-05-12 04:49:52.665259 Vxlan1 in 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:49:52.665283 port5 out 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:49:52.665727 port5 in 10.43.0.18 -> 10.43.0.8: icmp: echo reply
2021-05-12 04:49:52.665731 Vxlan1 out 10.43.0.18 -> 10.43.0.8: icmp: echo reply
2021-05-12 04:49:59.864173 Vxlan1 in 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:49:59.864186 port5 out 10.43.0.8 -> 10.43.0.18: icmp: echo request
2021-05-12 04:49:59.864529 port5 in 10.43.0.18 -> 10.43.0.8: icmp: echo reply
2021-05-12 04:49:59.864535 Vxlan1 out 10.43.0.18 -> 10.43.0.8: icmp: echo reply
