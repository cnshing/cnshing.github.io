---
layout: post
title:  "Exposing your VSCode web server via reverse proxy"
date:   2023-10-20 11:39:14 +0000
categories: software-development
tags: [server, vscode, webapp, proxy, dns, tls, encryption, nginx, htpasswd]
---
{% include image_finder.html %}

[Previously]({% link _posts/2023-10-18-how-to-run-vscode-on-your-browser.markdown %}), we talked about why VSCode on the browser can be an incredibly useful tool for your developer environment and how a VSCode dedicated server can be implemented. However, this basic installation of our code server has not been properly configured to be accessed securely through the browser. 

Today we will move forward and start securing our access to VSCode through a commonly utilized network architecture component referred to as a reverse proxy.

<!--more-->

## Proxy, Proxy, Proxy

Before we start delving into reverse proxies, let us first talk about what a VPN is, as understanding those will help contextualize the term "reverse" in "reverse proxy". A VPN is an application of a proxy by placing something between you and whatever you are trying to access on the Internet. Whatever is placed in the middle is then responsible for protecting your data by making it completely unreadable to anyone except for you, the VPN provider, and your app of choice. So whenever you are using a VPN, you are essentially using a proxy, a device acting as a representative of someone else by placing itself between origin and destination to intercept communications. In fact, if you were to go to google.com with a VPN, Google would not see your IP address but the VPN proxy's identity instead.  

![VPN Illustration](https://www.hostinger.com/tutorials/wp-content/uploads/sites/2/2022/05/what-is-a-vpn-01.webp)
*How a VPN works. The green locks indicate when the data has been made undecipherable through encryption. Courtesy of [hostinger](https://www.hostinger.com/tutorials/what-is-vpn).*

The first question would be, how does a proxy relate to a reverse proxy exactly? Well, consider that in our proxy scenario, there are always at least three individual agents: the client sending the data, the proxy intercepting the data, and the server receiving the data from our proxy. In our previous case, the proxy was represented in the client's place by forwarding its data to the server. However, unlike a forward proxy, a reverse proxy "flips" or *reverses* representation by acting as an intermediary for the *server*, not the client. Now, any client that wishes to talk to the server can only indirectly do so by communicating to the reverse proxy acting on behalf of the server.

> a reverse proxy "flips" or *reverses* representation by acting as an intermediary for the *server*, not the client.

## Reverse Proxy in the context of VSCode

With some understanding of what a reverse proxy is, our second question would be on how a reverse proxy can enable us to access VSCode conveniently and securely. There are a couple of reasons for why a reverse proxy is an ideal choice for this goal:

* An HTTP address is almost exclusively an IP address, while an HTTPS address is almost exclusively a domain. A reverse proxy can hit two birds with one stone by converting our random, hard to read number HTTP VSCode address into an easily accessible and HTTP-secured domain name. 
* Most modern reverse proxies are bundled with additional tools like authentication and plenty of support. You can even manage a reverse proxy over a [web GUI](https://nginxproxymanager.com/)!

Here is an overall view of our reverse proxy:
![Reverse Proxy]({{IMG}}Reverse_Proxy.svg)

When a browser wishes to enter VSCode via our domain, the computer will attempt to resolve the domain's IP address through DNS[^1]. We will then configure the DNS response such that the browser will attempt to communicate to the reverse proxy instead of directly through the code server by supplying the IP address with the reverse proxy's own IP of `206.190.175.42`. This reverse proxy is responsible for sending and receiving on the code server's behalf to the rest of the Internet. Since the reverse proxy and the code server are housed in the same machine, the reverse proxy can freely see and encrypt the previously private HTTP traffic into secure, public HTTPS traffic in a process called [SSL Termination](https://en.wikipedia.org/wiki/TLS_termination_proxy). Lastly, the reverse proxy will be configured such that the user will be blocked from entering the website unless proper credentials are supplied.

Now that we understand what is supposed to theoretically happen with a reverse proxy, let us implement it.

## Requirements
* Some basic knowledge of using and accessing a console. 
* A SSH accessible Debian machine with openvscode-server up and running.
* The public IP of the code server.
* A DNS configurable domain name.

### Variables
* `${DOMAIN_NAME}`: Your domain name, without any `https://``. Example: google.com
* `${DOMAIN_EMAIL}`: The email you used to register your domain name.
* `${USERNAME}`: Username of choice used to authenticate with openvscode-server.

### (Optional) Registering a new domain

Since this is for people who don't have a domain, anyone who already has one can feel free to skip this section.

Obtaining a new domain can vary in difficulty and cost. For example, you can purchase a domain for roughly $10/year via [Cloudflare](https://www.cloudflare.com/products/registrar/), or attempt to get a free one via [GetFreeDomain](https://www.getfreedomain.name/) with some restrictions and caveats. Personally, I use [DynuDNS](https://www.dynu.com/), one of many free dynamic DNS providers allowing you to link a domain to an IP, which I feel is a perfect use-case for this application.

Each domain provider has a different UI for domain management, so your mileage may vary. In this specific guide, DynuDNS will be used as the prime example.

In DynuDNS, go to the "[Add Dynamic DNS](https://www.dynu.com/en-US/ControlPanel/AddDDNS)" page and enter your desired domain details like so:

![Add Dynamic DNS]({{IMG}}Create_Domain.png)

### Altering DNS records

Create "A Record" of the domain name with the code-servers public IP:

![Add A Record]({{IMG}}Add_A_Record.png)

### Installing the reverse proxy

In your Debian machine, install the following dependencies for reverse proxy and authentication software.

```bash
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx nginx apache2-utils
```

Add a reverse proxy configuration to `/etc/nginx/sites-available/openvscode-server`.

```bash
sudo tee -a /etc/nginx/sites-available/openvscode-server > /dev/null <<-EOF
server {
    server_name ${DOMAIN_NAME};
    location / {
      proxy_pass http://localhost:3000; #openvscode-server IP:port
      proxy_read_timeout 10;
      proxy_connect_timeout 10;
      proxy_set_header Host $http_host; #Websockets
      proxy_set_header Upgrade $http_upgrade; 
      proxy_set_header Connection upgrade;
      proxy_set_header Accept-Encoding gzip;
      auth_basic "Authentication for openvscode-server";
      auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
EOF
```

Our reverse proxy configuration will hand off anything from our code server local IP `localhost:3000` to `${DOMAIN_NAME}`, refreshing its connection every `10` seconds. There are web sockets enabled since the openvscode-server requires it, and mandatory authentication whose secrets are located in `/etc/nginx/.htpasswd` so that you are the only one with a password to access it. Speaking of which, let us attempt to create an identity with the following command:

```bash
htpasswd -c /etc/nginx/.htpasswd ${USERNAME}
```

Then enter your password when prompted to create an account.

Our final step will be to enable and start the reverse proxy with TLS termination.

```bash
sudo ln -s /etc/nginx/sites-available/openvscode-server /etc/nginx/sites-enabled/openvscode-server
sudo certbot --non-interactive --redirect --agree-tos --nginx -d ${DOMAIN_NAME} -m ${DOMAIN_EMAIL}
```

If all goes well, you should see a response message similar to this:

```bash
root@debian:~# sudo certbot --non-interactive --redirect --agree-tos --nginx -d ${DOMAIN_NAME} -m ${DOMAIN_EMAIL}
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Certificate not yet due for renewal
Deploying certificate
Successfully deployed certificate for code-server.loseyourip.com to /etc/nginx/sites-enabled/openvscode-server
Congratulations! You have successfully enabled HTTPS on https://code-server.loseyourip.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

To test authentication, try visiting the HTTPS domain `https://${DOMAIN_NAME}`. You should see a native browser login popup asking for your credentials. You should be able to log in with your freshly created username and password and enter VSCode as you did before with an SSH tunnel. 

Congratulations! You have successfully created a secure and publicly accessible VSCode server! You may now reap the benefits of being able to code from any browser with just the Internet.


[^1]: Domain Name System, or DNS, is a complex mapping of domain names to IP addresses, allowing anyone to find the IP address of a particular domain. DNS is commonly referred to as "the phonebook of the Internet".