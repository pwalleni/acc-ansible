# Agent Deployment via Ansible

[Ansible](https://www.ansible.com/) is a simple but powerful DevOps tool that can be leveraged to deploy and configure the Agent Client Collector. This repository serves to host sample Ansible playbooks that can be used to install, configure, and uninstall our agent on its various supported platforms.

## Setup

### Prerequisites

- Agent 2.4.0+
- Ansible 2.11+
- Pywinrm 0.4.1+ (for Windows deployments)

### Configuration Parameters

Open up the [group_vars/all](group_vars/all) file and change the parameters according to the instance and MID Server your agents will be connecting to.

```vim group_vars/all```

```yaml
# Required configuration parameters for agent installation

version: 2.4.0
webserverIP: 10.11.145.209
webserverPort: 9900
instanceToConnect: https://myinstance.service-now.com
apiKey: b3c0382f533220103f5fddeeff7b12dd
```

- `version` - The version of ACC being installed
- `webServerIP` - The IP of the MID webserver the agent will connect to
- `webServerPort` - The port of the MID webserver the agent will connect to
- `instanceToConnect` - The URL of the ServiceNow instance that the agents will be connected to, used for one-line installer version
- `apiKey` - The MID Web Server API Key which can be found on the instance

```yaml
# Optional configuration parameters

linuxDownloadFolder: /tmp/agentInstallerPackage
signatureValidation: 1
enableAllowList: 1
```

- `linuxDownloadFolder` - The folder where linux agent installations will download to
- `signatureValidation` - Whether to enable signature validation 0 = disabled, 1 = enabled
- `enableAllowList` - Whether to enable the allow list 0 = disabled, 1 = enabled

### Host Configuration

The [hosts](hosts) file (default location: /etc/ansible/hosts) must be setup so that linux and windows hosts are grouped by OS accordingly in order for the following example playbooks to work out of the box. Additionally, the [manual linux installation playbook](#install-agent-on-linux-host) requires os-specific variables to be defined in order for the correct package to be downloaded and installed.

``` vim hosts ```

```yaml

[Debian_8]
10.11.0.1

[Debian_9]
10.11.0.2

[Debian_8:vars]
release=debian-8

[Debian_9:vars]
release=debian-9

[Debian:children]
Debian_8
Debian_9

[Debian:vars]
arch=amd64
extension=deb

[Ubuntu_16]
10.11.0.3

[Ubuntu_16:vars]
release=ubuntu-1604

[Ubuntu_18]
10.11.0.4

[Ubuntu_18:vars]
release=ubuntu-1804

[Ubuntu:children]
Ubuntu_16
Ubuntu_18

[Ubuntu:vars]
arch=amd64
extension=deb

[linux:children]
Ubuntu
Debian

[Win_2016]
10.12.0.1

[Win_2019]
10.12.0.2

[windows:children]
Win_2016
Win_2019
```

- `arch` - The archiecture of the OS, `amd64` for ubuntu/debian, `x86_64` for centos/SUSE/redhat
- `extension` - The file extension for the package download, `deb` for ubuntu/debian, `rpm` for centos/SUSE/redhat
- `release` - The specific release for the OS, `debian-8` for debian 8, `debian-9` for debian 9, `ubuntu-1404` for Ubuntu 14, `ubuntu-1604` for Ubuntu 16, `ubuntu-1804` for Ubuntu 18, `el7` for centos/SUSE/redhat

If using the single-line linux installer playbook, the preceeding host variable configuration is handled automatically and is not needed.

## Playbook Usage

### Linux Agents

#### Install agent on Linux host

There are two versions of the linux installation playbook, the [manual installation](linux_manual_install.yml) allows more control over how the installation is performed and can give a better idea on what is done during installation:

`ansible-playbook linux_manual_install.yml`

The [single-line installation](linux_single_line_install.yml) involves less Ansible tasks and uses an up-to-date installation script that is downloaded from the configured ServiceNow instance (using `instanceToConnect`):

`ansible-playbook linux_single_line_install.yml`

#### Upgrade agent on Linux host

`ansible-playbook linux_upgrade.yml`

#### Uninstall agent on Linux host

Similar to installation, there are two versions of the linux uninstallation playbook, the [manual uninstallation](linux_manual_uninstall.yml):

`ansible-playbook linux_manual_uninstall.yml`

And the [single-line uninstallation](linux_single_line_uninstall.yml)

`ansible-playbook linux_single_line_uninstall.yml`

### Windows Agents

#### Install agent on Windows host

`ansible-playbook windows_install.yml`

#### Upgrade agent on Windows host

Upgrading the agent on a windows host requires making sure the [group_vars/all](group_vars/all) file `version` is set to the latest release and then running the [windows installation playbook](#install-agent-on-windows-host) again. The msi installer will handle upgrading the agent if it was previously installed.

#### Uninstall agent on Windows host

`ansible-playbook windows_uninstall.yml`

## Advanced 

The following snippets are Ansible tasks that can be used to change the state of the agent and/or its configuration parameters post-deployment. The main configuration changes take place inside the acc.yml file and require the agent to be restarted in order to take effect. These examples serve as a template to follow for making such configuration changes.

### Windows

#### Stopping agent on Windows host

```yaml
    - name: Stop Agent Client Collector service
      ansible.windows.win_service:
        name: AgentClientCollector
        state: stopped
```

#### Editing acc.yml on Windows host

```yaml
    - name: Edit acc.yml to enable allow list
      win_lineinfile:
        path: 'C:\ProgramData\ServiceNow\agent-client-collector\config\acc.yml'
        regex: 'allow-list:'
        line: 'allow-list: C:\ProgramData\ServiceNow\agent-client-collector\config\check-allow-list.json'
      when: enableAllowList | int == 1
 ```

#### Starting agent on Windows host

```yaml
    - name: Start Agent Client Collector service
      ansible.windows.win_service:
        name: AgentClientCollector
        state: started
```

#### Ex: Configure agent allow list for Windows

For a windows agent that is already installed:

- Disable allow list:	`ansible-playbook windows_allow_list.yml --extra-vars "enableAllowList=0"`
- Enable allow list: 	`ansible-playbook windows_allow_list.yml --extra-vars "enableAllowList=1"`

### Linux

#### Stopping agent on Linux host

```yaml
    - name: Disable - Agent Client Collector service
      ansible.builtin.shell : systemctl disable acc
      ignore_errors: true

    - name: Stop - Agent Client Collector service
      ansible.builtin.shell : systemctl stop acc
      ignore_errors: true
```

#### Editing acc.yml on Linux host

```yaml
    - name: Enable Allow-List
      ansible.builtin.replace:
        path: '/etc/servicenow/agent-client-collector/acc.yml'
        regexp: '#allow-list: '
        replace: 'allow-list: '
      when: enableAllowList | int == 1
 ```

#### Starting agent on Linux host

```yaml
    - name: Enable - Agent Client Collector service
      ansible.builtin.shell: systemctl enable acc

    - name: Start - Agent Client Collector service
      ansible.builtin.shell: systemctl start acc
```

#### Ex: Enable/disable allow list for Linux

For a linux agent that is already installed:

- Disable allow list:	`ansible-playbook linux_allow_list.yml --extra-vars "enableAllowList=0"`
- Enable allow list: 	`ansible-playbook linux_allow_list.yml --extra-vars "enableAllowList=1"`

