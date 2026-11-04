# openshift-day2

Day-2 configuration for OpenShift clusters, to install common operators and get
the cluster from an freshly installed state to a consumable state.

## Prerequisites

- `oc` CLI installed and logged in to the target cluster
  (`oc login -u kubeadmin ...`) before running the playbook.
- `htpasswd` binary installed on the Ansible host (`apache2-utils` on Debian/
  Ubuntu or `httpd-tools` on Fedora/RHEL). OpenShift requires its `$2y$` bcrypt
  hashes; Python passlib's `$2b$` hashes are not accepted by the HTPasswd IDP.
- Ansible and its Python dependencies in a virtual environment (from this
  directory):
  ```
  python3 -m venv .venv
  . .venv/bin/activate
  python -m pip install ansible-core kubernetes jsonpatch
  ansible-galaxy collection install -r requirements.yml
  ```
  Run the playbook with the virtual environment active. The local tasks use the
  same Python interpreter as `ansible-playbook`, so installing `kubernetes` only
  in a different Python environment will not work.

## Optional manual steps

### Add any further SSH keys (manual)

If additional people or machines need SSH access to the OpenShift, add their
public keys for the `core` user through an OpenShift MachineConfig.

```
oc get machineconfig
oc edit machineconfig 99-master-ssh
```

### Wipe the storage disk (manual, destructive)

This is required if the node has more disks and LVM storage was used on it
before. Confirm `/dev/nvme0n1` is the **data disk, not the OS disk**, and that
`vg1` belongs only to it (see `lvms_device_paths` and `lvms_vg_name` in
`group_vars/all.yml`). Then, before running `lvm`:

```sh
$NODE=<node>
oc debug node/$NODE -- chroot /host sh -c 'vgchange -an vg1 && wipefs -fa /dev/nvme0n1'
```

Deactivating the old VG clears its live LVM mappings; `wipefs` clears the disk
signatures.

## Running the playbook

Prepare `secrets.yml` with an HTPasswd username and password, a Cloudflare API
token, and an email address you control for the Let's Encrypt ACME account:

```
cp -n secrets.yml.example secrets.yml
# if secrets.yml already exists, keep its credentials and add any missing entries
# edit secrets.yml with your HTPasswd username/password, token and email
```

Create the Cloudflare token under **Account API Tokens** and scope it to the
public DNS zone containing the cluster's ingress domain, with
**Zone → DNS → Edit** and **Zone → Zone → Read** permissions. The email address
is not secret, but is the ACME account contact;
do not use `noreply@redhat.com`. cert-manager discovers the default ingress
domain from OpenShift, so you do not need to enter the certificate DNS names
yourself. Check that this domain belongs to your Cloudflare-managed zone before
running:

```
oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.status.domain}'
```

Review the settings in `group_vars/all.yml` if the devices listed there match
the actual hardware.

```
vi group_vars/all.yml
```

Run all tasks in the playbook:

```
ansible-playbook -i localhost, playbook.yml
```

To run tasks selectively `--tags` or `--skip-tags` can be used:

| Operator / feature                  | Tag                  | What it configures                                                                                                                                      |
| ----------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cluster readiness check             | `readiness` (always) | Checks all ClusterOperators are Available, not Progressing, and not Degraded before any tasks run; at most 30 seconds per condition (90 seconds total). |
| Local HTPasswd admin                | `auth`               | Creates the user, grants `cluster-admin`, and waits for the OAuth rollout if the identity provider changes.                                             |
| cert-manager + Let's Encrypt        | `cert-manager`       | Installs the operator, issues a certificate for the default ingress domain and its wildcard via Cloudflare DNS-01, then configures the default router.  |
| LVM Storage                         | `lvm`                | Installs the operator and creates an `LVMCluster` on the configured disk(s); wipe them manually first.                                                  |
| Master node pod limit               | `podlimit`           | Sets `maxPods` to 500 and disables the per-core cap on master nodes; waits for the node limit to take effect.                                            |
| OpenShift Virtualization (KubeVirt) | `kubevirt`           | Installs NMState and Virtualization, then maps `br-ex` to `localnet1` for an untagged external VM network.                                               |
| Optional KubeVirt smoke test         | `kubevirt-smoketest` | Recreates a dedicated Fedora test VM, its untagged LAN attachment, and an LVM test disk; skipped during normal full runs.                               |

The default router certificate covers the console and routes under the default
ingress domain that do not specify their own certificate. It does **not**
replace the API server certificate, internal cluster certificates, or custom
route certificates. The router changes only after cert-manager reports the new
certificate Ready; first setting the default certificate rolls the router, while
later renewals hot-reload the secret. After a change, the role waits for the
router deployment to roll out and for all ClusterOperators to report Available,
not Progressing, and not Degraded. It does not restart the ingress operator
manually.

Applying `podlimit` changes kubelet configuration through the Machine Config
Operator. On a single-node cluster this can reboot the node and interrupt the
API; allow for downtime and have console access before running it. The role
waits for master nodes to report a pod capacity of 500 before the next role.

Each tagged section lives in `roles/<role>/tasks/main.yml`; the readiness check
runs first from `playbook.yml`.

For example:

```
# Fresh install without OpenShift Virtualization or its bridge mapping
ansible-playbook -i localhost, playbook.yml --skip-tags kubevirt

# Install NMState and Virtualization (including the bridge mapping)
ansible-playbook -i localhost, playbook.yml --tags kubevirt

# Repair an operator on an unhealthy cluster without the readiness check
ansible-playbook -i localhost, playbook.yml --tags kubevirt --skip-tags readiness
```

## Optional KubeVirt smoke test

The smoke test is the last role in `playbook.yml`, immediately after `kubevirt`,
but is skipped during normal full runs. After `kubevirt` and `lvm` are ready,
run it alone:

```sh
ansible-playbook -i localhost, playbook.yml --tags kubevirt-smoketest
```

**Each run deletes the entire `kubevirt-smoke` namespace first.** Use it only
for this test; do not put other workloads or data there. The playbook reads all
`core` SSH public keys from MachineConfig `99-master-ssh` and gives their
holders SSH access to the test VM. Optionally set `smoke_vm_password` in the
gitignored `secrets.yml` for **console-only** password login (requires `openssl`
on the Ansible host); SSH password authentication stays disabled. If there are
neither keys nor a password, it fails before deleting anything.

The test leaves a Fedora VM running with an untagged LAN attachment and a blank
LVM-backed disk. It needs access to `quay.io/containerdisks/fedora:latest` and
DHCP on the LAN. Check the VM's LAN IP (reported by the guest agent if present,
otherwise from your DHCP leases or VM console); the playbook prints it when
reported. Then ping or SSH as `fedora`
using a private key matching one in `99-master-ssh`. `Running` and a bound disk
do not by themselves prove that LAN connectivity works. The VM's boot disk is
temporary; its blank test disk is on `lvms-vg1`.

```sh
oc -n kubevirt-smoke get vmi fedora -o jsonpath='{.status.interfaces[?(@.name=="lan")].ipAddresses}'
# If the LAN address is not reported, use the VM console (requires virtctl):
virtctl console -n kubevirt-smoke fedora
# From your LAN, after finding the address:
ssh -i ~/.ssh/<matching-private-key> fedora@<lan-ip>
# When finished (or if a run was interrupted):
oc delete namespace kubevirt-smoke
```
