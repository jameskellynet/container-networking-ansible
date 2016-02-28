# container-networking-ansible
Ansible provisioning for container networking solutions using OpenContrail

This repository contains provisioning instructions to install OpenContrail
as a network overlay for container based cluster management solutions. 

Forked from https://github.com/Juniper/container-networking-ansible which was built for testing, this repo is my copy for setting a small cloud lab and demo environment. I have made the changes necessary to install OpenShift Origin with OpenContrail on EC2 and removed the Jenkins testing. As time allows I will update this readme with instructions to deploy this for yourself.

The opencontrail playbook consists of the following:
  - filter_plugins/ip_filters.py
  - roles/opencontrail{,_facts,_provision}

The playbooks are designed to be addons to the existing ansible provisioning for kubernetes and openshift.
<!---
### Kubernetes

#### Network segmentation and access control
When opencontrail is used as the kubernetes network plugin, it defaults to isolate all pods according to `namespace` and a user defined tag. External traffic is restricted to services that are annotated with a ExternalIP address or have "type" set to "LoadBalancer". This causes the opencontrail public to allocate an address on the public network and assign it to all the pods in this service.

Services in the `kube-system` namespace are also available to all Pods, irrespective of the namespace of the pod. This is configured via the `cluster-service` option in /etc/kubernetes/network.conf. The cluster-service network is also connected to the underlay network where masters and nodes are present.

Pods are expected to communicate with the master via its ClusterIP address.

#### Deployment
The kubernetes ansible playbook at https://github.com/kubernetes/contrib.

- edit ansible/group_vars/all.yml
```
networking: opencontrail
```

- inventory file:
```
[opencontrail:children]
masters
nodes
gateways

[opencontrail:vars]
opencontrail_public_subnet=192.0.2.0/24
opencontrail_kube_release=1.1

```

- patch ansible/cluster.yml according to:
https://github.com/kubernetes/contrib/pull/261

- run the ansible/cluster.yml playbook (e.g. via ansible/setup.sh)

-->

### OpenShift

#### Network segmentation and access control

There are several differences in design from a plain-vanilla kubernetes cluster deployment and an openshift deployment:
- OpenShift expects all external traffic to be delivered through the router service. The openshift router pod is a TCP load-balancer (ha-proxy by default) that performs SSL termination and delivers traffic to the pods that implement the service.
- OpenShift pods (builder/deployer) have the nasty habbit of trying to reach the master through its infrastructure IP address (rather than using the ClusterIP).
- OpenShift STI builder pods expect to be able to access external git repositories as well as package repositories for popular languages (python, ruby, etc...).
- OpenShift builder pods use the docker daemon in the node and expect it to be able to talk to the docker-repository service running as a pod (in the overlay).
- Deployer pods expect to be able to pull images from the docker-repository into the node docker daemon.

* In current test scripts, we expect the builder pods to use an http proxy in order to fetch software packages. The builder pods are spawned in the namespace of the user `project`. To provide direct external access, one would need to do so for all pods currently. Future versions of the contrail-kubernetes plugin should support source-nat for outbound access to the public network. It is also possible to add a set of prefixes that contain the software and artifact repositories used by the builder to the global `cluster-service` network.
* All the traffic between underlay and overlay is expected to occur based on the `cluster-service` gateway configured for ```default:default```

#### Deployment Steps

- I recommend creating a t2.micro node running Linux CentOS 7 on AWS as your "starting-point" node. This way you'll be subscribed to CentOS 7 and find out your region, EC2 image ID which you'll need anyway below. Download and copy your AWS keys to this node's ~/.ssh/ directory and install the AWS CLI. Fill in the <_blanks_>... 

```
sudo yum install -y wget
wget <_keys URL for your key.pem_>
mv <_keys file_> .ssh/<_key_>.pem
eval $(ssh-agent)
ssh-add .ssh/<_key_>.pem
export AWS_ACCESS_KEY_ID=<_get this from AWS IAM_>
export AWS_SECRET_ACCESS_KEY=<_get this from AWS IAM_>
curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
sudo yum install unzip
unzip awscli-bundle.zip
sudo ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws
```

- Try the AWS cli to check it works. There's all kinds of useful commands you'll find in its help.
```
aws ec2 describe-instances
 ```
 
- In the output from the last command, you'll see your region in the domain names and you'll see ami-d440a6e7 or something like that which is your image ID that you'll need below.
- Now let's install some prerequisite tools ansible python-boto and pyOpenSSL. To yum these you'll have to download and install EPEL. Also check you're using Ansible 1.94, not 2+  
```
wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo rpm -ivh epel-release-latest-7.noarch.rpm
sudo yum install -y ansible python-boto pyOpenSSL
ansible --version
```

- Now git clone this repo into a new src directory

```
mkdir ~/src
cd ~/src
git clone https://github.com/jameskellynet/container-networking-ansible.git
```

- Edit the file container-networking-ansible/test/ec2-origin/group_vars/all.yml which now contains

```
aws_region: us-west-2
ec2_image: ami-d440a6e7
ssh_user: centos
aws_vpc_name: openshift-demo-vpc
aws_public_network: public
aws_private_network: private
aws_keys: jamesk-keys
```

- In your edits, setup your AWS region (in case it's different), AWS VPC name, and the name of the AWS keys file (without the .pem extension ). I recommend creating your 2 AWS networks and calling them public and private, but if not, change the variables here too. If you are using a different AWS region than us-west-2, the EC2 image variable here might change. Use the appropriate image ID for a CentOS 7 machine with kernel version matching 3.10:
```
$ uname -r
3.10.0-229.14.1.el7.x86_64
```

- Finally let's kick off our Ansible playbook and get going. This sets up the VMs we will need. A deployer and gateway node in the public subnet, and a master, node1 and node2 in the private subnet.
```
cd container-networking-ansible/test/ec2-origin/
ansible-playbook -i localhost playbook.yml --tags=cluster
```
- That should have generated cluster.status, an Ansible inventory file. Check it out. Assuming it all worked, move on to setup the nodes in your new cluster.
```
ansible-playbook -i cluster.status playbook.yml --tags=deployer-install,workspace
```
- Open your cluster.status file and grab the name of the newly created and setup deployer node (see it under [deployer]) and ssh to it. Also node the IP of the master node for the next command (see it under [masters].
```
ssh <_something_>.compute.amazonaws.com -o ForwardAgent=yes
```
- You should now be on the deployer node. Test that from here, you can ssh into the master node in the other subnet using it's private IP.
```
ssh 10.<_something_>
```
-  If you can get into the master node, you're good. Otherwise, TODO. Close the connect to the master node, so you're back on the deployer.
-  From the deployer, we're going to kick off a few playbooks manually that are performed by Jenkins in the automated Juniper tests that I removed. It's normal to see an error in the command below validating stage 2 (run it nevertheless), which is why there's a workaround playbook run after that. 
```
cd src/openshift-ansible/
ansible-playbook -i inventory/byo/hosts playbooks/byo/system-install.yml
ansible-playbook -i inventory/byo/hosts playbooks/byo/opencontrail.yml
python playbooks/byo/opencontrail_validate.py --stage 1 inventory/byo/hosts
ansible-playbook -i inventory/byo/hosts playbooks/byo/config.yml
python playbooks/byo/opencontrail_validate.py --stage 2 inventory/byo/hosts
ansible-playbook -i inventory/byo/hosts playbooks/byo/systemd_workaround.yml
python playbooks/byo/opencontrail_validate.py --stage 2 inventory/byo/hosts
ansible-playbook -i inventory/byo/hosts playbooks/byo/opencontrail_provision.yml
python playbooks/byo/opencontrail_validate.py --stage 3 inventory/byo/hosts
ansible-playbook -i inventory/byo/hosts playbooks/byo/openshift_provision.yml
python playbooks/byo/opencontrail_validate.py --stage 4 inventory/byo/hosts
ansible-playbook -i inventory/byo/hosts playbooks/byo/applications.yml
python playbooks/byo/opencontrail_validate.py --stage 5 inventory/byo/hosts

```




