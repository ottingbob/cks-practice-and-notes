### Info

Network policies can allow ingress or egress traffic into / out of pods based on:
- pod selectors
- namespace selectors
- ip blocks

Inherently Network Policies deny any pod selections for a given policy type if no rules
are associated with the selection

Network Policy policy type rules will be OR'd together if there are multiple rules

If multiple Network Policies select the same pod, then the union of all NPs are applied.
Order does not affect the policy result.
When we say union, the results are again OR'd together.

### Example

```
# Get into the master node
$ gcloud compute ssh cks-master

# Create a frontend and backend pods
$ k run frontend --image nginx
$ k run backend --image nginx

# Expose both pods
$ k expose pod frontend --port 80
$ k expose pod backend --port 80

# Check connectivity from frontend -> backend
$ k exec frontend -- curl backend

# Check connectivity from backend -> frontend
$ k exec frontend -- curl backend
```

Now we create the default-deny policy like so:
```
# default-deny.yml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	namespace: default
	name: default-deny
spec:
	podSelector: {}
	policyTypes:
	- Ingress
	- Egress
```

Now we can apply the policy and have a timeout to see that it successfully
blocks the traffic from the pods:
```
# Apply the policy
$ k apply -f default-deny.yml

# Test the connection with a timeout
$ k exec frontend -- curl backend -m 10
$ k exec backend -- curl frontend -m 10
```

Create a policy to allow frontend -> backend traffic:
```
# frontend-to-backend-np.yml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	namespace: default
	name: frontend-to-backend-ingress
spec:
	podSelector:
		matchLabels:
			run: backend
	policyTypes:
	- Ingress
	ingress:
		- from:
			- podSelector:
					matchLabels:
						run: frontend
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	namespace: default
	name: frontend-to-backend-egress
spec:
	podSelector:
		matchLabels:
			run: frontend
	policyTypes:
	- Egress
	egress:
		- to:
			- podSelector:
					matchLabels:
						run: backend
```

This should work however the default deny policy still blocks DNS resolution
on the pods in the cluster.

To fix this we need to instead use the pod IP rather than using the service name.
```
$ k exec frontend -- curl $(k get pods backend -o wide --no-headers | awk '{ print $6 }') -m 10
```

To allow DNS resolution from the frontend / backend pods we could use the following
default deny policy:
```
# default-deny.yml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	namespace: default
	name: default-deny
spec:
	podSelector: {}
	policyTypes:
	- Ingress
	- Egress
	egress:
		- to:
		  ports:
				- protocol: TCP
					port: 53
				- protocol: UDP
					port: 53
```

We can now extend the example to have the backend pods allowed to talk to a database
server, in this instance cassandra

```
# Create the new namespace
$ k create ns cassandra

# Edit the namespace and add the label "ns: cassandra"
$ k edit ns cassandra

# Create the db pod (for simplicity it doesn't actually run cassandra)
$ k run -n cassandra cassandra --image=nginx
```

Enable the network policy for backend to reach cassandra pod:
```
# backend-to-cassandra-np.yml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	namespace: default
	name: backend-to-cassandra-egress
spec:
	podSelector:
		matchLabels:
			run: backend
	policyTypes:
	- Egress
	egress:
		- to:
			-	podSelector:
					matchLabels:
						run: cassandra
			-	namespaceSelector:
					matchLabels:
						ns: cassandra
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	namespace: cassandra
	name: backend-to-cassandra-ingress
spec:
	podSelector:
		matchLabels:
			run: cassandra
	policyTypes:
	- Ingress
	ingress:
		- from:
			-	podSelector:
					matchLabels:
						run: backend
			-	namespaceSelector:
					matchLabels:
						ns: default
```

This policy above will now allow for curling the cassandra pod either from the backend
pod with the respective label OR any pod in the default namespace
```
$ k exec backend -- curl $(k get pods -n cassandra -o wide cassandra --no-headers | awk '{print $6}') -m 10
```
