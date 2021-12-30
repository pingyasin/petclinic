


•	Created an IAM Role with name of rke-role to attach RKE nodes (instances) using rke-controlplane-policy and rke-etcd-worker-policy.
•	Created a security group for External Application Load Balancer of Rancher with name of rke-alb-sg and allow HTTP (Port 80) and HTTPS (Port 443) connections from anywhere.
•	Created a security group for RKE Kubernetes Cluster with name of rke-cluster-sg and defined following inbound and outbound rules.
    •	Inbound rules;
    •	Allow HTTP protocol (TCP on port 80) from Application Load Balancer.
    •	Allow HTTPS protocol (TCP on port 443) from any source that needs to use Rancher UI or API.
    •	Allow TCP on port 6443 from any source that needs to use Kubernetes API server(ex. Jenkins Server).
    •	Allow SSH on port 22 to any node IP that installs Docker (ex. Jenkins Server).
    •	Outbound rules;
    •	Allow SSH protocol (TCP on port 22) to any node IP from a node created using Node Driver.
    •	Allow HTTP protocol (TCP on port 80) to all IP for getting updates.
    •	Allow HTTPS protocol (TCP on port 443) to 35.160.43.145/32, 35.167.242.46/32, 52.33.59.17/32 for catalogs of git.rancher.io.
    •	Allow TCP on port 2376 to any node IP from a node created using Node Driver for Docker machine TLS port.
    •	Allow all protocol on all port from rke-cluster-sg for self communication between Rancher controlplane, etcd, worker nodes.

scp -i sshkey.pem rancher.key ec2-user@3.142.248.170:/home/ec2-user/.ssh


•	Logged into Jenkins Server and created rancher.key key-pair for Rancher Server using AWS CLI:
aws ec2 create-key-pair --region us-east-2 --key-name call-rancher.key --query KeyMaterial --output text > ~/.ssh/rancher.key
chmod 400 ~/.ssh/rancher.key

•	Launched an EC2 instance using Ubuntu Server 20.04 LTS (HVM) (64-bit x86) with t3a.medium type, 16 GB root volume, rke-cluster-sg security group, rke-role IAM Role, Name:Rancher-Cluster-Instance tag and rancher.key key-pair. Take note of subnet id of EC2.
subnet-dc1688b7 | Default in us-east-2a
vpc-5e80f635 (default)

•	Located the initial administrative password.
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

17df358fe8274c599d77c62bf12a1e2b

•	Entered the temporary password to unlock the Jenkins.
3.142.248.170:8080

•	Installed suggested plugins.
•	Created first admin user.
jenkins-admin
J3nkins
http://3.142.248.170:8080/ -- temporary

•	Open your Jenkins dashboard and navigate to Manage Jenkins >> Manage Plugins >> Available tab
•	Located GitHub Integration, Docker Plugin, Docker Pipeline, and Jacoco plugins, then Installed without restart.
•	Configured Docker as cloud agent by navigating to Manage Jenkins >> Manage Nodes and Clouds >> Configure Clouds and using tcp://localhost:2375 as Docker Host URI. (not really needed this) 

•	Attachd a tag to the nodes (intances), subnets and security group for Rancher with Key = kubernetes.io/cluster/Rancher and Value = owned.

ssh -i "rancher.key" ubuntu@ec2-18-222-20-209.us-east-2.compute.amazonaws.com

•	Log into Rancher-Cluster-Instance from Jenkins Server (Bastion host) and install Docker using the following script.
# Set hostname of instance
sudo hostnamectl set-hostname rancher-instance-1
# Updated OS 
sudo apt-get update -y
sudo apt-get upgrade -y
# Installed and start Docker on Ubuntu 19.03
# Updated the apt package index and install packages to allow apt to use a repository over HTTPS
sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
# Added Docker's official GPG key -- Rancher doest work with the latest docker, so I used version 19 or younger
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
# Used the following command to set up the stable repository
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# Update packages
sudo apt-get update
# List the versions available in your repo
apt-cache madison docker-ce

# Since Rancher is not compatible (yet) with latest version of Docker install version 19.03.15 or earlier version using the version string (exp: 5:19.03.15~3-0~ubuntu-focal) from the second column
sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
sudo systemctl start docker
sudo systemctl enable docker

# Add ubuntu user to docker group
sudo usermod -aG docker ubuntu
newgrp docker

•	Create a target groups with name of rancher-http-80-tg with following setup and add the rancher instances to it.
Target type         : instance
Protocol            : HTTP
Port                : 80

<!-- Health Checks Settings -->
Protocol            : HTTP
Path                : /healthz
Port                : traffic port
Healthy threshold   : 3
Unhealthy threshold : 3
Timeout             : 5 seconds
Interval            : 10 seconds
Success             : 200
•	Create Application Load Balancer with name of rancher-alb using rke-alb-sg security group with following settings and add rancher-http-80-tg target group to it.
Scheme              : internet-facing
IP address type     : ipv4

<!-- Listeners-->
Protocol            : HTTPS/HTTP
Port                : 443/80
Availability Zones  : Select AZs of RKE instances
Target group        : `rancher-http-80-tg` target group 
•	Configure ALB Listener of HTTP on Port 80 to redirect traffic to HTTPS on Port 443.
•	Create DNS A record for rancher.pingify.us (burada kendi dns isminizi kullanacaksınız) and attach the rancher-alb application load balancer to it.
•	Install RKE, the Rancher Kubernetes Engine, Kubernetes distribution and command-line tool) on Jenkins Server.
curl -SsL "https://github.com/rancher/rke/releases/download/v1.1.12/rke_linux-amd64" -o "rke_linux-amd64"
sudo mv rke_linux-amd64 /usr/local/bin/rke
chmod +x /usr/local/bin/rke
rke --version
