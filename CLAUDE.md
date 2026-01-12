# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Arista AVD (Ansible AVD) example demonstrating a dual data center L3 Leaf-Spine (L3LS) architecture with EVPN/VXLAN overlay. The project uses Arista's AVD collection to generate configurations for network devices across two data centers connected via DCI (Data Center Interconnect) links with EVPN DC Gateway functionality.

**Architecture**: Two separate data centers (DC1 and DC2) each containing spine switches, L3 leaf switches, and border leaf switches. Border leaves in each DC connect via DCI links and act as EVPN gateways to enable seamless EVPN route advertisement between data centers.

## First-Time Setup

**IMPORTANT:** This repository uses template files for sensitive credentials.

1. Copy template files and configure your credentials:
   ```bash
   cp deploy.yml.example deploy.yml
   cp group_vars/DC-L3LS-FABRIC.yml.example group_vars/DC-L3LS-FABRIC.yml
   ```

2. Edit the copied files to add your:
   - CloudVision API token (in `deploy.yml`)
   - Switch passwords (in `group_vars/DC-L3LS-FABRIC.yml`)
   - BGP passwords (in `group_vars/DC-L3LS-FABRIC.yml`)

See `SETUP.md` for detailed configuration instructions.

## Common Commands

### Build configurations
Generate device configurations and documentation without deploying:
```bash
ansible-playbook build.yml
```

This executes the `arista.avd.eos_designs` and `arista.avd.eos_cli_config_gen` roles to create:
- Structured configurations in `intended/structured_configs/`
- Device configurations in `intended/configs/`
- Documentation in `documentation/`

### Deploy configurations
Build and deploy configurations to devices (uses CloudVision for deployment):
```bash
ansible-playbook deploy.yml
```

Note: The deploy playbook is currently configured for CloudVision (CVaaS) deployment. Direct eAPI deployment is commented out.

### Check inventory
View the Ansible inventory structure:
```bash
ansible-inventory --graph
ansible-inventory --list
```

## Code Architecture

### Inventory Structure
The inventory (`inventory.yml`) uses a hierarchical group structure:
- `DC-L3LS-FABRIC` - Top-level fabric group
  - `DC1` - First data center
    - `DC1_SPINES` - Spine switches
    - `DC1_L3_LEAVES` - L3 leaf switches
    - `DC1_BORDER_LEAVES` - Border/gateway leaf switches
  - `DC2` - Second data center (same structure as DC1)
    - `DC2_SPINES`
    - `DC2_L3_LEAVES`
    - `DC2_BORDER_LEAVES`
- `NETWORK_SERVICES` - Logical group for devices that participate in tenant VRF/VLAN provisioning
- `CONNECTED_ENDPOINTS` - Logical group for devices with server connectivity

### Group Variables Organization

AVD uses a layered approach where variables are inherited from parent groups to children:

1. **DC-L3LS-FABRIC.yml** - Fabric-wide settings
   - Ansible connection parameters (eAPI via httpapi)
   - Underlay and overlay routing protocols (both eBGP)
   - BGP peer group passwords
   - DCI link definitions using `l3_edge` with IP pools, profiles, and P2P link assignments
   - Default interface mappings for different platform types

2. **DC1.yml / DC2.yml** - Data center-specific settings
   - Management gateway configuration
   - Spine defaults: platform, loopback pools, BGP AS
   - L3 leaf defaults: VTEP loopback pools, uplink configurations, MLAG peer pools
   - Node groups with MLAG pairs and individual node definitions
   - EVPN Gateway configuration on border leaves with `evpn_gateway` settings

3. **Device type files** (e.g., DC1_SPINES.yml, DC1_L3_LEAVES.yml)
   - Simple `type:` declaration (spine, l3leaf, etc.)
   - Device types map to AVD roles that define platform-specific behaviors

4. **NETWORK_SERVICES.yml** - Tenant, VRF, and VLAN definitions
   - Tenant definitions with MAC VRF VNI bases
   - VRF configurations with VRF VNIs and VTEP diagnostics
   - SVI definitions with anycast gateway IPs
   - Pure L2 VLAN definitions for bridged-only VLANs

5. **CONNECTED_ENDPOINTS.yml** - Server connectivity
   - Server adapter definitions with port-channel configurations
   - VLAN assignments, native VLANs, and trunk mode settings
   - Spanning tree portfast configurations

### Key AVD Concepts

**Node Groups**: When exactly two nodes are in a `node_groups` definition under l3leaf, AVD automatically generates MLAG configuration. The group shares a single BGP AS.

**ID-Based Calculations**: Each node has an `id` field used to calculate:
- Loopback0 IP addresses (for BGP peering)
- Loopback1 IP addresses (for VTEP)
- MLAG peer IPs
- Other derived values

**IP Pool Offsets**: The `loopback_ipv4_offset` is critical when spine and l3leaf use the same IP pool to prevent IP conflicts. Set it to at least the number of spine switches.

**EVPN DC Gateway**: Border leaves use `evpn_gateway` configuration with:
- `evpn_l2.enabled: true` - For EVPN type 2 routes (MAC-IP)
- `evpn_l3.enabled: true` with `inter_domain: true` - For EVPN type 5 routes (IP-PREFIX)
- `remote_peers` - Defines BGP peering to the remote DC's border leaves

**DCI Links**: Inter-DC connectivity is defined at the fabric level using `l3_edge`:
1. IP pools for P2P link addressing
2. Link profiles with BGP AS assignments
3. Explicit P2P link definitions between border leaves

### Generated Output Structure

These directories are created when you run `ansible-playbook build.yml` and are not committed to the repository:

- `intended/structured_configs/` - YAML structured configs per device (AVD data model)
- `intended/configs/` - EOS CLI configurations ready for deployment
- `documentation/fabric/` - Fabric-wide topology and P2P link documentation (CSV and markdown)
- `documentation/devices/` - Per-device configuration documentation

**Note:** The `documentation/` directory contains auto-generated topology diagrams and tables that reflect your actual configuration. Always refer to these for accurate topology information.

### Network Design Details

**IP Addressing Scheme**:
- Management: 172.20.20.0/24
- DC1 Loopback0 (BGP): 10.255.0.0/27
- DC1 Loopback1 (VTEP): 10.255.1.0/27
- DC1 P2P Underlay: 10.255.255.0/26
- DC2 Loopback0: 10.255.128.0/27 (offset by 128)
- DC2 Loopback1: 10.255.129.0/27
- DC2 P2P Underlay: 10.255.255.64/26
- DCI Links: 172.16.100.0/24

**BGP AS Assignments**:
- DC1 Spine: AS 65100
- DC1 Leaf Groups: AS 65101, 65102
- DC2 Spine: AS 65200
- DC2 Leaf Groups: AS 65201, 65202

**Platform Notes**:
- This example uses cEOS (containerized EOS)
- MTU is set to 1500 for virtual platforms (p2p_uplinks_mtu: 1500)
- Multi-agent routing model is required for EVPN (requires device reboot when first configured)

## Important Considerations

When modifying configurations:
- Always maintain consistency between DC1 and DC2 group_vars unless intentional architectural differences are needed
- Ensure border leaf `remote_peers` hostnames match inventory names for automatic BGP peering setup
- VRF VNI and VLAN-to-VNI mappings must be consistent across the fabric
- The `id` field must be unique within each node type in a data center
- BGP passwords in DC-L3LS-FABRIC.yml are base64 encoded (current password: "arista")

When adding new devices:
1. Add to inventory.yml under appropriate group
2. Add node definition to corresponding DC*.yml file
3. Ensure IP pools have sufficient space for new devices
4. For MLAG pairs, always define exactly two nodes in a node_group

## CloudVision Integration

The deploy.yml playbook includes CloudVision (CVaaS) integration:
- Uses token-based authentication (configure in your local `deploy.yml`)
- Automatic change control creation with `cv_run_change_control: true`
- Direct eAPI deployment role is commented out but available as alternative

## Security Notes

- `deploy.yml` and `group_vars/DC-L3LS-FABRIC.yml` contain sensitive credentials and are gitignored
- Template files (`.example` suffix) are provided in the repository
- Never commit files with real credentials
- See `SETUP.md` for configuration instructions
