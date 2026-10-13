Your Kubernetes environment is ready for the Keycloak lab.

Based on your output, we'll adapt the installation for your actual cluster:

- Kubernetes: v1.35.6, single node `devops-vm`
- Ingress controller: Traefik, so we'll use `ingress.className: traefik` instead of `nginx`.
- Storage: `microk8s-hostpath` is your default StorageClass.
- Hostname: `keycloak.local`
- Database: embedded PostgreSQL.
- Passwords: Kubernetes Secrets.
- Persistence: enabled for Keycloak and PostgreSQL.

We'll create the files using `mkdir -p` and `cat <<'EOF'`, as requested.

## 1. Create the lab directory

Run on `devops-vm`:

```
mkdir -p ~/keycloak-lab
cd ~/keycloak-lab

kubectl create namespace keycloak \
  --dry-run=client -o yaml | kubectl apply -f -
```

## 2. Create password Secrets

This generates random passwords without writing them into `values.yaml`.

```
ADMIN_PASSWORD="$(openssl rand -hex 24)"
POSTGRES_PASSWORD="$(openssl rand -hex 24)"

kubectl -n keycloak create secret generic keycloak-admin \
  --from-literal=admin-password="$ADMIN_PASSWORD" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n keycloak create secret generic keycloak-postgres \
  --from-literal=postgres-password="$POSTGRES_PASSWORD" \
  --dry-run=client -o yaml | kubectl apply -f -

unset ADMIN_PASSWORD POSTGRES_PASSWORD

kubectl get secrets -n keycloak
```

## 3. Create `values.yaml`

We'll use your default StorageClass and Traefik IngressClass.

```
cd ~/keycloak-lab

cat > values.yaml <<'EOF'
replicaCount: 1

keycloak:
  adminUser: admin
  existingSecret: keycloak-admin
  secretKeys:
    adminPasswordKey: admin-password

  hostname: http://keycloak.local
  hostnameStrict: false

  httpEnabled: true
  httpsEnabled: false
  production: false
  proxyHeaders: xforwarded

database:
  type: postgres
  name: keycloak
  username: postgres
  existingSecret: keycloak-postgres
  secretKeys:
    passwordKey: postgres-password

postgres:
  enabled: true
  auth:
    database: keycloak
    username: ""
    existingSecret: keycloak-postgres
  persistence:
    enabled: true
    size: 5Gi

persistence:
  enabled: true
  storageClass: ""
  size: 1Gi
  accessModes:
    - ReadWriteOnce

ingress:
  enabled: true
  className: traefik
  annotations: {}
  hosts:
    - host: keycloak.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

service:
  type: ClusterIP
  httpPort: 8080
  httpsPort: 8443
EOF
```

Review the file:

```
cat values.yaml
```

Important database detail: the supplied chart supports `database.existingSecret` for external PostgreSQL credentials and `postgres.auth.existingSecret` for the embedded PostgreSQL dependency. Because this chart has its own dependency configuration, we should verify that its rendered resources use the expected Secret keys before relying on them.

## 4. Download and validate the chart

```
helm show values \
  oci://registry-1.docker.io/cloudpirates/keycloak \
  > chart-default-values.yaml
```

Check the embedded PostgreSQL settings:

```
grep -n -A25 '^postgres:' chart-default-values.yaml
grep -n -A25 '^database:' chart-default-values.yaml
```

Render the chart:

```
helm template my-keycloak \
  oci://registry-1.docker.io/cloudpirates/keycloak \
  --namespace keycloak \
  -f values.yaml > rendered.yaml
```

If rendering succeeds, inspect the generated resources:

```
grep -n -E 'kind: (Deployment|StatefulSet|Ingress|PersistentVolumeClaim)' \
  rendered.yaml
```

## 5. Install Keycloak

```
helm install my-keycloak \
  oci://registry-1.docker.io/cloudpirates/keycloak \
  --namespace keycloak \
  -f values.yaml
```

Check the installation:

```
helm status my-keycloak -n keycloak
kubectl get pods -n keycloak -o wide
kubectl get svc,ingress,pvc -n keycloak
```

Wait for the workloads:

```
kubectl get pods -n keycloak -w
```

Press `Ctrl+C` after the pods become ready.

## 6. Configure `keycloak.local`

First find the node IP and Traefik service:

```
kubectl get nodes -o wide
kubectl get svc -A | grep -i traefik
kubectl get ingress -n keycloak
```

Since your cluster is a single-node VM, use the VM's reachable IP address for browser access. If you access the browser on the VM itself, you may be able to use its node IP directly; otherwise, use the VM's reachable network address.

On the machine running the browser, add this line to its hosts file, replacing `VM_IP`:

```
VM_IP keycloak.local
```

On Ubuntu, edit it with:

```
sudo sh -c 'cat >> /etc/hosts <<EOF
VM_IP keycloak.local
EOF'
```

Replace `VM_IP` before running the command.

## 7. Access Keycloak

Open:

[http://keycloak.local](http://keycloak.local)

Log in with:

- Username: `admin`
- Password: stored in the `keycloak-admin` Kubernetes Secret.

To retrieve the generated password when needed:

```
kubectl get secret keycloak-admin -n keycloak \
  -o jsonpath='{.data.admin-password}' | base64 -d
echo
```

## 8. Verify persistence and troubleshoot

```
kubectl get pvc -n keycloak
kubectl describe ingress -n keycloak
kubectl get endpoints -n keycloak
kubectl get events -n keycloak --sort-by=.lastTimestamp
```

If Keycloak fails to start:

```
kubectl get pods -n keycloak
kubectl logs -n keycloak deploy/my-keycloak
```

If the PostgreSQL pod fails, get its actual name from `kubectl get pods` and inspect its logs.

One final note: `microk8s-hostpath` is suitable for this single-node lab, but it is node-local storage rather than highly available storage. Also, HTTP is suitable only for local testing; use HTTPS before exposing Keycloak beyond your trusted lab network.

