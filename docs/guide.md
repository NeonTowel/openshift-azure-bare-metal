# OpenShift Bare Metal on Azure via Nested Virtualization

## Architecture

- **VNet Address Spaces:** `172.16.0.0/23` and `172.16.2.0/24`
- **Subnets:**
  - `ocp-bm`: `172.16.0.0/23` — Internal routing / LAN traffic
  - `ocp-tenants`: `172.16.2.0/24` — Internet-facing NAT subnet
- **Hyper-V Host VM:** Dual-NIC router straddling both subnets
  - **NIC1** (`ocp-tenants`): Direct Standard Static Public IP for inbound RDP and outbound internet
  - **NIC2** (`ocp-bm`): Internal routing to Azure VNet and nested VMs
- **No NAT Gateway required:** The direct Public IP on the host overrides subnet-level NAT Gateway for all outbound traffic from the host and nested VMs

## Host VM Requirements

1. Deploy a Windows Server VM on a size supporting nested virtualization (e.g., Dv3 or Ev3 series).
2. Attach two NICs: NIC1 to `ocp-tenants`, NIC2 to `ocp-bm`.
3. **Enable IP Forwarding on NIC2** in the Azure Portal.

## Security Groups

| Interface | NSG Rule | Details |
|-----------|----------|---------|
| NIC1 | Allow RDP Inbound | TCP 3389 from your specific public IP only |
| NIC2 | Allow Nested VM Traffic | Between `172.16.0.0/23` and nested VM range (e.g., `192.168.10.0/24`) |

## RRAS and Hyper-V Configuration

1. Install roles and reboot:
   ```powershell
   Install-WindowsFeature -Name Hyper-V, DirectAccess-VPN, Routing -IncludeManagementTools -Restart
   ```

2. Create an **Internal** Virtual Switch named `Nested-vSwitch`.
3. Assign the virtual adapter IP `192.168.10.1/24` (no default gateway).
4. Open **Routing and Remote Access**, choose **Custom Configuration**, enable **NAT** and **LAN Routing**.
5. Under **IPv4 → NAT**:
   - Add NIC1 as **Public interface connected to the Internet** with NAT enabled
   - Add `vEthernet (Nested-vSwitch)` as **Private interface connected to private network**
6. Add static route in RRAS for `172.16.0.0/23` via NIC2 gateway (e.g., `172.16.0.1`).

## Azure Fabric Routing

Create a **Route Table** and associate it with the `ocp-bm` subnet:

| Destination | Next Hop Type | Next Hop IP |
|-------------|---------------|-------------|
| `192.168.10.0/24` | Virtual Appliance | `172.16.0.4` (NIC2 IP) |

This ensures return traffic from the Azure VNet reaches nested VMs via the routing host.

## Nested OpenShift Nodes

1. Attach nested VM network adapters to `Nested-vSwitch`.
2. Configure each node:
   - **IP:** `192.168.10.x`
   - **Subnet:** `255.255.255.0`
   - **Gateway:** `192.168.10.1`
3. Internet traffic exits via RRAS NAT on NIC1.
4. Intra-VNet traffic routes through NIC2 without NAT.

## MTU Planning

| Phase | MTU | Notes |
|-------|-----|-------|
| Initial | 1500 | Default Azure VNet MTU; OVN-Kubernetes sets pod MTU to 1400 |
| Upgrade | 9000 | Requires MANA-enabled VM series (e.g., Dv6) with Accelerated Networking |
| OpenShift Shift | 8900 | Rolling migration via Cluster Network Operator; reserves 100 bytes for OVN encapsulation |

**Caveat:** 9000 MTU works only for intra-VNet traffic. Internet-bound traffic must fragment to 1500 at the host NAT layer.
