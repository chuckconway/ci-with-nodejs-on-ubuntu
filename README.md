I went through blood, sweat and tears to bring this to you. I suffered the scorching heat of Death Valley and summited the peaks of Mount McKinley. I’ve sacrificed much. 

Much of content shared in the post is not my original work. Where I can, I link back to the original work. 

This article assumes you can get around Linux. 

I could not find a comprehensive guide on hosting and managing Nodejs applications on Ubuntu in a production capacity. I’ve pulled together multiple articles on the subject. By the end of this article I hope you’ll be able to setup up your own Ubuntu server and have Nodejs deploying via a continuous integration server.

## Environment ##
I am using TeamCity on Windows which then deploys code from GitHub to Ubuntu hosted on AWS.

## Technologies ##
For this article I used the following technologies:

- Ubuntu 14.04 on AWS
- Plink 0.64
- TeamCity 9.1
- GitHub
- Nginx 1.9.3

## Setting up Ubuntu ##
I’m not going into detail here. Amazon Web Services (AWS) makes this pretty easy to do. It doesn’t matter where it’s at or if it’s on your own server. 

I encountered a few gotchas. First, make sure port 80 is opened. I made the foolish mistake of trying to connect with port 80 closed. Once I discovered my mistake, I felt like a rhinoceros's ass. 

## Installing NodeJs From Source ##

Nodejs is a server technology using Google’s V8 javascript engine. Since it’s release in 2010, its become widely popular.

The following instructions originally came from a [Digital Ocean](www.digitalocean.com) [post](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-node-js-application-for-production-on-ubuntu-14-04)).

You always have the option to install Nodejs from the apt-get, but it will be a few versions behind. To get the latest bits, install Nodejs from the source.

At this send of this section we will have downloaded the latest stable version of node (as of this article), we will have build the source and installed Nodejs.

Log into your server. We’ll start by updating the package lists.

	sudo apt-get update

I’m also suggesting that you upgrade all the packages. This is not necessary, for Nodejs but it is good practice to keep your server updated.

	sudo apt-get upgrade

Your server is all up to date. It’s time download the source.

	cd ~

As of the writing 12.7 is the latest stable release of Nodejs. Check out [nodejs.org](https://nodejs.org/) for the latest version.

	wget https://nodejs.org/dist/v0.12.7/node-v0.12.7.tar.gz

Extract the archive you’ve downloaded.

	tar xvf node-v*

Move into the newly created directory

	cd node-v*

Configure and build Nodejs.

	./configure

	make

Install Nodejs

	sudo make install

To remove the downloaded and the extracted files. Of course, this is optional.

	cd ~

	rm -rf node-v*

Congrats! Nodejs is now installed! And it wasn’t very hard.


## Setting up Nginx ##
[Source](http://stackoverflow.com/questions/5009324/node-js-nginx-what-now)

Nodejs can act as a web server, but it’s not what I would want to expose to the world. An industrial, harden, feature rich web server is better suited for this. I’ve turned to Nginx for this task. 

It’s a mature web server with the features we need. To run more than one instance of Nodejs, we’ll need to port forwarding.

You might be thinking, why do we need more than one instance of Nodejs running at the same-time. That’s a fair question… In my scenario, I have one server and I need to run DEV, QA and PROD on the same machine. Yeah, I know not ideal, but I don’t want to stand up 3 servers for each environment.

To start let’s install Nginx

	sudo -s

	add-apt-repository ppa:nginx/stable

	apt-get update 

	apt-get install nginx


Once Nginx is has successfully installed we need to set up on the domains. I’m going to assume you’ll want to have each of your sites on it’s own domain/sub domain. If you don’t and want to use different sub-folders, that’s doable and very easy to do. I am not going to cover that scenario here. There is a ton of documentation on how to do that. There is very little documentation on setting up different domains and port forwarding to the corresponding Nodejs instances. This is what I’ll be covering.

Now that Nginx is installed, create a file for yourdomain.com at `/etc/nginx/sites-available/`

	sudo nano /etc/nginx/sites-available/yourdomain.com

Add the following configuration to your newly created file


	# the IP(s) on which your node server is running. I chose port 9001.
	upstream app_myapp1 {
	    server 127.0.0.1:9001;
	    keepalive 8;
	}
	
	# the nginx server instance
	server {
	    listen 80;
	    server_name yourdomain.com;
	    access_log /var/log/nginx/yourdomain.log;
	
	    # pass the request to the node.js server with the correct headers
	    # and much more can be added, see nginx config options
	    location / {
	        proxy_http_version 1.1;
	        proxy_set_header X-Real-IP $remote_addr;
	        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	        proxy_set_header Host $http_host;
	        proxy_set_header X-NginX-Proxy true;
	
	        proxy_pass http://app_myapp1;
	
	    }
	 }

Make sure you replace "yourdomain.com" with your actual domain. Save and exit your editor.

Create a symbolic link to this file in the sites-enabled directory.

	cd /etc/nginx/sites-enabled/ 
	
	ln -s /etc/nginx/sites-available/yourdomain.com yourdomain.com

To test everything is working correctly, create a simple node app and save it to `/var/www/yourdomain.com/app.js` and run it.

Here is a simple nodejs app if you don’t have one handy.

	var http = require('http');
	
	http.createServer(function (req, res) {
	    res.writeHead(200, {'Content-Type': 'text/plain'});
	    res.end('Hello World\n');}).listen(9001, "127.0.0.1");
	console.log('Server running at http://127.0.0.1:9001/');

Let’s restart Nginx.

	sudo /etc/init.d/nginx restart

Don’t forget to start your Nodejs instance, if you haven’t already.

	cd /var/www/yourdomain/ && node app.js

If all is working correctly, when you navigate to yourdomain.com you’ll see “Hello World.”

To add another domain for a different Nodejs instance your need to repeat the steps above. Specifically you’ll need to change the upstream name, the port and the domain in your new Nginx config file. The proxy_pass address must match the upstream name in the nginx config file. Look at the upstream name and the proxy_pass value and you’ll see what I mean.

To recap, we’ve installed NodeJS from source and we just finished installing Nginx. We’ve configured and tested port forwarding with Nginx and Nodejs 

## Installing PM2 ##

You might be asking “What is PM2?” as I did when I first heard about. PM2 is a process manager for Nodejs applications. Nodejs doesn’t come with much. This is part of it’s appeal. The downside to this, is well, you have to provide the layers in front of it. PM2 is one of those layers.

PM2 manages the life of the Nodejs process. When it’s terminated, PM2 restarts it. When the server reboots PM2 restarts all the  Nodejs processes for you. It also has extensive development lifecycle process. We won’t be covering this aspect of PM2. I encourage you to read well [written documentation](https://github.com/Unitech/PM2/blob/master/ADVANCED_README.md).

Assuming you are logged into the terminal, we’ll start by installing PM2 via NPM. Npm is Nodejs package manager (npm). It was installed when you installed Nodejs.

	sudo npm install pm2 -g
  
That’s it. PM2 is now installed.

## Using PM2 ##

PM2 is easy to use. 

The hello world for PM2 is simple.

	pm2 start hello.js

This adds your application to PM2’s process list. This list is output each time an application is started.

![](/images/content/2015/08/03/pm2list.png)


In this example there are two Nodejs applications running. One called api.dev and api.pre.

PM2 automatically assigns the name of the app to the “App name” in the list.

Out of the box, PM2 does not configure itself to startup when the server restarts. The command is different for the different flavors of Linux. I’m running on Ubuntu, so I’ll execute the Ubuntu command.

	pm2 start ubuntu

We are not quite done yet. We have to add a path to the PM2 binary. Fortunately, the output of the previous command tells us how to do that.

 Output:

    [PM2] You have to run this command as root
    [PM2] Execute the following command :
    [PM2] sudo env PATH=$PATH:/usr/local/bin pm2 startup ubuntu -u sammy
	Run the command that was generated (similar to the highlighted output above) to set PM2 up to start on boot (use the command from your own output):

	 sudo env PATH=$PATH:/usr/local/bin pm2 startup ubuntu -u sammy

Examples of other PM2 usages (optional)

Stopping an application by the app name

	pm2 stop example

Restarting by the app name

	pm2 restart example

List of current applications managed by PM2

	pm2 list

Specifying a name when starting a process. If you call, PM2 uses the javascript file as the name. This might not work for you. Here’s how to specify the name.

	pm2 start www.js --name api.pre

That should be enough to get you going with PM2. To learn more about PM2’s capabilities, visit the [GitHub Repo](https://github.com/Unitech/PM2).


## Setting up and Using Plink ##

You are probably thinking, “What in the name of Betsey's cow is Plink?” At least that’s thought. I’m still not sure what to think of it. I’ve never seen anything like it. 

You ever watched the movie Wall-e? Wall-e pulls out a spork. First he tries to put it with the forks, but it doesn’t fix and then he tries to put it with the spoons, but it doesn’t fit. Well that’s Plink. It’s a cross between Putty (SSH) and the Windows Command Line. 

[Plink](http://the.earth.li/~sgtatham/putty/0.52/htmldoc/Chapter7.html) basically allows you to run bash commands via the Windows command line while logged into a Linux (and probably Unix) shell.

Start by downloading [Plink](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html). It’s just an executable. I recommend putting it in `C:/Program Files (x86)/Plink`. We’ll need to reference it later.

If you are running an Ubuntu instance in AWS. You’ll already have a cert setup for Putty (I’m assuming you are using Putty). 

If you are not, you’ll need to ensure you have a compatible ssh cert for Ubuntu in [AWS](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html). 

If you are not using AWS, you can specify the username and password in the command line and won’t to worry about the ssh certs.

Here is an example command line that connects to Ubuntu with Plink.

	"C:\Program Files (x86)\Plink\plink.exe" -ssh ubuntu@xx.xx.xx.xx -i "C:\Program Files (x86)\Plink\ssh certs\aws-ubuntu.ppk" 

![](![](/images/content/2015/08/03/plink.png))


This might be getting ahead of ourselves, but to run an ssh script on the Ubuntu server we add the complete path to the end of the Plink command.

	"C:\Program Files (x86)\Plink\plink.exe" -ssh ubuntu@xx.xx.xx.xx -i "C:\Program Files (x86)\Plink\ssh certs\aws-ubuntu.ppk" /var/www/deploy-dev-ui.sh

And that, dear reader, is Plink. 

## Understanding NODE_ENV ##

 `NODE_ENV` is an environment variable made popular by expressjs. Before your start the node instance, set the `NODE_ENV` to the environment. In the code you can load specific files based on the environment.

    Setting NODE_ENV
    Linux & Mac: export NODE_ENV=PROD
    Windows: set NODE_ENV=PROD


The environment variable is retrieved inside a Nodejs instance by using `process.env.NODE_ENV`.

example
	var environment = process.env.NODE_ENV

or with expressjs

	app.get('env')

*Note: app.get(‘env’) defaults to “development”.


Bringing it all together

`Nodejs`, `PM2`, `Nginx` and `Plink` are installed and hopefully working. We now need to bring all these pieces together into a continuous integration solution.

Clone your GitHub repository in `/var/www/yourdomain.com`. Although SSH is more secure than HTTPS, I recommend using HTTPS.  I know this isn’t ideal, but I couldn’t get Plink working with GitHub on Ubuntu. Without going into too much detail Plink and GitHub SSH cert formats are different and calling GitHub via Plink through SSH didn’t work. If you can figure out the issue let me know! 

To make the GitHub pull handsfree, the username and password will need to be a part of the origin url.

Here’s how you set your origin url. Of course you’ll need to substitute your information where appropriate. 

	git remote set-url origin  https://username:password@github.com/username/yourdomain.git

Clone your repository.
	cd /var/www/yourdomain.com
	git clone https://username:password@github.com/username/yourdomain.git .

Note, that if this directory is not completely empty, including hidden files Git will not clone the repo to this directory.

To find hidden files in the directory run this command

	ls -a

For the glue, we are using a shell script. Here is a copy of my script.

	#!/bin/bash
	
	echo "> Current PM2 Apps"
	pm2 list
	
	echo "> Stopping running API"
	pm2 stop api.dev
	
	echo "> Set Environment variable."
	export NODE_ENV=DEV
	
	echo "> Changing directory to dev.momentz.com."
	cd /var/www/yourdomain.com
	
	echo "> Listing the contents of the directory."
	ls -a
	
	echo "> Remove untracked directories in addition to untracked files."
	git clean -f -d
	
	echo "> Pull updates from Github."
	git pull
	
	echo "> Install npm updates."
	sudo npm install
	
	echo "> Transpile the ECMAScript 2015 code"
	gulp babel
	
	echo "> Restart the API"
	pm2 start transpiled/www.js --name api.dev
	
	echo "> List folder directories"
	ls -a
	
	echo "> All done."

I launch this shell script with TeamCity, but you can launch with anything.

Here is the raw command.

	"C:\Program Files (x86)\Plink\plink.exe" -ssh ubuntu@xx.xx.xx.xx -i "C:\Program Files (x86)\Plink\ssh certs\aws-ubuntu.ppk" /var/www/deploy-yourdomain.sh
	exit
	>&2


That’s it. 

## In Closing ##

This process has some rough edges... I hope to polish those edges in time. If you have suggestions please leave them in the comments.

This document is in my GitHub Repository. Technologies change, so if you find an error please update it. I will then update this post. 