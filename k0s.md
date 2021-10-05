# k0s and k0sctl notes

```bash
# Prepare a k0s cluster name
K0S_CLUSTER=valyria

# Find VMs IPs
cd ~/git/terraform-libvirt-vm
VM_PRIVATE_KEY=$(terraform output --raw ssh_private_key_filename)

VM_NODES=$(terraform output --json network | jq -r '.master[].addresses[]') # Assuming just one node for now

# All nodes (lowest IP becomes controller):
# IFS=$'\n' VM_NODES=($(terraform output --json network | jq '.[] | map(select(.addresses))' | jq -r '.[].addresses' | sort))


# Download k0sctl for creating k0s cluster
export K0SCTL_VERSION='0.10.4'
sudo curl --location --output /usr/local/bin/k0sctl https://github.com/k0sproject/k0sctl/releases/download/v${K0SCTL_VERSION}/k0sctl-linux-x64
sudo chmod +x /usr/local/bin/k0sctl

# Create k0s configuration for one node
k0sctl init --k0s --cluster-name ${K0S_CLUSTER} --user ansible --key-path ${VM_PRIVATE_KEY} $VM_NODES > /tmp/k0sctl.yaml
# With only one node, you need to edit /tmp/k0sctl.yaml
# so the node is both a controller and a worker node
sed -i 's/^.*role: controller$/    role: "controller+worker"/g' /tmp/k0sctl.yaml

# If using all nodes (lowest IP becomes controller):
#k0sctl init --k0s --cluster-name ${K0S_CLUSTER} --user ansible --key-path ${VM_PRIVATE_KEY} ${VM_NODES[@]} > /tmp/k0sctl.yaml


# Create k0s cluster
k0sctl apply --config /tmp/k0sctl.yaml

# Create KUBECONFIG
k0sctl kubeconfig --config /tmp/k0sctl.yaml > ~/.kube/${K0S_CLUSTER}.config

# Test cluster
export KUBECONFIG=~/.kube/${K0S_CLUSTER}.config
kubectl get nodes -o wide
kubectl get pods -A
```