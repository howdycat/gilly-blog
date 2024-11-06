+++
title = 'Setting up a self-hosted Server with Miniflux, start to finish'
date = 2024-11-06
draft = false
+++

As I explained in [[Ditch Subscription Apps and Self Host]], I suggested that self-hosting is a great alternative to the rise of subscription-based apps. The biggest barrier to self  hosting is the setup; however, once it is done, the opportunities are limitless. Here I will explain how I setup my server, connected it to a domain, configured nginx, and installed my first service (miniflux). I hope this gives you a starting point for your self-hosted journey.


# Get a Server
To self-host, you need a server. You can use your own machine, but here I will focus on how I rented one.

1. Pick a provider and create an account. I'm using [Hetzner Cloud](https://www.hetzner.com/cloud/) because they are cheap. 
	Hetzner will require you to use your ID to verify your identity. If this bothers you, you may want to consider a different provider.

2. Follow the [instructions for your provider](https://docs.hetzner.com/cloud/servers/getting-started/creating-a-server/) to create a server. For Hetzner, this involves creating a **project** with a **server**. In the server creation, you will be able to choose a **location**, **OS**, **CPU**, **IPs**, **volumes**, and **ssh keys**.
	- For CPU configuration, I recommend going with a shared vCPU at first. You'll be able to see your estimated price per month as you setup the server.
	- It is **necessary** that you get a **public IPv4**. Without it, I found that accessing my server was very difficult.

3. Once your server is live, you can access it via the web dashboard. If you have added an SSH key, **you can skip these steps**. Otherwise, access to your server will need to be set up.
	- On Hetzner, you'll need to navigate to the 'Rescue' tab of your server dashboard and reset the root password.
	- Once this is done, you can go to the 'Actions' dropdown, run the web console, and log in as root using the provided password.
	- Here, you should most certainly [add a new user account](https://www.liquidweb.com/help-docs/adding-users-and-granting-root-privileges-in-linux/), generate an ssh-key on your local machine, and [add the contents of the public ssh key to the new user account's .ssh/authorized_keys](https://community.hetzner.com/tutorials/add-ssh-key-to-your-hetzner-cloud)

4. Test your connection. From your local command line, you should be able to execute the following command to connect to the server:
	`ssh <username you created>@<ip of your server>`

# Get a Domain Name
Remembering the ip of your server is complicated. We can simplify this by getting a domain name. For this, I am using Cloudflare. Cloudflare has some... complications, so bare with me. If you choose another domain provider, chances are this section will be more straightforward.

1. [Buy your domain](https://www.cloudflare.com/products/registrar/). It can be anything you want.
2. [Set up your DNS records](). 
	- You'll want one entry to be like the one below, with a type of A, your domain, and the ipv4 address from Hetzner
		![[{DDD0B061-0CEB-49D9-867C-EEEC787B7D8D}.png]]
	- Optionally, you can add subdomains if you intend to host multiple services. For this, use the same IPv4 address and the name of the subdomain. So if I wanted rss.gilly.garden, I would put 'rss' in the name field.
	- Finally, you can set up an 'ssh' subdomain in the same way. For this to work, you'll have to disable the proxy. This will allow you to ssh into your server using the subdomain.

# Install Miniflux
Now that our machine is setup, let's install [Miniflux](https://miniflux.app/). From this point on, you may face errors - if you aren't sure how to fix them, check a few things:
- Does your user account have the proper permissions?
- Are you running the command as sudo?
- Google! It is impossible for me to cover all the errors you may face, but many I was able to solve via stack overflow and google - I am confident you will be able to solve them too :)

Miniflux can be installed with Docker or as package. For my case, I will not be using Docker, but if you are more familiar with that, by all means go ahead.

Follow the [instructions](https://miniflux.app/docs/installation.html) provided by Miniflux. Specifically, make sure you have a [database configured](https://miniflux.app/docs/database.html). Then, using your package manager, install the `miniflux` package. 

# Install and Configure nginx
Miniflux is installed, but it is not yet exposed to the web. Nginx is a web server. Basically, it'll allow you to expose your services to the web. To install it, you can use your distro's package manager. More information on the installation can be found [here](http://nginx.org/en/docs/install.html), but it is pretty straightforward.

The less straightforward part is the configuration. In order to expose miniflux to your domain, you need to create a nginx configuration.

1. First, create a file with the name of your domain (i.e `gilly.garden`) in `/etc/nginx/sites-enabled`. You'll probably need sudo permissions for this.
2. In this file, you'll need to paste the following config with the necessary modifications to fit your server.  I used the [instructions from nginx](https://miniflux.app/docs/howto.html#reverse-proxy) to generate this config.

```nginx
# This block will setup HTTPS redirecting. Chances are, your server will use https
server {
        listen 80;
        listen [::]:80;
        server_name _;
        return 301 https://$host$request_uri;
        }
        
# This block will link your domain to your miniflux service.
server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        ssl_certificate         /etc/ssl/cert.pem;
        ssl_certificate_key     /etc/ssl/key.pem;

        server_name <domain, i.e. rss.gilly.garden>

        location / {
                proxy_pass http://127.0.0.1:8080;
                proxy_redirect off;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
		        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		        proxy_set_header X-Forwarded-Proto $scheme;
        }
}
```

# Start services  and deploy!
In order to access Miniflux, you'll need to make sure the necessary services are running.

```
sudo service postgresql start
sudo service miniflux start
sudo systemctl start nginx
```

You should be able to access Miniflux at your domain. From here, you can login with the user you set up during the database configuration.

