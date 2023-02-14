### Info

The Open Policy Agent allows us to write custom policies in k8s for all our needs!

What we will be covering:
- What is the OPA and Gatekeeper
- Enforce Labels
- Enforce Pod Replica Counts

> Recap:
> 
> The request workflow either initiated by a human or pod hitting the kube-apiserver we
> take the following steps:
>
> 1) Authentication - who is the user
> 2) Authorization - what is the user allowed to do
> 3) Admission Control - are there any controls around this action such as pod limits etc.
> 4) Perform action / request

The OPA lives around action number 3 and acts as an admission controller.

The OPA is an open source, general-purpose policy engine that enables unified, context-aware
policy enforcement across the entire stack.

- This means that the OPA is _NOT_ k8s specific
- Allows for easy implementation of policies using the `Rego` language
- Works with JSON / YAML
- In k8s it uses Admission Controllers
- Does not know concepts like pods or deployments

The OPA Gatekeeper uses the OPA to make it easier to use / integrate with k8s.
It does so by creating k8s CRDs for OPA.

OPA Gatekeeper works with the concept of a `Constraint Template` and a `Constraint`:
```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
	name: k8srequiredlabels

---
apiVersion: constraints.gatekeeper.sh/v1
kind: K8sRequiredLabels
metadata:
	name: pod-must-have-gk
```

The `Constraint Template` ends up creating a _NEW_ CRD of the `Constraint` that we
can use as an implementation of the template.

This could be something along the lines of now a pod must have a label of a certain type.

### Example 1

Install the OPA Gatekeeper

First check if there are no more admission plugins enabled:
```bash
$ vim /etc/kubernetes/manifests/kube-apiserver.yaml

# Comment out the line related to `enable-admission-plugins`
```

Next install the gatekeeper:
```bash
$ k create -f https://raw.githubusercontent.com/killer-sh/cks-course-environment/master/course-content/opa/gatekeeper.yaml

# And check out the installed components
$ k get all -n gatekeeper-system
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/gatekeeper-audit-649cbfdc8f-45cs9                1/1     Running   0          72s
pod/gatekeeper-controller-manager-57575f6fc7-g5hlv   1/1     Running   0          72s
pod/gatekeeper-controller-manager-57575f6fc7-khktk   1/1     Running   0          72s
pod/gatekeeper-controller-manager-57575f6fc7-v4lr8   1/1     Running   0          72s

NAME                                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/gatekeeper-webhook-service   ClusterIP   10.96.69.77   <none>        443/TCP   73s

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gatekeeper-audit                1/1     1            1           73s
deployment.apps/gatekeeper-controller-manager   3/3     3            3           73s

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/gatekeeper-audit-649cbfdc8f                1         1         1       72s
replicaset.apps/gatekeeper-controller-manager-57575f6fc7   3         3         3       72s

# We can also see the related webhook:
$ k get validatingwebhookconfigurations.admissionregistration.k8s.io
NAME                                          WEBHOOKS   AGE
gatekeeper-validating-webhook-configuration   2          2m47s
```

### Example 2

Create a Deny-All / Approve-All policies

First take a look at the related CRDs we have related to the OPA Gatekeeper:
```bash
$ k get crd | grep gate
configs.config.gatekeeper.sh                          2023-02-01T18:11:31Z
constraintpodstatuses.status.gatekeeper.sh            2023-02-01T18:11:31Z
constrainttemplatepodstatuses.status.gatekeeper.sh    2023-02-01T18:11:31Z
constrainttemplates.templates.gatekeeper.sh           2023-02-01T18:11:31Z
```

Now lets create a template and constraints that apply to said template.

Here is an example to the always deny template.
To throw a violation, every condition must be TRUE, in this case 1 > 0.
```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
	name: k8salwaysdeny
spec:
	crd:
		spec:
			names:
				kind: K8sAlwaysDeny
			validation:
				# Schema for the `parameters` field
				openAPIV3Schema:
					properties:
						message:
							type: string
	targets:
		- target: admission.k8s.gatekeeper.sh
			rego: |
				package k8salwaysdeny

				violation[{"msg": msg}] {
					1 > 0
					msg := input.parameters.message
				}
```

Here is an example to the always deny constraint:
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAlwaysDeny
metadata:
	name: pod-always-deny
spec:
	match:
		kinds:
			- apiGroups: [""]
				kinds: ["Pod"]
	parameters:
		message: "ACCESS DENIED"
```

This template should always throw the violation on a pod so we try it out
by creating a pod:
```bash
$ k run pod --image=nginx
Error from server ([pod-always-deny] ACCESS DENIED): admission webhook "validation.gatekeeper.sh" denied the request: [pod-always-deny] ACCESS DENIED

# Now we can change the message and see what happens:
$ vim always-deny-constraint.yaml

...
	parameters:
		message: "GET REKT!"

# Apply the constraint again:
$ k apply -f always-deny-constraint.yaml

# Create the pod again:
$ k run pod --image=nginx
Error from server ([pod-always-deny] GET REKT!): admission webhook "validation.gatekeeper.sh" denied the request: [pod-always-deny] GET REKT!
```

Now we can create the approve-all policy:
```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
	name: k8salwaysdeny
spec:
	crd:
		spec:
			names:
				kind: K8sAlwaysDeny
			validation:
				# Schema for the `parameters` field
				openAPIV3Schema:
					properties:
						message:
							type: string
	targets:
		- target: admission.k8s.gatekeeper.sh
			rego: |
				package k8salwaysdeny

				violation[{"msg": msg}] {
					# This will always be true
					1 > 0
					# This will always be false
					1 > 2
					msg := input.parameters.message
				}
```

Now once we apply it we can safely create pods again:
```bash
$ k apply -f always-deny-constraint.yaml
constrainttemplate.templates.gatekeeper.sh/k8salwaysdeny configured

$ k run pod --image nginx
pod/pod created

$ k get pods pod
NAME       READY   STATUS    RESTARTS        AGE
pod        1/1     Running   0               35s
```

### Example 3

All namespaces created need to have the label `cks`.

To start remove the examples from the other section.

Here is the template we will create:
```yaml req-labels-template.yml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
	name: k8srequiredlabels
spec:
	crd:
		spec:
			names:
				kind: K8sRequiredLabels
			validation:
				# Schema for the `parameters` field
				openAPIV3Schema:
					properties:
						labels:
							type: array
							items: string
	targets:
		- target: admission.k8s.gatekeeper.sh
			rego: |
				package k8srequiredlabels

				violation[{"msg": msg, "details": {"missing_labels": missing}}] {
					# We look at the JSON of the created k8s resource
					# input.review.object == k8s resource
					provided := {label | input.review.object.metadata.labels[label]}
					# input.parameters.labels are the labels defined in the constraint
					# the `_` character means grab all the labels defined in the parameter
					# of type array
					required := {label | label := input.parameters.labels[_]}
					missing := required - provided
					count(missing) > 0

					msg := sprintf("you must provide labels: %v", [missing])
				}
```

And here are the related constraints
```yaml req-labels-pod-constraint.yml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
	name: pod-must-have-cks
spec:
	match:
		kinds:
			- apiGroups: [""]
				kinds: ["Pod"]
	parameters:
		labels: ["cks"]
```

```yaml req-labels-ns-constraint.yml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
	name: ns-must-have-cks
spec:
	match:
		kinds:
			- apiGroups: [""]
				kinds: ["Namespace"]
	parameters:
		labels: ["cks"]
```

Once we apply them we can attempt to create a NS:
```bash
$ k create ns denied
Error from server ([ns-must-have-cks] you must provide labels: {"cks"}): admission webhook "validation.gatekeeper.sh" denied the request: [ns-must-have-cks] you must provide labels: {"cks"}
```

When we edit one to have the required labels we will have more luck:
```bash
$ k create ns allowed --dry-run=client -o yaml > allowed-ns.yml

$ vim allowed-ns.yml
metadata:
	labels:
		cks: "gotem"

$ k apply -f allowed-ns.yml
namespace/allowed created
```

### Example 4

All deployments created need to have minimum replica count.

Again we delete the previous template / constraint.

Here is the template we will create:
```yaml req-replica-count-template.yml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
	name: k8sminreplicacount
spec:
	crd:
		spec:
			names:
				kind: K8sMinReplicaCount
			validation:
				# Schema for the `parameters` field
				openAPIV3Schema:
					properties:
						min:
							type: integer
	targets:
		- target: admission.k8s.gatekeeper.sh
			rego: |
				package k8sminreplicacount

				violation[{"msg": msg, "details": {"missing_replicas": missing}}] {
					provided := input.review.object.spec.replicas
					required := input.parameters.min
					missing := required - provided
					missing > 0

					msg := sprintf("you must provide %v more replicas", [missing])
				}
```

```yaml req-deploy-count-constraint.yml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sMinReplicaCount
metadata:
	name: deploy-must-have-min-replicas
spec:
	match:
		kinds:
			- apiGroups: ["apps"]
				kinds: ["Deployment"]
	parameters:
		min: 2
```

After applying both we can test it out:
```bash
$ k create deploy test --replicas 1 --image nginx
error: failed to create deployment: admission webhook "validation.gatekeeper.sh" denied the request: [deploy-must-have-min-replicas] you must provide 1 more replicas

$ k create deploy working --replicas 2 --image nginx
deployment.apps/working created
```

### Example 5

The Rego Playground allows us to test out policies in a sandbox environment.

This repo has examples such as not creating resources in the default namespace:
github.com/BouweCeunen/gatekeeper-policies

The Rego Playground can be found at:
play.openpolicyagent.org
