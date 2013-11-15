# Getting started with Chef-Server tutorial:

## What is Vagrant ?
####You can read about it [here](http://docs.vagrantup.com/v2/getting-started/) .

## What is Chef-Server ?
####You can read about it [here](http://docs.opscode.com/chef_overview.html ).

### Goal : Understand Chef Server and its concepts.

In this demo, our goal is to setup the following cluster of machines

1. chef.chef-demo.com (Chef-Server): This machine acts as the chef server. It is provisioned by chef-solo.

2. node.chef-demo.com (A node): This machine is an ordinary Ubuntu instance that get provisioned by chef-server.

Both the vms are Ubuntu 64 bit virtual boxes. The login/password combination should you need to ssh into them is vagrant/vagrant

### Step 1 : Create Virtual Machines using Vagrant

Create a file called 'Vagrantfile' in which we have to specify the configuration settings for our cluster.
 
     $ vi Vagrantfile  

This is our configuration file.

    # -*- mode: ruby -*- 
	# vi: set ft=ruby :  
	# Vagrantfile API/syntax version. Don't touch unless you know what you're doing! 
	VAGRANTFILE_API_VERSION = "2" 

	Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
	  config.berkshelf.enabled = true
	  config.omnibus.chef_version = :latest

	  config.vm.box = "precise64"
	  config.vm.box_url = "http://files.vagrantup.com/precise64.box"

	  config.vm.provider :virtualbox do |provider|
	    provider.customize ["modifyvm", :id, "--memory", "1024"]
	  end

	  config.vm.define :chef_server_node do |chef_config|
	    chef_config.vm.network :private_network, ip: "10.33.33.33"
	    chef_config.vm.network :forwarded_port, guest: 80, host: 1234
	    chef_config.vm.network :forwarded_port, guest: 443, host: 8443
	    chef_config.vm.hostname = 'chef.xyz.com'

	    chef_config.vm.provision :chef_solo do |chef|
	       chef.cookbooks_path = ["site-cookbooks", "cookbooks"]
	       chef.roles_path = "roles"
	       chef.data_bags_path = "data_bags"
	       chef.log_level = :debug
	       chef.add_role "chef"
	    end
	  end

	  config.vm.define :chef_client_node do |chef_client_config|
	    chef_client_config.vm.network :private_network, ip: "10.33.33.50"
	    chef_client_config.vm.hostname = 'node.abc.com'
	  end
	end 

Additionally, I add the following in my machine's /etc/hosts

  `10.33.33.33 chef.chef-demo.com`
  `10.33.33.50 node.chef-demo.com`

Now we need to create a role called 'chef' for the chef-server.
	 $ mkdir roles
	 $ vi roles/chef.json

	 The contents of the chef.json file is as below:

	 {
	   "name": "chef",
	   "chef_type": "role",
	   "json_class": "Chef::Role",
	   "description": "The base role for Chef Server",
	   "default_attributes": {
	     "chef-server": {
	       "version": "latest",
	       "configuration": {
	         "chef_server_webui": {
	           "enable": true
	         }
	       }
	     }
	   },
	   "run_list": [
	     "recipe[chef-server::default]"
	   ]
	 }

After creating a file we start our vagrant. So let's do it.

	 $ vagrant up 

It starts the process of bringing up the virtual machine. So after few seconds/minutes your vagrant should be up and runninig. `vagrant up` also runs `vagrant provision`, which is basically the process of installing the desired software. In the case of chef.chef-demo.com , it will have installed the chef-server software. You can check it by accessing `https://chef.chef-demo.com` . Ignore the SSL cert warnings. Login with the credentials provided on the right. After logging in, you will be asked to change the password, but you can ignore it as we are just learning our way through the demo. Click on links such as nodes, roles etc.., you will find that they are empty at this point.

Pause. Its time to internalize what we've achieved so far. If you look at the provisioning of chef.chef-demo.com, we can reinforce a few concepts:

1) The provisioning is done by chef-solo. chef-solo is a clean and easy way to provision a vm. As mentioned earlier, this is the only machine that we use chef-solo on. The node machine gets provisioned by the chef-server.
2) We have provided the path for cookbooks, roles and data bags. These are concepts in Chef which you should read up on the side for better appreciation. Feel free to switch and look up what these directories contain.
3) We have applied a role called 'chef' on the node. Roles are modular ways of defining behavior on machines. Inspect the roles/chef.json file to get a feel of the behavior defined by this role. At the very bottom you would see a `run_list` key that contains the recipes that need to be cooked on machines that have this role.


### Step 2 :  Install Rubygems chef,knife-solo and berkshelf

Create a file called 'Gemfile' in the root of the directory.

	$ vi Gemfile 

This is the contents of our Gemfile
``` 
source :rubygems

gem 'chef', "~> 11.4.0"
gem 'knife-solo'
gem 'berkshelf'
``` 
Now we have to install the gems by running:

    $ bundle install  

### Step3 : Install the cookbooks using berkshelf

Cookbooks are nothing but chef recipes. For example if you have to install nginx on the nodes you can just use the Opscode nginx cookbook recipe or you can write your own. So to download the cookbooks we use berkshelf. Berkshelf's functionality is almost similar to Rubygem's.

Specify the cookbooks which you are going to use including the cookbooks created by you.

    $ vi Berksfile
This is the contents of our Berksfile
 	
	site :opscode

	cookbook 'chef-server', path: 'cookbooks/chef-server'
	cookbook 'nginx'
	cookbook 'apt'
	cookbook 'build-essential'
	cookbook 'postgresql'
	cookbook 'mysql'

Now we can install cookbooks by running:
		
	$ berks install --path cookbooks

It installs the cookbooks which are specified in the Berksfile.

### Step 4: Initalize the knife and edit knife config file

Note: You dont have to run the commands in this step as the generated directories/content have already been setup in this repo. But do read the step as you will have to run this on a brand new setup.

Knife is a tool used with chef-server. We have to initialise knife, so that it prepares the directories and files.

	$ knife solo init .

This creates a set of directories and files. 
 	.
	|-- Berksfile
	|-- cookbooks
	|-- data_bags
	|-- environments
	|-- nodes
	|-- roles
	`-- site-cookbooks

After this we will provision the vagrant so that the chef-server node gets initialised
and we can access the web UI of the chef-server.

 	$ vagrant provision

Now open your web browser and specify the FQDN (chef.chef-demo.com) or IP address (10.33.33.33) of the chef-server node to access the Web UI.

##### SSH Keys :
After installation Chef Server with default settings, Chef will generate pem keys, which will be used for knife (command line tool for Chef) and Chef clients for authentication with server. We should copy keys from our Chef Server node to “.chef” directory in the project.

 	$ vagrant ssh chef_server_node

Now you will get the shell access to the chef-server node. From the server node's shell run this command.

    $ sudo cp /etc/chef-server/*.pem /vagrant/.chef/

This command copies the public keys into the shared folder /vagrant which is mapped to the root of this repo in your host file system.


##### Knife configuration:

Next we should create for knife configuration file (knife should know how to communicate with Chef Server).:
<br>

    $ knife configure -i
    Overwrite /Users/ravibhim/git/chef-server-demo/.chef/knife.rb? (Y/N) Y
    Please enter the chef server URL: [https://Ravis-MacBook-Air.local:443] https://10.33.33.33
    Please enter a name for the new user: [ravibhim] 
    Please enter the existing admin name: [admin] 
    Please enter the location of the existing admin's private key: [/etc/chef-server/admin.pem] /Users/ravibhim/git/chef-server-demo/.chef/admin.pem
    Please enter the validation clientname: [chef-validator] 
    Please enter the location of the validation key: [/etc/chef-server/chef-validator.pem] /Users/ravibhim/git/chef-server-demo/.chef/chef-validator.pem
    Please enter the path to a chef repository (or leave blank): 
    Creating initial API user...
    Please enter a password for the new user: 
    Created user[ravibhim]
    Configuration file written to /Users/ravibhim/git/chef-server-demo/.chef/knife.rb

<br>
As a result, you should have a file “.chef/knife.rb” with similar content:
<br>

    log_level                :info
    log_location             STDOUT
    node_name                'ravibhim'
    client_key               '/Users/ravibhim/git/chef-server-demo/.chef/ravibhim.pem'
    validation_client_name   'chef-validator'
    validation_key           '/Users/ravibhim/git/chef-server-demo/.chef/chef-validator.pem'
    chef_server_url          'https://10.33.33.33'
    syntax_check_cache_path  '/Users/ravibhim/git/chef-server-demo/.chef/syntax_check_cache'

The knife configure command is not cognizant of the chef-server public keys and has also created a new chef-server admin user 'ravibhim'. You can go to the web console and check the added user in the User's tab.

Let’s check knife configuration:
	
    $ knife client list

    It lists the clients
	- chef-validator
	- chef-webui
	 
###Step 5: Bootstrap first Client node

Once the Chef Server node is configured, it can be used to install chef-client on one or more nodes using a Knife bootstrap. The “knife bootstrap” command is used to SSH into the target machine, and then prepare the node to allow the chef-client to run on the node. It will install the chef-client executable, generate keys, and register the node with the Chef Server node. The bootstrap operation requires the IP address or FQDN of the target node or system, the SSH credentials (username, password or identity file) for an account that has root access to the node, and (if the operating system is not Ubuntu, which is the default distribution used by knife bootstrap) the operating system running on the target system.

So let’s do this:

	 $ knife bootstrap 10.33.33.50 -x vagrant -P vagrant --sudo --node-name web_server
		Bootstrapping Chef on 10.33.33.50
		10.33.33.50 Starting Chef Client, version 11.8.0
		10.33.33.50 Creating a new client identity for web_server using the validator key.
		10.33.33.50 resolving cookbooks for run list: []
		10.33.33.50 Synchronizing Cookbooks:
		10.33.33.50 Compiling Cookbooks...
		10.33.33.50 [2013-11-14T10:44:25+00:00] WARN: Node web_server has an empty run list.
		10.33.33.50 Converging 0 resources
		10.33.33.50 Chef Client finished, 0 resources updated

Let’s check clients on Chef Server:

		$ knife client list
		chef-validator
		chef-webui
		web_server

As you can see we get new client 'web_server'.

     $ knife client show web_server
     
		admin:      false
		chef_type:  client
		json_class: Chef::ApiClient
		name:       web_server
		public_key: -----BEGIN PUBLIC KEY-----
		MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxcSZNlVMEM3nFjQRyb6x
		3sVdd333vxbtXJP3eT019hJ5sI37gYKn6fNaoiA5oes4peew07g6GbEHEaQITboQ
		s54zXpkdrjP1x0H73FJFxLbEekTUosQdZ7sjCNeBhMi4QG9LPK9WuBNhq7g18QNW
		UY62VgApWTnOGFWZ0Yri0wf2+dlB2fnjvdFO6FPPkBQzY6kFy077X4JS7eWK8F1Y
		FiuJq8SQOpGthktLJBzszB0r7WFDwgeg6f/2RjazL5RqgFXu3eG+830FoLdBgWmn
		a/Nj7eM9p3v67I2Sj/ji2oouV2vWSrqmJ7Fw0D2vCrmsct3l8x4ynxICrrWgwULZ
		9wIDAQAB
		-----END PUBLIC KEY-----

When a node runs the chef-client for the first time, it generally does not yet have 
an API client identity, and so it cannot make authenticated requests to the server. This is where the validation client—known as the chef-validator—comes in. When the chef-client runs, it checks if it has a “client_key”. If the client key does not exist, it then attempts to borrow the identity of the chef-validator to register	itself with the server (“validation_key”).

###Step 6: Cookbooks, roles and nodes on chef server

In comparison with Chef Solo, Chef Server store all information on server and use only this information for “cooking” nodes. So we should know, how to upload our roles, cookbooks and nodes on server. First of all we should install vender cookbooks localy by using Berkshelf.

     $ berks install --path cookbooks

As you can see dependencies downloaded automatically. Right now we have this cookbooks only in our local directory “cookbooks”. Let’s upload its to Chef Server. To do this task we can use knife:

    $ knife cookbook upload --all --cookbook-path cookbooks

    $ knife cookbook list
      apt               2.3.0
      build-essential   1.4.2
      chef-server       2.0.0
      mysql             3.0.4
      nginx             1.8.0
      ohai              1.1.12
      openssl           1.1.0
      postgresql        3.2.0
      runit             1.2.0
      yum               2.3.2
 
 Don't forget about own custom cookbooks in 'site-cookbooks' folder

    $ knife cookbook upload --all --cookbook-path site-cookbooks

Now we can see all the uploaded cookbooks from the Web UI of the Chef Server.
The next step would be creating a role and adding a role to a particular node.
For this we will be using knife.

Set the default editor in your knife config to start.

    $ vi .chef/knife.rb
Add the below line to the end of the 'knife.rb' file. 

`knife[:editor]="vim"`

Note: You can use any editor of your choice but here we are using 'vim'. 

So everything is set we can start creating the role. Use the below syntax to create a new role.

		$ knife role create web

		{
		  "name": "web",
		  "description": "Nginx Web server node",
		  "json_class": "Chef::Role",
		  "default_attributes": {
		  },
		  "override_attributes": {
		  },
		  "chef_type": "role",
		  "run_list": [
		    "recipe[apt]",
		    "recipe[build-essential]",
		    "recipe[nginx]"
		  ],
		  "env_run_lists": {
		  }
		}

This will open the temporary file in the text editor. Here we have to add the recipes to the run_list[]. You can even specify the other details such as default_attributes and as well as environment based run_lists[]. After adding the details save the file and exit to succesffuly create a role. You can verify this by running below command.

     $ knife role list

The next step would be applying this role to a client node. We can see the existing nodes by running below command.

    $ knife nodes list
      web_server

You wonder how this node has been created ? This node was created when we ran the knife bootstrap.

So we will edit the existing node settings by using knife.
    
		$ knife node edit web_server

		{
		  "name": "web_server",
		  "chef_environment": "_default",
		  "normal": {
		    "tags": [

		    ]
		  },
		  "run_list": [
		    "role[web]"
		  ]
		}

This will open a temporary file in a text editor. Here add the role to the run_list[].
After adding the details save and exit to apply to the node.

Login to the client node's shell using

     $ vagrant ssh <client_node_name>

From clients prompt exectue the chef-client to update the client node.

    vagrant@node $ sudo chef-client

After running this successfully you should be able to see that the recipes got executed. We can repeat this procedure for creating other nodes. 

#### Voila ! we are done.