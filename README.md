# devops-provisioning
Provisioning container for the CyGlass analytics app

This container uses Terraform to lay down the infrastructure in AWS and Ansible to provision and configure the application 

The lifecycle of the container is controlled using Make.
The version for the container is controlled by the ```VERSION``` environment variable.  The default value is ```latest```

To build the container:

```
make build
```

To start the container, and get a shell:

```
make shell
```

To deploy the container to AWS ECR:
``` 
export VERSION=<version>
make push
```

## provisioning process details

The provisioning process uses the following envionment variables

| variable | description | example |
|---       |---          |---      |
| org      | The organization in CyGlass. The hostname for the site is *org*.cyglass.com.  Sets the ```Company``` tag in AWS | foobar |
| customer_name | The full name of the customer | FooBar Inc |
| bill_by | Sets the ```BillByOrg``` tag for all of the AWS objects used by this site | foobar |
| deployment_type | Sets the ```Customer``` tag for all of the AWS used by the site (CUSTOMER, POC, DEV) | POC |
| nodes    | The number of nodes in the cluster (1,3,4) | 1 |
| region   | The AWS region to deploy the app | us-east-1 |
| AWS_DEFAULT_REGION | The default AWS Region | us-east-1 |
| availability_zone | The AWS availability zone to deploy the app | us-east-1a |
| partner_role | *Optional* arn of the AWS IAM role to assign the AWS instance.  By default, a new IAM role is created | arn:aws:iam::501228773606:role/CyGlass|
| vpc_id | *Optional* id of the VPC to deploy the app.  By default, the app is deployed in the default VPC for the AZ | vpc-cc2336b4 |
| subnet_id | *Optional* id of the Subnet to deploy the app.  By default, the app is deployed in the default subnet of the VPC | subnet-de093d95 |
| version | The version of the application to deploy | 3.1.479 |
| CYGLASS_ENV | The environment (dev, staging, prod) to deploy | prod |
| CYGLASS_TAG | The tagged version of the Rundeck and Icinga servers (dev,staging,prod) to manage this deployment | prod |
| CYGLASS_ANSIBLE_INVENTORY | The path inside the container for the Ansible inventory script | /opt/inventory/production |
| ZD_SEND_TICKETS | A flag indicating that when there are errors in the Ansible provisioning jobs should it result in the creation of tickets in support.cyglass.com | True |
| RUNDECK_IP | The IP address of the Rundeck server that will need access to this server to operate properly | 34.239.75.208 |


The provision process is as follows:

- Start the provisioning process with a command like:
```
        docker run -e org=foobar \
           -e region="us_east_1" -e nodes=1 -e availability_zone="us_east_1a" \
           -e customer_name="'Foo Bar Inc'" -e bill_by="foobar" -e deployment_type=DEV \
           -e AWS_DEFAULT_REGION="us_east_1a" -e ZD_SEND_TICKETS=True -e RUNDECK_IP=34.239.75.208 \
           -e version=3.1.497 cyglass/provisioning provisionRole.sh
```
- The ```provisionRole.sh``` sets up the partner AWS environment for a single node.  It then calls ```provisionAws.sh```
- The ```provisionAws.sh``` does the following:
    - Selects and fills in the values of the Terraform template and runs in.  
    - Waits for the machine to start.
    - If it's a 3 or 4 node machine does some post deploy configuration of the cluster
    - Registers the public IP address of the machine with GoDaddy using ```godaddy.py```
    - Calls ```install_ansible.sh``` to install the DevOps ssh key on the instance
    - Calls ```create_invetory.sh``` to create an Ansible host inventory
    - Runs the ```install_cyglass.yml``` Ansible playbook - this downloads and runs the installer
    - Calls the ```post_install.sh``` this runs the following Ansible playbooks
        - ```icinga-client-config.yml``` - The configures the Icinga master to monitor the instance
        - ```post-install.yml``` - This performs the following steps
            - Registers the instance with the Teleport server 
            - Install Teleport client on the instance
            - Installs Kibana
            - Installs the CyGlass Kibana dashboards

## updating or creating kibana dashboard
1. Create the provisioning container
  a. Download or clone [devops-provisioning](https://github.com/CyGlass/devops-provisioning)
  b. Open in terminal and run “make build”
2. Download the dashboard
  1. Create folder to store the dashboard in inside provisioning/dashboards
  2. Open this folder in terminal
..c. Run ../download.sh website.cyglass.com "Dashboard Name"
3. Upload changes to git
⋅⋅a. Ensure that the new dashboard and all its visualizations are accounted for in the folder.
⋅⋅b. Use eclipse to push changes(The dashboard files) to github.
4. Create the container by running Jenkins [devops](http://buildserver:8080/view/DevOps%20Containers/job/Build%20dev-latest%20DevOps%20Environment/)
5. Push to dev with Rundeck: [Install Kibana and Teleport](https://dev-latest-rundeck.cyglass.com:4443/project/cyglass/job/show/c7e818fc-13e4-4031-bf06-5fc58fec6004)
6. Push to staging with rundeck: [staging-staging-rundeck.cyglass.com](staging-staging-rundeck.cyglass.com)
7. Push to production with rundeck

 
