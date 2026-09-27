# lqd-dth
Lab Environment Deployment for OT Asset Virtualization, Cyber Security Testing, and other use cases...

I would love to say that I got this up and running in less than two hours, but alas, I'm too upfront. This initial setup took me about two to three days of fiddling, since I had never used Kubernetes or Multipass before.

# Requirments
- A semi-24/7 hosting environment for various use cases trafficked through a WireGuard tunnel.
- The lab would run on my home (Windows) PC and be on 24/7 (given Windows OS quirks & for when I want to use my full PC resources).
- Hardened ASF

If I think of more, I'll add them here...

# Set-Up

(Throughout the explanation, I write lines of code that I either ran through my VM console directly or through my Windows PowerShell; you can tell which is which from the lines that include 'sudo')

## Part One (Baselines)

I first had to upgrade to Windows 11 Pro (go figure, thanks, Microsoft) and enable Hyper-V
I then created an external virtual network switch through Windows Hyper-V Manager

- Initialize VM and start dashboard proxy
```
microk8s install
microk8s status --wait-ready
microk8s dashboard-proxy
```
- Inspect internal subsystems & storage mounts (at first I had issues with storage not initializing properly)
```
multipass exec microk8s-vm -- microk8s inspect
```
- Connect Cluster Mapping & Sync KubeConfig Locally
```
New-Item -ItemType Directory -Force -Path "$HOME\.kube"
multipass exec microk8s-vm -- sudo microk8s config > "$HOME\.kube\config"
kubectl get nodes
```
- Reconfigure VM hardware allocations
```
multipass stop microk8s-vm
multipass set local.microk8s-vm.cpus=4
multipass set local.microk8s-vm.disk=50G
multipass set local.microk8s-vm.memory=8G
```
- Route network through external virtual network switch for split tunnel (ip a, multipass list, etc.)
- Retrieve updated kubeconfig following bridge change
```
multipass exec microk8s-vm -- sudo microk8s config > "$HOME\.kube\config"
```
- Confirm Node Connection
```
kubectl get nodes
```
- Add new IP address to Certificate Signing Request (CSR) template for proxy access
```
sudo nano /var/snap/microk8s/current/certs/csr.conf.template
(under alt names) IP.100 = <YOUR_VM_IP>
```
- Clear any locks (if any) and refresh both API Server & CA certificates
```
sudo rm -f /var/snap/microk8s/current/var/lock/no-cert-reissue
sudo microk8s refresh-certs --cert server.crt
sudo microk8s refresh-certs --cert ca.crt
```
## Part Two (Configurations/Personal Customizations)

- I created a Persistent Admin ServiceAccount Token for easier local cluster management
```
kubectl create serviceaccount admin-user -n kube-system
kubectl create clusterrolebinding admin-user-binding --clusterrole=cluster-admin --serviceaccount=kube-system:admin-user

@'
apiVersion: v1
kind: Secret
metadata:
 name: admin-user-token
 namespace: kube-system
 annotations:
 kubernetes.io/service-account.name: "admin-user"
type: kubernetes.io/service-account-token
'@ | kubectl apply -f -

kubectl -n kube-system get secret admin-user-token -o jsonpath="{.data.token}" | % { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```
(make sure to hold onto that token for dear life)

- Preliminary Microk8s add-ons
```
microk8s enable dns storage
microk8s enable rbac ingress
microk8s enable cis-hardening
```
- Run CIS Benchmark
```
sudo microk8s kube-bench (I found so many warnings, no failures though!)
```
- Let's fix those errors with some Firewall configurations
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 16443/tcp
sudo ufw allow in on lo
sudo ufw allow out on lo
sudo ufw allow from XXX.XXX.XXX.XXX/24
sudo ufw allow from XXX.XXX.XXX.XXX/16
sudo ufw allow in on cali+
sudo ufw allow out on cali+
sudo ufw allow 10443/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 30000:32767/tcp
sudo ufw allow from XXX.XXX.XXX.XXX/24
sudo ufw enable
```
- We need an internal FW policy for cluster health
```
kubectl create namespace asset-test

@'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: default-deny-all
 namespace: asset-test
spec:
 podSelector: {}
 policyTypes:
    Ingress
    Egress
'@ | kubectl apply -f -
```
- Kubebench was being annoying post-hardening, so I wrote some symbolic links for kube-bench, then exported benchmark results for review a little more cleanly
```
sudo ln -s /snap/bin/microk8s.kubectl /usr/local/bin/kubectl
sudo ln -s /var/snap/microk8s/current/args/kubelet /usr/local/bin/kubelet
sudo microk8s kube-bench > ~/cis-benchmark-results.txt
grep -E "\[WARN\]|\[FAIL\]" ~/cis-benchmark-results.txt
```
- Don't forget to change your default namespace. I quarantined mine with some network isolation
```
@'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: default-deny-all
 namespace: default
spec:
 podSelector: {}
 policyTypes:
  Ingress
  Egress
'@ | kubectl apply -f -
```
- I want to be able to work on the cluster remotely
```
ssh-keygen -t ed25519 -b 4096 -C "XXX"
multipass exec microk8s-vm -- bash -c "mkdir -p ~/.ssh && echo '$(cat ~/.ssh/id_ed25519.pub)' >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
multipass info microk8s-vm
ssh ubuntu@<YOUR_VM_IP>
```
- Having kubectl installed on MacBook makes life easier
```
brew install kubernetes-cli
kubectl version --XXX
mkdir -p ~/.kube
ssh ubuntu@<YOUR_VM_IP> "sudo microk8s config" > ~/.kube/config
kubectl config set-cluster microk8s-cluster --server=https://<YOUR_VM_IP>:16443
kubectl get nodes
kubectl get networkpolicy -n asset-test
```
- More tools to install and use for the next git commit :)
```
brew install k9s
brew install wireshark
```
# My Miseries
At the bottom, I'm including my lessons learned/rants on the process. I had a tough time getting an environment up and running with the standard installation launchers for micro8ks. From times when I forgot my host-machine VPN was connected, to how even basic custom configurations in that launcher gave me so many errors. Including the hair-pulling that came up when I tried to use a bridged network for the environment. I ended up doing everything manually, in the sense of just using the CLI after getting the kubectl dependencies I needed to properly install microk8s, etc.
