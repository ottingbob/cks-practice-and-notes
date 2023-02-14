


### Helpful commands

Generate a certificate & key for securing an ingress:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout cert.key -out cert.crt -subj "/CN=world.universe.mine/O=world.universe.mine"
```

Generate a certificate key:
```bash
openssl genrsa -out my-cert.key 2048
```

Generate a certificate from said key:
```bash
openssl req -new -key my-cert.key -out my-cert.csr

# Make sure you include the Common Name with a realistic value:
# Common Name (e.g. server FQDN or YOUR name) []: user@internal.users
```

Manually sign a CSR with a CA file to generate a CRT:
```bash
openssl x509 -req -in my-cert.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out my-cert.crt -days 500
```

Now add the cert & key to the kube config in the form of a context:
```bash
k config set-credentials user@internal.users --client-key=my-cert.key --client-certificate=my-cert.crt
k config set-context user@internal.users --cluster=kubernetes --user=user@internal.users
k config use-context user@internal.users
```
