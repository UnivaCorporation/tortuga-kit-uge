# tortuga-kit-uge

## Overview

The Univa Grid Engine kit for Tortuga allows automated set up, configuration,
and management of a Univa Grid Engine cluster within the Tortuga environment.

This document describes possible installation customizations and administrative
workflows that exist beyond the basic Univa Grid Engine tools.

## Downloading Univa Grid Engine trial kit

The Univa Grid Engine 8.5.4 trial kit for Tortuga is available as a free
download from <http://www.univa.com/resources/univa-navops-launch-trial-kits.php>.

The kit is installed on the Tortuga installer using the `install-kit` command
as follows:

```shell
install-kit kit-uge-8.5.4-0.tar.bz2
```

## Types of installations/integrations

Before proceeding with any Univa Grid Engine kit installation, it is strongly
recommended to become familiar with the basic concepts of Tortuga as well as
familiarizing one's self with the Tortuga installation process.

See the [Tortuga Installation and Administration
Guide](https://github.com/UnivaCorporation/tortuga/blob/master/doc/tortuga-6-admin-guide.md)
for configuration details. The "Quickstart Installation" section details an
installation of Tortuga and Univa Grid Engine.

### Cloud-based

A slight variation from the traditional installation with the difference being
that the Tortuga installer is cloud-based and used to deploy cloud-based Univa
Grid Engine compute nodes. 

This installation type allows a full cloud-based Univa Grid Engine environment
with no on-premise (local) node requirement. It can be brought up,
expanded/contracted (compute nodes added/removed), and terminated on an
on-demand basis or left running indefinitely.

### On-premise (phsyical nodes/virtual machines)

This is the traditional installation where Tortuga is used to deploy a Univa
Grid Engine cluster on local (on-premise), usually physical, nodes.

In a typical Tortuga/Univa Grid Engine installation, the Univa Grid Engine
`qmaster` runs on the same host as the Tortuga installer.

### Hybrid

The hybrid installation is one in which the Univa Grid Engine cluster features
any combination of local (on-premise) nodes, and cloud-based nodes. This can
include integration with an existing Univa Grid Engine cluster.

The included recipe uses Amazon EC2-based compute instances as Univa Grid
Engine execution hosts, however the configuration steps are identical for any
supported Tortuga resource adapter.

The following scenarios are supported to enable Tortuga-managed execution hosts
running on:

* Amazon EC2
* Google Compute Engine
* Microsoft Azure
* Oracle Cloud Infrastructure
* VMware vSphere
* OpenStack (hosted or on-premise)

or any combination of the above. This allows, for example, a single Univa Grid
Engine cluster to span multiple cloud-providers.

The key to this configuration is network accessibility and DNS configuration:

* The NFS shared Univa Grid Engine spool directory must be accessible by the
  cloud instances.
* Nodes started on hypervisors and/or cloud-based instances must share the same
  DNS namespace as the (existing) Univa Grid Engine `qmaster` host.

### Prerequisites

For our example Univa Grid Engine cluster, we know the following:

| Item                     | Value                  |
| ------------------------ | ---------------------- |
| qmaster host name        | qmaster-01.example.com |
| SGE\_ROOT                | `/opt/uge-8.5.4`       |
| SGE\_CELL                | default                |
| Grid Engine admin user   | `sge`                  |
| Grid Engine admin UID    | 1500                   |
| Grid Engine admin group  | `sge`                  |
| Grid Engine admin GID    | 1500                   |

Ensure the example UID and GID do not conflict with existing UIDs/GIDs.

#### Univa Grid Engine installation directory shared over NFS

The path `/opt/uge-8.5.4` is shared by NFS from the qmaster host `qmaster-01.example.com`.

```shell
[root@qmaster-01 ~]# showmount -e
Export list for qmaster-01:
/opt/uge-8.5.4 *
```

#### Tortuga AWS resource adapter

We describe the AWS integration as example. The other supported cloud adapters
can be used in analogy.

Ensure Tortuga is able to create EC2 instances in your pre-configured [Amazon
VPC][] before proceeding with the Univa Grid Engine integration. Tortuga
EC2-based compute nodes must be able to automatically reach "Installed" state
after being added with the Tortuga `add-nodes` command.

Other cloud providers have similar concepts of a "virtual network" comparable
in functionality to [Amazon VPC][]. Consult the cloud provider's
documentation for additional information. Tortuga does **not** automatically
create or configure virtual networks.

#### Network connectivity/VPN

Network connectivity must exist between the Tortuga installer, Univa Grid Engine
qmaster, and any EC2 instances.

This will require properly configured VPN or [Amazon Direct Connect][],
security group(s), and AWS access tokens.

#### Amazon VPC

Set domain name and domain name server, as necessary, in "DHCP options set"
associated with Amazon VPC. This step is imperative to ensure the assigned
host names are resolvable on both the Univa Grid Engine qmaster host and the
Tortuga installer host.

The Univa Grid Engine qmaster host and Tortuga installer hosts must have the
same DNS settings.

EC2 instances started as part of the Amazon VPC must have a host name
resolvable on both the qmaster host and the Tortuga installer host. This
may require predefining DNS entries for all possible IP addresses in the
Amazon VPC subnet.

One possibility is for the DNS server managed by Tortuga to be the canonical
DNS server.  This would require `/etc/resolv.conf` on the qmaster host to be
configured to use the Tortuga managed DNS server.  For example:

```
search ...
options ...
nameserver <IP address of Tortuga installer>
nameserver ...
```

This is only one option and may **NOT** be required for your installation.

Consult your network administrator for further information.

### Configuration steps for (existing) Univa Grid Engine cluster

1. On a Univa Grid Engine administrative host, ensure the Tortuga installer host is added to
   the list of trusted hosts:

        qconf -ah <public hostname or FQDN of Tortuga installer>

     For example:

        qconf -ah tortuga.example.com

    This gives Tortuga the ability to add/remove `execd` hosts from the
    existing Univa Grid Engine cluster.

### Configuration steps for Tortuga installer

1. Ensure user/group of owner of NFS shared Univa Grid Engine cell directory exists on Tortuga installer.

    This may require either connecting the Tortuga installer to an
    existing NIS domain or LDAP configuration or simply creating a local
    user and group with matching uid and gid as that of the Univa Grid Engine
    administrator on the qmaster host.

    For example, if the NFS shared cell directory has ownership of
    `sge`:`sge` (uid: `1500`, gid: `1500`), then a local user 'sge' (uid:
    `1500`) and local group 'sge' (gid: `1500`) must be created.

    To create a local group/user `sge` with uid/gid `1500`/`1500`:

        groupadd --gid 1500 sge
        useradd --uid 1500 --gid 1500 sge

1. Create the mount point on the Tortuga installer to correspond with `SGE_ROOT`

        mkdir -p /opt/uge-8.5.4

    **Note:** It is necessary that this path is identical to that of `SGE_ROOT` on
    the qmaster host because the environment scripts
    (`$SGE_ROOT/$SGE_CELL/common/settings.sh` and
    `$SGE_ROOT/$SGE_CELL/common/settings.csh`) are configured this way.

1. Mount the shared Univa Grid Engine installation using NFS

    In this example, "qmaster-01.example.com" is the qmaster for the
    existing Univa Grid Engine cluster:

        mount qmaster-01.example.com:/opt/uge-8.5.4 /opt/uge-8.5.4

1. Source the Univa Grid Engine environment

        . /opt/uge-8.5.4/default/common/settings.sh

1. Copy directory helper scripts to `$SGE_ROOT`

    This is a set of "helper" scripts used by Tortuga to manage Univa Grid Engine.

    **Note:** due to network security and/or restrictions, the `root` user on the
    Tortuga installer may not have write permissions in the NFS-mounted
    `$SGE_ROOT`. In that case, copy these files as the Univa Grid Engine
    adminisrtrative user (`sge`, in this example). Alternatively, use
    `rsync` or `scp` to copy files to the `qmaster` host.

        su - -s /bin/bash sge
        source /opt/uge-8.5.4/default/common/settings.sh
        rsync -av /etc/puppetlabs/code/environments/production/modules/tortuga_kit_uge/files/setup $SGE_ROOT

    This will create a directory `setup` under $SGE_ROOT.

1. Configure Univa Grid Engine on Tortuga

    Create `default` Univa Grid Engine cluster

        uge-cluster create default

    Add settings for newly created Univa Grid Engine cluster:

        uge-cluster update default \
            --var unmanaged_qmaster=true \
            --var uge_user=sge \
            --var uge_uid=1500 \
            --var uge_group=sge \
            --var uge_gid=1500 \
            --var sge_cell_netpath=qmaster-01.example.com:/opt/uge-8.5.4/default

    **Note:** the NFS path specified by `sge_cell_netpath` must contain an
    accessible IP address or DNS resolvable host name. If using a host
    name, it must be DNS resolvable *from* the EC2 instance.  It may or
    may not require a fully-qualified domain name.

    In our basic example, all nodes contained in software profiles with the
    `execd` component enabled will be added to the existing Univa Grid Engine
    cluster.

1. Enable the `execd` component on the desired software profile(s)

        enable-component --software-profile execd execd
        uge-cluster update default --add-execd-swprofile execd

    Repeat this step if multiple software profiles will contain `execd` nodes.

1. Add the "tortuga" complex resource to the Univa Grid Engine cluster

        qconf -sc >/tmp/tortuga_complexes.txt
        echo "tortuga tortuga BOOL == YES NO FALSE 0" >>/tmp/tortuga_complexes.txt
        qconf -Mc /tmp/tortuga_complexes.txt
        rm -f /tmp/tortuga_complexes.txt

    All nodes added by Tortuga have the complex value `tortuga=TRUE`.
    This is to distinguish between pre-existing Univa Grid Engine compute nodes,
    and compute nodes added by Tortuga.

1. Add nodes in Tortuga

    Any nodes added to a Tortuga software profile with the `execd`
    component enabled should now be automatically added to the existing
    Univa Grid Engine cluster with the qmaster hosted on the host
    `qmaster-01.example.com`.

    Conversely, nodes deleted from Tortuga will automaticaly removed from
    the Univa Grid Engine cluster.

        add-nodes -n<count> --software-profile execd --hardware-profile <name>

1. Display all Univa Grid Engine nodes added by Tortuga

    Since all Tortuga-managed resources have the associated complex
    resource `tortuga`, they can be queried as follows:

        qstat -f -l tortuga

\newpage

## Univa Grid Engine cluster configuration

Univa Grid Engine cluster configuration under Tortuga is managed using the
`uge-cluster` command-line tool.

### Command reference

#### Display all UGE clusters

List Grid Engine clusters using `uge-cluster list`.

For example:

~~~shell
[root@c7inst ~]# uge-cluster list
default
~~~

Adding the `--verbose` option will display all clusters, software profiles
and settings for each cluster.

~~~shell
[root@c7inst ~]# uge-cluster list --verbose
Cluster name: default
  Qmaster software profiles:
    - Installer (Installer software profile)
  Execd software profiles:
    - execd (execd Nodes)
  Settings:
    - shared_install: true
    - manage_nfs: false
    - uge_user: ugeadmin
    - sge_cell_netpath: %(qmaster)s:%(sge_root)s/%(cell_name)s
    - uge_group: ugeadmin
    - uge_uid: 2002
    - uge_gid: 2002
    - sge_root: /opt/uge-lives-here
~~~

#### Display Univa Grid Engine cluster settings

    uge-cluster show <cluster name>

For example, the following command displays the Univa Grid Engine cluster configuration
for the default` cluster:

    uge-cluster show default

Adding the `--verbose` argument will display Univa Grid Engine node details.

~~~shell
[root@c7inst ~]# uge-cluster show default --verbose
Cluster name: default
  Qmaster software profiles:
    - Installer (Installer software profile)
        - c7inst.univa.com
  Execd software profiles:
    - execd (execd Nodes)
        - compute-01.private
        - compute-02.private
  Settings:
    - shared_install: true
    - manage_nfs: false
    - uge_user: ugeadmin
    - sge_cell_netpath: %(qmaster)s:%(sge_root)s/%(cell_name)s
    - uge_group: ugeadmin
    - uge_uid: 2002
    - uge_gid: 2002
    - sge_root: /opt/uge-lives-here
~~~

#### Add `execd` software profile to Univa Grid Engine cluster

Tortuga is not limited to having a single software profile designated as
an `execd` software profile. Enabling the `execd` component on additional
software profiles allows, for example, multiple different operating systems
and/or software stacks to become Univa Grid Engine compute nodes:

    uge-cluster update default \
        --add-execd-swprofile <software profile name>

#### Remove `execd` software profile from Univa Grid Engine cluster

First, disable the `execd` component from the software profile:

    disable-component --software-profile <software profile name>

Update Univa Grid Engine cluster configuration:

    uge-cluster update default \
        --delete-execd-swprofile <software profile name>
    /opt/puppetlabs/bin/puppet agent --no-daemonize --verbose --onetime

#### Support for additional Univa Grid Engine clusters

Tortuga and Univa Grid Engine support multiple concurrent clusters.

Creating additional Univa Grid Engine clusters with Tortuga is as simple as creating
new software and hardware profiles to represent nodes comprising the new Univa Grid
Engine cluster.

##### Automatically create hardware/software profiles

Use the following command to create a Univa Grid Engine cluster named "cluster2" and
automatically create associated hardware and software profiles:

    uge-cluster create cluster2 --create-profiles

**Note:** the hardware and software profiles are named similarly, however
they are unique profiles. It is possible to rename them in order to
prevent possible confusion. See the documentation for
`update-hardware-profile` and `update-software-profile` for syntax for
renaming profiles.

After running this command, add node to "cluster2-qmaster":

    add-nodes -n1 \
        --software-profile cluster2-qmaster \
        --hardware-profile cluster2-qmaster

Add compute (`execd`) nodes to the new cluster as follows:

    add-nodes -n3 \
        --software-profile cluster2-execd \
        --hardware-profile cluster2-execd

##### Manually create hardware/software profiles

1. Create profile(s) for `qmaster`, as necessary

        create-software-profile ... --name cluster2-qmaster

    **Hint:** use `copy-software-profile` to copy an existing software profile.

    Do not forget to map the software profile to an existing hardware
    profile or create new hardware profile as necessary.

    Enable `qmaster` component on software profile:

        enable-component --software-profile cluster2-qmaster qmaster

1. Create profile(s) for `execd` nodes, as necessary.

        create-software-profile ... --name cluster2-execd

    Enable `execd` component on software profile:

        enable-component --software-profile cluster2-execd execd

1. Register new Univa Grid Engine cluster

        uge-cluster create cluster2
        uge-cluster update cluster2 \
            --add-qmaster-swprofile cluster2-qmaster
        uge-cluster update cluster2 \
            --add-execd-swprofile cluster2-execd
        uge-cluster update cluster2 \
            --var sge_cell_netpath="%(qmaster)s:%(sge_root)s/%(cell_name)s"
        uge-cluster update cluster2 --var manage_nfs=true

    Adjust `sge_cell_netpath` configuration as appropriate for site-specific
    configuration (i.e. network attached storage). If using customized settings,
    set value of `manage_nfs` to `false`, which implies that Tortuga will
    automatically manage the NFS shared spool directory for `execd` hosts.

1. Add node(s) to the Univa Grid Engine cluster, as appropriate

    Add `qmaster` node(s):

        add-nodes ... --software-profile cluster2-qmaster ...

    Add `execd` node(s):

        add-nodes ... --software-profile cluster2-execd ...

\newpage

## Advanced Topics

### How do I automatically add execution hosts to a Univa Grid Engine hostgroup?

Create a file `$SGE_ROOT/$SGE_CELL/common/config.<NAME>` (where `NAME` is the
software profile name) with the Univa Grid Engine hostgroup name as the only
entry in the file.

For example, to have nodes in the "execd" software profile automatically join
the hostgroup "@myhosts", run the following command:

    echo "@myhosts" >$SGE_ROOT/$SGE_CELL/common/config.execd

Software profile names are case-sensitive and must match the exact name
displayed by `get-software-profile-list`.

All hosts added to the "execd" software profile will now be automatically added
to the "@myhosts" hostgroup.

**Tip:** this process can be repeated for any number of software profiles. This
makes it possible to create a separate software/hardware profile for each
instance/VM type/size, Univa Grid Engine hostgroup(s) associated with that and,
if desired, a separate Univa Grid Engine queue associated with those
hostgroup(s).

**Note:** the hostgroup "@myhost" must already exist. Use the Univa Grid Engine
command `qconf -ahgrp` to create a new hostgroup. Consult Univa Grid Engine
documentation for further information on `qconf` command syntax.

### Adding Tortuga-managed hosts to a specific queue

Create a queue in Univa Grid Engine using `qconf -aq` and associate that queue with
the hostgroup that Tortuga automatically adds nodes to (see above)

For example:

    qconf -mattr queue hostlist "@myhosts" all.q

Slot availability can be displayed using `qstat -f`.

### How do I set the slot count on execd hosts?

Since Tortuga 6.3.0, the slot counts for `execd` hosts are automatically
maintained on AWS, Google Compute Engine, and Microsoft Azure instances.

### Managing Univa Grid Engine submit hosts

Enable the `submit` component (from the Univa Grid Engine kit) on desired
software profile(s):

    enable-component --software-profile <software profile name> submit
    uge-cluster update <Grid Engine cluster> \
        --add-submit-swprofile <software profile name>
    /opt/puppetlabs/bin/puppet agent --onetime --no-daemonize --verbose

Any new nodes added to specified software profile will become Univa Grid Engine submit hosts.

**Note:** do not enable the Univa Grid Engine `submit` component on hosts that have the `execd` component already enabled.

### Univa Grid Engine REST Webservice

The Grid Engine REST webservice can be enabled as follows (assuming the
`qmaster` is running on the same host as the Tortuga installer):

    enable-component -p webservice
    /opt/puppetlabs/bin/puppet agent --onetime --no-daemonize --verbose

If the `qmaster` is running on a standalone node, the `webservice` component
must be enabled on the appropriate software profile. For example, if the
`qmaster` software profile was named `qnaser`:

    enable-component --software-profile qmaster webservice

If the `qmaster` node already exists, run `schedule-update` from the Tortuga
installer to synchronize the `qmaster` node.

Please refer to official Univa Grid Engine REST webservice documentation for
further information.

### Univa Grid Engine installation customizations

After enabling the `qmaster` component and *prior* to running the Puppet
sync, the installation file template for Univa Grid Engine may be modified to
change installation defaults.

For example, if using non-default spooling method, different network port
numbers, etc., these values can be changed such that Tortuga deploys
customized Univa Grid Engine installation.

The Puppet feature 

This file is managed by Puppet and found in
`/etc/puppetlabs/code/environments/production/modules/...
tortuga_kit_uge/templates/qmaster.conf.tmpl`.

**Note:** the values for `ADMIN_HOST_LIST`, `SUBMIT_HOST_LIST`, and
`EXEC_HOST_LIST` will be automatically managed by Tortuga.

Refer to official Univa Grid Engine documentation for further details on
available customiztaions.

#### Setting Univa Grid Engine user/group/uid/gid

When integrating into an existing Univa Grid Engine cluster, it is necessary
for the Tortuga managed Univa Grid Engine user and group settings to match the
existing user and group.

These settings are configured through Puppet module configuration. This is done
through the use of [Hiera](https://puppet.com/docs/puppet/5.4/hiera_intro.html)
configuration. Hiera is a key/value lookup used for separating data from Puppet
code.

Add the following settings to
`/etc/puppetlabs/code/environments/production/hieradata/tortuga-common.yaml`:

For example, assuming the Univa Grid Engine user is `ugeuser` with `uid: 1234`
and group `ugegroup` with `gid: 4567`:

```yaml
tortuga_kit_uge::uge_user: ugeuser
tortuga_kit_uge::uge_uid: 1234
tortuga_kit_uge::uge_group: ugegroup
tortuga_kit_uge::uge_gid: 4567
```

**Note:** do **not** make user/group changes to an existing cluster! This will
cause failure of the `qmaster` and/or `execd` ndoes.

Optionally, management of the Univa Grid Engine user/group can be disabled with
the setting:

```yaml
tortuga_kit_uge::qmaster::manage_user: false
```

This is required when the Univa Grid Engine user/group is managed by a
directory service, such as NIS or LDAP.

#### NFS client mount options

Custom NFS client mount options are required for executions hosts using Hiera.

Set the value in
`/etc/puppetlabs/code/environments/production/hieradata/tortuga-common.yaml`
(or in a software profile specific Hiera file) as follows:

```yaml
tortuga_kit_uge::execd::nfs_mount_options:
  - defaults
  - vers=3
  - tcp
  - intr
```

The value of `tortuga_kit_uge::execd::nfs_mount_options` can also be a properly
formatted string (comma-separated values in the format of the `fstab` mount
options):

```yaml
tortuga_kit_uge::execd::nfs_mount_options: vers=3,intr
```

#### Shared vs. standalone execd installation

This option allows sharing of the Univa Grid Engine installation directory over
NFS. Default behaviour is to install Univa Grid Engine binaries locally on all
execd nodes.

Use the following command to enable NFS shared installations:

```shell
uge-cluster update <UGE CLUSTER NAME> --var shared_install=true
```

All execution nodes in the specified cluster will NFS mount the Univa Grid
Engine root directory (which must be manually exported by NFS from the Tortuga
installer/qmaster host), instead of installing Univa Grid Engine on the local
filesystem.

**Hint:** it is recommended not to use this installation type when running
cloud-based execd nodes in an hybrid configuration. This would cause
bottlenecks in executing Univa Grid Engine binaries over the VPN/WAN connection
to the cloud provider. Also for larger clusters it is recommended not to use
this option because of similar bottlenecks when loading binaries.

### Univa Grid Engine Puppet module reference

The Tortuga Univa Grid Engine Puppet module exposes many user configurable
settings for customizing the Univa Grid Engine installation within Tortuga.
These configurable settings are applied through the use of a Puppet feature
named [Hiera](https://puppet.com/docs/puppet/5.4/hiera_intro.html). Hiera is a
key/value lookup used to separate data from Puppet code.

The following settings can be configured through Hiera
(`/etc/puppetlabs/code/environments/production/hieradata/common.yaml` or a
software profile specific Hiera configuration):

#### Common settings

##### Disable management of Univa Grid Engine user/group

```yaml
tortuga_kit_uge::manage_user: false
```

This is necessary for environments where Univa Grid Engine user/group
information is stored in NIS, LDAP, or other directory service.

##### User/group settings

```yaml
tortuga_kit_uge::uge_user: "sge"
tortuga_kit_uge::uge_uid: 269
tortuga_kit_uge::uge_group: "sge"
tortuga_kit_uge::uge_gid: 269
```

##### Cluster name

The key `tortuga_kit_uge::cluster_name` is used to define the cluster name (not
to be confused with the cell name). It defaults to `tortuga` and is generally
not changed.

```yaml
tortuga_kit_uge::cluster_name: "tortuga"
```

##### Disable managed installation

Automatic installation of Univa Grid Engine can be disabled by the following
setting:

```yaml
tortuga_kit_uge::managed_install: false
```

This would be used, for example, where the Univa Grid Engine binaries are already
installed in the compute nodes.

##### Disable managed NFS

```yaml
tortuga_kit_uge::manage_nfs: false
```

##### Configure `qmaster` and `execd` network ports

```yaml`
tortuga_kit_uge::sge_qmaster_port: 6444
tortuga_kit_uge::sge_execd_port: 6445
```

#### qmaster

```yaml
tortuga_kit_uge::qmaster::sge_root
tortuga_kit_uge::qmaster::cell_name
tortuga_kit_uge::qmaster::manage_service
tortuga_kit_uge::qmaster::manage_nfs
tortuga_kit_uge::qmaster::manage_user
tortuga_kit_uge::qmaster::managed_install
tortuga_kit_uge::qmaster::sge_qmaster_port
tortuga_kit_uge::qmaster::sge_execd_port
```

#### execd

```yaml
tortuga_kit_uge::execd::sge_root
tortuga_kit_uge::execd::cell_name
tortuga_kit_uge::execd::managed_install
tortuga_kit_uge::execd::manage_user
tortuga_kit_uge::execd::manage_service
tortuga_kit_uge::execd::shared_install
```

\newpage

## Upgrading Univa Grid Engine

The Univa Grid Engine installation deployed by Tortuga can be upgraded using
the standard Univa Grid Engine upgrade procedure (see Univa Grid Engine
documentation for further details).

The procedure for upgrading the Univa Grid Engine kit is as follows:

1. Use `disable-component` to disable all Univa Grid Engine components
    (`qmaster`, `execd`, `submit`, and `webservice`). Make a note of which
    software profile(s) have which components enabled.

    For example:

        disable-component -p qmaster
        disable-component -p webservice
        disable-component --software-profile execd execd

2. Remove the existing (old) Univa Grid Engine kit using `delete-kit`
3. Install updated (new) Univa Grid Engine kit using `install-kit`
4. Re-enable the components on the profiles noted from the first step.
5. Copy Univa Grid Engine distribution tarballs into `$TORTUGA_ROOT/www_int`
6. Install updated Puppet module

        puppet module install --force $TORTUGA_ROOT/kits/kit-uge-8.5.4-0/univa-tortuga_kit_uge-8.5.4.tar.gz

7. Follow official Univa Grid Engine upgrade procedure

    Please note, this procedure will not upgrade existing execution nodes without
    manual intervention (notably, changing the NFS exported spool directory),
    although with some manual steps it is possible.

[Amazon VPC] https://aws.amazon.com/vpc/ "Amazon VPC"
[Amazon Direct Connect] https://aws.amazon.com/directconnect/ "AWS Direct Connect"
