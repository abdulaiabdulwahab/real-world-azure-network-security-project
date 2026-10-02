# real-world-azure-network-security-project

# Secure Azure Production Network with Azure Firewall

## Project Overview

This project demonstrates how to secure a multi-tier Azure environment using **network segmentation, Network Security Groups, User Defined Routes, and Azure Firewall**.

The goal is to build a production-style network where communication between application tiers is restricted according to least-privilege principles and outbound internet traffic is routed through a centralized firewall.

## Architecture

```text
                         Internet
                            |
                            v
                    +----------------+
                    | Azure Firewall |
                    +----------------+
                            ^
                            |
                       0.0.0.0/0
                            |
              +-------------+-------------+
              |             |             |
              v             v             v

          Web Subnet     App Subnet    Data Subnet
         10.20.1.0/24   10.20.2.0/24  10.20.3.0/24

             |              |              |
          nsg-web         nsg-app        nsg-data
             |              |              |
           Web VM          App VM         Data VM
```

Traffic rules:

```text
Web Tier
   |
   | TCP 8080
   v
Application Tier
   |
   | TCP 5432
   v
Database Tier
```

Direct Web-to-Database communication is blocked.

Internet-bound traffic is routed through Azure Firewall.

---

## Technologies Used

- Microsoft Azure
- Azure CLI
- Azure Virtual Network
- Azure Subnets
- Network Security Groups
- Azure Firewall
- User Defined Routes
- Route Tables
- Public IP Addresses
- Azure Network Watcher
- Azure Virtual Machines
- Azure Firewall Application Rules

---

## Project Objectives

The project demonstrates how to:

- Build a segmented Azure network
- Separate workloads into Web, Application, and Data tiers
- Apply least-privilege NSG rules
- Control east-west network traffic
- Block unnecessary lateral communication
- Deploy Azure Firewall
- Route outbound traffic through Azure Firewall
- Restrict outbound access using firewall rules
- Inspect effective network routes
- Troubleshoot Azure network connectivity

---

# Network Design

The Virtual Network uses:

```text
VNet
10.20.0.0/16
```

Subnets:

| Subnet | Address Range | Purpose |
|---|---|---|
| AzureFirewallSubnet | 10.20.0.0/26 | Azure Firewall |
| snet-web | 10.20.1.0/24 | Web workloads |
| snet-app | 10.20.2.0/24 | Application workloads |
| snet-data | 10.20.3.0/24 | Database workloads |

---

# Step 1 — Create the Resource Group

```bash
az group create \
  --name "$RG2" \
  --location "$LOCATION"
```

---

# Step 2 — Create the Virtual Network

```bash
az network vnet create \
  --resource-group "$RG2" \
  --name "$VNET2" \
  --location "$LOCATION" \
  --address-prefixes 10.20.0.0/16 \
  --subnet-name AzureFirewallSubnet \
  --subnet-prefixes 10.20.0.0/26
```

Create the workload subnets:

```bash
az network vnet subnet create \
  --resource-group "$RG2" \
  --vnet-name "$VNET2" \
  --name "$WEB_SUBNET" \
  --address-prefixes 10.20.1.0/24
```

```bash
az network vnet subnet create \
  --resource-group "$RG2" \
  --vnet-name "$VNET2" \
  --name "$APP_SUBNET" \
  --address-prefixes 10.20.2.0/24
```

```bash
az network vnet subnet create \
  --resource-group "$RG2" \
  --vnet-name "$VNET2" \
  --name "$DATA_SUBNET" \
  --address-prefixes 10.20.3.0/24
```

---

# Step 3 — Create Network Security Groups

Create separate NSGs for each workload tier:

```bash
az network nsg create \
  --resource-group "$RG2" \
  --name "$WEB_NSG" \
  --location "$LOCATION"
```

```bash
az network nsg create \
  --resource-group "$RG2" \
  --name "$APP_NSG" \
  --location "$LOCATION"
```

```bash
az network nsg create \
  --resource-group "$RG2" \
  --name "$DATA_NSG" \
  --location "$LOCATION"
```

---

# Step 4 — Secure the Application Tier

Only the Web subnet should be able to communicate with the Application subnet on TCP port `8080`.

```bash
az network nsg rule create \
  --resource-group "$RG2" \
  --nsg-name "$APP_NSG" \
  --name Allow-Web-To-App \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 10.20.1.0/24 \
  --destination-port-ranges 8080
```

Block other Virtual Network traffic:

```bash
az network nsg rule create \
  --resource-group "$RG2" \
  --nsg-name "$APP_NSG" \
  --name Deny-Other-VNet \
  --priority 200 \
  --direction Inbound \
  --access Deny \
  --protocol "*" \
  --source-address-prefixes VirtualNetwork \
  --destination-port-ranges "*"
```

---

# Step 5 — Secure the Data Tier

Only the Application subnet should be able to reach the Data subnet on TCP port `5432`.

```bash
az network nsg rule create \
  --resource-group "$RG2" \
  --nsg-name "$DATA_NSG" \
  --name Allow-App-To-Database \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 10.20.2.0/24 \
  --destination-port-ranges 5432
```

Block other Virtual Network traffic:

```bash
az network nsg rule create \
  --resource-group "$RG2" \
  --nsg-name "$DATA_NSG" \
  --name Deny-Other-VNet \
  --priority 200 \
  --direction Inbound \
  --access Deny \
  --protocol "*" \
  --source-address-prefixes VirtualNetwork \
  --destination-port-ranges "*"
```

---

# Step 6 — Associate NSGs with Subnets

Web subnet:

```bash
az network vnet subnet update \
  --resource-group "$RG2" \
  --vnet-name "$VNET2" \
  --name "$WEB_SUBNET" \
  --network-security-group "$WEB_NSG"
```

Application subnet:

```bash
az network vnet subnet update \
  --resource-group "$RG2" \
  --vnet-name "$VNET2" \
  --name "$APP_SUBNET" \
  --network-security-group "$APP_NSG"
```

Data subnet:

```bash
az network vnet subnet update \
  --resource-group "$RG2" \
  --vnet-name "$VNET2" \
  --name "$DATA_SUBNET" \
  --network-security-group "$DATA_NSG"
```

---

# Step 7 — Deploy Azure Firewall

Create the Firewall public IP:

```bash
az network public-ip create \
  --resource-group "$RG2" \
  --name "$FW_PIP" \
  --location "$LOCATION" \
  --allocation-method Static \
  --sku Standard
```

Create Azure Firewall:

```bash
az network firewall create \
  --resource-group "$RG2" \
  --name "$FW" \
  --location "$LOCATION" \
  --sku AZFW_VNet \
  --tier Standard
```

Create its IP configuration:

```bash
az network firewall ip-config create \
  --resource-group "$RG2" \
  --firewall-name "$FW" \
  --name fw-ipconfig \
  --public-ip-address "$FW_PIP" \
  --vnet-name "$VNET2"
```

Update the Firewall:

```bash
az network firewall update \
  --resource-group "$RG2" \
  --name "$FW"
```

Retrieve the Firewall private IP:

```bash
FW_PRIVATE_IP=$(az network firewall show \
  --resource-group "$RG2" \
  --name "$FW" \
  --query "ipConfigurations[0].privateIPAddress" \
  --output tsv)

echo "$FW_PRIVATE_IP"
```

---

# Step 8 — Create the Route Table

Create a route table:

```bash
az network route-table create \
  --resource-group "$RG2" \
  --name "$ROUTE_TABLE" \
  --location "$LOCATION"
```

Route internet traffic to Azure Firewall:

```bash
az network route-table route create \
  --resource-group "$RG2" \
  --route-table-name "$ROUTE_TABLE" \
  --name default-to-firewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address "$FW_PRIVATE_IP"
```

This creates the following path:

```text
Workload
   |
   v
0.0.0.0/0
   |
   v
Azure Firewall
   |
   v
Internet
```

---

# Step 9 — Associate the Route Table

Web subnet:

```bash
az network vnet subnet update \
  --resource-group "$RG2" \
  --vnet-name "$VNET2" \
  --name "$WEB_SUBNET" \
  --route-table "$ROUTE_TABLE"
```

Application subnet:

```bash
az network vnet subnet update \
  --resource-group "$RG2" \
  --vnet-name "$VNET2" \
  --name "$APP_SUBNET" \
  --route-table "$ROUTE_TABLE"
```

Data subnet:

```bash
az network vnet subnet update \
  --resource-group "$RG2" \
  --vnet-name "$VNET2" \
  --name "$DATA_SUBNET" \
  --route-table "$ROUTE_TABLE"
```

---

# Step 10 — Configure Firewall Application Rules

Allow access only to approved websites:

```bash
az network firewall application-rule create \
  --resource-group "$RG2" \
  --firewall-name "$FW" \
  --collection-name Approved-Websites \
  --name Allow-Microsoft \
  --protocols Http=80 Https=443 \
  --target-fqdns \
    www.microsoft.com \
    learn.microsoft.com \
  --source-addresses \
    10.20.1.0/24 \
    10.20.2.0/24 \
    10.20.3.0/24 \
  --priority 200 \
  --action Allow
```

This demonstrates centralized outbound filtering.

---

# Verification

## List NSG Rules

```bash
az network nsg rule list \
  --resource-group "$RG2" \
  --nsg-name "$APP_NSG" \
  --output table
```

```bash
az network nsg rule list \
  --resource-group "$RG2" \
  --nsg-name "$DATA_NSG" \
  --output table
```

---

## List Network Interfaces

```bash
az network nic list \
  --resource-group "$RG2" \
  --query "[].{Name:name,PrivateIP:ipConfigurations[0].privateIPAddress}" \
  --output table
```

---

## View Effective Routes

```bash
az network nic show-effective-route-table \
  --resource-group "$RG2" \
  --name <NIC-NAME>
```

Look for:

```text
0.0.0.0/0
Next Hop: VirtualAppliance
Next Hop IP: Azure Firewall private IP
```

---

## View Effective NSG Rules

```bash
az network nic list-effective-nsg \
  --resource-group "$RG2" \
  --name <NIC-NAME>
```

This helps determine which security rules are actually affecting a workload.

---

# Troubleshooting Scenarios

## Scenario 1 — Internet Access Stops

If internet connectivity fails after applying the route table:

Check the effective routes:

```bash
az network nic show-effective-route-table \
  --resource-group "$RG2" \
  --name <NIC-NAME>
```

Verify that:

```text
0.0.0.0/0
     |
     v
Virtual Appliance
     |
     v
Azure Firewall private IP
```

Then verify Firewall rules.

---

## Scenario 2 — Web Tier Cannot Reach Application Tier

Inspect the Application NSG:

```bash
az network nsg rule list \
  --resource-group "$RG2" \
  --nsg-name "$APP_NSG" \
  --output table
```

Verify that the allow rule has a higher priority than the deny rule.

Correct:

```text
100 Allow Web -> App TCP/8080
200 Deny Other VNet traffic
```

Incorrect:

```text
90  Deny Other VNet traffic
100 Allow Web -> App TCP/8080
```

Because lower numbers are processed first, the deny rule would block the traffic.

---

## Scenario 3 — Web Tier Can Reach the Database Directly

This violates the intended architecture.

The Data NSG should permit only:

```text
App subnet
10.20.2.0/24
      |
      | TCP 5432
      v
Data subnet
10.20.3.0/24
```

Check:

```bash
az network nsg rule list \
  --resource-group "$RG2" \
  --nsg-name "$DATA_NSG" \
  --output table
```

---

## Scenario 4 — NSG Rules Look Correct but Connectivity Still Fails

Check the troubleshooting path:

```text
Application
     |
     v
Operating System
     |
     v
NSG
     |
     v
Route Table
     |
     v
Azure Firewall
     |
     v
Destination
```

Useful Azure CLI commands include:

```bash
az network nic list-effective-nsg \
  --resource-group "$RG2" \
  --name <NIC-NAME>
```

and:

```bash
az network nic show-effective-route-table \
  --resource-group "$RG2" \
  --name <NIC-NAME>
```

---

# Security Concepts Demonstrated

## Network Segmentation

Workloads are separated according to their function:

```text
Web
 |
 v
App
 |
 v
Database
```

Each tier has its own subnet and security controls.

---

## Least Privilege

Only required communication paths are permitted.

For example:

```text
Web -> App :8080
```

and:

```text
App -> Database :5432
```

Everything else is restricted.

---

## East-West Traffic Control

East-west traffic is communication between workloads inside the network.

NSGs restrict unnecessary lateral communication between the Web, Application, and Data tiers.

---

## North-South Traffic Control

North-south traffic is communication between Azure workloads and external networks.

Azure Firewall provides centralized control over outbound traffic.

---

## Defense in Depth

This project uses multiple security layers:

```text
Network Segmentation
        +
       NSGs
        +
      Routes
        +
 Azure Firewall
        +
 Monitoring
```

A single security control is not relied upon to protect the environment.

---

# Skills Demonstrated

This project demonstrates:

- Azure Network Security
- Azure Virtual Networks
- Subnet segmentation
- Network Security Groups
- NSG priority management
- Least-privilege networking
- East-west traffic filtering
- Azure Firewall
- Firewall application rules
- User Defined Routes
- Route Tables
- Centralized internet egress
- Azure CLI
- Effective route troubleshooting
- Effective NSG troubleshooting
- Defense-in-depth architecture

---

# Azure DevOps Integration

In a production environment, this architecture could be deployed using Infrastructure as Code with:

```text
GitHub / Azure Repos
        |
        v
Azure Pipelines / GitHub Actions
        |
        v
Terraform / Bicep
        |
        v
Azure
        |
        +--> VNets
        +--> NSGs
        +--> Routes
        +--> Azure Firewall
```

A DevSecOps pipeline could also scan the infrastructure for insecure configurations such as:

```text
0.0.0.0/0 -> SSH

0.0.0.0/0 -> RDP

Overly permissive NSGs

Missing network segmentation

Unrestricted outbound traffic
```

This helps move network-security validation earlier into the deployment lifecycle.

---

# Cleanup

Azure Firewall incurs ongoing charges, so delete the lab after completing the project unless you intentionally want to keep it running.

```bash
az group delete \
  --name "$RG2" \
  --yes \
  --no-wait
```

Check that deletion has started:

```bash
az group exists \
  --name "$RG2"
```

Eventually this should return:

```text
false
```

---

# Key Takeaway

This project demonstrates how multiple Azure network-security controls can work together to protect a production-style environment.

Instead of allowing unrestricted communication inside a Virtual Network, the environment uses:

```text
Segmentation
     +
Least Privilege
     +
NSGs
     +
User Defined Routes
     +
Azure Firewall
```

The result is a more controlled network architecture where each workload communicates only with the systems and destinations it requires.