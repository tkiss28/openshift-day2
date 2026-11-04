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

### Wipe the storage disk (manual, destructive — not automated)

Before running the day-2 playbook, make sure the disk(s) listed in
`group_vars/all.yml` under `lvms_device_paths` are clean of any leftover
filesystem/LVM signatures from a previous install, e.g.:

```
oc debug node/<node> -- chroot /host wipefs -fa /dev/nvme0n1
```

This is deliberately a manual step — it's destructive and worth eyeballing the
device name each time rather than having a playbook wipe a disk unattended.

## Running the playbook

Prepare `secrets.yml` with an HTPasswd username and password, a Cloudflare API
token, and an email address you control for the Let's Encrypt ACME account:

```
cp -n secrets.yml.example secrets.yml
# if secrets.yml already exists, keep its credentials and add any missing entries
# edit secrets.yml with your HTPasswd username/password, token and email
```

Scope the Cloudflare token to the public DNS zone containing the cluster's
ingress domain, with **Zone → DNS → Edit** and **Zone → Zone → Read**
permissions. The email address is not secret, but is the ACME account contact;
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
| NMState                             | `nmstate`            | Installs the operator and starts the handler.                                                                                                           |
| LVM Storage                         | `storage`            | Installs the operator and creates an `LVMCluster` on the configured disk(s); wipe them manually first.                                                  |
| OpenShift Virtualization (KubeVirt) | `virtualization`     | Installs the operator and the default `HyperConverged` configuration.                                                                                   |
| External VM network mapping         | `virtualization`     | Maps `br-ex` to `localnet1` via NMState; requires the `nmstate` section on a fresh install.                                                             |

The default router certificate covers the console and routes under the default
ingress domain that do not specify their own certificate. It does **not**
replace the API server certificate, internal cluster certificates, or custom
route certificates. The router changes only after cert-manager reports the new
certificate Ready; first setting the default certificate rolls the router, while
later renewals hot-reload the secret. After a change, the role waits for the
router deployment to roll out and for all ClusterOperators to report Available,
not Progressing, and not Degraded. It does not restart the ingress operator
manually.

Each tagged section lives in `roles/<role>/tasks/main.yml`; the readiness check
runs first from `playbook.yml`.

For example:

```
# Fresh install without OpenShift Virtualization or its bridge mapping
ansible-playbook -i localhost, playbook.yml --skip-tags virtualization

# Install NMState and Virtualization (including the bridge mapping)
ansible-playbook -i localhost, playbook.yml --tags nmstate,virtualization

# Repair an operator on an unhealthy cluster without the readiness check
ansible-playbook -i localhost, playbook.yml --tags nmstate --skip-tags readiness
```
