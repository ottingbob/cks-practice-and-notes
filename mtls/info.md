### Info

Going to talk about the following subjects:
- Using mTLS
- mTLS and Pod to Pod communication
- Service Meshes

mTLS is mutual authentication or two-way authentication.

Two parties authenticate with each other at the same time.

In k8s every pod can communicate with every other pod by default unencrypted.

When communicating with pods externally over an ingress most-likely the request will
use HTTPS and the request will have TLS termination and the traffic is decrypted
and passed inside the cluster.

With mTLS between pods we would need to have each pod have a client cert which
communicates with another pod that has a server cert and both certs would need to
be signed by a common CA.

In addition we would ideally want to rotate the certificates which adds another layer
of complexity.

Many service meshes accomplish this cert generation / rotation for mTLS by using a
sidecar proxy container.

The proxy sidecar containers are responsible for sending the traffic as encrypted and
receiving the encrypted traffic, decrypting it, and forwarding it to the sidecar application
container on the receiving end.

Sidecar proxy containers are injected and managed externally by a manager / operator that
handles this integration for us. This includes creating the CA cert, rotating certs, applying
them, and intercepting traffic from application containers. The operator / manager is usually
considered a type of service mesh.

The proxy works under the hood by creating `iptable` rules to route traffic via the proxy
such as using an `initContainer` using the `NET_ADMIN` capability.


### Example

Creating a proxy sidecar with the `NET_ADMIN` capability

```bash
# First ssh to the instance
$ gcloud compute ssh cks-master

# Switch to a root shell
$ sudo su -

# Create a ping pod
$ k run app --image=bash --command -o=yaml --dry-run=client > app.yaml -- sh -c 'ping google.com'

# Now create the proxy sidecar for the application by editing the `app.yaml` file
$ vim app.yaml
```

```yaml
# Add another container:
containers:
	- name: proxy
		image: ubuntu
		command:
		- sh
		- -c
		# install IP tables and add an entry
		- 'apt update && apt install -y iptables && iptables -L && sleep 1d'
		securityContext:
			capabilities:
				add:
					- NET_ADMIN
	- name: app
	# ... Rest of already defined container
```

```bash
# Now we can apply and check the logs of the container
$ k apply -f app.yaml

$ k logs -f app -c proxy

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

