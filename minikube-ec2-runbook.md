# Runbook: minikube on EC2 (Amazon Linux 2023)

Run a lightweight single-node Kubernetes cluster on an EC2 instance, pull images
from **Docker Hub** and **Amazon ECR**, deploy them, and access the web app from
your laptop.

> **Run all commands as `ec2-user`** (no sudo for docker/minikube).
>
> **⚠️ CPU architecture matters.** Graviton EC2 instances (t4g, c7g, m7g, …) are
> **ARM64**; Intel/AMD instances are **x86_64**. Downloading the wrong binary
> gives `cannot execute binary file: Exec format error`. Check with:
>
> ```bash
> uname -m        # aarch64 = ARM/Graviton (arm64),  x86_64 = Intel/AMD (amd64)
> ```
>
> The install steps below **auto-detect** the arch, so they work on either.

---

## 0. Build & publish the demo image (buildx, multi-arch)

The workshop app lives at
`/home/ec2-user/.kiro/crew/workspace/workshop-repo-devops-haitran-main` — a
static site served by nginx. It builds into a **cross-platform** image
(`linux/amd64` + `linux/arm64`) with `docker buildx` and publishes to the
**public** Docker Hub repo `tranminhhai/swe4006-minikube-demo`.

Files added to the repo:

| File                | Purpose                                              |
|---------------------|------------------------------------------------------|
| `Dockerfile`        | `nginx:stable-alpine` serving `index.html` + logo.   |
| `.dockerignore`     | Keeps screenshots / AWS config out of the image.     |
| `build-and-push.sh` | buildx multi-arch build + push wrapper.              |

**Log in to Docker Hub from the CLI** — a browser login is not enough for a
`docker push`:

```bash
docker login            # username: tranminhhai, then password / access token
```

**Build and push (one command):**

```bash
cd /home/ec2-user/.kiro/crew/workspace/workshop-repo-devops-haitran-main
./build-and-push.sh             # pushes :latest
./build-and-push.sh v1          # also pushes :v1
```

What the script does:

```bash
# creates a buildx builder that supports multi-platform, then:
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag tranminhhai/swe4006-minikube-demo:latest \
  --push \
  .
```

**Verify the manifest lists both architectures:**

```bash
docker buildx imagetools inspect tranminhhai/swe4006-minikube-demo:latest
```

> `buildx --push` builds and uploads in one step — a multi-arch image cannot be
> held in the local `docker images` list, so there is no separate `docker push`.

---

## 1. Install Docker

```bash
sudo dnf update -y
sudo dnf install -y docker
sudo systemctl enable --now docker
```

**Run Docker as `ec2-user` without `sudo`** — add the user to the `docker`
group, then start a fresh login shell so the new group membership is picked up:

```bash
sudo usermod -aG docker ec2-user

# the group only applies to NEW shells, so do one of:
newgrp docker            # switch group in the current shell, OR
exit                     # log out and SSH back in

# verify it works with no sudo:
id -nG | tr ' ' '\n' | grep -x docker   # should print: docker
docker ps                               # should run with NO sudo and NO error
docker run --rm hello-world             # full round-trip test
```

> If `docker ps` still says *permission denied* after `newgrp`, you did not start
> a fresh shell — log out of SSH and back in. Every command below assumes
> sudo-free docker, and **minikube's docker driver requires it** (never run
> `minikube` with `sudo`).

## 2. Install kubectl

```bash
# auto-pick amd64 or arm64 from the machine's arch
ARCH=$([ "$(uname -m)" = "aarch64" ] && echo arm64 || echo amd64)

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/${ARCH}/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

> Got `Exec format error`? You downloaded the wrong arch. Delete the file
> (`rm kubectl`) and re-run the block above — it reads `uname -m` so it picks the
> right one.

## 3. Install minikube

```bash
# reuse ARCH from step 2, or re-detect:
ARCH=$([ "$(uname -m)" = "aarch64" ] && echo arm64 || echo amd64)

curl -LO "https://storage.googleapis.com/minikube/releases/latest/minikube-linux-${ARCH}"
sudo install "minikube-linux-${ARCH}" /usr/local/bin/minikube
minikube version
```

## 4. Start the cluster (Docker driver)

```bash
minikube start --driver=docker
minikube status
kubectl get nodes   # should show Ready
```

> Run `minikube` as `ec2-user`, **never with sudo**. The docker driver uses your
> user's rootless docker access (step 1); running it as root creates a root-owned
> `~/.minikube` and breaks later `kubectl` calls made as `ec2-user`.

**A successful first start looks like this** (it downloads ~780 MB of preload +
base images the first time, then is instant on later starts):

```
🔥  Creating docker container (CPUs=2, Memory=2200MB) ...
📦  Preparing Kubernetes v1.37.0 on containerd 2.3.4 ...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster ...
```

Confirm the cluster is healthy:

```bash
kubectl get nodes      # 1 node, STATUS Ready
minikube status        # host / kubelet / apiserver all Running
```

> **Error `PROVIDER_DOCKER_NEWGRP ... permission denied ... docker.sock`?** Your
> shell is not yet in the `docker` group. Fix and retry:
>
> ```bash
> sudo usermod -aG docker $USER
> newgrp docker          # or log out of SSH and back in
> docker ps              # must succeed with NO sudo
> minikube start --driver=docker --cpus=2 --memory=2200mb
> ```

On a small instance, cap resources:

```bash
minikube start --driver=docker --cpus=2 --memory=2200mb
```

---

## 5a. Deploy a Docker Hub image

Public images need no login.

```bash
kubectl create deployment hub-web --image=nginx:latest
kubectl expose deployment hub-web --port=80 --type=ClusterIP
```

## 5b. Deploy an ECR image

ECR requires auth. Two parts: the EC2 must be allowed to reach ECR, and the
cluster needs a pull secret.

**Give EC2 permission** — attach an IAM role to the instance with
`AmazonEC2ContainerRegistryReadOnly`, or a policy allowing:

- `ecr:GetAuthorizationToken`
- `ecr:BatchGetImage`
- `ecr:GetDownloadUrlForLayer`

**Create a pull secret from a fresh ECR token:**

```bash
ACCOUNT=123456789012
REGION=ap-southeast-1
REGISTRY=$ACCOUNT.dkr.ecr.$REGION.amazonaws.com

kubectl create secret docker-registry ecr-cred \
  --docker-server=$REGISTRY \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region $REGION)"
```

> ⚠️ The ECR token expires after **12 hours**. Re-run the secret command if a
> later pull fails with `401`.

**Deploy with the secret:**

```bash
kubectl create deployment ecr-web --image=$REGISTRY/my-app:latest
kubectl patch deployment ecr-web -p \
  '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"ecr-cred"}]}}}}'
kubectl expose deployment ecr-web --port=80 --type=ClusterIP
```

## 5c. Deploy THIS workshop image (public Docker Hub)

The image published in step 0 is public, so no secret is needed.

```bash
kubectl create deployment swe-demo --image=tranminhhai/swe4006-minikube-demo:latest
kubectl expose deployment swe-demo --port=80 --type=ClusterIP

# optional: run 2 replicas to show scaling
kubectl scale deployment swe-demo --replicas=2
```

Watch it come up:

```bash
kubectl get pods -l app=swe-demo -w   # Ctrl+C once Running
```

---

## 6. Check it is running

```bash
kubectl get pods
kubectl get svc
kubectl logs deploy/hub-web
kubectl describe pod <pod-name>   # shows pull errors if any
```

| Pod status        | Meaning                              |
|-------------------|--------------------------------------|
| `Running`         | Good.                                |
| `ImagePullBackOff`| Auth failed or image name is wrong.  |
| `ErrImagePull`    | Could not pull the image.            |

### 6b. Quickest test from the EC2 box (curl)

Replace `hub-web` with `swe-demo` / `ecr-web` for the other deployments.

**Option 1 — port-forward the service, then curl:**

```bash
kubectl port-forward svc/hub-web 8080:80 &   # & backgrounds it
curl http://localhost:8080                   # should print the HTML
# stop it later:  kill %1   (or fg then Ctrl+C)
```

**Option 2 — throwaway curl pod (no port-forward, hits service by name):**

```bash
kubectl run tmp --rm -it --image=curlimages/curl --restart=Never -- curl -s http://hub-web
```

It spins up, curls the service inside the cluster, prints the page, and deletes
itself.

**Logs & health** — a Service has no logs; its *pods* do:

```bash
kubectl logs deploy/hub-web            # nginx/app logs
kubectl logs deploy/hub-web -f         # follow live
kubectl get pods -l app=hub-web        # STATUS = Running
kubectl get endpoints hub-web          # must list a pod IP:port, not <none>
```

> `endpoints` showing `<none>` = the service matches no pod (label mismatch),
> which is why a curl would hang.

---

## 7. Access the web app

The cluster runs **inside** the minikube Docker container, so a plain NodePort
is not reachable from your laptop. Use port-forward.

**On the EC2 box** (run as `ec2-user`, no sudo):

```bash
kubectl port-forward deployment/swe-demo 8080:80 --address 0.0.0.0
```

- `8080:80` = EC2 port 8080 → container port 80.
- `--address 0.0.0.0` makes it listen on all interfaces.
- Swap `swe-demo` for `hub-web` / `ecr-web` to reach the other deployments.

**From your laptop — two options:**

### Option A — SSH tunnel (recommended, nothing exposed publicly)

```bash
ssh -i key.pem -L 8080:localhost:8080 ec2-user@<EC2_PUBLIC_IP>
# then open http://localhost:8080 in your browser
```

With a tunnel you can drop `--address 0.0.0.0` and just run
`kubectl port-forward deployment/hub-web 8080:80`.

### Option B — open the port (quick demo, less safe)

Add an inbound rule for **TCP 8080** in the EC2 security group **from your IP
only**, then open `http://<EC2_PUBLIC_IP>:8080`.

> Prefer the SSH tunnel. Opening 8080 to the internet exposes the app to
> everyone until you remove the rule.

---

## 8. Clean up

```bash
kubectl delete deployment hub-web ecr-web
kubectl delete svc hub-web ecr-web
minikube stop      # keep the cluster for later
minikube delete    # remove it entirely
```

---

## Quick reference

| Task                     | Command                                                       |
|--------------------------|---------------------------------------------------------------|
| Start cluster            | `minikube start --driver=docker`                              |
| Cluster status           | `minikube status`                                             |
| Deploy image             | `kubectl create deployment <name> --image=<image>`            |
| Expose (internal)        | `kubectl expose deployment <name> --port=80 --type=ClusterIP` |
| ECR login token          | `aws ecr get-login-password --region <region>`                |
| List pods                | `kubectl get pods`                                            |
| Pod logs                 | `kubectl logs deploy/<name>`                                  |
| Port-forward             | `kubectl port-forward deployment/<name> 8080:80`              |
| Stop / delete cluster    | `minikube stop` / `minikube delete`                           |

---

_SWE40006 Software Deployment and Evolution — DevOps / CI/CD runbook._
