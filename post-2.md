---
ID: 3
Date: 2025-05-30T12:00:31Z
Title: "Post 2"
Author: "Jeremy Novak"
Summary: "Part one of my home lab bare metal Kubernetes cluster with Calico"
Slug: post-2
Tags: 
 - homelab
 - kubernetes
 - ansible
Published: true
---

I had a goal to create an inexpensive Kubernetes cluster on bare metal in my home lab for 2025.
The budget I have for this home lab means sticking with the most machine I can get for under $200
per node. That's how I landed on the decision to purchase the [Beelink Mini S Pro Mini PC, N100 Processor, 16G DDR4 RAM, 500G SSD](https://www.amazon.com/dp/B0BWLRXY84?ref=ppx_yo2ov_dt_b_fed_asin_title&th=1).

I bought one to be the control-plane, and two to be worker nodes. Although not necessary to get a 
kubernetes cluster going, I bought a fourth that I have some future plans for to run some additional services *on-prem*.

I chose Ubuntu Server 24.04 for the server OS on the nodes. Ansible will be used for some basic configuration but for
the install of Kubernetes and Calico, I chose to do it all by hand using `kubeadm`.

## Ansible

### Creating the ansible user on the homelab nodes

I'm going to create a user named `ansible` on each of the nodes that Ansible uses when we run tasks. 

Before I do that though, I need to create an SSH key for my own account and copy it to the `authorized_keys` file for 
my account on each of the nodes.

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/jgn_rsa -N ""
```

The public key can be added for my account on each node with a simple loop like this.

```bash
for node in 192.168.1.{100..103}; do
  ssh-copy-id -i ~/.ssh/jgn_rsa.pub jgn@$node
done
```

I created a playbook to run against all nodes in the cluster to create the `ansible` user and give it `sudo` privileges
that allow it run all commands without a password.

```yml
---
- name: Set up ansible user and sudo privileges
  hosts: homelab
  become: yes
  vars:
    ansible_password: "AnsibleUserPasswordHere"
  tasks:
    - name: Create ansible user with password and home directory
      ansible.builtin.user:
        name: ansible
        password: "{{ ansible_password | password_hash('sha512') }}"
        shell: /bin/bash
        create_home: yes
        state: present

    - name: Grant ansible user sudo privileges via sudoers.d
      ansible.builtin.copy:
        content: "ansible ALL=(ALL) NOPASSWD:ALL" # Grant ansible ability to run any command with no password
        dest: /etc/sudoers.d/ansible
        mode: "0440"
        owner: root
        group: root
        validate: "/usr/sbin/visudo -cf %s"
```

I will have to run this playbook as myself, but going forward running playbooks will use the `ansible` user.

```bash
ansible-playbook -i inventory users/ansible.yml --user jgn --ask-pass
```

Now that the `ansible` user is on all the nodes with `sudo` permissions that allow it to run commands we can proceed
with making Ansible a little nicer to use in the home lab.

### Completing the Ansible configuration 

I added the four nodes in my home lab to my `/etc/hosts` file (Linux/Unix, MacOS) so that instead of typing `192.168.1.100` the
hostname can be used instead. If you are on Windows this file is located at `c:\windows\system32\drivers\etc\hosts`.

```txt
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1       localhost
255.255.255.255 broadcasthost
::1             localhost

# Home lab hosts
192.168.1.100 docbar
192.168.1.101 steeldust
192.168.1.102 reride
192.168.1.103 roughstock
```

Next we need an *inventory* file so we that Ansible knows what we mean when we address the nodes by their group name (e.g. "homelab")
and it lets us specify a few other things for convenience. I just called mine `inventory` and it is found at the top level directory
where all my Ansible playbooks are located at `~/projects/homelab/ansible`

```ini
[homelab]
docbar ansible_host=192.168.1.100
steeldust ansible_host=192.168.1.101
reride ansible_host=192.168.1.102
roughstock ansible_host=192.168.1.103

[all:vars]
ansible_python_interpreter=/usr/bin/python3.12
ansible_ssh_private_key_file=~/.ssh/ansible_rsa

[admin]
docbar

[kubernetes]
steeldust
reride
roughstock

[contol-plane]
steeldust

[workers]
reride
roughstock
```
Now any playbook can target the nodes *homelab*, *admin*, *kubernetes*, *control-plane*, or *workers* and Ansible will know
what machines we are referring to.

I'm also going to create an `ansible.cfg` which lets me specify some defaults. This file will be located at the top level directory
with the `inventory` file at `~/projects/homelab/ansible`. This lets me do less typing since the remote_user, inventory file,
and private key are already specified.

```txt
[defaults]
inventory = ./inventory
remote_user = ansible
private_key_file = ~/.ssh/ansible_rsa
host_key_checking = false
```

Similar to what I did above, now I need to create an SSH key for the `ansible user`.

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible_rsa -N ""
```

And copy then I copy the public key to each node.

```bash
for node in 192.168.1.{100..103}; do
  ssh-copy-id -i ~/.ssh/ansible_rsa.pub ansible@$node
done
```

Let's test it with a *ping* and make sure every node in the homelab responds with no warnings or errors.

```bash
ansible -m ping all
roughstock | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
reride | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
steeldust | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
docbar | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Creating an update playbook

This playbook will run the equivalent of logging into each node in the homelab and doing `sudo apt update -y && sudo apt upgrade -y`.
I created this playbook at `~/projects/homelab/ansible/update/all.yml`

```yml
---
- name: Update and patch Ubuntu nodes
  hosts: homelab
  become: yes
  tasks:
    - name: Update package cache
      ansible.builtin.apt:
        update_cache: yes
      register: update_result

    - name: Upgrade all packages to the latest version
      ansible.builtin.apt:
        upgrade: dist
      when: update_result.changed # Only run if cache was updated

    - name: Check if a reboot is required
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required

    - name: Reboot the node if required
      ansible.builtin.reboot:
        reboot_timeout: 300
      when: reboot_required.stat.exists
```

Now updating all hosts in the homelab is done with this playbook. How convenient!

```bash
ansible-playbook update/all.yml
```

### Configure Kubernetes pre-requisites

I'm going to stand up the cluster by hand using `kubeadm` with  [calico](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart) for the CNI.

There are a few [pre-requisites](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) that we need to handle before running `kubeadm init`.

Since this needs to be done on all three nodes that will make up the cluster, it is another good candidate for an Ansible playbook.
In the playbook I install required packages, disable swap, enable kernel networking drivers, install containerd and get the correct
packages for Kubernetes version 1.32.

```yml
---
- name: Install Kubernetes pre-requisites
  hosts: kubernetes
  become: yes
  tasks:
    # Update package cache to ensure we can install prerequisites
    - name: Update package cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

    # Install basic tools required for adding the Kubernetes repo
    - name: Install required packages
      ansible.builtin.apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gpg
        state: present

    # Disable swap as Kubernetes requires it off
    - name: Disable swap
      ansible.builtin.command: swapoff -a
      changed_when: true

    # Remove swap from fstab to prevent re-enabling on reboot
    - name: Remove swap entry from /etc/fstab
      ansible.builtin.lineinfile:
        path: /etc/fstab
        regexp: '^.*\sswap\s'
        state: absent

    # Load kernel modules for container networking
    - name: Load required kernel modules
      ansible.builtin.modprobe:
        name: "{{ item }}"
        state: present
      loop:
        - overlay
        - br_netfilter

    # Persist kernel modules for boot
    - name: Persist kernel modules
      ansible.builtin.lineinfile:
        path: /etc/modules-load.d/k8s.conf
        line: "{{ item }}"
        create: yes
        mode: "0644"
      loop:
        - overlay
        - br_netfilter

    # Configure sysctl settings for Kubernetes networking
    - name: Set sysctl parameters for Kubernetes
      ansible.builtin.sysctl:
        name: "{{ item.name }}"
        value: "{{ item.value }}"
        state: present
        sysctl_file: /etc/sysctl.d/k8s.conf
        reload: yes
      loop:
        - { name: "net.bridge.bridge-nf-call-iptables", value: "1" }
        - { name: "net.bridge.bridge-nf-call-ip6tables", value: "1" }
        - { name: "net.ipv4.ip_forward", value: "1" }

    # Install containerd as the container runtime
    - name: Install containerd
      ansible.builtin.apt:
        name: containerd
        state: present

    # Configure containerd to use systemd cgroup driver
    - name: Ensure containerd config directory exists
      ansible.builtin.file:
        path: /etc/containerd
        state: directory
        mode: "0755"

    # Generate and configure containerd config.toml
    - name: Generate and configure containerd config.toml for systemd
      ansible.builtin.shell:
        cmd: "containerd config default > /etc/containerd/config.toml"
        creates: /etc/containerd/config.toml
      register: config_generated
      changed_when: config_generated.rc == 0

    # Set systemdCgroup = true in containerd config
    - name: Ensure containerd is using systemdCgroup
      ansible.builtin.lineinfile:
        path: /etc/containerd/config.toml
        regexp: '^(\s*)systemd_cgroup\s*=\s*false'
        line: '\1systemd_cgroup = true'
        backrefs: yes
      notify: Restart containerd

    # Restart containerd to apply changes
    - name: Restart containerd
      ansible.builtin.systemd:
        name: containerd
        state: restarted
        enabled: yes

    # Run the curl | gpg command from the Kubernetes docs
    - name: Add Kubernetes APT key with curl and gpg
      ansible.builtin.shell:
        cmd: "curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg"
        creates: /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      become: yes
      changed_when: true

    # Set permissions on the keyring file
    - name: Set keyring file permissions
      ansible.builtin.file:
        path: /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        mode: "0644"

    # Add the Kubernetes APT repository exactly as in the docs
    - name: Add Kubernetes APT repository
      ansible.builtin.shell:
        cmd: "echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list"
        creates: /etc/apt/sources.list.d/kubernetes.list
      changed_when: true

    # Update APT cache after adding the repo
    - name: Update APT cache after adding Kubernetes repo
      ansible.builtin.apt:
        update_cache: yes

    # Install Kubernetes 1.32 packages
    - name: Install Kubernetes packages
      ansible.builtin.apt:
        name:
          - kubelet=1.32.0-*
          - kubeadm=1.32.0-*
          - kubectl=1.32.0-*
        state: present

    # Hold Kubernetes packages to prevent unwanted upgrades
    - name: Hold Kubernetes packages at current version
      ansible.builtin.dpkg_selections:
        name: "{{ item }}"
        selection: hold
      loop:
        - kubelet
        - kubeadm
        - kubectl

    # Reboot to apply all changes
    - name: Reboot nodes to apply changes
      ansible.builtin.reboot:
        reboot_timeout: 300

```

All I need to do now to get these hosts ready to do a manual installation with `kubeadm` is just run the playbook.

```bash
ansible-playbook kubernetes/install_dependencies.yml
```

## Running kubeadm to initialize the cluster

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=192.168.1.101 --v=5
```

### Uh oh... it isn't working!?

```text
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
[preflight] Some fatal errors occurred:
failed to create new CRI runtime service: validate service connection: validate CRI v1 runtime API for endpoint "unix:///var/run/containerd/containerd.sock": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
error execution phase preflight
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).Run.func1
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:262
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).visitAll
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:450
k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow.(*Runner).Run
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/phases/workflow/runner.go:234
k8s.io/kubernetes/cmd/kubeadm/app/cmd.newCmdInit.func1
        k8s.io/kubernetes/cmd/kubeadm/app/cmd/init.go:129
github.com/spf13/cobra.(*Command).execute
        github.com/spf13/cobra@v1.8.1/command.go:985
github.com/spf13/cobra.(*Command).ExecuteC
        github.com/spf13/cobra@v1.8.1/command.go:1117
github.com/spf13/cobra.(*Command).Execute
        github.com/spf13/cobra@v1.8.1/command.go:1041
k8s.io/kubernetes/cmd/kubeadm/app.Run
        k8s.io/kubernetes/cmd/kubeadm/app/kubeadm.go:47
main.main
        k8s.io/kubernetes/cmd/kubeadm/kubeadm.go:25
runtime.main
        runtime/proc.go:272
runtime.goexit
        runtime/asm_amd64.s:1700
```

To make a long story short, the `containerd` package that comes with Ubuntu Server 24.04 doesn't seem to have CRI enabled
and after trying everything I could think of ended up ripping and replacing it with another package `containerd.io` provided by Docker.

### Installing containerd.io

First I need to remove the version provided by Ubuntu since we couldn't get it working.

```bash
sudo apt remove --purge containerd -y
```
Next I'll grab the gpg key from Docker and add it to the apt repository.

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

An install the `containerd.io` package.

```bash
sudo apt install -y containerd.io
```

There are a couple of changes to `config.toml` I need to make so that it will work correctly. I saved the default config
to disk as `/etc/containerd/config.toml` so I can make my edits.

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

The main thing we need to do is enable SystemdCgroup by changing `false` to `true`.

```bash
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

### Trying kubeadm init again

```text
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=192.168.1.101 --v=5
```

SUCCESS!

### Adding Calico CNI

Installing Calico on the cluster is pretty straight forward, we run two `kubectl create -f` commands with supplied yaml files.

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml
```

### Setting up kube config

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Hmmm... something isn't quite right yet

The cluster is doing *okay*, but the calico components and coredns are not happy.

It turns out that I used the wrong CIDR when I ran `kubeadm init` for Calico. I used `10.244.0.0` and
the CIDR that Calico expects is `192.168.0.0/16`.

To fix this I need to download the yaml file provided by Calico and edit it to use the CIDR that I specified when I created the cluster.

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml > calico.yaml
```

Now I'll just update the `cidr:` block with the correct CIDR for this cluster.

```yaml
# This section includes base Calico installation configuration.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

... and apply.

```bash
kubectl apply -f calico.yaml
```

Checking the nodes and pods.

```bash
jgn@steeldust:~$ kubectl get nodes
NAME         STATUS   ROLES           AGE     VERSION
reride       Ready    <none>          4h48m   v1.32.0
roughstock   Ready    <none>          4h15m   v1.32.0
steeldust    Ready    control-plane   5h7m    v1.32.0


jgn@steeldust:~$ kubectl get pods -A
NAMESPACE          NAME                                       READY   STATUS    RESTARTS   AGE
calico-apiserver   calico-apiserver-7dc9f69d88-95jrg          1/1     Running   0          4h40m
calico-apiserver   calico-apiserver-7dc9f69d88-hbl4k          1/1     Running   0          4h40m
calico-system      calico-kube-controllers-869ddb5bc5-rp4nd   1/1     Running   0          4h38m
calico-system      calico-node-gf88r                          1/1     Running   0          4h16m
calico-system      calico-node-j42kb                          1/1     Running   0          4h37m
calico-system      calico-node-kt9xs                          1/1     Running   0          4h37m
calico-system      calico-typha-6cf8b4fd74-8h52d              1/1     Running   0          4h38m
calico-system      calico-typha-6cf8b4fd74-ncpw5              1/1     Running   0          4h16m
calico-system      csi-node-driver-768ql                      2/2     Running   0          4h38m
calico-system      csi-node-driver-9t8wz                      2/2     Running   0          4h38m
calico-system      csi-node-driver-hh8x6                      2/2     Running   0          4h16m
kube-system        coredns-668d6bf9bc-7v8rr                   1/1     Running   0          5h8m
kube-system        coredns-668d6bf9bc-vz6q5                   1/1     Running   0          5h8m
kube-system        etcd-steeldust                             1/1     Running   0          5h8m
kube-system        kube-apiserver-steeldust                   1/1     Running   0          5h8m
kube-system        kube-controller-manager-steeldust          1/1     Running   0          5h8m
kube-system        kube-proxy-8556g                           1/1     Running   0          4h16m
kube-system        kube-proxy-j75jf                           1/1     Running   0          4h49m
kube-system        kube-proxy-wpwhw                           1/1     Running   0          5h8m
kube-system        kube-scheduler-steeldust                   1/1     Running   0          5h8m
tigera-operator    tigera-operator-ccfc44587-f4qs9            1/1     Running   0          4h41m
```

### Testing with nginx

I won't know for sure if the cluster is actually working until I run a pod, see it pull and image and fire up.

```bash
kubectl run nginx --image=nginx
pod/nginx created
```

And it works...

```bash
kubectl get pods -n default
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          24s
```

SUCCESS!

I now have a working bare metal Kubernetes cluster with Calico providing the CNI. The basics for administering the homelab
with Ansible are also in place and ready for any future playbooks that I need to write. 

> The cluster is hours old at the time of grabbing the output shown above, but it accurately reflects the
state of the cluster after `kubeadm init` and the Calico installation completed.

## Up next

Now that we have a Kubernetes cluster installed with Calico in our home lab and Ansible basics working we can start
using it to deploy some homelab projects. Check back periodically for the next installment in the blog series.
