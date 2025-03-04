# Introduction

This document provides a comprehensive checklist and requirements for deploying NSX for Supervisors in a vSphere Environment. It covers the steps for both VCF and non-VCF Environments. The intended audience consists of the VI administrators, network administrators, and platform admins responsible for managing the platform and making the various services available for end users to consume.

## Standard checklist
* Make sure that all hostnames of all ESXi nodes servers are in lowercase.
* The minimum MTU required for the entire network path (VMkernel ports, virtual switches, physical switches, and routers) is 1600; however, the recommended value is 1700. Reference Documentation[https://techdocs.broadcom.com/us/en/vmware-cis/nsx/vmware-nsx/4-2/installation-guide/transport-zones-and-transport-nodes/mtu-guidance.html]


## NSX Configuration for VCF Environment 
This section outlines the VCF Environment process where SDDC Manager workflows do the NSX Configuration. 

### NSX Manager
VMware Cloud Foundation domains/clusters have NSX Managers deployed and are prepared for NSX by default. Sometimes, when the vSphere environment is converted to VCF via Brownfield Import/convert and does not have NSX installed, the converted environment would only have NSX-VLAN networking configured, not overlay.
Management Domain NSX Manager is deployed as medium-sized by default, which supports up to 128 hypervisors or five vSphere Clusters.  NSX Managers can be resized to Large or XL-Large based on the requirement. Reference - NSX Manager VM and Host Transport Node System Requirements
Workload Domain NSX Managers should be deployed as Large or X-Large to support the supervisor in a scaled deployment environment.
Note—An SDDC Manager only supports 1000 Hosts, so a large or X-large NSX Manager should meet the requirements. 

