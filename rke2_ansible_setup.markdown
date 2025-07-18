# Deploying RKE2, Helm, and Rancher with Ansible

This guide provides a step-by-step process to deploy a high-availability (HA) RKE2 Kubernetes cluster with 3 master nodes and 3 worker nodes, install Helm, and deploy Rancher using Ansible. The setup is production-ready and automates the configuration for scalability and reliability.

## Overview

- **Components**: RKE2 (Kubernetes distribution), Helm (package manager), Rancher (management UI), Ansible _(automation)_
- **Cluster Setup**: 3 master nodes (control-plane, etcd), 3 worker nodes
- **Goal**: Automate deployment for an HA Kubernetes cluster with Rancher management

## Prerequisites

- **Nodes**: 6 Linux servers (e.g., Ubuntu 22.04 or CentOS Stream 8) with static IPs:
  - Masters: `192.168.1.10–12` (2 vCPUs, 8GB RAM)
  - Workers: `192.168.1.13–15` (1 vCPU, 2GB RAM)
- **Load Balancer**: Layer 4 load balancer (e.g., HAProxy, MetalLB, or AWS ELB) for ports 6443 (Kubernetes API) and 9345 (RKE2 registration)
- **DNS**: Record (e.g., `rke2.example.com`) pointing to the load balancer
- **Ansible**: Installed on the control machine (`pip install ansible`)
- **Tools**: `kubectl` and `helm` on the control machine
- **SSH**: Key-based access with root/sudo privileges

```bash
# Install kubectl and Helm on control machine
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Ansible Inventory

Create an inventory file (`inventory.yml`) to define nodes and variables:

```yaml
all:
  hosts:
    master-1:
      ansible_host: 192.168.1.10
      ansible_user: ubuntu
      node_name: master-1
      role: server
    master-2:
      ansible_host: 192.168.1.11
      ansible_user: ubuntu
      node_name: master-2
      role: server
    master-3:
      ansible_host: 192.168.1.12
      ansible_user: ubuntu
      node_name: master-3
      role: server
    worker-1:
      ansible_host: 192.168.1.13
      ansible_user: ubuntu
      node_name: worker-1
      role: agent
    worker-2:
      ansible_host: 192.168.1.14
      ansible_user: ubuntu
      node_name: worker-2
      role: agent
    worker-3:
      ansible_host: 192.168.1.15
      ansible_user: ubuntu
      node_name: worker-3
      role: agent
  vars:
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
    rke2_version: v1.24.10+rke2r1
    primary_master: master-1
    load_balancer_url: rke2.example.com
    cluster_token: my-shared-secret
    cni_plugin: canal
```

**Key Variables**:
- `rke2_version`: RKE2 version (e.g., `v1.24.10+rke2r1`)
- `primary_master`: First master node to bootstrap the cluster
- `load_balancer_url`: DNS name for the load balancer
- `cluster_token`: Shared secret for node registration
- `cni_plugin`: Network plugin (e.g., `canal`, `calico`)

## Ansible Playbook

Create a playbook (`deploy_rke2_rancher.yml`) to automate the deployment:

```yaml
---
- name: Deploy RKE2, Helm, and Rancher
  hosts: all
  become: true
  vars:
    rke2_config_dir: /etc/rancher/rke2
    rke2_bin_dir: /usr/local/bin
    helm_version: v3.11.0
    cert_manager_version: v1.11.0
    rancher_hostname: rancher.example.com
    rancher_version: v2.7.1
  tasks:
    - name: Update package cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"
    - name: Install prerequisites
      ansible.builtin.apt:
        name:
          - curl
          - open-iscsi
          - nfs-common
        state: present
      when: ansible_os_family == "Debian"
    - name: Set hostname
      ansible.builtin.hostname:
        name: "{{ node_name }}"
        use: systemd
    - name: Install RKE2 binaries
      ansible.builtin.shell:
        cmd: curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION={{ rke2_version }} sh -
        creates: "{{ rke2_bin_dir }}/rke2"
      register: rke2_install
      changed_when: rke2_install.rc == 0
    - name: Create RKE2 config directory
      ansible.builtin.file:
        path: "{{ rke2_config_dir }}"
        state: directory
        mode: '0755'
    - name: Configure primary master node
      ansible.builtin.copy:
        dest: "{{ rke2_config_dir }}/config.yaml"
        content: |
          node-name: "{{ node_name }}"
          token: "{{ cluster_token }}"
          cni: "{{ cni_plugin }}"
          tls-san:
            - "{{ load_balancer_url }}"
            - "{{ ansible_ssh_host }}"
          node-external-ip: "{{ ansible_ssh_host }}"
          node-ip: "{{ ansible_ssh_host }}"
      when: inventory_hostname == primary_master
    - name: Configure additional master nodes
      ansible.builtin.copy:
        dest: "{{ rke2_config_dir }}/config.yaml"
        content: |
          node-name: "{{ node_name }}"
          server: "https://{{ hostvars[primary_master].ansible_ssh_host }}:9345"
          token: "{{ cluster_token }}"
          cni: "{{ cni_plugin }}"
          tls-san:
            - "{{ load_balancer_url }}"
            - "{{ ansible_ssh_host }}"
          node-external-ip: "{{ ansible_ssh_host }}"
          node-ip: "{{ ansible_ssh_host }}"
      when: inventory_hostname != primary_master and role == "server"
    - name: Configure worker nodes
      ansible.builtin.copy:
        dest: "{{ rke2_config_dir }}/config.yaml"
        content: |
          node-name: "{{ node_name }}"
          server: "https://{{ hostvars[primary_master].ansible_ssh_host }}:9345"
          token: "{{ cluster_token }}"
          node-external-ip: "{{ ansible_ssh_host }}"
          node-ip: "{{ ansible_ssh_host }}"
      when: role == "agent"
    - name: Start and enable RKE2 server service
      ansible.builtin.systemd:
        name: rke2-server
        state: started
        enabled: true
      when: role == "server"
    - name: Start and enable RKE2 agent service
      ansible.builtin.systemd:
        name: rke2-agent
        state: started
        enabled: true
      when: role == "agent"
    - name: Wait for primary master to be ready
      ansible.builtin.shell:
        cmd: "{{ rke2_bin_dir }}/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes | grep {{ primary_master }} | grep Ready"
        cmd_timeout: 300
      register: master_ready
      retries: 30
      delay: 10
      until: master_ready.rc == 0
      when: inventory_hostname == primary_master
    - name: Copy kubeconfig to control machine
      ansible.builtin.fetch:
        src: /etc/rancher/rke2/rke2.yaml
        dest: ~/.kube/config
        flat: yes
      when: inventory_hostname == primary_master
    - name: Install Helm
      ansible.builtin.shell:
        cmd: |
          curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
          chmod 700 get_helm.sh
          ./get_helm.sh --version {{ helm_version }}
        creates: /usr/local/bin/helm
      when: inventory_hostname == primary_master
    - name: Add Helm repositories
      ansible.builtin.shell:
        cmd: |
          helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
          helm repo add jetstack https://charts.jetstack.io
          helm repo update
      when: inventory_hostname == primary_master
    - name: Install cert-manager
      ansible.builtin.shell:
        cmd: |
          helm install cert-manager jetstack/cert-manager \
            --namespace cert-manager \
            --create-namespace \
            --version {{ cert_manager_version }} \
            --set installCRDs=true
      when: inventory_hostname == primary_master
    - name: Install Rancher
      ansible.builtin.shell:
        cmd: |
          helm install rancher rancher-latest/rancher \
            --namespace cattle-system \
            --create-namespace \
            --set hostname={{ rancher_hostname }} \
            --set bootstrapPassword=admin \
            --set replicas=3
      when: inventory_hostname == primary_master
    - name: Verify Rancher deployment
      ansible.builtin.shell:
        cmd: "{{ rke2_bin_dir }}/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get pods -n cattle-system | grep rancher | grep Running"
        cmd_timeout: 300
      register: rancher_ready
      retries: 30
      delay: 10
      until: rancher_ready.rc == 0
      when: inventory_hostname == primary_master
```

**Playbook Tasks**:
- Installs prerequisites (`curl`, `open-iscsi`, `nfs-common`)
- Sets unique hostnames for nodes
- Installs RKE2 binaries and configures nodes (primary master, additional masters, workers)
- Starts RKE2 services (`rke2-server` or `rke2-agent`)
- Installs Helm, cert-manager, and Rancher on the primary master
- Verifies cluster and Rancher deployment

## Running the Playbook

1. Save the inventory as `inventory.yml` and the playbook as `deploy_rke2_rancher.yml`.
2. Verify SSH connectivity:
   ```bash
   ansible all -i inventory.yml -m ping
   ```
3. Run the playbook:
   ```bash
   ansible-playbook -i inventory.yml deploy_rke2_rancher.yml
   ```

## Post-Installation Steps

1. **Verify Cluster**:
   ```bash
   kubectl get nodes
   ```
   Expected output:
   ```
   NAME       STATUS   ROLES                  AGE   VERSION
   master-1   Ready    control-plane,etcd     10m   v1.24.10+rke2r1
   master-2   Ready    control-plane,etcd     10m   v1.24.10+rke2r1
   master-3   Ready    control-plane,etcd     10m   v1.24.10+rke2r1
   worker-1   Ready    <none>                 10m   v1.24.10+rke2r1
   worker-2   Ready    <none>                 10m   v1.24.10+rke2r1
   worker-3   Ready    <none>                 10m   v1.24.10+rke2r1
   ```
   Check pod status:
   ```bash
   kubectl get pods -A
   ```

2. **Access Rancher**:
   - Visit `https://rancher.example.com`
   - Log in with the bootstrap password (`admin`) and set a new password
   - Note: Self-signed certificates may require bypassing browser warnings

3. **Load Balancer Configuration**:
   Configure a load balancer for ports 6443 (Kubernetes API), 9345 (RKE2 registration), 80, and 443 (Rancher UI).

   Example HAProxy configuration:
   ```haproxy
   frontend rke2-api
      bind *:6443
      mode tcp
      default_backend rke2-api-backend
   backend rke2-api-backend
      mode tcp
      server master-1 192.168.1.10:6443 check
      server master-2 192.168.1.11:6443 check
      server master-3 192.168.1.12:6443 check
   frontend rke2-registration
      bind *:9345
      mode tcp
      default_backend rke2-registration-backend
   backend rke2-registration-backend
      mode tcp
      server master-1 192.168.1.10:9345 check
      server master-2 192.168.1.11:9345 check
      server master-3 192.168.1.12:9345 check
   frontend rancher-http
      bind *:80
      mode tcp
      default_backend rancher-http-backend
   backend rancher-http-backend
      mode tcp
      server master-1 192.168.1.10:80 check
      server master-2 192.168.1.11:80 check
      server master-3 192.168.1.12:80 check
   frontend rancher-https
      bind *:443
      mode tcp
      default_backend rancher-https-backend
   backend rancher-https-backend
      mode tcp
      server master-1 192.168.1.10:443 check
      server master-2 192.168.1.11:443 check
      server master-3 192.168.1.12:443 check
   ```

## Additional Notes

- **Networking**: RKE2 includes an NGINX Ingress controller by default
- **Security**: Uses hardened images and secure defaults
- **TLS**: Self-signed certificates via cert-manager; for production, add `--set ingress.tls.source=letsEncrypt --set letsEncrypt.email=your-email@example.com` to Rancher Helm install
- **Firewall**: Open ports 6443, 9345, 80, 443, and CNI ports (e.g., 8472/UDP for Canal)
- **Cleanup**: Reset nodes with:
  ```bash
  /usr/local/bin/rke2-killall.sh
  /usr/local/bin/rke2-uninstall.sh
  ```

## References

- [RKE2 Documentation](https://docs.rke2.io)
- [Rancher Installation Guide](https://ranchermanager.docs.rancher.com)
- [Ansible RKE2 Setup Example](https://www.tech-couch.com)

---

*Generated on July 18, 2025, by Grok 3 | [xAI](https://x.ai)*