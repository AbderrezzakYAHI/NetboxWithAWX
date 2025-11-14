
🌐 Exploring Netbox as a Dynamic Inventory for Ansible
This project demonstrates how to use NetBox as a dynamic inventory source for Ansible to automate network configuration management. By leveraging the netbox.netbox.nb_inventory plugin, you can dynamically target hosts and generate device-specific configuration files using Jinja2 templates.

🚀 Environment Setup
The environment for this example uses a virtualized setup within VMware Workstation  and consists of two virtual machines:


VM_Netbox: Running Ubuntu 24 with NetBox Community installed.
VM_Ansible: Running Ubuntu 24 with Ansible installed, acting as the control machine.
Both VMs are connected via NAT.

Prerequisites:
Ensure Ansible is installed on the control machine (VM_Ansible). The version used in this example is Ansible [core 2.9.6].
Verify that VM_Ansible can communicate with VM_Netbox using a ping test.

⚙️ NetBox Configuration (Data Requirements)
Before using NetBox as an inventory, you must configure certain data fields for the devices you wish to manage. These fields are used by the Ansible NetBox inventory plugin to populate host variables and groups.

Required Data	NetBox Location	Purpose for Ansible
Device Name	Devices → Device Name	
Used as hostname in the inventory.

Primary IP Address	Primary IP → Assign Interface	
Used as the Ansible variable ansible_host.

Platform (with slug)	Platform	
Tells Ansible how to connect (e.g., IOS, Junos).

Status	Status field	
Must be set to "Active" to be included in the inventory.

Device Role	Device Role	
Can be used for grouping (e.g., router, switch, firewall).

Site	Site	
Used for location grouping.

Device Type	Device Type	
Represents the platform or model.

Tags (optional)	Tags	
Used to group hosts dynamically.

Custom Fields (opt.)	Custom Fields	
Can be used for variables like ansible_user or ansible_port.

API Token Generation
An API token is required for Ansible to authenticate and interact with NetBox.

Go to the "Admin" section in the left-hand sidebar.

Click "API Tokens" under the AUTHENTICATION menu.

Fill out the token form (specify the User and optionally enable Write enabled for write operations).

Click "Create".


Crucially, copy the generated Key immediately, as you won't be able to see it again after saving.

📂 Project Structure
On the VM_Ansible control machine, the project is named Netbox-Ansible and contains the following file structure:

Netbox-Ansible/
├── ansible.cfg
├── netbox_inventory.yml
├── output/
│   ├── R91100-ESSBA.cfg (Example output file)
│   └── R92160-HDSBA.cfg (Example output file)
├── Playbook_Generating_config_with-netbox.yml
└── templates/
    └── config_template.j2
The output directory is where the generated configurations will be saved, and templates contains the Jinja2 template.

1. netbox_inventory.yml
This file configures the Ansible NetBox dynamic inventory plugin (plugin: netbox).

YAML
plugin: netbox
api_endpoint: http://192.168.127.37:8000
token: 9d4ba55384e148382b29ffd39fc6ec733a65345a  # IMPORTANT: Replace with your actual token
validate_certs: false
group_by:
  - sites
  - device_roles
compose:
  ansible_host: primary_ip4

api_endpoint: Specifies the NetBox URL.


token: Your generated API token (example shown).

group_by: Defines how hosts are grouped in the Ansible inventory. Here, hosts are grouped by their site and device role.


compose: Creates the Ansible variable ansible_host using the device's primary_ip4 address.

Testing the Inventory: You can test the dynamic inventory using the command:

Bash
ansible-inventory -i netbox_inventory.yml --list
This command outputs the structured inventory data obtained from NetBox.

2. ansible.cfg
This file ensures that Ansible correctly uses the dynamic inventory and sets other default behavior.

Ini, TOML
[defaults]
inventory = ./netbox_inventory.yml
collections_paths = ~/.ansible/collections
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml

inventory: Sets the path to the dynamic inventory file.


host_key_checking: Disabled for simplicity in this lab environment.


stdout_callback: Sets the output format to YAML.

3. Playbook_Generating_config_with-netbox.yml
This playbook targets devices with the NetBox device role "Router" and generates configuration files using a Jinja2 template.

YAML
---
- name: Create config using netbox
  hosts: device_roles_Router 
  gather_facts: False
  connection: local
  tasks:
    - name: "Generate configs using data obtained from netbox"
      template:
        src: "./templates/config_template.j2"
        dest: "./output/{{ inventory_hostname }}_{{ 'Y-%m-%d_%Hh-%Mm-%Ss' | strftime }}.cfg"

hosts: device_roles_Router: This is a key feature, as it dynamically targets only devices assigned the "Router" role in NetBox.

The template module generates the file.


dest: The output filename includes the hostname (inventory_hostname) and a timestamp for traceability.

4. config_template.j2
The Jinja2 template uses variables populated by the NetBox inventory plugin to create the configuration.

Code snippet
interface Gi2
  ip address {{ ansible_host.address.split('/')[0] }} 255.255.255.0
  description "Production line"
  no shutdown

{{ ansible_host.address.split('/')[0] }}: This extracts the primary IP address (e.g., 192.168.3.148/24) and strips the subnet mask to get just the IP (192.168.3.148).

▶️ Execution
Execute the Ansible playbook using the following command:

Bash
ansible-playbook Playbook_Generating_Config_with-netbox.yml
Execution Result
The playbook successfully generates configuration files for the targeted router devices (R91100-ESSBA and R92160-HDSBA).

Example Output File (R91100-ESSBA_2025-05-27_00h-29m-17s.cfg):

interface Gi2
ip address 192.168.3.148 255.255.255.0
description "Production line"
no shutdown
The IP address (192.168.3.148) is dynamically pulled from NetBox for the specific device.
