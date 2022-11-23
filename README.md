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
