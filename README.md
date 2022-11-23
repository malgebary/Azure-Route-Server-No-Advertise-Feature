# **Azure Route Server No-Advertise BGP Community**

# Contents
[Introduction](#introduction)

[Notes](#Notes)

[Lab](#lab)


## Introduction
Azure Route Server (ARS) has recently started supporting the BGP community of No Advertise, when a No-Advertise community is attached to a route, the BGP speaker won't advertise the route to any internal or external BGP peers. So when ARS gets a route tagged with this community, ARS will learn the route but will not forwarded it after that to any peers including the VPN gateway or the ExpressRoute gateway. The main advantage of this feature is to control the number of routes being pushed to the ExpressRoute gateway where it has limit for number of routes advertised toward this gateway from the VNet which is 1000 routes, so in scenarios where there is SDWAN NVA in the VNet and you are using ARS to inject SDWAN learned routes to the VM's effective route table and at the same time you don't want to push those routes down to on-prem then you can use the No Advertise community, this community also helpful in case you want to advertise 0/0 in Azure but not to be passed down to on-prem.

## Notes

In this Lab you will creat the following components: 
- This Lab is built using Azure Cli
- Hub VNet (10.1.0.0/16), Spoke VNet (172.16.3.0), and on-prem VNet (192.168.0.0/16).
- In the HUB VNet we will have two CSRs behind ILB also will have ARS, the CSRs will peer with ARS and use the Next Hop ip feature to point the traffic to the ILB front end ip for load balancing, and will also have VPN GW. 
- On-prem VNet will have another CSR that represnet on-prem edge router that will termiante the VPN tunnel with the HUB VPN GW, and the advertised routes will be verfied using this CSR.
- Spoke VNet will have a VM to test connectivity

![image](https://user-images.githubusercontent.com/78562461/203482792-8b22d70e-57b9-4ae7-b8c0-83f718678528.png)

## Lab

#Paramaters

```
loc=SouthCentralUS
rg=Nexthop
username=azureuser
password="MyP@SSword123!"
vmsize=Standard_D2_v2

#Create the RG
az group create -n $rg -l $loc --output none

#Create the VNETs (Hubs+Spoke) and VPN Branches
az network vnet create --address-prefixes 10.1.0.0/16 -n hub -g $rg -l $loc --output none
az network vnet subnet create --name RouteServerSubnet --address-prefixes 10.1.0.0/24 --resource-group $rg --vnet-name hub --output none
az network vnet subnet create --name csroutside --address-prefixes 10.1.2.0/24  --resource-group $rg --vnet-name hub --output none
az network vnet subnet create --name csrinside --address-prefixes 10.1.1.0/24 --resource-group $rg --vnet-name hub --output none
az network vnet subnet create --name GatewaySubnet --address-prefixes 10.1.3.0/24 --resource-group $rg --vnet-name hub --output none


az network vnet create --address-prefixes 172.16.3.0/24 -n spoke -g $rg -l $loc --output none 
az network vnet subnet create --name vmsubnet --address-prefixes 172.16.3.240/29 --resource-group $rg --vnet-name spoke --output none


#Create the NVAs (CSR) in each hub
az vm image terms accept --urn cisco:cisco-csr-1000v:17_3_4a-byol:latest --output none

az network nic create --name CSROutsideIntprimary -g $rg --subnet csroutside --vnet hub --ip-forwarding true --location $loc --output none
az network nic create --name CSRInsideIntprimary -g $rg --subnet csrinside --vnet hub --ip-forwarding true --location $loc --output none
az vm create --resource-group $rg --location $loc --name hubCSR1 --size $vmsize --nics CSROutsideIntprimary CSRInsideIntprimary --image cisco:cisco-csr-1000v:17_3_4a-byol:latest --admin-username azureuser --admin-password MyP@SSword123!

az vm image terms accept --urn cisco:cisco-csr-1000v:17_3_4a-byol:latest --output none

az network nic create --name CSROutsideIntsecondary -g $rg --subnet csroutside --vnet hub --ip-forwarding true --location $loc --output none
az network nic create --name CSRInsideIntsecondary -g $rg --subnet csrinside --vnet hub --ip-forwarding true --location $loc --output none
az vm create --resource-group $rg --location $loc --name hubCSR2 --size $vmsize --nics CSROutsideIntsecondary CSRInsideIntsecondary --image cisco:cisco-csr-1000v:17_3_4a-byol:latest --admin-username azureuser --admin-password MyP@SSword123!


az network public-ip create --resource-group $rg --name ExtLBIP --sku Standard --zone 1 2 3
az network lb create --resource-group $rg --name ExtLB --sku Standard --public-ip-address ExtLBIP --frontend-ip-name FrontEndIP --backend-pool-name csrpool
az network lb probe create --resource-group $rg --lb-name ExtLB --name HealthProbe --protocol tcp --port 22

az network lb rule create --resource-group $rg --lb-name ExtLB --name HTTPRule --protocol tcp --frontend-port 80 --backend-port 80 --frontend-ip-name FrontEndIP --backend-pool-name csrpool --probe-name HealthProbe --disable-outbound-snat true --idle-timeout 15 --enable-tcp-reset true

#Create the peering between spoke and hub vnets
az network vnet peering create -g $rg -n HubtoSpoke --vnet-name hub --remote-vnet spoke --allow-vnet-access --allow-forwarded-traffic --output none
az network vnet peering create -g $rg -n SpoketoHub --vnet-name spoke --remote-vnet hub --allow-vnet-access --allow-forwarded-traffic --output none

echo Creating Azure Internal Load Balancer...
az network lb create --name Int_LB --resource-group $rg --sku Standard --private-ip-address 10.1.1.254 --vnet-name hub --subnet csrinside --backend-pool-name ILBBackendpool

echo Creating ILB health probe to connect to port 80 with minimal possible interval...
az network lb probe create --resource-group $rg --lb-name Int_LB --name ILBHealthProbe --protocol tcp --port 22 --interval 5 --threshold 2

echo Creating ILB rule to forward all packets to the CSRs...
az network lb rule create --resource-group $rg --lb-name Int_LB --name ILBPortsRule --protocol All --frontend-port 0 --backend-port 0 --backend-pool-name ILBBackendpool --probe-name ILBHealthProbe

#Create the ARS in hubs and spoke
#Create the Pips
az network public-ip create --name HubRouteServerIP --resource-group $rg  --version IPv4 --sku Standard --output none

#Get the ARS SubnetIds
arshubsubnet_id=$(az network vnet subnet show --name RouteServerSubnet --resource-group $rg --vnet-name hub --query id -o tsv)
echo @arshubsubnet_id

az network routeserver create --name HubRouteServer --resource-group $rg --hosted-subnet $arshubsubnet_id --public-ip-address HubRouteServerIP --output none

az network routeserver peering create --name hubCSR1 --peer-ip 10.1.1.4 --peer-asn 65002 --routeserver HubRouteServer --resource-group $rg --output none
az network routeserver peering create --name hubCSR2 --peer-ip 10.1.1.5 --peer-asn 65002 --routeserver HubRouteServer --resource-group $rg --output none


az network routeserver show --name HubRouteServer --resource-group $rg
   10.1.0.4
   10.1.0.5

router bgp 65002 <--HubCSR
neighbor 10.1.0.4 remote-as 65515
neighbor 10.1.0.4 ebgp-multihop 255
neighbor 10.1.0.5 remote-as 65515
neighbor 10.1.0.5 ebgp-multihop 255

	 !
address-family ipv4
network 0.0.0.0/0
neighbor 10.1.0.4 activate
neighbor 10.1.0.5 activate
neighbor 10.1.0.4 route-map lbnexthop out
neighbor 10.1.0.5 route-map lbnexthop out
exit-address-family
	!
route-map lbnexthop permit 10

set ip next-hop 10.1.1.254

!

line vty

!
add static route to ARS subnet that point to the default gateway of the CSR Internal subnet to avoid recursive routing failure for ARS BGP endpoints learned via BGP
ip route 10.1.0.0 255.255.255.0 10.1.1.1

az network routeserver peering list-learned-routes --name hubcsr1 --routeserver HubRouteServer --resource-group $rg


router bgp 65002 <--Hub2CSR
neighbor 10.1.0.4 remote-as 65515
neighbor 10.1.0.4 ebgp-multihop 255
neighbor 10.1.0.5 remote-as 65515
neighbor 10.1.0.5 ebgp-multihop 255

!
address-family ipv4
neighbor 10.1.0.4 activate
neighbor 10.1.0.5 activate
exit-address-family
!
add static route to ARS subnet that point to the default gateway of the CSR Internal subnet to avoid recursive routing failure for ARS BGP endpoints learned via BGP
ip route 10.1.0.0 255.255.255.0 10.1.1.1

# Add Virtual Network Gateway to the HUb VNet:

To allow route transit between ARS and virtual network gateway, the latter need to be in active-active mode.

az network public-ip create --name HUB-VNG-PIP1 --resource-group $rg --allocation-method Dynamic --location $loc
az network public-ip create --name HUB-VNG-PIP2 --resource-group $rg --allocation-method Dynamic --location $loc
az network vnet-gateway create --name HUB-VNG --public-ip-address HUB-VNG-PIP1 HUB-VNG-PIP2 --resource-group $rg --vnet HUB --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --asn 65515  --location $loc --bgp-peering-address 10.1.3.4 10.1.3.5


# Create On-Prem VNet:

az network vnet create --resource-group $rg --name On-PremVNet --location $loc --address-prefixes 192.168.0.0/16 --subnet-name External --subnet-prefix 192.168.0.0/24 
az network vnet subnet create --address-prefix 192.168.1.0/24 --name Internal --resource-group $rg --vnet-name On-PremVNet

# Configure On-prem CSR:

az network public-ip create --name OnPremCSRPublicIP -g $rg --idle-timeout 30 --allocation-method Static --location $loc
az network nic create --name OnPremCSROutsideInterface -g $rg --subnet External --vnet On-PremVNet --public-ip-address OnPremCSRPublicIP --ip-forwarding true --private-ip-address 192.168.0.4 --location $loc
az network nic create --name OnPremCSRInsideInterface -g $rg --subnet Internal --vnet On-PremVNet --ip-forwarding true --private-ip-address 192.168.1.4 --location $loc

az vm create -g $rg --location $loc --name On-PremCSR --size Standard_DS3_v2 --nics OnPremCSROutsideInterface OnPremCSRInsideInterface --image cisco:cisco-csr-1000v:17_2_1-byol:17.2.120200508 --admin-username $username --admin-password $password

# After the HUB-VNG gateway and On-PremCSR have been created, document the public IP address for both, we will use them to build the IPsec tunnel

az network public-ip show -g $rg -n HUB-VNG-PIP1 --query "{address: ipAddress}"
az network public-ip show -g $rg -n OnPremCSRPublicIP --query "{address: ipAddress}"


#Login to on OnPremCSR, we will need to get into the configuration mode to configure IPsec and BGP, so type (conf t):

onpremCSR#conf t

Now you in configuration mode:

onpremCSR(config)#

Paste in below configuration one block at a time, make sure to replace CSRPublicIP and Azure-VNGpubip with ips you got from earlier:


crypto ikev2 proposal Azure-Ikev2-Proposal
 encryption aes-cbc-256
 integrity sha1 sha256
 group 2
!
crypto ikev2 policy Azure-Ikev2-Policy
 match address local 192.168.0.4 
 proposal Azure-Ikev2-Proposal
!
crypto ikev2 keyring to-onprem-keyring
 peer Azure-VNGpubip
  address HUB-VNG-PIP1
  pre-shared-key Routeserver
!
crypto ikev2 profile Azure-Ikev2-Profile
 match address local 192.168.0.4 
 match identity remote address HUB-VNG-PIP1 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local to-onprem-keyring
 lifetime 28800
 dpd 10 5 on-demand
!
crypto ipsec transform-set to-Azure-TransformSet esp-gcm 256
 mode tunnel
!
crypto ipsec profile to-Azure-IPsecProfile
 set transform-set to-Azure-TransformSet
 set ikev2-profile Azure-Ikev2-Profile
!
interface Loopback11
 ip address 172.16.1.1 255.255.255.255
!
interface Tunnel11
 ip address 172.16.2.1 255.255.255.255
 ip tcp adjust-mss 1350
 tunnel source 192.168.0.4
 tunnel mode ipsec ipv4
 tunnel destination HUB-VNG-PIP1
 tunnel protection ipsec profile to-Azure-IPsecProfile
!
router bgp 65005
 bgp log-neighbor-changes
 neighbor 10.1.3.4 remote-as 65515
 neighbor 10.1.3.4 ebgp-multihop 255
 neighbor 10.1.3.4 update-source Loopback11
 !
 address-family ipv4
  network 192.168.0.0 mask 255.255.0.0
  neighbor 10.1.3.4 activate
 exit-address-family
!
!Static route to HUB-VNG BGP ip pointing to Tunnel11, so that it would be reachable
ip route 10.1.3.4 255.255.255.255 Tunnel11

