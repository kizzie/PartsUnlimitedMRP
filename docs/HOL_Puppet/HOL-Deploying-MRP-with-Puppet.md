# Deploying the Parts Unlimited MRP App via Puppet in Azure
In this hands-on lab you will deploy a Java app, the Parts Unlimited MRP App, using Puppet from [PuppetLabs](https://puppetlabs.com/). Puppet is a configuration
management system that allows you to automate provisioning and configuration of machines by describing the state of your infrastructure
as code. Infrastructure as Code is an important pillar of good DevOps.

## Prerequisites
- An SSH client such as PuTTY
- An Azure subscription

## Tasks

In this lab you will work with two machines: a Puppet Master machine and another machine known as a _node_
which will host the MRP application. The only task you will perform on the node is to install the Puppet
agent - the rest of the configuration will be applied by instructing Puppet how to configure the node 
though _puppet programs_ on the Puppet Master. 

1. Provisioning a Puppet Master and node (both Ubuntu VMs) in Azure using ARM templates
1. Retrieve the Puppet Master admin password
1. Install Puppet Agent on the node
1. Configure the Puppet Production Environment
1. Test the Production Environment Configuration
1. Create a Puppet program to describe the environment for the MRP application

## Task 1: Provision the Lab
This lab calls for the use of two machines. The Puppet Master server must be a Linux machine, but the puppet
agent can run on Linux or Windows. For this lab, the _node_ that we will be configuring is an Ubuntu VM.

Instead of manually creating the VMs in Azure, we are going to use an Azure Resource Management (ARM) template.
Simply click the Deploy to Azure button below and follow the wizard to deploy the two machines. You will need
to log in to the Azure Portal.

The VMs will be deployed to a Resource Group along with a virtual network (VNET) and some other required resources. You can 
delete the resource group in order to remove all the created resources at any time.

You will need to select a subscription and region to deploy the Resource Group to and to supply an admin username 
and password and unique name for both machines. The Puppet Master will be a Standard D2_V2 while the partsmrp machine
will be a Standard A2.

Make sure you make a note of the region as well as the usernames and passwords for the machines. Allow
about 10 minutes for deployment and then another 10 minutes for the Puppet Master to configure Puppet. 

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fcolindembovsky%2FPartsUnlimitedMRP%2Fa10629e79974ed278ce50552d007acfdeb8c074d%2Fdocs%2FHOL_Puppet%2Fenv%2FPuppetPartsUnlimitedMRP.json" target="_blank">
    <img src="http://azuredeploy.net/deploybutton.png"/>
</a>
<a href="http://armviz.io/#/?load=https%3A%2F%2Fraw.githubusercontent.com%2Fcolindembovsky%2FPartsUnlimitedMRP%2Fa10629e79974ed278ce50552d007acfdeb8c074d%2Fdocs%2FHOL_Puppet%2Fenv%2FPuppetPartsUnlimitedMRP.json" target="_blank">
    <img src="http://armviz.io/visualizebutton.png"/>
</a>

![](<media/1.jpg>)

When the deployment completes, you should see the following resources in the Azure Portal:

![](<media/2.jpg>)

Click on the "partspuppetmaster" Public IP Address. Then make a note of the DNS name:

![](<media/3.jpg>)

The _dnsaddres_ will be of the form _machinename_._region_.cloudapp.azure.com. Open a browser to https://_dnsaddress_.
(Make sure you're going to http__s__, not http). You will be prompted about an invalid certificate - it is safe to
ignore this for the purposes of this lab. If the Puppet configuration has succeeded, you should see the Puppet Console
sign in page:

![](<media/4.jpg>)

>**Note:** The lab requires several ports to be open, such as the Puppet Server port, the Puppet console port, SSH
ports and the Parts Unlimited MRP app port on the partsmrp machine. The ARM template opens these ports on the
machines for you.

## Task 2: Retrieve the Puppet Master admin password
The Puppet Master VM is created from an image that PuppetLabs maintiains in the Azure MarketPlace. When a VM is created 
from the image, it installs and configures Puppet Server from an answerfile. One of the fields in the answerfile is the 
admin password. We will now SSH into the Puppet Master and retrieve the password.

SSH into the puppet master VM by opening PuTTY or some other SSH console and connecting to _dnsaddress_ as noted above.
Enter in the username and password that you configured when creating the resource group earlier. Once you have logged
in, enter the following command:

```sh
sudo cat /etc/puppetlabs/installer/database_info.install
```

![](<media/5.jpg>)

Make a note of the password with the key `q_puppet_enterpriseconsole_auth_password`. 

>**Note:** If this key isn't in the dump, then wait a few more moments for the Puppet Console install to complete and 
rerun the command.  

Now go back to the Puppet Console in your browser and enter the username `admin` and the password you just retrieved. 
When you log in, you should see a page like this:

![](<media/6.jpg>)

>**Note:** It is recommended that you change the admin password. In the toolbar, click 'admin->My Account' and then click
the "Reset Password" link in the upper right:
![](<media/7.jpg>)

In the SSH console, enter the following command to make a note of the internal IP address of the Pupper Master:

```sh
hostname -i
```

The IP address should be something like `10.0.0.4`.

## Task 3: Install Puppet Agent on the node
You are now ready to add the node to the Puppet Master. Once the node is added, the Puppet Master will be able to configure
the node.

Click on the "No Node Requests" button in the toolbar at the top of the console. The page that loads will show a command
that we need to run on the node. Make a note of the machine name (the internal VNet name of the puppet master, since we 
will have to add this address into the hosts file on the node so that the node will be able to resolve the IP address of 
the puppet master. In the example below, the puppet master machine name is:

```
partspuppetmaster.nqkkrckzqwwu1p5pu4ntvzrona.cx.internal.cloudapp.net
```
![](<media/8.jpg>)

Now open another SSH terminal and this time log into the node using the 2nd username and password that you selected when
the resource group was created.

Once you have logged in, enter the following command to edit the hosts file:

```sh
sudo nano /etc/hosts
```

Enter a new line on line 2 with the following:

_IP-of-puppet-master_ puppetmaster _internal-name-of-puppet-master_. For example:
```
10.0.0.4 partspuppetmaster partspuppetmaster.nqkkrckzqwwu1p5pu4ntvzrona.cx.internal.cloudapp.net
```
![](<media/9.jpg>)


Then press `cntrl-X`, `y` and `enter` to save the changes. Test your edits by entering the following command:
```sh
ping partspuppetmaster.nqkkrckzqwwu1p5pu4ntvzrona.cx.internal.cloudapp.net
```

You should see a reply from the ping. Press cntrl-C to stop the ping.

![](<media/10.jpg>)

Next, enter the command that you copied from the Puppet Console to add the node:
![](<media/11.jpg>)

The command will take a few moments to complete.

From here on, you will configure the node only from the Puppet Master, though you will use the partsmrp SSH
terminal to manually force Puppet to configure it.

Return to the Puppet Console and refresh the node requests page (where you previously go the node install command). You
should see a pending request. This request has come from the node and will authorize the certificate between the puppet
master and the node so that they can communicate securely. Press "Accept" to approve the node:
![](<media/12.jpg>)

Click on the "Nodes" tab in the Puppet Console to return to the nodes view. You should see 2 nodes listed: the puppet
master and the partsmrp node:

![](<media/13.jpg>)

>**Note:** It is possible to automate the install and configuration of the Puppet agent onto an Azure VM using the
[Puppet Agent extension](https://github.com/Azure/azure-quickstart-templates/tree/master/puppet-agent-windows) from the 
Azure Marketplace.

## Task 4: Configure the Puppet Production Environment
The Parts Unlimited MRP application is a Java application that requires [mongodb](https://www.mongodb.org/)
and [tomcat](http://tomcat.apache.org/) to be installed and configured on the partsmrp machine (the node). Instead of
installing and configuring manually, we will now write a puppet program that will instruct the node how to configure
itself.

Puppet Programs are stored in a particular folder in the puppet master. Puppet programs are made up of manifests
that describe the desired state of the node(s). The manifests can consume modules, which are pre-packaged Puppet
Programs. Users can create their own modules or consume modules from a marketplace maintained by PuppetLabs known
as the [Forge](http://forge.puppetlabs.com). Some modules on the Forge are official modules that are supported - 
others are open-source modules uploaded from the community.

There are two major ways to organise Puppet Programs. One is by _site_, and the other is by _environment_. Organizing
by site allows you to configure a group of nodes from a single catalog. Howeverm, it is better to organize by 
environment, allowing you to manage different catalogs for different environments such as dev, test and production.

For the purposes of this lab, we will treat the node as if it were in the production environment. We will also need to 
download a few modules from the Forge which we will consume to configure the node.

When the Puppet Server was installed in Azure, it configured a folder for managing the production environment
in `/etc/puppetlabs/puppet/environments/production`.

On the SSH terminal to the Puppet Master, cd to that folder now:

```sh
cd /etc/puppetlabs/puppet/environments/production
```

If you run `ls` you will see two folders: `manifests` and `modules`. The `manifests` folder contains descriptions
of machines that we will later apply to nodes. The `modules` folder contains any modules that are referenced
within the manifests.

We will now install some modules from the Puppet Forge that we will need to configure the `partsmrp` node. Run
the following 3 commands:

```sh
sudo puppet module install puppetlabs-mongodb --modulepath /etc/puppetlabs/puppet/environments/production/modules/
sudo puppet module install puppetlabs-tomcat --modulepath /etc/puppetlabs/puppet/environments/production/modules/
sudo puppet module install maestrodev-wget --modulepath /etc/puppetlabs/puppet/environments/production/modules/
```

>**Note:** We need to specify the `modulepath` since by default the modules are installed to the "site" modules
folder and not the environments module folder.

![](<media/14.jpg>)

>**Note:** The `mongodb` and `tomcat` modules are supported modules from the Forge. The `wget` module is
a user module and so is not officially supported.

We will now create a custom module that will configure the Parts Unlimited MRP app. Run the following commands
to template a module:

```sh
cd /etc/puppetlabs/puppet/environments/production/modules
sudo puppet module generate partsunlimited-mrpapp --environment production
```

This will start a wizard that will ask a series of questions as it scaffolds the module. Simply press `enter`
for each question (accepting blank or default) until the wizard completes.

When generating the module, you need to supply an author and module name (that's why we passed
`partsunlimited-mrpapp` as the name of the module). However, to use the module, the name must simply be the
module name without the author, so rename the folder from `partsunlimited-mrpapp` to `mrpapp`:

```sh
sudo mv partsunlimited-mrpapp mrpapp
```

Running `ls -la` should list the modules available so far, including `mrpapp`:

![](<media/15.jpg>)

We are going to define the node's configuration in the `mrpapp` module. The configuration of the nodes in the
production environment is defined in a `site.pp` file in the production `manifests` folder (the `.pp` extension
is short for "puppet program"). Let's edit the `site.pp` file and define the configuration for our node:

```sh
sudo nano /etc/puppetlabs/puppet/environments/production/manifests/site.pp
```

Open the Puppet console and go to the nodes page. Copy the FQDN of the partsmrp node (which will be something
like `partsmrp.nqkkrckzqwwu1p5pu4ntvzrona.cx.internal.cloudapp.net`. This is the _nodeFQDN_.

Scroll to the bottom of the file and edit the `node default` section. Edit it to look as follows:

```puppet
node default {
  class { 'mrpapp': }
}
```

Press `cntrl-X`, then `y` then `enter` to save the changes to the file.

This instructs Puppet to configure the `default` (that is, all nodes) with the `mrpapp` module. The module (though
currently empty) is in the `modules` folder of the production environment, so Puppet will know where to find
it.

## Task 5: Test the Production Environment Configuration
Before we fully describe the MRP app for the node, let's test that everything is hooked up correctly by 
configuring a "dummy" file in the `mrpapp` module. If Puppet executes and creates the dummy file, then we can
flesh out the rest of the module properly.

Let's edit the `init.pp` file of the `mrpapp` module (this is the entry point for the module):

```sh
sudo nano /etc/puppetlabs/puppet/environments/production/modules/mrpapp/manifests/init.pp
```

You can either delete all the boiler-plate comments or just ignore them. Scroll down to the `class mrpapp`
declaration and make it look as follows:

```puppet
class mrpapp {
  file { '/tmp/dummy.txt':
    ensure => 'present',
    content => 'Puppet rules!',
  }
}
```

Press `cntrl-X`, then `y` then `enter` to save the changes to the file.

>**Note:** Classes in Puppet programs are not like classes in Object Oriented Programming. They simply define
a "resource" that is conifgured on a node. In the `mrpapp` class (or resource), we have just instructed 
Puppet to ensure that a file exists at the path `/tmp/dummy.txt` that has the content "Puppet rules!". We 
will define more advanced resources within the `mrpapp` class as we progress.

Let's test our setup. Switch to the `partsmrp` SSH terminal and enter the following command:

```sh
sudo puppet agent --test
```

By default, the Puppet agents will query the Puppet Master for their configuration every 30 minutes. The
command you just entered forces the agent to ask the Puppet Master for its configuration. It then tests
itself against the configuration, and does whatever it needs to do in order to make itself match that
configuration. In this case, the configuration requires the `/tmp/dummy.txt` file, so the node creates
the file accordingly.

You should see a successful run on the node. `cat` the `/tmp/dummy.txt` file to inspect its contents:

```sh
cat /tmp/dummy.txt
```

![](<media/16.jpg>)

Let's delete the file and then re-run the test:

```sh
sudo rm /tmp/dummy.txt
sudo puppet agent --test
cat /tmp/dummy.txt
```

You should see the run complete successfully and the file should exist again. 

![](<media/17.jpg>)

You can also try to edit the contents of the file and re-run the `sudo puppet agent --test` command to see the 
contents update.

## Task 6: Create a Puppet Program to Describe the Prerequisites for the MRP Application
Now that we have hooked up the node (partsmrp) to the Puppet Master, we can begin to write the Puppet Program
that will describe the prerequisites for the Parts Unlimited MRP application.

>**Note:** For simplicity, we will describe the entire configuration in a single Puppet Program (init.pp from 
the mrpapp module we created earlier). However, the parts of the configuration could be split into multiple 
manifests or modules as they grow. This would promote reuse - just as in any good programming language.

### 6.1 Configure MongoDb ###
Let's add a class to configure mongodb. Once mongodb is configured, we want Puppet to donwload a mongo script
that contains some data for our application's database. We'll include this as part of the mongodb setup.

On the Puppet Master, edit the init.pp file of the mrpapp module:
```sh
sudo nano /etc/puppetlabs/puppet/environments/production/modules/mrpapp/manifests/init.pp
```

Add the following class at the bottom of the file:

```puppet
class configuremongodb {
  include wget
  class { 'mongodb': }->

  wget::fetch { 'mongorecords':
    source => 'https://raw.githubusercontent.com/Microsoft/PartsUnlimitedMRP/master/deploy/MongoRecords.js',
    destination => '/tmp/MongoRecords.js',
    timeout => 0,
  }->
  exec { 'insertrecords':
    command => 'mongo ordering /tmp/MongoRecords.js',
    path => '/usr/bin:/usr/sbin',
    unless => 'test -f /tmp/initcomplete'
  }->
  file { '/tmp/initcomplete':
    ensure => 'present',
  }
}
```

Let's examine this class:
- Line 1: We create a class (resource) called `configuremongodb`
- Line 2: We include the `wget` [module](https://forge.puppetlabs.com/maestrodev/wget) so that we can 
download files via `wget`
- Line 3: We invoke the `mondodb` resource (from the `mongodb` module we downloaded earlier). This installs
mongodb using defaults defined in the [Puppet mongodb module](https://forge.puppetlabs.com/puppetlabs/mongodb).
Believe it or not, that's all we have to do to install mondodb!
- Line 5: We invoke the `fetch` resource from the `wget` module, calling this resource `mongorecords`
- Line 6: We set the source of the file we need to download
- Line 7: We set the destination where the file must be downloaded to
- Line 10: We use the built-in Puppet resource `exec` to execute a command
- Line 11: We specify the command to execute
- Line 12: We set the path for the command invocation
- Line 13: We specify a condition using the keyword `unless`: we only want this command to execute once, so we
create a tmp file once we have inserted the records (Line 15). If this file exists, we don't execute the
command again.

>**Note**: The `->` notation on Lines 3, 8 and 13 is an "ordering arrow": it tells Puppet that it must apply the
"left" resource before invoking the "right" resource. This allows us to specify order when necessary.

Press `cntrl-O`, then `enter` to save the changes to the file without exiting.

### 6.2 Configure Java ###
Next we'll configure Java for the application. Add the following class below the `configuremongodb` class:

```puppet
class configurejava {
  include apt
  $packages = ['openjdk-8-jdk', 'openjdk-8-jre']

  apt::ppa { 'ppa:openjdk-r/ppa': }->
  package { $packages:
     ensure => 'installed',
  }
}
```

Let's examine this class:
- Line 2: We include the `apt` module, which will allow us to configure new Personal Package Archives (PPAs)
- Line 3: We create an array of packages that we need to install 
- Line 5: We add a PPA
- Lines 6 - 8: We tell Puppet to ensure that the package are installed. Puppet expands the array and essentially
does a for-each, installing each package in the array.

>**Note:** We can't use the Puppet `package` target to install Java since this will only install Java 7. That's
why we needed to add the PPA using the `apt` module.

Press `cntrl-O`, then `enter` to save the changes to the file without exiting.

Let's add a class below the `configurejava` class to configure `tomcat`:
```puppet
class configuretomcat {
  class { 'tomcat': }

  tomcat::instance { 'default':
    package_name => 'tomcat7',
    install_from_source => false,
  }->
  tomcat::service { 'default':
    use_jsvc => false,
    use_init => true,
    service_name => 'tomcat7',
  }->
  tomcat::config::server::connector { 'tomcat7-http':
    catalina_base => '/var/lib/tomcat7',
    port => '9080',
    protocol => 'HTTP/1.1',
    connector_ensure => 'present',
    server_config => '/etc/tomcat7/server.xml',
  }
}
```

Let's examine this class:
- Line 1: We create a class (resource) called `configuretomcat`
- Line 2: We invoke the `tomcat` resource (from the [tomcat module](https://forge.puppetlabs.com/puppetlabs/mongodb)
we downloaded earlier)
- Line 4: We need to override some default properties for the tomcat instance. We specify the tomcat package we
need (Line 5) and tell the `tomcat` class not to install from source (Line 6).
- Lines 8 - 12: We configure the Tomcat service
- Lines 13 - 19: We need to configure a connector for the Parts Unlimited MRP application. In Lines 9 - 13, we specify
the connector properties for Puppet to write to the tomcat server.xml file.

Press `cntrl-O`, then `enter` to save the changes to the file without exiting.

Go back to the top of the file and change the `mrpapp` class to look as follows:
```puppet
class mrpapp {
  class { 'configuremongodb': }
  class { 'configurejava': }
  class { 'configuretomcat': }
}
```

This tells Puppet to configure our node with the 3 classes (resources) we just defined to configure MongoDb, Java and Tomcat.

### 6.3 Run the Puppet Program ###
At this point we have configured the prerequisites for the MRP app. Let's run the puppet program on the agent to make
sure that everything is configured correctly so far.

On the partsmrp SSH session, force Puppet to update the node's configuration:
```sh
sudo puppet agent --test
```

This first run will take a few moments - there is lots to download and install for the first run! Next time the Puppet agent runs,
it will verify that the existing environment is correctly configured - that should be much quicker since the services will already
be installed and configured.

Once the run has completed, you can continue.

### 6.4 Verify that Tomcat is running ###
Open a browser and browse to port `9080` of the partsmrp machine. You can get the name of the machine by clicking on the Public IP
resource for the machine in Azure (just like you did to get the url of the puppet master earlier). Once you open the browser, you 
should see the following Tomcat confirmation page:

![](<media/20.jpg>)

## Task 7: Add Puppet Configuration to Deploy the Application
So far we have created the MongoDb database (and inserted some data) and configured Java and Tomcat. We'll now deploy the
application - not by copying manually, but by specifying the steps to deploy the applicaiton in our Puppet Program.

The Tomcat server serves pages from a Java application. The application is compiled to a war file that we need to deploy into
Tomcat. This sever then makes calls to an Ordering Service, which is a REST API managing orders. This service is compiled to a 
jar file. We'll need to copy the jar file to our node and then run it in the background so that it can listen for requests.
 
### 7.1 Deploy a WAR File
Let's specify a resource to deploy the war file for the site. Go back to the Puppet SSH session and edit the `init.pp` file. 
Add the following class at the bottom of the file:

```puppet
class deploywar {
  tomcat::war { 'mrp.war':
    catalina_base => '/var/lib/tomcat7',
    war_source => 'https://raw.githubusercontent.com/Microsoft/PartsUnlimitedMRP/master/builds/mrp.war',
  }
}
```

Let's examine this class:
- Line 1: We create a class (resource) called `deploywar`
- Line 3: We set the `catalina base` directory so that Puppet deploys the war to our Tomcat service
- Line 4: We use the tomcat module's `war` resource to deploy our war from the `war_source`

### 7.2 Start the Ordering Service
Now we need to make sure that the ordering service is running. Again we'll add a new class at the bottom of the `init.pp` file:

```puppet
class orderingservice {
  package { 'openjdk-7-jre':
    ensure => 'installed',
  }

  file { '/opt/mrp':
    ensure => 'directory'
  }->
  wget::fetch { 'orderingsvc':
    source => 'https://raw.githubusercontent.com/Microsoft/PartsUnlimitedMRP/master/builds/ordering-service-0.1.0.jar',
    destination => '/opt/mrp/ordering-service.jar',
    cache_dir => '/var/cache/wget',
    timeout => 0,
  }->
  exec { 'stoporderingservice':
    command => "pkill -f ordering-service",
    path => '/bin:/usr/bin:/usr/sbin',
    onlyif => "pgrep -f ordering-service"
  }->
  exec { 'stoptomcat':
    command => 'service tomcat7 stop',
    path => '/usr/bin:/usr/sbin',
    onlyif => "test -f /etc/init.d/tomcat7",
  }->
  exec { 'orderservice':
    command => 'java -jar /opt/mrp/ordering-service.jar &',
    path => '/usr/bin:/usr/sbin:/usr/lib/jvm/java-8-openjdk-amd64/bin',
  }->
  exec { 'wait':
    command => 'sleep 20',
    path => '/bin',
    notify => Tomcat::Service['default']
  }
}
```

Let's examine this class:
- Line 1: We create a class (resource) called `orderingservice`
- Lines 2 - 4: We install the Java JRE required to run the application using Puppet's `package` resource
- Lines 6 - 8: We ensure that the directory `/opt/mrp` exists (Puppet creates it if it doesnt)
- Lines 15 - 18: We stop the `orderingservice`, but only if it is running
- Lines 20 - 23: We stop the `tomcat7` service, but only if it is running
- Lines 25 - 27: We start the ordering service
- Lines 29 - 32: We sleep for 20 seconds to give the ordering service time to start up before notifying
the `tomcat` service, which triggers a refresh on the service - Puppet will re-apply the state we defined
for the service (i.e. start it if it is not running)

>**Note:** We need to wait after running the `java` command since this service needs to be running before we
start Tomcat.

### 7.3 Complete the mrpapp Resource ###
Go back to the top of the file and change the `mrpapp` class to look as follows to run all our resources:
```puppet
class mrpapp {
  class { 'configuremongodb': }
  class { 'configurejava': }
  class { 'configuretomcat': }
  class { 'deploywar': }
  class { 'orderingservice': }
}
```

Press `cntrl-O`, then `enter` to save the changes to the file without exiting.

### 7.4 Run the Puppet Configuration on the Node ###
On the partsmrp SSH session, again force Puppet to update the node's configuration:
```sh
sudo puppet agent --test
```

Now you can test the configuration by opening up a browser to the Parts Unlimited MRP application. The address
will be http://partsmrp-public-ip:9080/mrp where _partsmrp-public-ip_ is the public ip or DNS name of the
partsmrp VM (you can get it by clicking on the VM in the resource group in the Azure Portal).

![](<media/18.jpg>)

If you click on the Orders button, you should see the orders page:

![](<media/19.jpg>)

>**Note:** You can see the complete `init.pp` file [here](final/init.pp).

#Congratulations!
You've now completed this lab. Remember to delete the resource group you created when you are done in the Azure
Portal so that you are not charged for the resources.