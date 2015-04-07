---
layout: post
title: Installing Chef Server, Chef Development Kit and Chef Client
description: How to get started with Chef.
tags: chef azure
---

## Chef Server ##

At the time of writing this Chef and Microsoft [recently released](http://azure.microsoft.com/blog/2015/04/01/chef-server-in-marketplace-chef-azure-provisioning-and-more/) a ready to use Chef Server in the Azure Marketplace. Unfortunately it's not offered in all countries/regions, so it might not work with your subscription.

The steps below show how to manually install Chef Server 12 on CentOS 6.x and will work whether you use Azure or not. 

### Download package ###

    $ wget https://web-dl.packagecloud.io/chef/stable/packages/el/6/chef-server-core-12.0.7-1.el6.x86_64.rpm

Alternatively you can go to [https://downloads.chef.io/chef-server/](https://downloads.chef.io/chef-server/), pick "Red Hat Enterprise Linux" and download the package for "Red Hat Enterprise Linux 6".

### Install package ###

    $ sudo rpm -Uvh chef-server-core-12.0.7-1.el6.x86_64.rpm

### Start the services ###

    $ sudo chef-server-ctl reconfigure

### Create an administrator ###

    $ sudo chef-server-ctl user-create <username> <first-name> <last-name> <email> <password> --filename <FILE-NAME>.pem

> An RSA private key is generated automatically. This is the user’s private key and should be saved to a safe location. The --filename option will save the RSA private key to a specified path.

### Create an organization ###

    $ sudo chef-server-ctl org-create <short-name> <full-organization-name> --association_user <username> --filename <FILE_NAME>.pem

> The --association\_user option will associate the \<username\> with the admins security group on the Chef server.
> 
> An RSA private key is generated automatically. This is the chef-validator key and should be saved to a safe location. The --filename option will save the RSA private key to a specified path.

### Check the host name ###

This should return the FQDN:

    $ hostname -f

If they differ, run the following commands:

    $ sudo hostname <HOSTNAME>
    $ echo "<HOSTNAME>" | sudo tee /etc/hostname
    $ sudo bash -c "echo 'server_name = \"<HOSTNAME>\"
    api_fqdn server_name
    nginx[\"url\"] = \"https://#{server_name}\"
    nginx[\"server_name\"] = server_name
    lb[\"fqdn\"] = server_name
    bookshelf[\"vip\"] = server_name' > /etc/opscode/chef-server.rb"
	$ sudo opscode-manage-ctl reconfigure
    $ sudo chef-server-ctl reconfigure

### Premium features ###

Premium features are free for up to 25 nodes.

#### Management console ####

On Azure the management console will be available at `https://your-host-name.cloudapp.net`.

> Use Chef management console to manage data bags, attributes, run-lists, roles, environments, and cookbooks from a web user interface.

    $ sudo chef-server-ctl install opscode-manage
    $ sudo opscode-manage-ctl reconfigure
    $ sudo chef-server-ctl reconfigure

#### Reporting ####

> Use Chef reporting to keep track of what happens during every chef-client runs across all of the infrastructure being managed by Chef. Run Chef reporting with Chef management console to view reports from a web user interface.

    $ sudo chef-server-ctl install opscode-reporting
    $ sudo opscode-reporting-ctl reconfigure
    $ sudo chef-server-ctl reconfigure

----------

## Chef Development Kit ##

This will install the Chef Development kit on your Windows workstation.

Go to [https://downloads.chef.io/chef-dk/windows/#/](https://downloads.chef.io/chef-dk/windows/#/) and download the [installer](https://opscode-omnibus-packages.s3.amazonaws.com/windows/2008r2/x86_64/chefdk-0.4.0-1.msi "installer").

The default installation options will work just fine.

### Starter kit ###

If you're new to Chef, this is the easiest way to start interacting with the Chef server. Manual methods of setting up the chef-repo may be found [here](https://docs.chef.io/install_dk.html#set-up-the-chef-repo).  

We'll take the easy route so open the management console and download the starter kit from the "Administration" section. Extract the files your a suitable location on your machine, I'll assume you're extracting the contents to `c:\`.

### SSL certificate ###

This step is only necessary if you're using a self-signed certificate.

Chef Server automatically generates a self-signed certificate during the installation. To interact with the server we have to put it into the trusted certificates directory, which is done using the following command:

    c:\chef-repo>knife ssl fetch

### Azure publish settings ###

If you're using Azure, download your publish settings [here](https://manage.windowsazure.com/publishsettings), place the file in the `.chef` directory and add the following line to `knife.rb`:  

    knife[:azure_publish_settings_file] = "#{current_dir}/<FILENAME>.publishsettings"

and also install the Knife Azure plugin:

    c:\chef-repo>chef gem install knife-azure

> A knife plugin to create, delete, and enumerate Microsoft Azure resources to be managed by Chef.

### Check installation ###

    c:\chef-repo>chef verify

----------

## Chef Client ##

### Azure portal ###

Provision a new VM and add the Chef Client extension. The extension [isn't supported on CentOS](https://github.com/chef-partners/azure-chef-extension/issues/17) at the moment, it will only work with Windows or Ubuntu images. This can be very confusing if you're not aware of it, since it's allowed, but the bootstrapping will fail.

#### Validation key ####

Use the \<ORGANIZATION_NAME\>-validator.pem file from your workstation or the .pem-file you created when creating an organization above.

#### Client.rb ####

If you're using a self-signed certificate:

    chef_server_url "https://<FQDN>/organizations/<ORGANIZATION_NAME>"
    validation_client_name	"<ORGANIZATION_NAME>-validator"
    ssl_verify_mode    :verify_none

If not:

    chef_server_url "https://<FQDN>/organizations/<ORGANIZATION_NAME>"
    validation_client_name	"<ORGANIZATION_NAME>-validator"

#### Run list (optional) ####

Add the cookbooks, recipes or role that you want to run.

### Azure Cross-Platform Command-Line Interface ###

Instructions on how to install the xplat-cli can found [here](http://azure.microsoft.com/en-us/documentation/articles/xplat-cli/).

#### Linux ####

##### Create VM #####

    azure vm create --vm-size <VM-SIZE> --vm-name <VM-NAME> --ssh <SSH-PORT> <IMAGE-NAME> <USERNAME> <PASSWORD>

- Add `--connect <FQDN>` if you want to connect it to existing VMs.
- CentOS images: `azure vm image list | grep CentOS`.
- Ubuntu LTS images: `azure vm image list | grep "Ubuntu.*LTS" | grep -v DAILY`.

##### Bootstrap node #####

    knife bootstrap <FQDN> -p <SSH-PORT> -x <USERNAME> -P <PASSWORD> --sudo

- Add `--node-ssl-verify-mode none` if you're using a self-signed certificate.
- Add `--no-host-key-verify` if you don't want to verify the host key.    

#### Windows ####

##### Create VM #####

    azure vm create --vm-size <VM-SIZE> --vm-name <VM-NAME> --rdp <RDP-PORT> <IMAGE-NAME> <USERNAME> <PASSWORD>

Add `--connect <FQDN>` if you want to connect it to existing VMs.

Windows Server 2012 images: `azure vm image list | grep "Windows-Server-2012" | grep -v "Essentials\|Remote\|RDSH"`.

##### Bootstrap node #####

Before bootstrapping the node we'll have to configure WinRM. 

Start a PowerShell session on the node and run the following commands:

**This configuration should not be used in a production environment.**

    c:\>winrm qc
    c:\>winrm set winrm/config/winrs '@{MaxMemoryPerShellMB="300"}'
    c:\>winrm set winrm/config '@{MaxTimeoutms="1800000"}'
    c:\>winrm set winrm/config/service '@{AllowUnencrypted="true"}'
    c:\>winrm set winrm/config/service/auth '@{Basic="true"}'
    
The first command performs the following operations:

> - Starts the WinRM service, and sets the service startup type to auto-start.
> - Configures a listener for the ports that send and receive WS-Management protocol messages using either HTTP or HTTPS on any IP address.
> - Defines ICF exceptions for the WinRM service, and opens the ports for HTTP and HTTPS.

If we're not on the same subnet we also have to change the "Windows Remote Management" firewall rule.

Back on the workstation, create a WinRM endpoint:

    c:\chef-repo>azure vm endpoint create -n winrm <VM-NAME> <PUBLIC-PORT> 5985

Finally we can do the bootstrapping:

    c:\chef-repo>knife bootstrap windows winrm <FQDN> --winrm-user <USERNAME> --winrm-password <PASSWORD> --winrm-port <WINRM-PORT>

- Add `--node-ssl-verify-mode none` if you're using a self-signed certificate.

----------

## Knife Azure ##

When setting up our workstation above we installed the Knife Azure plugin. This allows us to provision a VM in Azure as well as bootstrap it in one step.

Here's an example using a CentOS 7.0 image, which will be connected to an existing resource group:

    c:\chef-repo>knife azure server create --azure-service-location '<service-location>' --azure-connect-to-existing-dns --azure-dns-name '<dns-name>' --azure-vm-name '<vm-name>' --azure-vm-size '<vm-size>' --azure-source-image '5112500ae3b842c8b9c604889f8753c3__OpenLogic-CentOS-70-20150325' --ssh-user <ssh-user> --ssh-password <ssh-password> --bootstrap-protocol 'ssh'

## Resources ##

- [https://www.chef.io/](https://www.chef.io/)
- [https://learn.chef.io/](https://learn.chef.io/)
- [https://docs.chef.io/azure_portal.html](https://docs.chef.io/azure_portal.html)
- [https://github.com/chef/knife-azure](https://github.com/chef/knife-azure)