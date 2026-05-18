## OSP17.1-Galera-garbd-arbitare

1. 1-deploy-garbd-direct.yaml (The Infrastructure & Quorum Layer)
Purpose: Pairs the baremetal host with the pre-existing Controller cluster at the Corosync/OS layer.

Key Actions: * Injects native, persistent firewall rules directly into /etc/sysconfig/nftables.conf on the Garbd node to permanently open ports 2224 (PCSD), 5404/5405 (Corosync), and 4567 (Galera replication) across reboots.

Authenticates and joins the node to the active cluster using pcs cluster node add.

Surgically edits /etc/corosync/corosync.conf on the fly to inject quorum_votes: 2 into the garbd node definition, then live-reloads Corosync without impacting running services.

Variables to Customize:
Located in the vars: block at the top of the playbook:

YAML
vars:
  garbd_ip: "172.16.2.5"           # The real internal API IP of your Garbd host
  garbd_votes: 2                   # Keep as 2 to maintain the 5-vote asymmetric quorum
  pacemaker_pass: "cSCHexcx70UKqwgv" # The dynamic 'hacluster' password of your overcloud
  
How to find your pacemaker_pass: 
PcsdPassword can be found inside ~/overcloud-deploy/overcloud/ (stack user).

2. deploy-garbd-resource.yaml (The Application & Pacemaker Layer)
Purpose: Configures Pacemaker to manage the Galera Arbitrator daemon process safely.

Key Actions: * Queries an active galera-bundle container on the Controllers to dynamically discover runtime variables (wsrep_cluster_name and wsrep_cluster_address).

Sanitizes and re-formats the cluster connection string to append explicit ports (:4567) to each FQDN, satisfying the strict RHEL 9 OCF parser syntax requirements.

Creates a native Pacemaker resource (ocf:heartbeat:garbd) pinned exclusively to the Garbd node via location constraints, ensuring it never attempts to run inside the Controllers' Podman space.

Variables to Customize:
This playbook relies almost entirely on dynamic runtime discovery (it scrapes the configuration directly out of the running database containers). 
There are no hardcoded IP or password variables needed.
However, you must verify your inventory groups match your environment:

YAML
hosts: overcloud_Controller,overcloud_Garbd  # Ensure these groups match your Undercloud inventory file
These hosts groups can be indetified from tripleo-ansible-inventory.yaml available in ~/overcloud-deploy/overcloud (stack user)


3. cleanup-garbd.yaml (The De-provisioning Layer)
Purpose: Provides a safe rollback and clean uninstall path without causing a disruptive cluster fence or quorum loss.

Key Actions:

Gracefully stops and deletes the garbd-service resource from Pacemaker.

Expels the node topology from the controllers using pcs cluster node remove garbd --skip-offline --force to seamlessly downscale the quorum weight back to 3 votes.

Purges configuration signatures (pcs cluster destroy) and tokens locally on the baremetal host.

Variables to Customize:
Like the resource playbook, this script dynamically targets the cluster endpoints. 
The only requirement is confirming that the hostnames and groups line up with your tripleo-ansible-inventory.yaml:

YAML
hosts: overcloud_Controller,overcloud_Garbd  # Must target your accurate Controller and Garbd node naming
