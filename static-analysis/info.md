### Info

Static Analysis of User Workloads

Information covered will be:
- Static Analysis: what & where it is used
- Manual approach of performing static analysis
- Tools for k8s and scenarios

Static analysis looks at source code and text files, checks & enforces against rules.

Some examples of static analysis rules:
- Always define resource requests and limits
- Pods should never use the default service account

Rules will depend on the use case / company / or project

Generally don't store sensitive data in plain k8s / docker files

We can run static analysis in CI/CD around a build or test phase.

We could also use Pod Security Policies / Open Policy Agent to run some static analysis
tools to enforce security after pipeline deployment.

Kubesec - kubesec.io

The Kubesec tool offers security risk analysis for k8s resources.

Opensource & opinionated with a fixed set of rules

It can run in the following forms:
- binary
- docker container
- kubectl plugin
- admission controller (kubesec-webhook)

OPA Conftest

We already used OPA Gatekeeper

Conftest is a binary that provides unit test framework for k8s configs

It uses the same `rego` language as OPA Gatekeeper
```go
package main

deny[msg] {
	input.kind = "Deployment"
	not input.spec.template.spec.securityContext.runAsNonRoot = true
	msg = "Containers must not run as root"
}
```

### Example 1

Using `kubesec` to perform static analysis

We will use the `kubesec` public docker image to run against pods on our cluster.

We start by creating a simple nginx pod to test against:
```bash
$ k run nginx --image nginx -o yaml --dry-run=client > nginx.yml

# Now we run the kubesec docker container against the generated yaml file:
$ docker run -i kubesec/kubesec:v2 scan /dev/stdin < nginx.yml
[
  {
	    "object": "Pod/nginx.default",
			"valid": true,
			"fileName": "STDIN",
			"message": "Passed with a score of 0 points",
			"score": 0,
			"scoring": {...}
	}
]
```

Now we can change our pod based on some of the scoring that it recommends:
```bash
$ vim nginx.yml
containers:
	- image: nginx
		securityContext:
			runAsNonRoot: true

# Now run the command again:
$ docker run -i kubesec/kubesec:v2 scan /dev/stdin < ./nginx.yml
[
  {
	    "object": "Pod/nginx.default",
			"valid": true,
			"fileName": "STDIN",
			"message": "Passed with a score of 1 points",
			"score": 1,
			"scoring": {
					"passed": [
						{
							"id": "RunAsNonRoot",
							"selector": "containers[] .securityContext .runAsNonRoot == true",
							"reason": "Force the running image to run as a non-root user to ensure least privilege",
							"points": 1
						}
					],
					"advise": [...]
				}
			]
		}
	}
]
```

### Example 2

We will now use conftest to check a k8s example

We will pull down the course example using a sparse checkout to get the correct setup.
```bash
git clone --filter=blob:none --sparse https://github.com/killer-sh/cks-course-environment.git
cd cks-course-environment
git sparse-checkout add course-content/supply-chain-security/static-analysis/conftest

cd ./cks-course-environment/course-content/supply-chain-security/static-analysis/conftest/kubernetes
```

Then we can run the examples on the files provided:
```bash
$ ./run.sh
2 tests, 2 passed, 0 warnings, 0 failures, 0 exceptions
```
