# Host Variables Directory

This directory contains host-specific variables for your libvirt hosts. You can define VMs and override default settings on a per-host basis.

## Usage

Create a YAML file named after your host (exactly as it appears in your inventory):

- For host `aarch64-002.sys.eng.rdu2.dc.redhat.com` → `aarch64-002.sys.eng.rdu2.dc.redhat.com.yml`
- For host `127.0.0.1` → `127.0.0.1.yml`
- For host alias `aarch64-dev` → `aarch64-dev.yml`

## Example: Defining VMs for a Host

Create `host_vars/aarch64-002.sys.eng.rdu2.dc.redhat.com.yml`:

```yaml
---
# VMs to create on this host
vms:
  - name: ocp-master-1
    memory: 16384  # MB
    vcpus: 8
    disk_size: 120  # GB
    autostart: true

  - name: ocp-worker-1
    memory: 32768
    vcpus: 16
    disk_size: 200
    autostart: true

  - name: dev-vm-001
    memory: 4096
    vcpus: 4
    disk_size: 50
    autostart: false
```

## Example: Overriding Storage Path

```yaml
---
# Custom storage path for this host
libvirt_storage_path: /mnt/nvme/libvirt

# VMs for this host
vms:
  - name: high-perf-vm
    memory: 65536
    vcpus: 32
    disk_size: 500
    autostart: true
```

## VM Properties

| Property | Required | Default | Description |
|----------|----------|---------|-------------|
| `name` | Yes | - | Unique name for the VM |
| `memory` | No | 2048 | RAM in MB |
| `vcpus` | No | 2 | Number of virtual CPUs |
| `disk_size` | No | 20 | Disk size in GB |
| `autostart` | No | false | Start VM automatically on host boot |

## Creating VMs from host_vars

After defining VMs in host_vars files, create them with:

```bash
# Create and configure VMs for all hosts with host_vars definitions
ansible-playbook -i inventory manage_host_vms.yml

# Create and configure VMs for specific host only
ansible-playbook -i inventory manage_host_vms.yml -l aarch64-002.sys.eng.rdu2.dc.redhat.com

# Create VMs without configuration (faster)
ansible-playbook -i inventory manage_host_vms.yml --skip-tags configure

# Only configure already-created VMs
ansible-playbook -i inventory manage_host_vms.yml --tags configure
```

## Example Files

See the `.example` files in this directory for complete examples:
- `aarch64-002.sys.eng.rdu2.dc.redhat.com.yml.example` - OpenShift cluster setup example
