### Info


### Examples

TODO: Include steps on how to get the certificate from the CertificateSigningRequest
& the related k8s resource. This can be acheived through a jq command to pull out
the related data and pipe it into a file such as `jane.crt`

In order to create a new entry in the kube config file do as follows:
```bash
$ k config set-credentials jane --client-key=jane.key --client-certificate=jane.crt

# Optionally we can choose to embed the cert data into the kube config with
# the following command:
$ k config set-credentials jane --client-key=jane.key --client-certificate=jane.crt --embed-certs

# Then we can create the the related context to test out running commands as the
# newly created user
$ k config set-context jane --user=jane --cluster=kubernetes

# Now we switch to the newly created context
$ k config use-context jane
```

