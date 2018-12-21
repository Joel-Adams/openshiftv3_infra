# Openshiftv3 Workshop on Azure

![ansible](img/openshift_v3.png)

`Openshiftv3 Workshop` is a ansible playbook to provision Openshift in Azure. This playbook uses Ansible to deploy Azure templates, for provisioning Azure infrastructure and nodes. To find more info about the templates [check here](https://github.com/Microsoft/openshift-container-platform)

These modules all require that you have Azure ID's available to use to provision Azure resources. You also need to have Azure permissions set to allow you to create resources within Azure. There are several methods for setting up you Azure environment on you local machine.

This repo also requires that you have Ansible installed on your local machine. For the most upto date methods of installing Ansible for your operating system [check here](http://docs.ansible.com/ansible/intro_installation.html).

Make sure that you have pip installed and and on the latest version [check here](https://pip.pypa.io/en/stable/installing/).

Install necessary modules:
`pip install packaging msrestazure ansible[azure]`

## Prerequisites

### Prepare and Upload Red Hat Image to AZURE

If you don't have a Red Hat image stored in the Azure region you would like to deploy Openshift please see this documentation [click here](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/redhat-create-upload-vhd).

Using your own image will insure everything is up to standards and make sure you don't get double billed for the marketplace image.

### Generate SSH Keys

You'll need to generate an SSH key pair (Public / Private) in order to provision this template. Ensure that you do **NOT** include a passphrase with the private key. <br/><br/>
If you are using a Windows computer, you can download puttygen.exe.  You will need to export to OpenSSH (from Conversions menu) to get a valid Private Key for use in the Template.<br/><br/>
From a Linux or Mac, you can just use the ssh-keygen command.  Once you are finished deploying the cluster, you can always generate new keys that uses a passphrase and replace the original ones used during initial deployment.

Make sure to keep a local copy of the public and private key that was created for future access.

### Create Key Vault to store SSH Private Key

You will need to create a Key Vault to store your SSH Private Key that will then be used as part of the deployment.  This extra work is to provide security around the Private Key - especially since it does not have a passphrase.  I recommend creating a Resource Group specifically to store the KeyVault.  This way, you can reuse the KeyVault for other deployments and you won't have to create this every time you chose to deploy another OpenShift cluster.

**Create Key Vault using Azure CLI 2.0**<br/>
1.  Create new Resource Group: az group create -n \<name\> -l \<location\><br/>
    Ex: `az group create -n ResourceGroupName -l 'East US'`<br/>
1.  Create Key Vault: az keyvault create -n \<vault-name\> -g \<resource-group\> -l \<location\> --enabled-for-template-deployment true<br/>
    Ex: `az keyvault create -n KeyVaultName -g ResourceGroupName -l 'East US' --enabled-for-template-deployment true`<br/>
1.  Create Secret: az keyvault secret set --vault-name \<vault-name\> -n \<secret-name\> --file \<private-key-file-name\><br/>
    Ex: `az keyvault secret set --vault-name KeyVaultName -n SecretName --file ~/.ssh/id_rsa`<br/>

### Generate Azure Active Directory (AAD) Service Principal

To configure Azure as the Cloud Provider for OpenShift Container Platform, you will need to create an Azure Active Directory Service Principal.  The easiest way to perform this task is via the Azure CLI.  Below are the steps for doing this.

Assigning permissions to the entire Subscription is the easiest method but does give the Service Principal permissions to all resources in the Subscription.  Assigning permissions to only the Resource Group is the most secure as the Service Principal is restricted to only that one Resource Group.

**Azure CLI 2.0**

1. **Create Service Principal and assign permissions to Subscription**<br/>
  a.  az ad sp create-for-rbac -n \<friendly name\> --password \<password\> --role contributor --scopes /subscriptions/\<subscription_id\><br/>
      Ex: `az ad sp create-for-rbac -n openshiftcloudprovider --password Pass@word1 --role contributor --scopes /subscriptions/555a123b-1234-5ccc-defgh-6789abcdef01`<br/>

2. **Create Service Principal and assign permissions to Resource Group**<br/>
  a.  If you use this option, you must have created the Resource Group first.  Be sure you don't create any resources in this Resource Group before deploying the cluster.<br/>
  b.  az ad sp create-for-rbac -n \<friendly name\> --password \<password\> --role contributor --scopes /subscriptions/\<subscription_id\>/resourceGroups/\<Resource Group Name\><br/>
      Ex: `az ad sp create-for-rbac -n openshiftcloudprovider --password Pass@word1 --role contributor --scopes /subscriptions/555a123b-1234-5ccc-defgh-6789abcdef01/resourceGroups/00000test`<br/>

3. **Create Service Principal without assigning permissions to Resource Group**<br/>
  a.  If you use this option, you will need to assign permissions to either the Subscription or the newly created Resource Group shortly after you initiate the deployment of the cluster or the post installation scripts will fail when configuring Azure as the Cloud Provider.<br/>
  b.  az ad sp create-for-rbac -n \<friendly name\> --password \<password\> --role contributor --skip-assignment<br/>
      Ex: `az ad sp create-for-rbac -n openshiftcloudprovider --password Pass@word1 --role contributor --skip-assignment`<br/>

You will get an output similar to:

```javascript
{
  "appId": "2c8c6a58-44ac-452e-95d8-a790f6ade583",
  "displayName": "openshiftcloudprovider",
  "name": "http://openshiftcloudprovider",
  "password": "Pass@word1",
  "tenant": "12a345bc-1234-dddd-12ab-34cdef56ab78"
}
```

The appId is used for the aadClientId parameter.

### Azure Infrastructure Roles


### roles/azure.infra

To create infrastructure and a Openshift instance via Ansible:

First, copy group_vars/all_example to group_vars/all, and then edit `group_vars/all` and fill out the below variables:


```
#####################################################
Azure Information
#####################################################
azure_subscription_id:            ""
azure_client_id:                  ""
azure_client_secret:		          ""
azure_tenant_id:		              ""
azure_image_id:                   ""
ssh_public_key:                   "" # Full public key
key_vault_resource_group:         "" # Resource group where key vault is located
key_vault_name:                   "" # Name of the key vault
key_vault_secret:                 "" # Name of the secret key
#####################################################
Red Hat Information
#####################################################
username:                         ""
password:                         ""
pool_id:                          ""
```

## Configure Workshop Nodes
To call Ansible to provision the nodes and openshift environment run the first playbook.

```
ansible-playbook 1_provision.yml

```
Once the first playbook is created, go back into group_vars/all and update the bastion_fqdn. This can be found in the Azure portal -> bastion VM -> DNS name. Then run the second playbook.

```
ansible-playbook 2_loadinformation.yml

```

You might get an error stating `"Aborting, target uses selinux but python bindings (libselinux-python) aren't installed!"`. If so please install the libselinux-python an re-run the playbook.

To install and configure all of the needed software for the workshop run the below command replacing the `openshift_admin_username` and `private key location` with your own.

```
ansible-playbook -u {{ openshift_admin_username }} 3_setup_bastion_host.yml --private-key {{ /etc/ssh/oshift_key_rsa }}

```

## Login to the Openshift Console

Browse to the URL of the Master DNS load balancer. This can be found in the Azure portal -> master VM -> DNS name.

```
https://{{ master_fqdn }}/console

```
username: found in `group_vars/all` variable `openshift_admin_username`

password: found in `group_vars/all` variable `openshift_password`

![Openshift Login](img/oshit_console.png)

There is a web based ide (wetty) running on the bastion node that each user can log into. This will give them access to perform the Openshift commands on the workshop

```
https://{{ bastion_fqdn }}/wetty

```

![Wetty Login](img/wetty_login.png)

The login for the wetty access will be `labuser(01-50)` with a password of `RedHat123`.

![Wetty Logged In](img/wetty_loged_in.png)

## Setup the Nexus Repository

Log into the command line as the {{ openshift_admin_username }} using the wetty console or SSH from the bastion host. Then setup the nexus container.

```
oc new-project nexus
oc new-app sonatype/nexus3
oc expose svc/nexus3
oc get route
```
Log into the Nexus console using the route provided. User name `admin` password `admin123`

1. Go to settings -> Repositories
2. Select Create Repository -> maven2(proxy) <br/>
  a. Name: apache <br/>
  b. Remote storage: https://repo.maven.apache.org/maven2/ <br/>
  c. Create Repository <br/>
3. Select Create Repository -> maven2(proxy) <br/>
  a. Name: jboss <br/>
  b. Remote storage: https://maven.repository.redhat.com/ga/ <br/>
  c. Create Repository <br/>
4. Select Create Repository -> maven2(proxy) <br/>
  a. Name: jboss-releases <br/>
  b. Remote storage: https://maven.repository.redhat.com/earlyaccess/all/ <br/>
  c. Create Repository   <br/>
5. Select maven-public (This will give you the URL needed for the workshop exercises under `URL:`) <br/>
  a. If not already there, add the apache, jboss, and jboss-releases to the Available Member repositories: <br/>

Now the workshop environment is completely setup and ready to go.

## Hugo Server Workshop steps

There is also a hugo server that is deployed locally on the bastion host that can be used for the workshop steps.

Open up a browser to the {{ bastion_fqdn }}:8080 <br/>
Select the **Openshiftv3 on Azure** workshop. <br/>
Provide each student with a user number 1-50, bastion DNS name, master DNS name, and region. Once populated it will save the variables for each student to use during the workshop.


## Once the workshop is completed

You can either shut down the environment via the Azure Portal as to not incur VM charges, or delete the resource group that was created to not incur any additional charges. If you choose to delete the resource group be sure to first un-register each host from Red Hat subscription manager via [click here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/chap-subscription-management-unregistering), or log into access.redhat.com and delete the systems from your subscriptions.

If you choose to shut the environment down, there will be some necessary steps to take when spinning it back up.

First ssh into the bastion hosts using the private key. Then:
```
sudo su -
docker rm nginx-proxy wetty-cntr
cd /var/tmp
rm -rf github hugo
cd /home/{{ openshift_admin_username }}
rm -rf dockyard epel-release-latest-7.noarch.rpm
```
Once that is completed you can re-run the third playbook on your local system. After this completes the system will be ready to go once again.

```
ansible-playbook -u {{ openshift_admin_username }} 3_setup_bastion_host.yml --private-key {{ /etc/ssh/oshift_key_rsa }}

```
