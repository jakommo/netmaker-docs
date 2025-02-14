===============
Quick Install
===============

This quick install guide is an **opinionated** guide for getting up and running with Netmaker as quickly as possible.

If you are just looking to trial netmaker, we **highly recommend** using the 5-minute installer of Netmaker in the README on GitHub: https://github.com/gravitl/netmaker 

The following is a guided version of that script, with the option to add a custom domain (instead of nip.io):  

Assumptions
============================

We assume for this installation that you want all of the Netmaker features enabled, and you want your server to be accessible from anywhere.

This instance will not be HA. However, it should comfortably handle 100+ concurrent clients and support the most common use cases.

If you are deploying for a business or enterprise use case and this setup will not fit your needs, please contact us: https://www.netmaker.org/contact

By the end of this guide, you will have Netmaker installed on a public VM linked to your custom domain, secured behind a Caddy reverse proxy.

For information about deploying more advanced configurations, see the :doc:`Advanced Installation <./server-installation>` docs. 


0. Prerequisites
==================
-  **Virtual Machine**
   
   - Preferably from a cloud provider (e.x: DigitalOcean, Linode, AWS, GCP, etc.)
   
   - We **highly recommend** that Netmaker be deployed in a dedicated networking environment. It should not share a local network with the clients which it will be managing. This can cause routing issues.

   - We do not recommend Oracle Cloud, as VM's here have been known to cause network interference.

   - The machine should have a public, static IP address 
   
   - The machine should have at least 1GB RAM and 1 CPU (2GB RAM preferred for production installs)
   
   - 2GB+ of storage 
   
   - Ubuntu  20.04 Installed

- **Domain**

  - A publicly owned domain (e.x. example.com, mysite.biz) 
  - Permission and access to modify DNS records via DNS service (e.x: Route53)
  - **Important Note:** Some of our users like use Cloudflare for DNS. Cloudflare has limitations on subdomains you must be aware of, which can cause issues once Netmaker is deployed.

1. Prepare DNS
================

Create a wildcard A record pointing to the public IP of your VM. As an example, \*.netmaker.example.com. Alternatively, create records for these specific subdomains:

- dashboard.domain

- api.domain

- broker.domain

2. Install Dependencies
========================

.. code-block::

  ssh root@your-host
  sudo apt-get update
  sudo apt-get install -y docker.io docker-compose wireguard

At this point you should have all the system dependencies you need.
 
3. Open Firewall
===============================

Make sure firewall settings are set for Netmaker both on the VM and with your cloud security groups (AWS, GCP, etc). 

Make sure the following ports are open both on the VM and in the cloud security groups:

- **443 (tcp):** for Dashboard and REST API  
- **80 (tcp):** for LetsEncrypt  
- **ICMP (optional:** as of 0.14.0 this should not be necessary, but previous versions require nodes to ping the server over public IP  
- **53 (udp and tcp):** for CoreDNS - This is no longer necessary as of 0.10.0, but in some cases you may still want to use CoreDNS externally.  
- **51821-518XX (udp):** for WireGuard - Netmaker needs one port per network, starting with 51821, so open up a range depending on the number of networks you plan on having. For instance, 51821-51830.  
- **8883 (tcp):** for MQ Broker

.. code-block::

  sudo ufw allow proto tcp from any to any port 443 && sudo ufw allow 53/udp && sudo ufw allow 53/tcp && sudo ufw allow 51821:51830/udp && sudo ufw allow 8883/tcp

It is also important to make sure the server does not block forwarding traffic (it will do this by default on some providers). To ensure traffic will be forwarded:

.. code-block::

  iptables --policy FORWARD ACCEPT



**Again, based on your cloud provider, you may additionally need to set inbound security rules for your server (for instance, on AWS). This will be dependent on your cloud provider. Be sure to check before moving on:**
  - allow 443/tcp from all
  - allow 80/tcp from all
  - allow 8883/tcp from all
  - (optional) allow 53/udp and 53/tcp from all
  - (optional) allow icmp from all
  - allow 51821-51830/udp from all

4. Install Netmaker
========================

Prepare Docker Compose 
------------------------

.. code-block::

  ip route get 1 | sed -n 's/^.*src \([0-9.]*\) .*$/\1/p'

Now, insert the values for your base (wildcard) domain, public ip.

.. code-block::

  wget -O docker-compose.yml https://raw.githubusercontent.com/gravitl/netmaker/master/compose/docker-compose.yml
  sed -i 's/NETMAKER_BASE_DOMAIN/<your base domain>/g' docker-compose.yml
  sed -i 's/SERVER_PUBLIC_IP/<your server ip>/g' docker-compose.yml

Generate a unique master key and insert it:

.. code-block::

  tr -dc A-Za-z0-9 </dev/urandom | head -c 30 ; echo ''
  sed -i 's/REPLACE_MASTER_KEY/<your generated key>/g' docker-compose.yml

You may want to save this key for future use with the API.

Prepare Caddy
------------------------

.. code-block::

  wget -O /root/Caddyfile https://raw.githubusercontent.com/gravitl/netmaker/master/docker/Caddyfile

  sed -i 's/NETMAKER_BASE_DOMAIN/<your base domain>/g' /root/Caddyfile
  sed -i 's/YOUR_EMAIL/<your email>/g' /root/Caddyfile

Prepare MQ
------------------------


You must retrieve the MQ configuration file for Mosquitto.

.. code-block::

  wget -O /root/mosquitto.conf https://raw.githubusercontent.com/gravitl/netmaker/master/docker/mosquitto.conf


Start Netmaker
----------------

``sudo docker-compose up -d``

navigate to dashboard.<your base domain> to begin using Netmaker.

To troubleshoot issues, start with:

``docker logs netmaker``

Or check out the :doc:`troubleshoooting docs <./troubleshoot>`.
