### Info

Nginx Ingress will automatically generate Nginx configs based on the ingress YAML
you create so it takes a lot of simplicity out of exposing services through k8s.

For some background a ClusterIP service will always point to a Pod, not a deployment
or a daemon-set VIA a label.
ClusterIP means the service has an internal IP and is reachable by the given IP and
the DNS name.

NodePort service operates like a ClusterIP service in that connects to a pod via a
label with the added feature that it opens a port on EVERY node and requests coming
into that port will be forwarded to the respective pod
A NodePort service makes a ClusterIP service reachable from the outside.

A LoadBalancer service again is another level up where it creates a NodePort service
which would encapsulate a ClusterIP service but it is able to create that service
across every node in the cluster using a cloud providers load balancer

An ExternalName service maps the service to the contents of the `externalName` field
by returning a `CNAME` record with its value. There is NO proxying done by this type
of service.

> Important Note:
> Make sure you REMOVE any default-deny policy that you have setup previously
> otherwise it might make some of the working examples have issues.

### Examples

First install the nginx ingress:
```
# Install Nginx Ingress
$ k apply -f https://raw.githubusercontent.com/killer-sh/cks-course-environment/master/course-content/cluster-setup/secure-ingress/nginx-ingress-controller.yaml
```

This will create the new namespace `ingress-nginx` which has a service listed for us:
```
# Find the node port service
$ k get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.111.10.104   <none>        80:31330/TCP,443:31717/TCP   77s
ingress-nginx-controller-admission   ClusterIP   10.101.245.1    <none>        443/TCP                      77s
```

Then we can curl the external IP of the worker node by using the node port combo:
```
$ curl http://$(gcloud compute instances list --filter="name='cks-worker'" | grep 'cks-worker' | awk '{print $5}'):31330
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

Now we can create an ingress resource to expose a service / pod:
```
# ingress.yml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: my-ingress
	annotations:
		nginx.ingress.kubernetes.io/rewrite-target: /
spec:
	ingressClassName: nginx
	rules:
		- http:
				paths:
					-	path: /service1
						pathType: Prefix
						backend:
							service:
								name: service1
								port:
									number: 80
					-	path: /service2
						pathType: Prefix
						backend:
							service:
								name: service2
								port:
									number: 80

# Verify the ingress is created properly
$ k apply -f ingress.yml
ingress.networking.k8s.io/my-ingress created

$ k get ing
NAME         CLASS   HOSTS   ADDRESS      PORTS   AGE
my-ingress   nginx   *       10.128.0.3   80      76s
```

Now we need to create the pods / service combos:
```
# Create pod / service1
$ k run pod1 --image=nginx
$ k expose pod pod1 --name service1 --port 80

# Create pod / service2
$ k run pod2 --image=httpd
$ k expose pod pod2 --name service2 --port 80
```

Now we should get results on both services:
```
# Hit Nginx / service1
$ curl http://$(gcloud compute instances list --filter="name='cks-worker'" | grep 'cks-worker' | awk '{print $5}'):31330/service1 | tail -n 3
<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# Hit httpd / service2
$ curl http://$(gcloud compute instances list --filter="name='cks-worker'" | grep 'cks-worker' | awk '{print $5}'):31330/service2 
<html><body><h1>It works!</h1></body></html>
```

We can also hit the HTTPS endpoint too:
```
# Hit Nginx / service1
$ curl -k https://$(gcloud compute instances list --filter="name='cks-worker'" | grep 'cks-worker' | awk '{print $5}'):31717/service1 | tail -n3
<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# Hit httpd / service2
$ curl -k https://$(gcloud compute instances list --filter="name='cks-worker'" | grep 'cks-worker' | awk '{print $5}'):31717/service2
<html><body><h1>It works!</h1></body></html>
```

We can additionally change this to use our own certificate instead of the self-signed fake
certificate that is created automatically by ingress-nginx-controller

We create another self-signed certificate to test with.

```
# Create the self-signed certificate
$ openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# The only thing we need to enter is the CN / Common Name which we will call:
# `secure-ingress.com`

# Now we should have our cert / key combinations:
$ ls | grep '.pem'
cert.pem
key.pem

# Now create the secret with the cert / key combo:
$ k create secret tls secure-ingress --cert cert.pem --key key.pem

# Now edit the ingress to use the TLS secret
$ k edit ing my-ingress
spec:
  tls:
	- hosts:
		- secure-ingress.com
		secretName: secure-ingress
	rules:
	- host: secure-ingress.com
```

Now we should be able to curl and see the resolution come through as the self-signed
cert that we created:
```
$ curl -kv https://secure-ingress.com:31717/service2 --resolve secure-ingress.com:31717:$(gcloud compute instances list --filter="name='cks-worker'" | grep 'cks-worker' | awk '{print $5}')
```



