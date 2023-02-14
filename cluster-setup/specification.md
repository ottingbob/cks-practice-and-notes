
### Cluster specs

There will be a master node and a worker node with both:
- OS: Ubuntu 20.04 LTS
- Disk: 50GB
- CPU: 2
- RAM: 4GB

### GCP Setup

Enable the account with `yobcks@gmail.com`

Install gcloud
```
$ sudo apt-get install apt-transport-https ca-certificates gnupg

$ echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

$ curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -

$ sudo apt-get update && sudo apt-get install google-cloud-cli

$ gcloud init

$ gcloud auth login

$ gcloud projects list

$ gcloud config get project

$ gcloud compute instances list
```

Now create the two virtual machines to run the cluster
```
# Check out regions near you
https://cloud.google.com/compute/docs/regions-zones

# Create the master node
gcloud compute instances create cks-master --zone=us-central1-c \
	--machine-type=e2-medium \
	--image=ubuntu-2004-focal-v20220419 \
	--image-project=ubuntu-os-cloud \
	--boot-disk-size=50GB

# Create the worker node
gcloud compute instances create cks-worker --zone=us-central1-c \
	--machine-type=e2-medium \
	--image=ubuntu-2004-focal-v20220419 \
	--image-project=ubuntu-os-cloud \
	--boot-disk-size=50GB

# Install CKS master packages
gcloud compute ssh cks-master
sudo -i
bash <(curl -s https://raw.githubusercontent.com/killer-sh/cks-course-environment/master/cluster-setup/latest/install_master.sh)

# Install CKS worker packages
gcloud compute ssh cks-worker
sudo -i
bash <(curl -s https://raw.githubusercontent.com/killer-sh/cks-course-environment/master/cluster-setup/latest/install_worker.sh)

# Add the worker node
...

# Make cluster accessible from the outside by opening ports 30_000 - 40_000
$ gcloud compute firewall-rules create nodeports --allow tcp:30000-40000

# Stop your instances to save your free credits
gcloud compute instances stop cks-master cks-worker

# Containerd commands
$ crictl ps
```
