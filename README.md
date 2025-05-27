### Prerequisites for Harbor Registry Installation

Before installing Harbor (or Mirantis Secure Registry), ensure the following prerequisites are met:

##### 1. Kubernetes Cluster
- A running Kubernetes cluster (v1.18 or later is recommended).
- At least 3 nodes for HA setup (optional for basic single-node installation).
- Sufficient CPU and memory resources for registry, database, and other components.

##### 2. Helm
- Helm 3 installed and configured.
- Configure the StorageClass for Dynamic provisioning.
- Enabled the Nginx Ingress controller from MKE UI.


### Install NFS Subdir External Provisioner

#### Ensure that the NFS provisioner storageClass is configured. If not then below are the steps to configure.

##### Choose an NFS Server Node
Pick any node to act as the NFS server and execute the following commands based on your OS.

###### For RedHat-based Systems:
```sh
yum install nfs-utils -y
mkdir -p /var/nfs/general
chown -R nobody /var/nfs/general
chmod -R 755 /var/nfs/general
echo '/var/nfs/general *(rw,sync,no_root_squash,no_subtree_check)' > /etc/exports
systemctl restart nfs-server
systemctl enable nfs-server
```

###### For Debian-based Systems:
```sh
apt-get update -y
apt-get install nfs-server -y
mkdir -p /var/nfs/general
chown -R nobody /var/nfs/general
chmod -R 755 /var/nfs/general
echo '/var/nfs/general *(rw,sync,no_root_squash,no_subtree_check)' > /etc/exports
systemctl restart nfs-server
systemctl enable nfs-server
```

##### Install NFS Utilities on All Hosts

- For RedHat-based Systems:
```sh
sudo yum install -y nfs-utils
```
- For Ubuntu and Debian-based Systems:
```sh
sudo apt-get update -y; sudo apt-get install nfs-common -y
```

##### Add the NFS Provisioner Helm Repo
```sh
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
```

##### Install the NFS Provisioner
Change the IP address accordingly before executing:
```sh
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=????? \
  --set nfs.path=/var/nfs/general
```
##### Patch the storageClass nfs-client as default
```sh
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

###### Notes:
- Replace `?????` with the IP address of your NFS server.
- Ensure that the NFS path exists and is accessible by your cluster nodes.
- This assumes the `nfs-client` StorageClass was created by the Helm chart.

Let me know if you want to include a sample `values.yaml` for the provisioner or additional validation steps.


### Create the NameSpace, add node label where you want to install all the components related to MSR4 (Harbor).
```sh
kubectl create namespace msr4
kubectl label nodes <node-name> node-role.kubernetes.io/msr: "true"
```


### PostgreSQL HA Setup with Bitnami Helm Chart

##### Add the Bitnami Helm repository
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```
##### Install HA PostgreSQL using a custom values file
```sh
helm install postgresql bitnami/postgresql-ha -f postgresql-values.yaml -n msr4
```
##### Obtain the PostgreSQL password
```sh
export POSTGRES_PASSWORD=$(kubectl get secret --namespace msr4 postgresql-postgresql-ha-postgresql -o jsonpath="{.data.password}" | base64 -d)
echo $POSTGRES_PASSWORD
```
##### Login to PostgreSQL
```sh
kubectl run postgresql-postgresql-ha-client --rm --tty -i --restart='Never' --namespace msr4 \
  --image docker.io/bitnami/postgresql-repmgr:16.4.0-debian-12-r12 \
  --env="PGPASSWORD=$POSTGRES_PASSWORD" \
  --command -- psql -h postgresql-postgresql-ha-pgpool -p 5432 -U postgres -d postgres
```
##### Check the database in postgres.It wil prompt for "POSTGRES_PASSWORD"
```sh
kubectl exec -n msr4 -i $(kubectl get pods -n msr4 -l app.kubernetes.io/component=postgresql -o name | head -n1) -- psql -U postgres -c "SELECT datname FROM pg_database;"
```
##### Create the harbor_core database. Addionally if required then create "notary_signer" and "notary_server". It wil prompt for "POSTGRES_PASSWORD"
```sh
kubectl exec -n msr4 -i $(kubectl get pods -n msr4 -l app.kubernetes.io/component=postgresql -o name | head -n1) -- psql -U postgres -c "CREATE DATABASE harbor_core;"
kubectl exec -n msr4 -i $(kubectl get pods -n msr4 -l app.kubernetes.io/component=postgresql -o name | head -n1) -- psql -U postgres -c "CREATE DATABASE notary_signer;"
kubectl exec -n msr4 -i $(kubectl get pods -n msr4 -l app.kubernetes.io/component=postgresql -o name | head -n1) -- psql -U postgres -c "CREATE DATABASE notary_server;"
```

##### Find the pool service and port
```sh
kubectl get service -n msr4
```
###### Example output:
```
NAME                                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
postgresql-postgresql-ha-pgpool       ClusterIP   10.96.40.233   <none>        5432/TCP
```



### Redis HA Setup with Bitnami Helm Chart

##### Add the Bitnami Helm repository to Helm
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```

##### Install Redis with a custom values file
```sh
helm install redis bitnami/redis -f redis-values.yaml -n msr4
```
##### Save the Redis password as an environment variable
```sh
export REDIS_PASSWORD=$(kubectl get secret --namespace msr4 redis -o jsonpath="{.data.redis-password}" | base64 -d)
echo $REDIS_PASSWORD
```

##### Get the Redis service name and IP address
```sh
kubectl get service -n msr4
```
###### Example output:
```
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
redis-master   ClusterIP   10.96.84.98    <none>        6379/TCP
```



### Install HA Mirantis Secure Registry (MSR)

##### Get the Harbor values.yaml file
```sh
helm show values oci://registry.mirantis.com/harbor/helm/harbor > harbor-values.yaml
```
##### (Optional) Create custom TLS certificates

##### Step 1: Create a directory for certificates
```sh
mkdir certs
```

##### Step 2: Create a harbor.conf file in the certs directory
```sh
cat <<EOF > certs/harbor.conf
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
C = US
ST = State
L = City
O = Organization
OU = Organizational Unit
CN = msr

[v3_req]
keyUsage = digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
IP.1 = <IP-ADDRESS-OF-WORKERNODE>  # Replace with your actual IP address
EOF
```

##### Step 3: Generate the certificate and key
```sh
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/tls.key -out certs/tls.crt \
  -config certs/harbor.conf
```

##### Step 4: (Only if using custom certs) Create Kubernetes TLS secret
```sh
kubectl -n msr4 create secret tls <NAME-OF-YOUR-SECRET> \
  --cert=certs/tls.crt \
  --key=certs/tls.key
```

##### Modify the harbor-values.yaml. Update the postgres password, redis password, secret name, Ingress hostname & externalURL'msr4.example.com' and pv size as per requirement. 

##### Install MSR using Helm with the configured values file
```sh
helm install msr4 oci://registry.mirantis.com/harbor/helm/harbor -f <PATH-TO/harbor-values.yaml> -n msr4
```

###### Notes:
- Replace `<IP-ADDRESS-OF-WORKERNODE>` with your actual IP address.
- Replace `<NAME-OF-YOUR-SECRET>` with a meaningful name for your Kubernetes TLS secret.
- Replace `<PATH-TO/harbor-values.yaml>` with the actual path to your values file.

