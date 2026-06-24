# Simple AWX Buildout on Ubuntu with MicroK8s

AWX running on a single Ubuntu VM using MicroK8s (single-node Kubernetes). No cloud required.

## Requirements

- Ubuntu 22.04+ VM
- 4GB+ RAM (AWX + MicroK8s idle at ~2-3GB)
- 20GB+ disk (images + postgres data)
- Internet access from the VM

## 1. Expand Disk if Needed

If the VM is tight on space, check first:

```bash
df -h /
vgdisplay  # check free space in the volume group
```

If you have free space in the VG but not the filesystem:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

## 2. Install MicroK8s

```bash
sudo snap install microk8s --classic
sudo usermod -aG microk8s $USER
newgrp microk8s
microk8s status --wait-ready
```

Enable required addons:

```bash
microk8s enable hostpath-storage dns
```

## 3. Install kustomize (standalone binary)

Do NOT use the snap version — it runs confined and can't access `/tmp`. Install the binary directly:

```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
```

## 4. Export Kubeconfig

So system tools (kustomize) can talk to MicroK8s:

```bash
mkdir -p ~/.kube
microk8s config > ~/.kube/config
```

## 5. Deploy AWX Operator

```bash
mkdir -p /tmp/awx-operator
cat > /tmp/awx-operator/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - github.com/ansible/awx-operator/config/default?ref=2.19.1
namespace: awx
EOF

microk8s kubectl create namespace awx
kustomize build /tmp/awx-operator/ | microk8s kubectl apply -f -
```

**Gotcha:** `gcr.io/kubebuilder/kube-rbac-proxy` is deprecated. Patch it immediately after apply:

```bash
microk8s kubectl set image deployment/awx-operator-controller-manager \
  kube-rbac-proxy=registry.k8s.io/kubebuilder/kube-rbac-proxy:v0.15.0 \
  -n awx
```

Wait for the operator to be ready:

```bash
microk8s kubectl wait deployment/awx-operator-controller-manager \
  -n awx --for=condition=Available --timeout=300s
```

## 6. Deploy AWX Instance

```bash
cat > /tmp/awx-instance.yaml <<EOF
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: NodePort
  nodeport_port: 30080
EOF

microk8s kubectl apply -f /tmp/awx-instance.yaml
```

## 7. Fix Kubelet TLS Certificate

`kubectl logs` and `kubectl exec` will fail if the kubelet cert was generated with the wrong IP (e.g. old VirtualBox NAT IP instead of your bridged IP). Check:

```bash
sudo openssl x509 -in /var/snap/microk8s/current/certs/kubelet.crt -text -noout | grep -A 5 "Subject Alternative"
```

If your VM's real IP is missing, confirm it's in the template:

```bash
sudo cat /var/snap/microk8s/current/certs/csr.conf.template | grep -A 10 "alt_names"
```

Add your IP to `[alt_names]` if missing (e.g. `IP.3 = 192.168.1.57`), then regenerate:

```bash
sudo openssl req -new \
  -key /var/snap/microk8s/current/certs/kubelet.key \
  -out /tmp/kubelet.csr \
  -config /var/snap/microk8s/current/certs/csr.conf.template

sudo openssl x509 -req \
  -in /tmp/kubelet.csr \
  -CA /var/snap/microk8s/current/certs/ca.crt \
  -CAkey /var/snap/microk8s/current/certs/ca.key \
  -CAcreateserial \
  -out /var/snap/microk8s/current/certs/kubelet.crt \
  -days 3650 \
  -extensions v3_ext \
  -extfile /var/snap/microk8s/current/certs/csr.conf.template

sudo microk8s stop && sudo microk8s start
```

## 8. Watch Pods Come Up

```bash
sudo microk8s kubectl get pods -n awx -w
```

Expected sequence:

| Pod | Final State |
|-----|-------------|
| `awx-operator-controller-manager` | `2/2 Running` |
| `awx-postgres-15-0` | `1/1 Running` |
| `awx-migration-<version>` | `0/1 Completed` (appears mid-way, runs DB migrations) |
| `awx-task` | `4/4 Running` (after migration completes) |
| `awx-web` | `3/3 Running` |

`awx-task` will sit at `Init:0/2` until the migration pod completes — this is normal and takes 5–10 minutes.

## 9. Get Admin Password

```bash
microk8s kubectl get secret awx-admin-password -n awx \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

## 10. Access AWX

- URL: `http://<vm-ip>:30080`
- Username: `admin`
- Password: from step 9

## 11. Connect Your Ansible Repo

AWX pulls from git — it does not use local files.

1. **Project** → Add → Source Control Type: Git → enter your GitHub repo URL → Save
2. **Credentials** → Add a *Machine* credential with your SSH private key; add *Amazon Web Services* credential if using EC2 dynamic inventory
3. **Inventories** → Add → add hosts manually or add an Inventory Source pointing to `inventory/aws_ec2.yml`
4. **Templates** → Add Job Template → select Project + playbook + Inventory + Credential → Launch

## Automation

The full installation is automated in `install_awx.yml`. Run it against a fresh Ubuntu host:

```bash
ansible-playbook install_awx.yml
```

## Gotchas Summary

| Problem | Fix |
|---------|-----|
| MicroK8s bundled git conflicts with kustomize | Install kustomize as a standalone binary, not snap |
| `gcr.io/kubebuilder/kube-rbac-proxy` image pull fails | Patch deployment to `registry.k8s.io/kubebuilder/kube-rbac-proxy:v0.15.0` |
| `kubectl logs` fails with TLS cert error | Regenerate `kubelet.crt` using the CA and `csr.conf.template` with your VM's real IP |
| `awx-task` stuck at `Init:0/2` forever | Wait — the `awx-migration` job runs DB migrations first; task pod starts after |
| Disk full during image pulls | Expand LVM before starting: `lvextend` + `resize2fs` |
