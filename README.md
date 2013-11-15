# Getting started with Chef-Server tutorial:

## What is Vagrant ?
####You can read about it [here](http://docs.vagrantup.com/v2/getting-started/) .

## What is Chef-Server ?
####You can read about it [here](http://docs.opscode.com/chef_overview.html ).

### Step 1 : Create Virtual Machines using Vagrant

Create a file called 'Vagrantfile' in which we have to specify the configuration settings for our cluster.

In our example we are using two nodes in the cluster. A server node and a client node.
 
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

After creating a file we start our vagrant. So let's do it.

	 $ vagrant up 

It starts the process of bringing up the virtual machine. So after few seconds/minutes your vagrant should be up and runninig.

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

###Step3 : Install the cookbooks using berkshelf

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
		
	$ berks install

It installs the cookbooks which are specified in the Berksfile.

###Step 4: Initalize the knife and edit knife config file

Knife is a tool used with chef-server. We have to initailse the knife, so that it prepares the directories and files.

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

Now open your web browser and specify the FQDN or IP address of the chef-server node to access the Web UI.

#####SSH Keys :
After installation Chef Server with default settings, Chef will generate pem keys, which will be used for knife (command line tool for Chef) and Chef clients for authentication with server. We should copy keys from our Chef Server node to “.chef” directory in the project.

 	$ vagrant ssh chef_server_node

Now you will get the shell access to the chef-server node. From the server node's shell run this command.

    $ sudo cp /etc/chef-server/*.pem /vagrant/.chef/


#####Knife configuration:

Next we should create for knife configuration file (knife should know how to communicate with Chef Server):
<br>

    $ knife configure -i
        Overwrite /home/itsprdp/chef-server-demo/.chef/knife.rb? (Y/N) y
		Please enter the chef server URL: [https://Jaeger:443] https://chef.itsprdp.com
		Please enter a name for the new user: [itsprdp] admin
		Please enter the existing admin name: [admin] 
		Please enter the location of the existing admin's private key: [/etc/chef-server/admin.pem] /home/itsprdp/chef-server-demo/.chef/admin.pem
		Please enter the validation clientname: [chef-validator] 
		Please enter the location of the validation key: [/etc/chef-server/chef-validator.pem] /home/itsprdp/chef-server-demo/.chef/chef-validator.pem
		Please enter the path to a chef repository (or leave blank): 
		Creating initial API user...
<br>
As a result, you should have a file “.chef/knife.rb” with similar content:
<br>

    	log_level                :info
		log_location             STDOUT
		node_name                'admin'
		client_key               '/home/itsprdp/chef-server-demo/.chef/admin.pem'
		validation_client_name   'chef-validator'
		validation_key           '/home/itsprdp/chef-server-demo/.chef/chef-validator.pem'
		chef_server_url          'https://chef.itsprdp.com'
		syntax_check_cache_path  '/home/itsprdp/chef-server-demo/.chef/syntax_check_cache'

Let’s check knife configuration:
	
    $ knife client list

    It lists the clients
	- chef-validator
	- chef-webui
	 
###Step 5: Bootstrap first Client node

Once the Chef Server node is configured, it can be used to install Chef on one or more nodes using a Knife bootstrap. The “knife bootstrap” command is used to SSH into the target machine, and then prepare the node to allow the chef-client to run on the node. It will install the chef-client executable, generate keys, and register the node with the Chef Server node. The bootstrap operation requires the IP address or FQDN of the target node or system, the SSH credentials (username, password or identity file) for an account that has root access to the node, and (if the operating system is not Ubuntu, which is the default distribution used by knife bootstrap) the operating system running on the target system.

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
		web.node

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
	``` 
When a node runs the chef-client for the first time, it generally does not yet have 
an API client identity, and so it cannot make authenticated requests to the server. This is where the validation client—known as the chef-validator—comes in. When the chef-client runs, it checks if it has a “client_key”. If the client key does not exist, it then attempts to borrow the identity of the chef-validator to register	itself with the server (“validation_key”).

###Step 6: Cookbooks, roles and nodes on chef server

In comparison with Chef Solo, Chef Server store all information on server and use only this information for “cooking” nodes. So we should know, how to upload our roles, cookbooks and nodes on server. First of all we should install vender cookbooks localy by using Berkshelf.

     $ berks install --path cookbooks

As you can see dependencies downloaded automatically. Right now we have this cookbooks only in our local directory “cookbooks”. Let’s upload its to Chef Server. To solve this task we can use knife:

    $ knife cookbook upload --all --cookbook-path cookbooks
 
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

		$ knife role create <role_name>

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

	You wonder how this node has been created ? This node was created when we ran the knife bootstrap.

So we will edit the existing node settings by using knife.
    
		$ knife node edit <node_name>

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