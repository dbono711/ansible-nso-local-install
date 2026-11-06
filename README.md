# Ansible Role: NSO Local Installation

Ansible role for Cisco [Network Services Orchestrator](https://nso-docs.cisco.com/) (NSO) [Local](https://nso-docs.cisco.com/guides/nso-6.4/administration/installation-and-deployment/local-install) installation

## Overview

* Executes NSO 'Local' installation
* Compiles external YANG files
* Creates NSO runtime environment (idempotent; an existing runtime is never rebuilt)
  * Customized runtime configuration
  * Automatically unpack, build and load NED's and other NSO packages
    * Packages without a `src/Makefile` (precompiled) are loaded as-is, not built
  * Create and load NSO NETSIM's (IOS, IOS-XR, NX-OS)

### Restrictions

* The use of this role assumes the user can obtain the appropriate binaries as noted in [Prerequisites](#prerequisites)
  * Evaluation copies for NSO & various NSO NED's can be found on [DevNet](https://developer.cisco.com/docs/nso/#!getting-and-installing-nso/download-your-nso-free-trial-installer-and-cisco-neds)

### Prerequisites

* Ansible
  * ansible-core >= 2.19
    * The role uses `ansible_facts['...']` (no top-level `ansible_*` fact variables) and strictly boolean conditionals, and is ready for `INJECT_FACTS_AS_VARS=False`
* NSO
  * Python >= 3.9.6
    * Optionally a Python virtual environment for the NSO Python VM and package builds (see `nso_venv_python`)
  * Operating System requirements as described on [DevNet](https://developer.cisco.com/docs/nso/#!getting-and-installing-nso/requirements) (Java + Ant)
    * **NOTE:** If ```Java``` or ```Ant``` binaries are installed in a location other than the default per the Operating System requirements on DevNet, you can change the ```java_binary``` or ```ant_binary``` variables in the respective [vars](vars) file
    * **NOTE:** Assumes the target host has ```build-essentials``` (gcc, make, etc.) and ```xsltproc``` installed
  * NSO Signed Binary (*.signed.bin)
    * A **single** NSO signed binary **must** be placed within the (```files```) folder
  * NED / NSO Packages (*.tar.gz)
    * Any NED or package tarballs **must** be placed within the (```files```) folder
    * These are the `.tar.gz` packages contained **inside** the NED signed binaries; extract them first (signed `.bin` NED files are no longer used)
    * Files are matched by `ned_file_pattern` (default `'^ncs-.*\.tar\.gz$'`)
    * Packages are copied from the controller, so the same files work for local and remote targets

## Role Variables

All variables which can be overridden are found within [defaults/main](defaults/main)

### Setup ([setup.yml](defaults/main/setup.yml))

These variables are directly related to running the **NSO Local installation** and **NSO Runtime** environment

| Name | Default | Description |
| ---- | ------- | ----------- |
| `nso_root_dir` | "~/nso" | Root path to directory where the NSO installation, runtime, external YANG files, and NETSIM will reside.<br /><br /> In other words, ```nso_install_dir```, ```nso_runtime_dir```, ```nso_yang_dir```, and ```nso_netsim_dir``` are all derived from this variable within [vars/main.yml](vars/main.yml). |
| `nso_yang_files` | [] | External YANG files (i.e.; those not included as part of NSO installation) to be installed. Examples provided in the file. |
| `nso_venv_python` | | Full path to a Python interpreter inside a virtual environment (e.g. `~/venvs/nso/bin/python3`). When set, the NSO Python VM startup script (macOS) uses it, and package builds run with that virtual environment on `PATH`. |
| `ned_file_pattern` | '^ncs-.*\\.tar\\.gz$' | Regex used to find NED / package tarballs in `files`. Use **single quotes** in YAML so the backslashes are not treated as escape sequences. |

### NETSIM ([netsim-vars.yml](defaults/main/netsim-vars.yml))

These variables are directly related to the creation of NETSIM devices

| Name | Default | Description |
| ---- | ------- | ----------- |
| `nso_netsim` | [] | List of NETSIM devices to be created. Each entry takes `type` (`ios`, `iosxr` or `nxos`), `count`, and optionally `count_gige_intf` and `count_tengige_intf` (TenGigabitEthernet is not created for `nxos`). |

Each NETSIM `type` is matched to a NED package in the runtime by the prefix map `nso_netsim_ned_prefix` in [vars/main.yml](vars/main.yml):

| Type | NED package prefix |
| ---- | ------------------ |
| `ios` | `cisco-ios-cli-` |
| `iosxr` | `cisco-iosxr-cli-` |
| `nxos` | `cisco-nx-cli-` |

The role fails early if a requested `type` has no matching NED package. Non-NED packages (e.g. `resource-manager`) are ignored for NETSIM purposes.

### Runtime Configuration ([nso-runtime-config.yml](defaults/main/nso-runtime-config.yml))

These variables are directly related to the parameters used to construct the NSO runtime configuration, `ncs.conf`, abstracted from the various 'Configuration Parameters' found in the NSO Manual Pages.

| Name | Default | Description |
| ---- | ------- | ----------- |
| `nso_config_hide_group` | [] | Corresponds to the `/ncs-config/hide-group` section. |
| `nso_config_rollback` | {} | Corresponds to the `/ncs-config/rollback` section. |
| `nso_config_cli` | {} | Corresponds to the `/ncs-config/cli` section. |
| `nso_config_webui_tcp` | {} | Corresponds to the `/ncs-config/webui/transport/tcp` section. |
| `nso_config_webui_ssl` | {} | Corresponds to the `/ncs-config/webui/transport/ssl` section. |
| `nso_config_restconf` | {} | Corresponds to the `/ncs-config/restconf` section. |

## Tags

Task files are statically imported, so tags work directly from the command line.

| Tag | Runs |
| --- | ---- |
| `always` | OS variables, pre-flight checks, handler flush, cleanup (always run) |
| `install` | NSO installation and external YANG files |
| `runtime` | Installation and runtime creation / configuration |
| `nso_neds` | Unpack and build NED / NSO packages |
| `nso_netsim` | Create and load NETSIM devices |

## Execution Scenarios

The examples below assume a playbook (`nso-local.yml`) that imports this role tagged `setup`, and an `nso-config` role tagged `config`. `-K` (become password) and the vault options are only needed if your playbook uses `become` or vaulted variables (`secrets.yml`).

**Full install (everything: install, runtime, NED's, NETSIM's, then NSO configuration)**

```bash
ansible-playbook nso-local.yml -K --vault-id default@prompt -e @secrets.yml
```

**Full install of this role only (skip the `nso-config` role)**

```bash
ansible-playbook nso-local.yml --tags=setup -K --vault-id default@prompt -e @secrets.yml
```

**Re-create only the NETSIM's (after a full install)**

Deletes the existing NETSIM network and devices in NSO, then rebuilds and reloads them from `nso_netsim`.

```bash
ansible-playbook nso-local.yml --tags=nso_netsim -K --vault-id default@prompt -e @secrets.yml
```

**Add or update NED's (after dropping new tarballs into `files`)**

Unpacks and rebuilds packages, then reloads them in NSO. Add `nso_netsim` if the NETSIM's should be rebuilt against the new NED's.

```bash
ansible-playbook nso-local.yml --tags=nso_neds -K --vault-id default@prompt -e @secrets.yml
ansible-playbook nso-local.yml --tags=nso_neds,nso_netsim -K --vault-id default@prompt -e @secrets.yml
```

**Apply runtime configuration changes (`ncs.conf` variables)**

`runtime` also runs the install tasks, which are idempotent on an existing installation.

```bash
ansible-playbook nso-local.yml --tags=runtime -K --vault-id default@prompt -e @secrets.yml
```

**Apply only the NSO configuration role**

```bash
ansible-playbook nso-local.yml --tags=config -K --vault-id default@prompt -e @secrets.yml
```

**Preview what a tag combination will run**

Because task files are statically imported, `--list-tasks` shows every task that would execute, without running anything.

```bash
ansible-playbook nso-local.yml --tags=nso_netsim --list-tasks
```

Tasks tagged `always` (OS variables, pre-flight, handler flush, cleanup) run in every scenario.

## Sample Playbook

```yaml
---
- name: NSO Local Installation
  hosts: all
  gather_facts: true
  vars:
    nso_root_dir: "~/nso"
    nso_venv_python: "~/venvs/nso/bin/python3"
    nso_yang_files:
      - name: "ietf-routing-types.yang"
        uri: "https://raw.githubusercontent.com/YangModels/yang/master/standard/ietf/RFC/ietf-routing-types%402017-12-04.yang"
    nso_netsim:
      - type: ios
        count: 3
        count_gige_intf: 2
        count_tengige_intf: 2
      - type: iosxr
        count: 3
        count_gige_intf: 2
        count_tengige_intf: 2
    nso_config_hide_group:
      - name: debug
    nso_config_rollback:
      enabled: true
      directory: ./logs
      history_size: 500
    nso_config_cli:
      rollback_numbering: rolling
    nso_config_webui_ssl:
      enabled: true
      ip: 0.0.0.0
      port: 8888
      key_file: ${NCS_DIR}/var/ncs/webui/cert/host.key
      cert_file: ${NCS_DIR}/var/ncs/webui/cert/host.cert
  tasks:
    - name: Setup NSO
      ansible.builtin.import_role:
        name: dbono711.ansible_nso_local_install
      tags:
        - "setup"
```

**NOTE:** Use `import_role` (or `roles:`) rather than `include_role`. With `include_role`, the role's internal tags are hidden from `--tags` unless every tag is repeated on the include.

## Troubleshooting

* **Tasks silently missing from output:** if `ansible.cfg` sets `display_skipped_hosts = False`, skipped tasks print nothing. Check the `skipped=` count in the recap, or run with `ANSIBLE_DISPLAY_SKIPPED_HOSTS=true`.
* **`Cannot access directory ./state` on reload:** NSO was running when its runtime directory was deleted and recreated, so the daemon is still working from the deleted directory. Stop NSO (`ncs --stop`) and run the role again.
* **`ncs --status` failing during a run:** expected when NSO is stopped; the role records the status and does not treat it as a failure.

## Local Testing

Testing is accomplished with Molecule. After [installing](https://molecule.readthedocs.io/en/latest/installation/) Molecule in your local environment, you can execute the complete test suite via:

```molecule test```

## Additional Resources

[Learning NSO](https://developer.cisco.com/docs/nso/#!learning-nso)

[Documentation Guides](https://developer.cisco.com/docs/nso/guides/)

## Authors

* Darren Bono - [darren.bono@att.net](mailto://darren.bono@att.net)

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE.md) for details
