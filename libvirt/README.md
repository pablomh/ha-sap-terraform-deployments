# Terraform cluster deployment with Libvirt

# Table of content:

- [Requirements](#requirements)
- [Howto](#quickstart)
- [Monitoring](../doc/monitoring.md)
- [Netweaver](../doc/netweaver.md)
- [DRBD](../doc/drbd.md)
- [QA](../doc/qa.md)
- [Design](#design)
- [Specifications](#specifications)
- [Troubleshooting](#troubleshooting)

# Requirements

1. You need to have Terraform and the the Libvirt provider for Terraform. You may download packages from the
   [openSUSE Build Service](http://download.opensuse.org/repositories/systemsmanagement:/terraform/) or
   [build from source](https://github.com/dmacvicar/terraform-provider-libvirt)

   You will need to have a working libvirt/kvm setup for using the libvirt-provider. (refer to upstream doc of [libvirt provider](https://github.com/dmacvicar/terraform-provider-libvirt))

2. You need to fulfill the system requirements provided by SAP for each Application. At least 15 GB of free disk space and 512 MiB of free memory per node.

# Quickstart

1) Make sure you use terraform workspaces, create new one with: ```terraform workspace new $USER```

  For more doc, see: [workspace](../doc/workspaces-workflow.md).
  If you don't create a new one, the string `default` will be used as workspace name. This is however highly discouraged since the workspace name is used as prefix for resources names, which can led to conflicts to unique names in a shared server ( when using a default name).

2) Edit the `terraform.tfvars.example` file, following the Readme.md in the provider directory.

3) **[Adapt saltstack pillars](../pillar_examples/)**

4) Deploy with:

```bash
terraform workspace new myworkspace # The workspace name will be used to create the name of the created resources as prefix (`default` by default)
terraform init
terraform apply
terraform destroy
```

# Design

This project is mainly based in [sumaform](https://github.com/uyuni-project/sumaform/)

Components:

- **modules**: Terraform modules to deploy a basic two nodes SAP HANA environment.
- **salt**: Salt provisioning states to configure the deployed machines with the
all required components.


### Terraform modules
- [hana_node](modules/hana_node): Specific SAP HANA node defintion. Basically it calls the
host module with some particular updates.
- [netweaver_node](modules/netweaver_node): SAP Netweaver environment allows to have
a Netweaver landscape working with the SAP Hana database.
- [drbd_node](modules/drbd_node): DRBD cluster for NFS share.
- [iscsi_server](modules/iscsi_server): Machine to host a iscsi target.
- [monitoring](modules/monitoring): Machine to host the monitoring stack.
- [shared_disk](modules/shared_disk): Shared disk, could be used as a sbd device.

### Salt modules
- [pre_installation](../salt/pre_installation): Adjust the configuration needed for
defult module.
- [default](../salt/default): Default configuration for each node. Install the most
basic packages and apply basic configuration.
- [hana_node](../salt/hana_node): Apply SAP HANA nodes specific updates to install
SAP HANA and enable system replication according [pillar](../pillar_examples/libvirt/hana.sls)
data. You can also use the provided [automatic pillars](../pillar_examples/automatic/hana).
- [drbd_node](../salt/drbd_node): Apply DRBD nodes specific updates to configure
DRBD cluster for NFS share according [drbd pillar](../pillar_examples/libvirt/drbd/drbd.sls)
and [cluster pillar](../pillar_examples/libvirt/drbd/cluster.sls). You can also use the
provided [automatic pillars](../pillar_examples/automatic/drbd).
- [monitoring](../salt/monitoring): Apply prometheus monitoring service configuration.
- [iscsi_srv](../salt/iscsi_srv): Apply configuration for iscsi target.
- [netweaver_node](../salt/netweaver_node): Apply netweaver packages and formula.
- [qa_mode](../salt/qa_mode): Apply configuration for Quality Assurance testing.

# Specifications

* main.tf

**main.tf** stores the configuration of the terraform deployment, the infrastructure configuration basically. Here some important tips to update the file properly (all variables are described in each module variables file):

- **qemu_uri**: Uri of the libvirt provider.
- **base_image**: The cluster nodes image is selected updating the *image* parameter in the *base* module.
- **network_name** and **bridge**: If the cluster is deployed locally, the *network_name* should match with a currently available virtual network. If the cluster is deployed remotely, leave the *network_name* empty and set the *bridge* value with remote machine bridge network interface.
- **hana_inst_media**: Public media where SAP HANA installation files are stored.
- **iprange**: IP range addresses for the isolated network.
- **isolated_network_bridge**: A name for the isolated virtual network bridge device. It must be no longer than 15 characters. Leave empty to have it auto-generated by libvirt.
- **host_ips**: Each host IP address (sequential order).
- **shared_storage_type**: Shared storage type between iscsi and KVM raw file shared disk. Available options: `iscsi` and `shared-disk`.
- **iscsi_srv_ip**: IP address of the machine that will host the iscsi target (only used if `iscsi` is used as a shared storage for fencing)
- **iscsi_image**: Source image of the machine hosting the iscsi target (sles15 or above) (only used if `iscsi` is used as a shared storage for fencing)
- **monitoring_image**: Source image of the machine hosting the monitoring stack (if not set, the same image as the hana nodes will be used)
- **monitoring_srv_ip**: IP address of the machine that will host the monitoring stack
- **ha_sap_deployment_repo**: Repository with HA and Salt formula packages. The latest RPM packages can be found at [https://download.opensuse.org/repositories/network:/ha-clustering:/Factory/{YOUR OS VERSION}](https://download.opensuse.org/repositories/network:/ha-clustering:/Factory/)
- **devel_mode**: Whether or not to install HA/SAP packages from ha_sap_deployment_repo
- **scenario_type**: SAP HANA scenario type. Available options: `performance-optimized` and `cost-optimized`.
- **provisioner**: Select the desired provisioner to configure the nodes. Salt is used by default: [salt](../salt). Let it empty to disable the provisioning part.
- **background**: Run the provisioning process in background finishing terraform execution.
- **reg_code**: Registration code for the installed base product (Ex.: SLES for SAP). This parameter is optional. If informed, the system will be registered against the SUSE Customer Center.
- **reg_email**: Email to be associated with the system registration. This parameter is optional.
- **reg_additional_modules**: Additional optional modules and extensions to be registered (Ex.: Containers Module, HA module, Live Patching, etc). The variable is a key-value map, where the key is the _module name_ and the value is the _registration code_. If the _registration code_ is not needed, set an empty string as value. The module format must follow SUSEConnect convention:
    - `<module_name>/<product_version>/<architecture>`
    - *Example:* Suggested modules for SLES for SAP 15


          sle-module-basesystem/15/x86_64
          sle-module-desktop-applications/15/x86_64
          sle-module-server-applications/15/x86_64
          sle-ha/15/x86_64 (use the same regcode as SLES for SAP)
          sle-module-sap-applications/15/x86_64

For more information about registration, check the ["Registering SUSE Linux Enterprise and Managing Modules/Extensions"](https://www.suse.com/documentation/sles-15/book_sle_deployment/data/cha_register_sle.html) guide.

[Specific QA variables](../doc/qa.md#specific-qa-variables)

If the current *main.tf* is used, only *uri* (usually SAP HANA cluster deployment needs a powerful machine, not recommended to deploy locally) and *hana_inst_media* parameters must be updated.

* hana.sls

**hana.sls** is used to configure the SAP HANA cluster. Check the options in: [saphanabootstrap-formula](https://github.com/SUSE/saphanabootstrap-formula)

* cluster.sls

**cluster.sls** is used to configure the HA cluster. Check the options in: [habootstrap-formula](https://github.com/SUSE/habootstrap-formula)


# Troubleshooting

### Resources have not been destroyed

Sometimes it happens that created resources are left after running
`terraform destroy`. It happens especially when the `terraform apply` command
was not successful and you tried to destroy the setup in order of resetting the
state of your terraform deployment to zero.
It is often helpful to simply run `terraform destroy` again. However, even when
it succeeds in this case you might still want to check manually for remaining
resources.

For the following commands you need to use the command line tool Virsh. You can
retrieve the QEMU URI Virsh is currently connected to by running the command
`virsh uri`.

#### Checking networks

You can run `virsh net-list --all` to list all defined Libvirt networks. You can
delete undesired ones by executing `virsh net-undefine <network_name>`, where
`<network_name>` is the name of the network you like to delete.

#### Checking domains

For each node a domain is defined by Libvirt in order to address the specific
machine. You can list all domains by running the command `virsh list`. When you
like to delete a domain you can run `virsh undefine <domain_name>` where
`<domain_name>` is the name of the domain you like to delete.

#### Checking images

In case you experience issues with your images such as install ISOs for
operating systems or virtual disks of your machine check the following folder
with elevated privileges: `sudo ls -Faihl /var/lib/libvirt/images/`

#### Packages failures

If some package installation fails during the salt provisioning, the
most possible thing is that some repository is missing.
Add the new repository with the needed package and try again.
