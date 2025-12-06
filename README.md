# libvirt-playbooks

Ansible playbooks to automate virtual machine creation and management on Fedora 42+ using libvirt and KVM.

## Features

- **Automated libvirt setup** - Installs and configures all required packages
- **VM creation** - Easily create VMs with customizable specs
- **VM configuration** - Post-creation setup with SSH, packages, and hostname
- **Multiple network interfaces** - Configure VMs with multiple NICs, custom MAC addresses, and different virtual networks
- **Multiple disk volumes** - Attach multiple disks to VMs with custom sizes, formats, and device names
- **ISO mounting** - Mount ISO images as CDROM for OS installation
- **ISO management** - Download and manage ISO images from URLs
- **Storage management** - Automatic storage pool creation and management
- **Role-based structure** - Modular, reusable Ansible roles
- **Remote host support** - Manage VMs on remote Fedora 42+ hosts via SSH
- **Unified VM management** - Single playbook handles both single and multiple VM creation

## Requirements

- **Host**: Fedora 42 or higher
- **CPU**: Virtualization extensions enabled (Intel VT-x or AMD-V)
- **RAM**: Minimum 4GB (8GB+ recommended for multiple VMs)
- **Storage**: At least 20GB free space for VMs
- **Ansible**: 2.12+
- **Python**: 3.9+

## Installation

1. Clone the repository:
```bash
git clone https://github.com/jeffdyoung/libvirt-playbooks.git
cd libvirt-playbooks
```

2. Install Ansible and dependencies:
```bash
sudo dnf install -y ansible python3-libvirt python3-lxml
```

3. Install required Ansible collections:
```bash
ansible-galaxy collection install community.libvirt
ansible-galaxy collection install ansible.posix
```

4. Create your host-specific configuration:
```bash
# Copy example files as templates
cp host_vars/aarch64-002.sys.eng.rdu2.dc.redhat.com.yml.example \
   host_vars/your-hostname.yml

# Edit with your VM definitions
vim host_vars/your-hostname.yml
```

**Note**: Host-specific files (`host_vars/*.yml` and `inventory.yml`) are excluded from version control via `.gitignore` to prevent committing sensitive host data. Only example files are tracked.

## Project Structure

```
libvirt-playbooks/
├── ansible.cfg                    # Ansible configuration
├── inventory                      # Host inventory (INI format)
├── inventory.yml.example          # Extended YAML inventory example with VM definitions
├── setup_libvirt.yml             # Main libvirt setup playbook
├── manage_host_vms.yml           # Unified VM creation and configuration playbook
├── manage_host_storage.yml       # Storage pool and directory management playbook
├── manage_host_network.yml       # Virtual network management playbook
├── reset_host.yml                # Reset host by removing all VMs, networks, and storage
├── complete_workflow.yml         # Combined setup + VM creation
├── setup_remote_host.sh          # Remote host preparation script
├── README.md                     # This file
├── templates/                    # Network XML templates
│   ├── network_nat.xml.j2        # NAT network template
│   ├── network_isolated.xml.j2   # Isolated network template
│   └── network_bridge.xml.j2     # Bridge network template
├── roles/
│   ├── libvirt-setup/            # Role to setup libvirt on host
│   │   ├── tasks/
│   │   ├── templates/
│   │   └── handlers/
│   ├── vm-create/                # Role to create VMs
│   │   ├── tasks/
│   │   └── templates/
│   └── vm-configure/             # Role to configure VMs
│       └── tasks/
├── group_vars/
│   └── libvirt_hosts.yml         # Default variables for all hosts
├── host_vars/                    # Host-specific variables and VM definitions
│   ├── *.yml                     # Actual host configurations (gitignored)
│   ├── *.yml.example             # Example host variable files
│   └── README.md                 # Documentation for host_vars
└── group_files/
    └── vm_examples.yml           # Example VM definitions
```

## Quick Start

> **Note**: This project uses Ansible playbooks exclusively. All operations are performed via `ansible-playbook` commands.

### Quick Reference

| Task | Command |
|------|---------|
| Setup libvirt on hosts | `ansible-playbook -i inventory setup_libvirt.yml` |
| Setup storage pools | `ansible-playbook -i inventory manage_host_storage.yml` |
| Download ISOs | `ansible-playbook -i inventory manage_host_storage.yml --tags iso` |
| Create virtual networks | `ansible-playbook -i inventory manage_host_network.yml` |
| Create single VM | `ansible-playbook -i inventory manage_host_vms.yml -e "vm_name=myvm"` |
| Create VMs from inventory | `ansible-playbook -i inventory manage_host_vms.yml` |
| Create VMs (skip config) | `ansible-playbook -i inventory manage_host_vms.yml --skip-tags configure` |
| Reset host (remove all VMs) | `ansible-playbook -i inventory reset_host.yml -l hostname` |
| List all VMs | `virsh list --all` |
| List all networks | `virsh net-list --all` |
| Start VM | `virsh start <vm_name>` |
| Stop VM | `virsh shutdown <vm_name>` |

### 1. Configure Inventory

Edit the [inventory](inventory) file to specify your target host(s):

**Local operations** (via SSH to 127.0.0.1):
```ini
[libvirt_hosts]
127.0.0.1 ansible_user=root ansible_port=22
```

**Remote hosts**:
```ini
[libvirt_hosts]
aarch64.host.com ansible_user=<youruser> ansible_port=22
x86.host.com ansible_user=<youruser> ansible_port=22
```

See [inventory.yml.example](inventory.yml.example) for YAML format with VM definitions.

### 2. Setup libvirt on your host

```bash
ansible-playbook -i inventory setup_libvirt.yml
```

This will:
- Update system packages
- Install libvirt, QEMU, and related tools
- Enable and start the libvirt service
- Create storage directories
- Configure storage pools
- Set up proper permissions

### 3. Create VMs

```bash
ansible-playbook -i inventory manage_host_vms.yml
```

## Usage Examples

### Example 1: Simple setup and VM creation

```bash
# Step 1: Setup libvirt (run once)
ansible-playbook -i inventory setup_libvirt.yml

# Step 2: Create and configure a development VM
ansible-playbook -i inventory manage_host_vms.yml \
  -e "vm_name=dev-box" \
  -e "vm_memory_mb=4096" \
  -e "vm_vcpus=4" \
  -e "vm_disk_size_gb=50"

# Or skip configuration step
ansible-playbook -i inventory manage_host_vms.yml \
  -e "vm_name=dev-box" \
  --skip-tags configure
```

### Example 2: Create multiple VMs

Create a file `extra_vars.yml`:
```yaml
vm_definitions:
  - name: web-server
    memory: 2048
    vcpus: 2
    disk_size: 30
  - name: db-server
    memory: 4096
    vcpus: 4
    disk_size: 50
  - name: app-server
    memory: 3072
    vcpus: 3
    disk_size: 40
```

Then run:
```bash
ansible-playbook -i inventory complete_workflow.yml -e "@extra_vars.yml"
```

### Example 3: Create single VM without configuration

Sometimes you only want to create VMs quickly without waiting for configuration:

```bash
# Create VM without configuring it
ansible-playbook -i inventory manage_host_vms.yml \
  -e "vm_name=test-vm" \
  --skip-tags configure

# Or create multiple VMs from inventory without configuring them
ansible-playbook -i inventory.yml manage_host_vms.yml \
  --skip-tags configure
```

### Example 4: Multi-host operations

Target specific hosts from inventory:
```bash
# Run on local host only
ansible-playbook -i inventory setup_libvirt.yml -l 127.0.0.1

# Run on specific remote host
ansible-playbook -i inventory setup_libvirt.yml -l aarch64-002.sys.eng.rdu2.dc.redhat.com

# Run on all hosts
ansible-playbook -i inventory setup_libvirt.yml
```

### Example 5: Define VMs in inventory (YAML format)

Create or edit `inventory.yml` with VM definitions per host:

```yaml
libvirt_hosts:
  hosts:
    aarch64-dev:
      ansible_host: aarch64-002.sys.eng.rdu2.dc.redhat.com
      ansible_user: root
      ansible_port: 22
      # Define VMs for this host
      vms:
        - name: dev-vm-001
          memory: 4096
          vcpus: 4
          disk_size: 50
          autostart: true
          networks:
            - name: default
              mac: "52:54:00:11:22:01"
            - name: storage-network
              mac: "52:54:00:aa:bb:01"
        - name: dev-vm-002
          memory: 4096
          vcpus: 4
          disk_size: 50
          autostart: true
          networks:
            - name: default
              mac: "52:54:00:11:22:02"
        - name: test-vm-001
          memory: 2048
          vcpus: 2
          disk_size: 30
          autostart: false
          # networks parameter is optional - defaults to single NIC on 'default' network
```

Then create and configure all VMs:
```bash
# Create and configure all VMs defined in inventory for all hosts
ansible-playbook -i inventory.yml manage_host_vms.yml

# Create and configure VMs for specific host only
ansible-playbook -i inventory.yml manage_host_vms.yml -l aarch64-dev

# Create VMs without configuration (faster)
ansible-playbook -i inventory.yml manage_host_vms.yml --skip-tags configure
```

### Example 6: Define VMs using host_vars

Create `host_vars/aarch64-002.sys.eng.rdu2.dc.redhat.com.yml`:

```yaml
vms:
  - name: ocp-master-1
    memory: 16384
    vcpus: 8
    disk_size: 120
    autostart: true
    networks:
      - name: default
        mac: "52:54:00:11:22:01"
      - name: storage-network
        mac: "52:54:00:aa:bb:01"
  - name: ocp-worker-1
    memory: 32768
    vcpus: 16
    disk_size: 200
    autostart: true
    networks:
      - name: default
        mac: "52:54:00:11:22:02"
      - name: storage-network
        mac: "52:54:00:aa:bb:02"
```

Then create and configure the VMs:
```bash
# Create and configure all VMs
ansible-playbook -i inventory manage_host_vms.yml -l aarch64-002.sys.eng.rdu2.dc.redhat.com

# Only create VMs (skip configuration)
ansible-playbook -i inventory manage_host_vms.yml -l aarch64-002.sys.eng.rdu2.dc.redhat.com --skip-tags configure

# Only configure already-created VMs
ansible-playbook -i inventory manage_host_vms.yml -l aarch64-002.sys.eng.rdu2.dc.redhat.com --tags configure
```

### Example 7: Multi-NIC VMs for OpenShift cluster

For complex setups like OpenShift clusters with separate networks for management, storage, and application traffic:

Create `host_vars/aarch64-002.sys.eng.rdu2.dc.redhat.com.yml`:

```yaml
vms:
  - name: ocp-master-1
    memory: 16384
    vcpus: 8
    disk_size: 120
    autostart: true
    networks:
      - name: default              # Management network
        mac: "52:54:00:10:00:01"
      - name: storage-network       # Storage/Ceph network
        mac: "52:54:00:20:00:01"
      - name: app-network          # Application traffic network
        mac: "52:54:00:30:00:01"

  - name: ocp-worker-1
    memory: 32768
    vcpus: 16
    disk_size: 200
    autostart: true
    networks:
      - name: default
        mac: "52:54:00:10:00:02"
      - name: storage-network
        mac: "52:54:00:20:00:02"
      - name: app-network
        mac: "52:54:00:30:00:02"

  - name: ocp-worker-2
    memory: 32768
    vcpus: 16
    disk_size: 200
    autostart: true
    networks:
      - name: default
        mac: "52:54:00:10:00:03"
      - name: storage-network
        mac: "52:54:00:20:00:03"
      - name: app-network
        mac: "52:54:00:30:00:03"
```

Then create the cluster:
```bash
# Create all VMs with their network configurations
ansible-playbook -i inventory manage_host_vms.yml -l aarch64-002.sys.eng.rdu2.dc.redhat.com
```

**Note**: Ensure the custom networks (`storage-network`, `app-network`) are created in libvirt before running the playbook. See the [Network Configuration](#network-configuration) section for details on creating virtual networks.

## VM Configuration

VM configuration happens automatically after creation when using `manage_host_vms.yml`. The configuration step will:
- Wait for SSH connectivity
- Set hostname
- Update system packages
- Install common tools
- Configure SSH key-based authentication

You can control configuration with tags:

```bash
# Create and configure (default behavior)
ansible-playbook -i inventory manage_host_vms.yml -e "vm_name=myvm"

# Only create, skip configuration
ansible-playbook -i inventory manage_host_vms.yml -e "vm_name=myvm" --skip-tags configure

# Only configure existing VMs (without creating)
ansible-playbook -i inventory manage_host_vms.yml --tags configure
```

## Defining VMs

There are multiple ways to define VMs in this project:

### Method 1: Command-line (single VM)

Create a single VM using command-line parameters:
```bash
ansible-playbook -i inventory manage_host_vms.yml \
  -e "vm_name=myvm" \
  -e "vm_memory_mb=4096" \
  -e "vm_vcpus=4"
```

### Method 2: Inventory YAML (multiple VMs per host)

Define VMs directly in a YAML inventory file:
```yaml
libvirt_hosts:
  hosts:
    aarch64-dev:
      ansible_host: aarch64-002.sys.eng.rdu2.dc.redhat.com
      ansible_user: root
      vms:
        - name: vm1
          memory: 4096
          vcpus: 4
          disk_size: 50
          networks:
            - name: default
              mac: "52:54:00:11:22:01"
        - name: vm2
          memory: 2048
          vcpus: 2
          disk_size: 30
          networks:
            - name: default
```

Use: `ansible-playbook -i inventory.yml manage_host_vms.yml`

### Method 3: host_vars directory (multiple VMs per host)

Create `host_vars/<hostname>.yml`:
```yaml
vms:
  - name: vm1
    memory: 4096
    vcpus: 4
    disk_size: 50
    networks:
      - name: default
        mac: "52:54:00:11:22:01"
      - name: storage-network
        mac: "52:54:00:aa:bb:01"
  - name: vm2
    memory: 2048
    vcpus: 2
    disk_size: 30
    networks:
      - name: default
```

Use: `ansible-playbook -i inventory manage_host_vms.yml -l <hostname>`

### Method 4: Extra vars file (cluster setup)

Create a vars file with VM definitions:
```yaml
# cluster.yml
vm_definitions:
  - name: master
    memory: 4096
    vcpus: 4
  - name: worker1
    memory: 2048
    vcpus: 2
```

Use: `ansible-playbook -i inventory complete_workflow.yml -e "@cluster.yml"`

## Available Variables

### Global variables (group_vars/libvirt_hosts.yml)

| Variable | Default | Description |
|----------|---------|-------------|
| `libvirt_enabled` | true | Enable libvirt |
| `libvirt_storage_path` | `/home/{{ ansible_user }}/libvirt` | Default storage path for all libvirt data |
| `libvirt_network_name` | default | Network name |
| `vm_memory_mb` | 2048 | Default VM memory in MB |
| `vm_vcpus` | 2 | Default number of vCPUs |
| `vm_disk_size_gb` | 20 | Default disk size in GB |
| `vm_os_variant` | fedora42 | OS variant for virt-install |
| `vm_network_bridge` | virbr0 | Network bridge |
| `vm_dns_servers` | 8.8.8.8, 8.8.4.4 | DNS servers |

**Storage Configuration:**
- By default, all libvirt data (VM images, ISOs, cloud images) is stored in a single directory: `libvirt_storage_path`
- Override the default path in host_vars: `libvirt_storage_path: /mnt/nvme/libvirt`
- Create additional custom storage pools using the `storage_pools` variable (see Storage Pools section)

### VM-specific variables

| Variable | Required | Description |
|----------|----------|-------------|
| `vm_name` | Yes | Name of the VM |
| `vm_memory_mb` | No | Memory in MB |
| `vm_vcpus` | No | Number of vCPUs |
| `vm_disk_size_gb` | No | Disk size in GB (primary OS disk) |
| `vm_autostart` | No | Enable autostart (true/false) |
| `vm_hostname` | No | Hostname inside the VM |
| `vm_user` | No | Default user (default: root) |
| `vm_networks` | No | List of network configurations (see Network Configuration section) |
| `vm_volumes` | No | List of additional disk volumes (see Volume Configuration section) |
| `vm_iso` | No | ISO filename or full path to mount as CDROM (see ISO Mount section) |
| `vm_mac` | No | (Deprecated) Use `vm_networks` instead for network configuration |

## Storage Pools Configuration

By default, all libvirt data is stored in a single directory (`libvirt_storage_path`). You can create additional custom storage pools for different purposes or storage devices.

### Default storage

The default storage pool is automatically created at:
```yaml
libvirt_storage_path: /home/<user>/libvirt
```

All VM images, ISOs, and cloud images are stored here by default.

### Overriding default storage location

Change the default location in host_vars:
```yaml
libvirt_storage_path: /mnt/nvme/libvirt
```

### Creating custom storage pools

Define additional storage pools for high-performance storage, archives, or different mount points:

```yaml
storage_pools:
  - name: nvme-fast
    path: /mnt/nvme/libvirt
  - name: ssd-pool
    path: /mnt/ssd/libvirt
  - name: archive-pool
    path: /mnt/archive/libvirt
```

### Storage pool parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `name` | Yes | Unique pool name |
| `path` | Yes | Full path to storage directory |

### Using custom pools with VMs

Specify the full path when defining VM volumes:

```yaml
vms:
  - name: high-performance-vm
    memory: 16384
    vcpus: 8
    disk_size: 100
    volumes:
      - name: fast-data.qcow2
        size: 500
        path: /mnt/nvme/libvirt/high-performance-vm-fast-data.qcow2
```

**Notes:**
- Default pool is always created
- Custom pools are created in addition to the default
- All pools autostart automatically
- SELinux contexts applied automatically
- Ownership based on path (/home → user, others → root)

See [host_vars/storage_pools_examples.yml.example](host_vars/storage_pools_examples.yml.example) for complete examples.

## Network Configuration

VMs can be configured with multiple network interfaces, each with custom MAC addresses and attached to different virtual networks.

### Basic network configuration

By default, if no `networks` parameter is specified, VMs are created with a single NIC attached to the `default` libvirt network with an auto-generated MAC address.

### Single network interface

```yaml
vms:
  - name: myvm
    memory: 4096
    vcpus: 4
    disk_size: 50
    networks:
      - name: default
```

### Single network with custom MAC address

```yaml
vms:
  - name: myvm
    memory: 4096
    vcpus: 4
    disk_size: 50
    networks:
      - name: default
        mac: "52:54:00:11:22:33"
```

### Multiple network interfaces

```yaml
vms:
  - name: myvm
    memory: 4096
    vcpus: 4
    disk_size: 50
    networks:
      - name: default
        mac: "52:54:00:11:22:01"
      - name: storage-network
        mac: "52:54:00:aa:bb:01"
      - name: management-network
        mac: "52:54:00:cc:dd:01"
```

### Network configuration options

Each network in the `networks` list can have:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `name` | No | Virtual network name (default: "default") |
| `mac` | No | MAC address (auto-generated if not specified) |

**Note**: MAC addresses must be unique across all VMs and follow the format `XX:XX:XX:XX:XX:XX`. The prefix `52:54:00` is commonly used for KVM virtual machines.

### Creating virtual networks

Before using custom networks, ensure they exist in libvirt:

```bash
# List existing networks
virsh net-list --all

# Define a new network from XML
virsh net-define /path/to/network.xml

# Start the network
virsh net-start storage-network

# Set network to autostart
virsh net-autostart storage-network
```

Example network XML (`storage-network.xml`):
```xml
<network>
  <name>storage-network</name>
  <bridge name='virbr1'/>
  <forward mode='nat'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.2' end='192.168.100.254'/>
    </dhcp>
  </ip>
</network>
```

## Playbooks

This project includes several playbooks for different aspects of VM management:

### setup_libvirt.yml
Initial setup of libvirt on hosts. Installs packages, configures services, sets up storage pools, and configures permissions.

```bash
# Setup all hosts
ansible-playbook -i inventory setup_libvirt.yml

# Setup specific host
ansible-playbook -i inventory setup_libvirt.yml -l hostname

# Only install packages
ansible-playbook -i inventory setup_libvirt.yml --tags packages
```

### manage_host_storage.yml
Standalone storage pool, directory, and ISO management. Can be used to create, update, or reconfigure storage independently of VM operations, and download ISO images for VM installation.

```bash
# Setup storage for all hosts
ansible-playbook -i inventory manage_host_storage.yml

# Setup storage for specific host
ansible-playbook -i inventory manage_host_storage.yml -l hostname

# Only create directories
ansible-playbook -i inventory manage_host_storage.yml --tags directories

# Only configure storage pool
ansible-playbook -i inventory manage_host_storage.yml --tags pool

# Only download ISOs
ansible-playbook -i inventory manage_host_storage.yml --tags iso

# Setup everything (storage + ISOs)
ansible-playbook -i inventory manage_host_storage.yml
```

**Features:**
- Creates storage directories with proper permissions
- Handles both `/var/lib` and `/home` user paths
- Sets SELinux contexts (virt_image_t)
- Creates and activates libvirt storage pools
- Can rebuild pools if paths change
- Downloads ISO images from URLs with optional checksum verification
- Manages ISO file permissions and ownership
- Idempotent downloads (won't re-download existing files)

**ISO Configuration:**

Define ISOs to download in host_vars:
```yaml
iso_images:
  - url: "https://download.fedoraproject.org/pub/fedora/linux/releases/43/Server/x86_64/iso/Fedora-Server-dvd-x86_64-43-1.6.iso"
    alias: "fedora43x86.iso"
  - url: "https://releases.ubuntu.com/22.04/ubuntu-22.04.3-live-server-amd64.iso"
    alias: "ubuntu-22.04-server.iso"
    checksum: "sha256:abc123..."  # Optional
```

ISO parameters:
- `url`: Direct download URL for the ISO image (required)
- `alias`: Filename to save as in libvirt_storage_path (required)
- `checksum`: Optional checksum for verification (format: "algorithm:hash")

See [host_vars/iso_examples.yml.example](host_vars/iso_examples.yml.example) for complete examples.

### manage_host_network.yml
Virtual network management for libvirt. Create, configure, or destroy virtual networks with support for NAT, isolated (host-only), and bridge modes.

```bash
# Create networks from host_vars
ansible-playbook -i inventory manage_host_network.yml

# Create networks for specific host
ansible-playbook -i inventory manage_host_network.yml -l hostname

# List all networks
ansible-playbook -i inventory manage_host_network.yml --tags list

# Destroy specific network
ansible-playbook -i inventory manage_host_network.yml -e "network_name=storage-network" --tags destroy
```

**Supported Network Types:**
- **NAT**: Network with outbound connectivity via NAT, isolated from physical network
- **Isolated**: Host-only network, VMs can communicate with each other and host only
- **Bridge**: Direct connection to physical network via existing bridge interface

**Features:**
- Define networks in host_vars with custom IP ranges
- DHCP configuration with optional static assignments
- Automatic network startup and autostart configuration
- Support for multiple networks per host
- Network templates for each type

**Network Configuration Example:**
```yaml
networks:
  - name: storage-network
    type: nat
    bridge: virbr1
    ip: 192.168.100.1
    netmask: 255.255.255.0
    dhcp_start: 192.168.100.10
    dhcp_end: 192.168.100.254
    autostart: true
    dhcp_hosts:  # Optional static DHCP
      - mac: "52:54:00:aa:bb:01"
        name: storage-node-1
        ip: 192.168.100.11

  - name: management-network
    type: isolated
    bridge: virbr2
    ip: 192.168.200.1
    netmask: 255.255.255.0
    dhcp_start: 192.168.200.10
    dhcp_end: 192.168.200.254
    autostart: true

  - name: external-bridge
    type: bridge
    bridge: br0  # Must exist on host
    autostart: true
```

See [host_vars/network_examples.yml.example](host_vars/network_examples.yml.example) for complete examples.

## Volume Configuration

VMs can be configured with multiple disk volumes beyond the primary OS disk. This is useful for database servers, storage nodes, or any scenario requiring separate data volumes.

### Basic volume configuration

By default, every VM is created with a single primary disk specified by `vm_disk_size_gb`. This becomes the `vda` device.

### Single VM with additional volumes

```yaml
vms:
  - name: database-server
    memory: 8192
    vcpus: 4
    disk_size: 50  # Primary OS disk (vda)
    volumes:       # Additional data volumes
      - name: data01.qcow2
        size: 100  # Size in GB
      - name: data02.qcow2
        size: 200
```

### Multiple volumes with custom configuration

```yaml
vms:
  - name: storage-server
    memory: 4096
    vcpus: 2
    disk_size: 40
    volumes:
      - name: storage01.qcow2
        size: 500
        device: vdb  # Optional: specify device name
        format: qcow2  # Optional: disk format
      - name: storage02.qcow2
        size: 500
        device: vdc
      - name: logs.qcow2
        size: 50
        path: /home/jeyoung/libvirt/custom/logs.qcow2  # Optional: custom path
        device: vdd
```

### Volume configuration options

Each volume in the `volumes` list can have:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `name` | Yes | Volume filename (e.g., data01.qcow2) |
| `size` | Yes | Size in GB |
| `device` | No | Device name (vdb, vdc, etc.) - auto-assigned if not specified |
| `format` | No | Disk format (qcow2, raw, etc.) - defaults to qcow2 |
| `path` | No | Full path to volume - defaults to `<storage_pool>/<vm_name>-<name>` |

**Notes:**
- The primary OS disk (`vm_disk_size_gb`) is always created as `vda`
- Additional volumes are automatically assigned `vdb`, `vdc`, `vdd`, etc. in order
- Volume paths default to: `<storage_pool>/<vm_name>-<volume_name>`
  - Example: `/home/jeyoung/libvirt/images/database-server-data01.qcow2`
- You can override device assignment and paths as needed
- All volumes inherit the storage pool path unless `path` is explicitly specified

### Complete example with volumes and networks

```yaml
vms:
  - name: database-cluster-1
    memory: 16384
    vcpus: 8
    disk_size: 50
    autostart: true
    networks:
      - name: default
        mac: "52:54:00:10:00:01"
      - name: storage-network
        mac: "52:54:00:20:00:01"
    volumes:
      - name: postgres-data.qcow2
        size: 200
        device: vdb
      - name: postgres-wal.qcow2
        size: 50
        device: vdc
      - name: backups.qcow2
        size: 500
        device: vdd
```

See [host_vars/volumes_examples.yml.example](host_vars/volumes_examples.yml.example) for complete examples.

## ISO Mount Configuration

VMs can be created with ISO images mounted as CDROM devices. This is useful for OS installation from ISO or providing additional installation media.

### Basic ISO mount

Mount an ISO from the storage path:

```yaml
vms:
  - name: fedora-install
    memory: 4096
    vcpus: 2
    disk_size: 50
    iso: "fedora43x86.iso"  # Filename from libvirt_storage_path
```

### ISO with full path

Specify a full path to the ISO file:

```yaml
vms:
  - name: custom-install
    memory: 4096
    vcpus: 2
    disk_size: 40
    iso: "/home/jeyoung/libvirt/iso/ubuntu-22.04-server.iso"
```

### Complete installation example

VM configured for OS installation with ISO, multiple networks, and data volumes:

```yaml
vms:
  - name: database-install
    memory: 16384
    vcpus: 8
    disk_size: 50
    iso: "fedora43x86.iso"
    autostart: false  # Don't autostart during installation
    networks:
      - name: default
        mac: "52:54:00:10:00:10"
      - name: storage-network
        mac: "52:54:00:20:00:10"
    volumes:
      - name: data01.qcow2
        size: 200
      - name: data02.qcow2
        size: 200
```

### ISO mount behavior

When `vm_iso` is specified:
- **Boot order**: CDROM first, then hard disk
- **Device assignment**: ISO mounted as `sda` on SATA bus (read-only)
- **Disk devices**: Regular disks still use virtio (`vda`, `vdb`, etc.)
- **Path resolution**:
  - Filename only (e.g., `"fedora43x86.iso"`) → looked up in `libvirt_storage_path`
  - Absolute path (e.g., `"/home/user/custom.iso"`) → used as-is

### Installation workflow

1. **Download ISO** (if not already present):
   ```bash
   ansible-playbook -i inventory manage_host_storage.yml --tags iso
   ```

2. **Create VM with ISO mounted**:
   ```bash
   ansible-playbook -i inventory manage_host_vms.yml --skip-tags configure
   ```

3. **Install OS** via virt-manager console or VNC

4. **After installation**, remove ISO and redefine VM:
   - Edit host_vars file, remove the `iso:` line
   - Redefine VM: `ansible-playbook -i inventory manage_host_vms.yml --tags vm-create`
   - Or manually eject via virt-manager

**Tips:**
- Set `autostart: false` during installation to prevent automatic boot before setup
- Use `--skip-tags configure` when creating installation VMs
- ISOs must exist before VM creation (download first with manage_host_storage.yml)

See [host_vars/iso_mount_examples.yml.example](host_vars/iso_mount_examples.yml.example) for complete examples.

### manage_host_vms.yml
Unified VM creation and configuration. Handles both single VMs (command-line) and multiple VMs (inventory-based).

```bash
# Create VMs from host_vars
ansible-playbook -i inventory manage_host_vms.yml

# Create single VM
ansible-playbook -i inventory manage_host_vms.yml -e "vm_name=myvm"

# Create VMs for specific host
ansible-playbook -i inventory manage_host_vms.yml -l hostname

# Only create VMs (skip configuration)
ansible-playbook -i inventory manage_host_vms.yml --skip-tags configure

# Only configure existing VMs
ansible-playbook -i inventory manage_host_vms.yml --tags configure
```

**Features:**
- Unified logic for single and multiple VMs
- Architecture-aware (x86_64 and ARM64/aarch64)
- UEFI firmware support for ARM64
- Multiple network interfaces
- Multiple disk volumes with custom configuration
- Custom MAC addresses
- Automatic VM configuration after creation
- Waits for IP address assignment

### reset_host.yml
**DESTRUCTIVE**: Removes all VMs, networks, and storage from a host. Use with caution!

```bash
# Reset specific host (REQUIRED)
ansible-playbook -i inventory reset_host.yml -l hostname

# Reset all hosts (requires confirmation)
ansible-playbook -i inventory reset_host.yml --extra-vars "confirm_reset_all=yes"

# Only remove VMs
ansible-playbook -i inventory reset_host.yml -l hostname --tags vms

# Only remove networks
ansible-playbook -i inventory reset_host.yml -l hostname --tags networks

# Only remove storage
ansible-playbook -i inventory reset_host.yml -l hostname --tags storage
```

**Safety Features:**
- Requires explicit host limit or confirmation
- 5-second warning pause before execution
- Clear display of what will be removed
- Summary of removed and remaining resources

**What it removes:**
- All VMs (destroys running VMs, removes definitions and NVRAM)
- All custom networks (preserves default network)
- All storage pools
- Storage directories and VM disk images
- NVRAM firmware files
- Temporary VM definition files

## Advanced Usage

### Using tags to run specific tasks

```bash
# Only setup packages
ansible-playbook -i inventory setup_libvirt.yml --tags packages

# Only create VM disks
ansible-playbook -i inventory manage_host_vms.yml --tags disk

# Only start VMs
ansible-playbook -i inventory manage_host_vms.yml --tags start
```

### Debugging

Run with verbose output:
```bash
ansible-playbook -i inventory setup_libvirt.yml -vvv
```

Check facts from host:
```bash
ansible -i inventory libvirt_hosts -m setup | grep -i virt
```

### Verify nested virtualization

Intel CPUs:
```bash
cat /sys/module/kvm_intel/parameters/nested
```

AMD CPUs:
```bash
cat /sys/module/kvm_amd/parameters/nested
```

## Troubleshooting

### Permission denied errors

Ensure your user is in the libvirt group:
```bash
sudo usermod -aG libvirt $USER
newgrp libvirt
```

### Nested virtualization not enabled

For Intel:
```bash
sudo modprobe -r kvm_intel
sudo modprobe kvm_intel nested=1
```

For AMD:
```bash
sudo modprobe -r kvm_amd
sudo modprobe kvm_amd nested=1
```

To make permanent, create `/etc/modprobe.d/kvm.conf`:
```
options kvm_intel nested=1
# or for AMD:
options kvm_amd nested=1
```

### VM fails to get IP address

Check libvirt network:
```bash
virsh net-list
virsh net-dumpxml default
```

Restart libvirt:
```bash
sudo systemctl restart libvirtd
```

### SSH connection issues

Verify VM is running:
```bash
virsh list
```

Get VM IP:
```bash
virsh domifaddr <vm_name>
```

Check console:
```bash
virsh console <vm_name>
```

## Contributing

Feel free to submit issues and enhancement requests!

## License

This project is open source and available under the MIT License.

## References

- [libvirt Documentation](https://libvirt.org/)
- [Fedora Virtualization Guide](https://docs.fedoraproject.org/en-US/fedora/latest/system-administrators-guide/virtualization/)
- [Ansible libvirt Module](https://docs.ansible.com/ansible/latest/collections/community/libvirt/virt_module.html)
- [QEMU Documentation](https://www.qemu.org/documentation/)
