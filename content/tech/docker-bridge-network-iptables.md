---
title: "Docker Bridge Networks and iptables"
date: 2023-01-10
summary: "How DNS resolution works, Network isolation is provided and exposing a container service to outside world works in Docker bridge networks with iptables"
tags: ["docker","iptables"]
author: "Clement Thomas"
toc: true
---

Docker provides support for multiple network drivers. The default network driver being ___bridge___ network. Docker creates ___docker0___ bridge and connects all the docker containers created in the host to it, unless we specify a different network with ___--network___ option. 

Docker's bridge network [documentation](https://docs.docker.com/network/bridge/) recommends user-defined bridge over using the default docker created bridge, some of the advantages of user-defined bridges are:

* user-defined bridges provide automatic dns resolution. Containers can resolve each other by using the name or alias. In default docker bridge, we should use ___--link___ option to achieve this functionality, which is deprecated already. 

* user-defined bridges provide better isolation since only the containers connected to the same bridge network can communicate with each other. Docker adds iptables rules to prevent communication between bridges. 

Docker's bridge network [documentation](https://docs.docker.com/network/bridge/) provides few more advantages of using user-defined bridges. But in this article,we will see

1. how dns resolution happens and works for resolving containers and other dns names.
2. how is network isolation provided?
3. how are we able to access the service(s) running in a container,by exposing it on a port.

### Demo setup

The below tests are done on ubuntu 20.04 vm running Docker engine version 20.10.17 

Let us create a user-defined bridge network called ___demo-net___ and create an nginx container named ___web___ and alphine container named ___cli___, both attached to demo-net. Our linux vm has an ethernet connection ___eth0___ with ip ___10.44.54.64___ and the dns server configured in /etc/resolv.conf is ___10.40.50.60___

![Setup](/img/tech/docker-bridge/docker-bridge-network.png)

{{< highlight shell >}}
docker network create --driver bridge demo-net

docker run -d --network demo-net -p 0.0.0.0:8080:80 --name web nginx

docker run -dit --network demo-net --name cli alpine ash

{{< /highlight >}}

When inspecting the demo-net bridge network, we can see both web and cli containers attached to it. Also to access our web container from outside,we are using ___-p 0.0.0.0:8080:80___ so anyone can reach the web container by connecting to port 8080 on the vm. 

{{< highlight json >}}
docker network create --driver bridge demo-net
docker network inspect demo-net
"Containers": {
            "303c085ffb99ab2d429a4f0b58138ba4f9a2560f133343c769b5593a1192c50c": {
                "Name": "cli",
                "EndpointID": "afad3f78d70a3c571d88dfed716564297b3903d6726e92dd5a2087519f1f5d0f",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            },
            "aea6a91131135c9f0de56686ea5fa465524f13d6e325415858ba653786166a93": {
                "Name": "web",
                "EndpointID": "4efde27f5d0529776ec9c20515b2bb00dddd0f0c4d0f3ac739b89dca6efcc755",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
{{< /highlight >}}

### DNS resolution

From the cli container, lets try to ping web and google.com

{{< highlight shell >}}
docker container attach cli

/ # ping -c 3 web
PING web (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.215 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.110 ms
64 bytes from 172.18.0.2: seq=2 ttl=64 time=0.160 ms

--- web ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.110/0.161/0.215 ms

/ # ping -c 3 google.com
PING google.com (142.250.194.142): 56 data bytes
64 bytes from 142.250.194.142: seq=0 ttl=115 time=3.610 ms
64 bytes from 142.250.194.142: seq=1 ttl=115 time=3.749 ms
64 bytes from 142.250.194.142: seq=2 ttl=115 time=3.804 ms

--- google.com ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 3.610/3.721/3.804 ms

{{< /highlight >}}

while in another terminal running a tcpdump to inspect dns traffic.

{{< highlight shell >}}
sudo tcpdump -i any -nn udp and port 53
[sudo] password for user:
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked v1), capture size 262144 bytes
08:45:43.502942 IP 172.18.0.3.51782 > 10.40.50.60.53: 9599+ A? google.com. (28)
08:45:43.502942 IP 172.18.0.3.51782 > 10.40.50.60.53: 9599+ A? google.com. (28)
08:45:43.502985 IP 172.18.0.3.42132 > 10.40.50.60.53: 9912+ AAAA? google.com. (28)
08:45:43.503012 IP 10.44.54.64.51782 > 10.40.50.60.53: 9599+ A? google.com. (28)
08:45:43.502985 IP 172.18.0.3.42132 > 10.40.50.60.53: 9912+ AAAA? google.com. (28)
08:45:43.503039 IP 10.44.54.64.42132 > 10.40.50.60.53: 9912+ AAAA? google.com. (28)
08:45:43.503288 IP 10.40.50.60.53 > 10.44.54.64.51782: 9599 1/0/0 A 142.250.194.142 (44)
08:45:43.503301 IP 10.40.50.60.53 > 172.18.0.3.51782: 9599 1/0/0 A 142.250.194.142 (44)
08:45:43.503308 IP 10.40.50.60.53 > 172.18.0.3.51782: 9599 1/0/0 A 142.250.194.142 (44)
08:45:43.503328 IP 10.40.50.60.53 > 10.44.54.64.42132: 9912 0/0/0 (28)
08:45:43.503332 IP 10.40.50.60.53 > 172.18.0.3.42132: 9912 0/0/0 (28)
{{< /highlight >}}

Here there is no dns request, sent for web but our container still resolved the correct ip for the web container. But for google.com we can see tcpdump showing the actual dns query being made by the cli container.

The DNS resolution for container names or aliases is handled by the ___Docker's embedded DNS server___ and so no actual dns call is made and hence there is no output in terminal running tcpdump. More on this in [here](https://docs.docker.com/config/containers/container-networking/)

Though the cli container makes the dns request, we can see the same request is sent out through our eth0 interface (also bridge network is not an actual network visible outside our linux host).

### Network isolation

The dns resolution done above, network isolation and also the exposing of the container service in a port on the host vm are all achieved through ___iptables___ rules.  

Let us inspect the iptables rules in the vm. Though ___iptables -nL___ gives the rules, the output of ___iptables-save___ is bit more easier to understand and so we will use the latter.

{{< highlight shell >}}
root@linux-vm:~# iptables-save
# Generated by iptables-save v1.8.4 on Wed Jan  4 09:07:50 2023
*filter
:INPUT ACCEPT [1023555:889414063]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [506329:266904587]
:DOCKER - [0:0]
:DOCKER-ISOLATION-STAGE-1 - [0:0]
:DOCKER-ISOLATION-STAGE-2 - [0:0]
:DOCKER-USER - [0:0]
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-ISOLATION-STAGE-1
-A FORWARD -o br-44a58be29ddf -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o br-44a58be29ddf -j DOCKER
-A FORWARD -i br-44a58be29ddf ! -o br-44a58be29ddf -j ACCEPT
-A FORWARD -i br-44a58be29ddf -o br-44a58be29ddf -j ACCEPT
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A DOCKER -d 172.18.0.2/32 ! -i br-44a58be29ddf -o br-44a58be29ddf -p tcp -m tcp --dport 80 -j ACCEPT
-A DOCKER-ISOLATION-STAGE-1 -i br-44a58be29ddf ! -o br-44a58be29ddf -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -j RETURN
-A DOCKER-ISOLATION-STAGE-2 -o br-44a58be29ddf -j DROP
-A DOCKER-ISOLATION-STAGE-2 -o docker0 -j DROP
-A DOCKER-ISOLATION-STAGE-2 -j RETURN
-A DOCKER-USER -j RETURN
COMMIT

*nat
:PREROUTING ACCEPT [451:184498]
:INPUT ACCEPT [13:1070]
:OUTPUT ACCEPT [6508:559391]
:POSTROUTING ACCEPT [6510:559535]
:DOCKER - [0:0]
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 172.18.0.0/16 ! -o br-44a58be29ddf -j MASQUERADE
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A POSTROUTING -s 172.18.0.2/32 -d 172.18.0.2/32 -p tcp -m tcp --dport 80 -j MASQUERADE
-A DOCKER -i br-44a58be29ddf -j RETURN
-A DOCKER -i docker0 -j RETURN
-A DOCKER ! -i br-44a58be29ddf -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.18.0.2:80
COMMIT
{{< /highlight >}}

Thats a lot of rules, Lets take few rules at a time to understand. But before a quick summary about iptables. 

iptables is the command used to configure the linux kernel's packet filtering subsystem. 

* iptables comes with three built-in tables ___filter,mangle and nat___. and if we dont specify
any table in the command, ___filter___ is the default. 
* Each table has one or more ___built-in chains(INPUT,OUTPUT,PREROUTING,POSTROUTING,FORWARD)___ which in turn has rules which determine what to do with a packet. ie. ACCEPT, DROP etc. 
* In addition to pre-configured chains, we can also create custom chains and use it. The following are the tables and their built-in chains. Also each chain has the default policy, ie. what to do when none of the rules match it. 


{{< highlight shell >}}
*filter
:INPUT ACCEPT [1023555:889414063]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [506329:266904587]
:DOCKER - [0:0]
:DOCKER-ISOLATION-STAGE-1 - [0:0]
:DOCKER-ISOLATION-STAGE-2 - [0:0]
:DOCKER-USER - [0:0]
...
...
*nat
:PREROUTING ACCEPT [451:184498]
:INPUT ACCEPT [13:1070]
:OUTPUT ACCEPT [6508:559391]
:POSTROUTING ACCEPT [6510:559535]
:DOCKER - [0:0]
{{< /highlight >}}

Here ___*filter___ and ___*nat___ are table names and the list of chains that it has (both built-in and custom chains) are shown below with their default policies. Docker has created ___DOCKER___, ___DOCKER-USER___, ___DOCKER-ISOLATION-STAGE-1___ and ___DOCKER-ISOLATION-STAGE-2___ custom chains. 

The following are the rules that provide isolation,so that we cant talk from one bridge network to another. 

{{< highlight shell >}}
-A FORWARD -j DOCKER-ISOLATION-STAGE-1
...
-A DOCKER-ISOLATION-STAGE-1 -i br-44a58be29ddf ! -o br-44a58be29ddf -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -j RETURN
-A DOCKER-ISOLATION-STAGE-2 -o br-44a58be29ddf -j DROP
-A DOCKER-ISOLATION-STAGE-2 -o docker0 -j DROP
-A DOCKER-ISOLATION-STAGE-2 -j RETURN
...
{{< /highlight >}}

Here ___docker0___ represents the default bridge created by docker and bridge interface name on the vm for demo-net is ___br-44a58be29ddf___

in ___FORWARD___ chain of filter table, the first rule says jump (-j) to ___DOCKER-ISOLATION-STAGE-1___ chain. In ___DOCKER-ISOLATION-STAGE-1___ we check if the packet originates from 
a bridge interface and is not to the same bridge interface,if so, we pass the control to ___DOCKER-ISOLATION-STAGE-2___ where the packet gets dropped.

### Network Address translation to expose container service

The following are the rules that will send out the packets originating from bridge to outside world using ___Network address translation___(MASQUERADE is a special form of Source NATing in which the NAT host, vm in our case, modifies the sender address to that of itself before sending the packet out),so containers can reach outside world. 

{{< highlight shell >}}
-A POSTROUTING -s 172.18.0.0/16 ! -o br-44a58be29ddf -j MASQUERADE
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
{{< /highlight >}}

Here if the packet originates from the bridge ip range, but is not destined for the bridge network itself, then MASQUERADE (NAT the request)

And finally, anyone from outside world can reach the service running in a container because of the following rule. 

{{< highlight shell >}}
*nat
...
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
...
-A DOCKER ! -i br-44a58be29ddf -p tcp -m tcp --dport 8080 -j DNAT --to-destination 172.18.0.2:80
{{< /highlight >}}

so when the traffic is not from within bridge network and is for the published port, then do the destination NATing to modify the destination address to that of the container. 

Thats it for now folks.
