# openshift-day2

Day-2 configuration for OpenShift clusters, to install common operators and get
the cluster from an freshly installed state to a consumable state. 

## Prerequisites

- `oc` CLI installed and logged in to the target cluster
  (`oc login -u kubeadmin ...`) before running the playbook.
- Ansible and its Python dependencies in a virtual environment (from this
  directory):
  ```
  python3 -m venv .venv
  . .venv/bin/activate
  python -m pip install ansible-core kubernetes jsonpatch passlib 'bcrypt<4.1'
  ansible-galaxy collection install -r requirements.yml
  ```
  Run the playbook with the virtual environment active. The local tasks use the
  same Python interpreter as `ansible-playbook`, so installing `kubernetes`
  only in a different Python environment will not work. `bcrypt<4.1` keeps
  passlib's bcrypt hashing compatible with its backend.

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

Prepare `secrets.yml` with real passwords:

```
cp secrets.yml.example secrets.yml
# edit secrets.yml with a real htpasswd password
```

Review the settings in `group_vars/all.yml` if the devices listed there match
the actual hardware.

```
vi `group_vars/all.yml`
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
| NMState                             | `nmstate`            | Installs the operator and starts the handler.                                                                                                           |
| LVM Storage                         | `storage`            | Installs the operator and creates an `LVMCluster` on the configured disk(s); wipe them manually first.                                                  |
| OpenShift Virtualization (KubeVirt) | `virtualization`     | Installs the operator and the default `HyperConverged` configuration.                                                                                   |
| External VM network mapping         | `virtualization`     | Maps `br-ex` to `localnet1` via NMState; requires the `nmstate` section on a fresh install.                                                             |
| cert-manager                        | `cert-manager`       | Installs the operator; does not configure an issuer or certificates.                                                                                    |

Each tagged section lives in `roles/<role>/tasks/main.yml`; the readiness
check runs first from `playbook.yml`.

For example:

```
# Fresh install without OpenShift Virtualization or its bridge mapping
ansible-playbook -i localhost, playbook.yml --skip-tags virtualization

# Install NMState and Virtualization (including the bridge mapping)
ansible-playbook -i localhost, playbook.yml --tags nmstate,virtualization

# Repair an operator on an unhealthy cluster without the readiness check
ansible-playbook -i localhost, playbook.yml --tags nmstate --skip-tags readiness
```
