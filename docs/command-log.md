# Doc 04 CA

## 4 CA
Create the CA configuration file:
```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```
Create the CA certificate signing request:

```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "New York",
      "O": "Refinery29",
      "OU": "CA",
      "ST": "New York"
    }
  ]
}
EOF
```
Generate the CA certificate and private key:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```
Results:

```
ca-key.pem
ca.pem
```
## Client and Server Certificates

###The Admin Client Certificate
Create the admin client certificate signing request:

```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "New York",
      "O": "system:masters",
      "OU": "Refinery29 Kubernetes",
      "ST": "New York"
    }
  ]
}
EOF
```
Generate the admin client certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```
Results:

```
admin-key.pem
admin.pem
```

### The Kubelet Client Certificates

Generate a certificate and private key for each Kubernetes worker node:

```
for instance in compute10.cloud.rf29.net compute11.cloud.rf29.net
do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "New York",
      "O": "system:nodes",
      "OU": "Refinery29 Kubernetes",
      "ST": "New York"
    }
  ]
}
EOF

EXTERNAL_IP=$(ssh $instance ip a | awk '/10.10.210/ {gsub("/.*", "", $2); print $2}')
INTERNAL_IP=$(ssh $instance ip a | awk '/10.10.220/ {gsub("/.*", "", $2); print $2}')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```
Results:

```
compute10.cloud.rf29.net-key.pem
compute10.cloud.rf29.net.pem
compute11.cloud.rf29.net-key.pem
compute11.cloud.rf29.net.pem
```
### The kube-proxy Client Certificate
Create the kube-proxy client certificate signing request:

```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "New York",
      "O": "system:node-proxier",
      "OU": "Refinery29 Kubernetes",
      "ST": "New York"
    }
  ]
}
EOF
```
Generate the kube-proxy client certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```
Results:

kube-proxy-key.pem
kube-proxy.pem

### The Kubernetes API Server Certificate
The kubernetes-the-hard-way static IP address will be included in the list of subject alternative names for the Kubernetes API Server certificate. This will ensure the certificate can be validated by remote clients.

Retrieve the kubernetes-the-hard-way static IP address:

```
KUBERNETES_PUBLIC_ADDRESS=kubernetes.cloud.rf29.net
```
Create the Kubernetes API Server certificate signing request:

```
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```
Generate the Kubernetes API Server certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```
Results:

```
kubernetes-key.pem
kubernetes.pem
```

### Distribute the Client and Server Certificates
Copy the appropriate certificates and private keys to each worker instance:

```
for instance in $(awk '/\[worker\]/{flag=1;next}; /^\[/{flag=0}flag' hosts)
do
  scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done
```
Copy the appropriate certificates and private keys to each controller instance:

```
for instance in $(awk '/\[controller\]/{flag=1;next}; /^\[/{flag=0}flag' hosts)
do
  scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem ${instance}:~/
done
```


## 5 kubeconfigs

```
for instance in $(awk '/\[worker\]/{flag=1;next}/^\[/{flag=0}flag' hosts)
do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

### The kube-proxy Kubernetes Configuration File
Generate a kubeconfig file for the kube-proxy service:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig
```
```
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
```
```
kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
```
```
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

### Distribute the Kubernetes Configuration Files
Copy the appropriate kubelet and kube-proxy kubeconfig files to each worker instance:
```
for instance in $(awk '/\[worker\]/{flag=1;next}/^\[/{flag=0}flag' hosts)
do
  scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```

## 6 Generating the Data Encryption Config and Key
Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to encrypt cluster data at rest.

In this lab you will generate an encryption key and an encryption config suitable for encrypting Kubernetes Secrets.

### The Encryption Key
Generate an encryption key:

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File
Create the encryption-config.yaml encryption config file:

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```
Copy the encryption-config.yaml encryption config file to each controller instance:

```
for instance in $(awk '/\[controller\]/{flag=1;next}; /^\[/{flag=0}flag' hosts)
do
   scp encryption-config.yaml ${instance}:~/
done
```
...
