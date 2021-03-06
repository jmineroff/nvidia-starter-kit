## Domino NVIDIA Starter Kit

Domino NVIDIA Starter Kit provides a fast and simple installation of Kubernetes, Domino, and all other dependencies on a single node for
use in demos of Domino software on [DGX hardware](https://www.nvidia.com/en-us/data-center/dgx-systems/).

For this project, we're using the [NVIDIA DeepOps](https://github.com/NVIDIA/deepops) tooling to install hardware
drivers, the NVIDIA container runtime, and Kubernetes 1.18.

Domino 4.3.3+ can then be installed from official release channels via the fleetcommand-agent.

### How to use Domino NVIDIA Starter Kit

- Quickly stand up Domino as an ML development and deployment layer on a single DGX machine
- Demo Domino on a variety of DGX hardware
- Demo Multi-Instance GPU capabilities on a machine with A100 Tensor Core modules, such as a p4.24xlarge EC2 instance

### How NOT to use Domino NVIDIA Starter Kit

- Since Domino NVIDIA Starter Kit deployments have no redundancy or backups, they should not be treated as durable production data science platforms
- Domino NVIDIA Starter Kit deployments can not currently be used to build multi-node clusters

**Contents**

* [Live development installation](#live-development-installation)
* [Installation prerequisites](#installation-prerequisites)
* [Kubernetes and dependency installation](#kubernetes-and-dependency-installation)
* [Domino installation](#domino-installation)


<br />
<br />


### Live development installation

As of 11-24-2020 the single-node development installation that was done manually on a p3.8xlarge EC2 instance is
available for testing and inspection at https://Domino NVIDIA Starter Kit.support-team-sandbox.domino.tech (requires
[Aviatrix VPN](https://dominodatalab.atlassian.net/wiki/spaces/ENG/pages/456196343/Aviatrix+User+Guide) and uses a
self-signed certificate). **This instance is STOPPED as of 11-30-2020.**

| Instance Details      |                                                    |
| --------------------- | -------------------------------------------------- |
| Type                  | `p3.8xlarge`                                       |
| ID                    | `i-0dae28f3b2a4c0934`                              |
| AMI                   | `ami-00ddb0e5626798373`                            |
| Hostname              | `ec2-18-207-237-15.compute-1.amazonaws.com`        |
| User                  | `ubuntu`                                           |
| Key                   | `/key/Domino NVIDIA Starter Kit.pem`                                  |

To connect (requires
[Aviatrix VPN](https://dominodatalab.atlassian.net/wiki/spaces/ENG/pages/456196343/Aviatrix+User+Guide)), run:

```
ssh -i ./key/Domino NVIDIA Starter Kit.pem ubuntu@ec2-18-207-237-15.compute-1.amazonaws.com
```

You can see Domino and supporting services running with `kubectl get pods --all-namespaces`.


<br />
<br />


### Installation prerequisites

- **Infrastructure**

  This process is only suitable for installing Domino on a **single NVIDIA GPU node**. The node should be running a
  DeepOps-compatible OS (Ubuntu 18.04 recommended) with Git installed, have unencumbered access to Domino release
  channels over the Internet, and have at least one
  [NVIDIA A100 Tensor Core GPU](https://www.nvidia.com/en-us/data-center/a100/) module installed.

  Record the IP address or hostname of this machine. You'll need it when configuring DNS for the Domino application.

- **DNS**

  Domino is a web application with multiple interlinked UI components, and must accordingly have a fully qualified domain
  name with which to self-reference. The installation process will generate a self-signed certificate for this name and
  configure Domino to serve it. Ideally, this would be a name for which you can control DNS for your organization,
  however you can use any name and override DNS in your `/etc/hosts` file if preferred.

  Record the FQDN you want to use to access Domino. You'll need it when running the Domino software installer.

- **Credentials**

  In order to download Domino component images from quay.io release channels, you will need a username and password
  provided by Domino. If you do not know your username and password, contact Domino at support@dominodatalab.com.

  Record these credentials. You'll need them when running the Domino software installer.




<br />
<br />


### Kubernetes and dependency installation

These steps install Kubernetes 1.18 plus NVIDIA GPU drivers and utilities on the node.

1. Connect to the node via SSH.

2. Install `pip`:

   ```
   sudo apt update
   sudo apt install python-pip
   ```

3. Clone the `NVIDIA/deepops` repository:

   ```
   git clone https://github.com/NVIDIA/deepops.git
   ```

4. Change directory into the cloned repository:

   ```
   cd deepops
   ```

5. Set up Ansible and dependencies by running the DeepOps setup script:

   ```
   bash ./scripts/setup.sh
   ```

6. Edit the Ansible configuration file `ansible.cfg` to set up running the playbook against the local host. This
   requires adding `transport = local` to the `[defaults]`. The file should look like this:

   ```
   [defaults]
   roles_path = ./roles/galaxy:./roles:./submodules/kubespray/roles
   library = ./submodules/kubespray/library
   inventory = ./config/inventory
   host_key_checking = False
   gathering = smart
   fact_caching = jsonfile
   fact_caching_connection = /var/tmp/ansible_cache
   fact_caching_timeout = 86400
   deprecation_warnings = False
   #vault_password_file = ./config/.vault-pass
   timeout=60
   stdout_callback = yaml
   bin_ansible_callbacks = True
   local_tmp=/tmp
   remote_tmp=/tmp
   forks = 25
   transport = local

   [ssh_connection]
   pipelining = True
   ssh_args = -o ControlMaster=auto -o ControlPersist=5m -o ConnectionAttempts=100 -o UserKnownHostsFile=/dev/null
   control_path = ~/.ssh/ansible-%%r@%%h:%%p
   ```

7. Replace the default DeepOps playbook inventory file `config/inventory` with the following local-only inventory:

   ```
   ######
   # ALL NODES
   # NOTE: Use existing hostnames here, DeepOps will configure server hostnames to match these values
   ######
   [all]
   local     ansible_host=127.0.0.1

   ######
   # KUBERNETES
   ######
   [kube-master]
   local

   # Odd number of nodes required
   [etcd]
   local

   # Also add mgmt/master nodes here if they will run non-control plane jobs
   [kube-node]
   local

   [k8s-cluster:children]
   kube-master
   kube-node
   ```

8. Update the Multi-instance GPU (MIG) strategy in `config/group_vars/k8s-cluster.yml` to single mode:

   ```
   k8s_gpu_mig_strategy = "single"
   ```

9. Validate your configuration by running the following command from the `deepops` directory:

   ```
   ansible all -m raw -a "hostname"
   ```

   You should see the following:

   ```
   ansible all -m raw -a "hostname"
   [WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details

   PLAY [Ansible Ad-Hoc] ***********************************************************************************************

   TASK [raw] **********************************************************************************************************
   changed: [local]

   PLAY RECAP **********************************************************************************************************
   local                    : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
   ```

   If this step fails, double-check your `ansible.cfg` to ensure `transport = local`.

10. Run the Kubernetes playbook with the following command from the `deepops` directory:

    ```
    ansible-playbook -l k8s-cluster playbooks/k8s-cluster.yml
    ```

    This will progress through driver installation and then halt, since a reboot is required to initalize the
    drivers, which would interrupt the playbook. You should see the playbook stop with the following message:

    ```
    TASK [nvidia.nvidia_driver : reboot after driver install] *****************************************
    fatal: [local]: FAILED! => changed=false
      elapsed: 0
      msg: Running reboot with local connection would reboot the control node.
      rebooted: false
    ```

    You should then manually reboot the machine by running:

    ```
    sudo reboot
    ```

11. After the machine has rebooted, reconnect to it via SSH and then resume the playbook by running the following
    command from the `deepops` directory again:

    ```
    ansible-playbook -l k8s-cluster playbooks/k8s-cluster.yml
    ```

    After the playbook completes, you should wait about 5 minutes for the NVIDIA device plugin to initialize before
    proceeding.

11. Next, run the playbook to enable [Multi-Instance GPUs](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html):

    ```
    ansible-playbook playbooks/nvidia-software/nvidia-mig.yml
    ```

    The node must be rebooted for this change to take effect:

    ```
    sudo reboot
    ```

12. After the reboot completes, reconnect to the node via SSH. The next step is to initialize virtual MIG GPU resources
    on your A100 devices. This requires invoking `nvidia-smi mig -cgi` and specifying the virtual device geometry you
    want to create. Read the [MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html) to
    learn more.

    ```
    sudo nvidia-smi mig -cgi 19,19,19,19,19,19,19 -C
    ```

    The example above splits 7 A100 GPUs into 8 virtual GPUs each, for a total of 56 virtual devices. If this completes
    successfully, you should see MIG devices listed when running `nvidia-smi`.

13. Next, run the device plugin and feature discovery playbooks to restart the Kubernetes GPU controllers and detect
    the new virtual devices:

    ```
    ansible-playbook playbooks/k8s-cluster/nvidia-k8s-gpu-device-plugin.yml playbooks/k8s-cluster/nvidia-k8s-gpu-feature-discovery.yml
    ```

14. After all the deployed components initialize, your node should be an operational Kubernetes controller and worker.
    Run the following command to confirm:

    ```
    kubectl describe node local
    ```

    You should see allocatable resources like the following:

    ```
    Allocatable:
      cpu:                95800m
      ephemeral-storage:  479584865101
      hugepages-1Gi:      0
      hugepages-2Mi:      0
      memory:             1175724668Ki
      nvidia.com/gpu:     56
      pods:               110
    ```

15. Run the included DeepOps Ceph installation script:

    ```
    bash ./scripts/k8s/deploy_rook.sh
    ```

    You can then poll for Ceph to become available with the `-w` flag to the same script. This will return once Ceph
    has initialized:

    ```
    bash ./scripts/k8s/deploy_rook.sh -w
    ```

    Once this completes, you are ready to install Domino.


<br />
<br />


### Domino installation

These steps install pre-release Domino 4.4.0 onto your single-node cluster.

1. Clone the `dominodatalab/nvidia-starter-kit` repository:

   ```
   git clone https://github.com/dominodatalab/nvidia-starter-kit.git
   ```

2. Change directory into the cloned repository:

   ```
   cd nvidia-starter-kit
   ```

3. Run the `domino-nvidia-starter-kit.sh` script, passing in the FQDN, username, and password you recorded when setting
   up the prerequisites:

   ```
   bash domino-nvidia-starter-kit.sh example.domino.tech example-username example-password
   ```

   Once the installation completes, you can access Domino by creating DNS records (or /etc/hosts entries) that
   resolve the Domino FQDN to the IP address of your node. To get started, browse to a path of `/auth/`
   (the trailing `/` is important) on your Domino FQDN to open the authentication service. Then follow
   [these instructions in the Domino administrator's guide](https://admin.dominodatalab.com/en/4.3.2/keycloak.html) to
   retrieve credentials and log in to the auth service, from which you can create users or enable independent signups.
