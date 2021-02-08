# JINX

JINX is an end-to-end automated deployment of OpenShift 4.x UPI using Ansible. JINX will create a bastion to be used to interface with cluster resources, a registry to serve cluster resources in a restricted fastion, all the other resources required to create an OpenShift 4.x cluster deployment, and **MOST IMPORTANTLY** the OpenShift 4.x cluster itself!

## User Guide

These instructions will get you a copy of the project up and running on
your local machine for development and testing purposes.

### Prerequisites

#### Configure the epel.repo
sudo cat << EOF >> /etc/yum.repos.d/epel.repo
[epel]
name= epel
baseurl=https://mirrors.sonic.net/epel/8/Everything/x86_64/
gpgcheck=0
enabled=1

#### Install Python
Ensure Python (supported version of 2.7.x or 3.x) is installed.

#### Install unzip
```
sudo dnf install unzip -y 
```
#### Install git
```
sudo dnf install git -y 
```
#### Clone the project 
```
git clone https://gitlab.consulting.redhat.com/navy-black-pearl/jinx
cd jinx 
```
#### Install Ansible >= 2.9.10
##### RHEL, CentOS, or Fedora
*On Fedora:*
```
sudo dnf install ansible -y
```
*On RHEL and CentOS:*
```
sudo yum install ansible -y
```
##### Configure ansible 
```
ansible-galaxy collection install amazon.aws community.aws community.crypto
```
##### MacOS
*On MacOS:*
[Installation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-macos)

#### Install AWS CLI >= 1.18.198
##### AWS CLI version 1
[Installation](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv1.html)

**NOTE:** AWS CLI version 1.18.198 is the only version of the AWS CLI that has been tested

##### AWS CLI version 2
[Installation](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

#### Login to AWS via CLI
```
aws configure
AWS Access Key ID [****************YYYY]: <aws-access-key-id>
AWS Secret Access Key [****************ZZZZ]: <aws-secret-access-key>
Default region name [us-gov-west-1]: us-gov-west-1
Default output format [None]:
```

#### Install Required Libraries
##### pyOpenSSL
```
pip3 install pyOpenSSL --user
```

##### boto
```
pip3 install boto --user
```

##### boto3 and botocore
```
pip3 install botocore boto3 --user
```

### Deploy Cluster with Automation

1. Edit the file located at `inventory/group_vars/all/vars.yml` to include the necessary values such as the following:

| Variable             | Use                                                         | Type   | Example Value(s) |
|----------------------|-------------------------------------------------------------|--------|------------------|
| `openshift_version`  | Version of OpenShift to deploy                              | String | 4.6.15           |
| `aws_ssh_key`        | Preferred name for AWS SSH KeyPair                          | String | rh-dev-blackpearl-us |
| `cluster_name`       | Preferred name of OpenShift cluster (and existing VPC name) | String | vpc01            |
| `cluster_domain`     | Subdomain for cluster to be hosted on                       | String | rh.dev.blackpearl.us |
| `cluster_subnet_ids` | Three subnets for cluster resources to be deployed on       | Array  | - subnet-xxxxxxxxxxxxxxxxx<br>-  subnet-xxxxxxxxxxxxxxxxx<br>-  subnet-xxxxxxxxxxxxxxxxx  |
| `bastion_subnet_id`  | Subnet ID for bastion to be deployed to                     | String | subnet-xxxxxxxxxxxxxxxxx  |
| `vpc_cidr_block`     | VPC CIDR block for the cluster                              | String | 10.126.126.0/23  |
| `vpc_id`             | AWS ID for VPC for cluster                                  | String | vpc-xxxxxxxxxxxxxxxxx  |

2. Create an Ansible vault file at `inventory/group_vars/all/vault.yml` as follows:
  1. Create file:
  ```
  ansible-vault create inventory/group_vars/all/vault.yml
  ```
  <br>**NOTE**: Use the `EDITOR` environment variable to use a different CLI editor than the default, for example:
  ```
  EDITOR=vim ansible-vault create inventory/group_vars/all/vault.yml
  ```

  2. Once prompted, enter your desired password for the vault file and confirm it

  3. Enter the following required variables and close the file: (using variable: value format)
  
  | Variable                | Use                                                         | Type   | Example Value(s)                       |
|-------------------------|-------------------------------------------------------------|--------|----------------------------------------|
| `quay_pull_secret`      | The quay pull secret to pull the necessary openshift images | String | The value located [here](https://cloud.redhat.com/openshift/install/aws/user-provisioned) under step **1 What you need to get started** in the *Pull secret* section. For example:<br><br> `'{"auths":{"cloud.openshift.com":{"auth":"secret","email":"example@example.com"},"quay.io":{"auth":"secret","email":"example@example.com"},"registry.connect.redhat.com":{"auth":"secret","email":"example@example.com"},"registry.redhat.io":{"auth":"secret","email":"example@example.com"}}}'`<br><br> **NOTE:** Make note of the surrounding single quotes  |
| `aws_access_key_id`     | AWS access key ID                                           | String | AXXXXXXXXXXXXXXXXXXXX                  |
| `aws_secret_access_key` | AWS secret access key                                       | String | BXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX |

3. Run the playbook.yml file as follows:
```
ansible-playbook -i inventory playbook.yml --ask-vault-pass -vvv
```

4. SSH to registry box from the bastion via the SSH keys located on both machines, under .ssh/{aws_ssh_key}

5. Enter the konductor container:
```
 podman exec -it konductor connect
```

6. Wait for the Apps ELB to come up: 
```
watch -d -n 5 -c "oc get svc -n openshift-ingress | awk '/router-default/{print $4}'"
```

7. Once this command prints a value for the ELB that appears like this, internal-xxxxxxxxxx.us-gov-w
est-1.elb.amazonaws.com, create a Route53 DNS CNAME record for `*.apps.cluster.domain.com` with this ELB name as the value.

8. To watch the cluster finish coming up:
```
 watch oc get co
```

9. Celebrate because you have a new cluster!

### Possible Limitations and Pitfalls
We have not tested these possible issues with the automation yet, so they may help to know:
* The playbook may fail if the registry or bastion EC2 instances are in the shutting down state when being run
* 2 or less subnets for cluster
* AWS CLI version 2 (we are very sure this should work)
* Ansible >= 2.10
* A cluster and VPC name that are different

## Authors

  - **Griffin College** - *Red Hat*
  - **Sean Nelson** - *Red Hat*
  - **Jonny Rickard** - *Red Hat*
  - **James Radtke** - *Red Hat*
  - **Kevin O'Donnell** - *Red Hat*
  - **Kat Morgan** - *Red Hat*
  - **Jon Hultz** - *Red Hat*
