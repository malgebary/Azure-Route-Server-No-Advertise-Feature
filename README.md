# **Azure Route Server No-Advertise Feature**

# Contents
[Introduction](#introduction)

[Lab](#lab)

## Introduction
Azure Route Server (ARS) has recently started supporting the BGP community of No Advertise, when a No-Advertise community is attached to a route, the BGP speaker won't advertise the route to any internal or external BGP peers. So when ARS gets a route tagged with this community, ARS will learn the route but will not forwarded it after that to any peers including the VPN gateway or the ExpressRoute gateway. The main advantage of this feature is to control the number of routes being pushed to the ExpressRoute gateway where the limits of routes advertised toward this gateway from the VNet is 1000 routes, so in scenarios where there is SDWAN NVA in the VNet and you are using ARS to inject SDWAN learned routes to the VM's effective route table and at the same time you don't want to push those routes down to on-prem then you can use the No Advertise community, this community also helpful in case you want to advertise 0/0 in Azure but not to be passed down to on-prem.
