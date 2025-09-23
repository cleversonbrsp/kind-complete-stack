## Local Kubernetes Lab with Kind + Ingress + MetalLB

This project brings up a local 3-node Kubernetes cluster using Kind, installs the NGINX Ingress Controller, and configures MetalLB to simulate `LoadBalancer` Services. It includes a sample app and local access via `/etc/hosts`. You can also use an external domain service to expose it publicly if you wish (not detailed here).

- **Kind** (Kubernetes in Docker): `https://kind.sigs.k8s.io/`

### Why use this setup
- **Realistic**: same components used in production (Ingress, LoadBalancer).
- **Fast and reproducible**: `make rebuild` creates everything from scratch in minutes.
- **No cloud cost**: great for development, POCs, and learning.
- **CI-ready**: easy to create ephemeral clusters for E2E tests.

### Included components
- 3 nodes (1 control-plane, 2 workers)
- NGINX Ingress Controller (default class `nginx`)
- MetalLB with an IP pool for `LoadBalancer` Services
- Demo manifests (`hello-ingress.yaml`)

---

## Requirements
- Docker and permissions to run containers
- `kind`, `kubectl`, `helm`

---

## Repository structure

```text
kind-complete-stack/
├── Makefile
├── kind-cluster.yaml
├── deploy-nginx-ingress.sh
├── deploy-metallb.sh
├── hello-ingress.yaml
├── metrics-server.yaml
└── README.md
```

---

## How to bring up the cluster

```bash
make rebuild
```

This command performs, in order: destroy previous cluster, create a new one (`kind-cluster.yaml`), install NGINX Ingress and MetalLB.

If you get an error about ports 80/443 being in use, see "Troubleshooting > Ports 80/443 in use" below.

---

## Useful commands (Makefile)

- `make up`: create the cluster from `kind-cluster.yaml`
- `make ingress`: install the NGINX Ingress Controller
- `make metallb`: install and configure MetalLB (IPAddressPool + L2Advertisement)
- `make destroy`: remove the cluster
- `make rebuild`: recreate everything from scratch

Note: the `demo` and `hosts` targets are not active in the `Makefile`. You can apply the demo manually with `kubectl apply -f hello-ingress.yaml` and manage `/etc/hosts` as described below.

---

## Ways to access your application

### Local development via `/etc/hosts` (with MetalLB)
1. Bring up the cluster: `make rebuild`
2. Install the demo (optional):
   ```bash
   kubectl apply -f hello-ingress.yaml
   ```
3. Get the LoadBalancer IP of the NGINX Ingress:
   ```bash
   kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
   ```
4. Add the host to `/etc/hosts` (replace `$IP`):
   ```bash
   echo "$IP hello.local" | sudo tee -a /etc/hosts
   ```
5. Access: `http://hello.local`

> Tip: You can change the `host` in `hello-ingress.yaml` to another local domain, e.g., `app.local`.

### Using a domain service (optional)
You can use an external domain service (for example, a DNS provider) to point a name like `www.your-domain.com` to this cluster. This setup depends on your environment and provider and is not detailed here.

---

## Troubleshooting

### Ports 80/443 in use when creating the cluster
Symptom (Kind/Docker):

```
failed to bind host port for 0.0.0.0:80 ... address already in use
```

Cause: `kind-cluster.yaml` maps host ports 80 and 443 to the control-plane node. If the host already has `nginx`, `apache`, or another process on 80/443, creation fails.

Fix options:
- Stop the host service and recreate the cluster:
  ```bash
  sudo systemctl stop nginx apache2 httpd traefik caddy haproxy 2>/dev/null || true
  make rebuild
  ```
- Change the host ports (e.g., 8080/8443):
  ```yaml
  # kind-cluster.yaml
  nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 8080
      - containerPort: 443
        hostPort: 8443
  ```
  Access via `http://localhost:8080`.
- Remove `extraPortMappings` and use only the MetalLB IP (recommended for this stack). See the section
  "Local development via `/etc/hosts`" to map the host to the Ingress IP.

How to identify which process is using the port:
```bash
sudo lsof -nP -iTCP:80 -sTCP:LISTEN
sudo lsof -nP -iTCP:443 -sTCP:LISTEN
```

### MetalLB IP Pool
The `deploy-metallb.sh` script creates an `IPAddressPool` and an `L2Advertisement` with a default range. Adjust the range according to the Docker `kind` network subnet:
```bash
docker network inspect kind | grep Subnet
```
Edit the range in the script if necessary.

### Demo and custom domains
Change the host in `hello-ingress.yaml` to the desired domain (e.g., `www.devops.lab.com.br`). You can integrate an external domain service for public access if needed.

---

## Cleanup
```bash
make destroy
```

---

## Notes
- Kind does not support ingress-nginx admission webhooks by default; the install script disables the admission webhooks.
- To enable metrics and HPA, you can apply `metrics-server.yaml` (adjust as needed for your environment).

