---
layout: post
title:  "How to run VSCode on your browser"
date:   2023-10-18 16:00:00 +0000
categories: software-development
tags: [server, vscode, webapp, ssh, port-forward, systemd]
feature_image: https://user-images.githubusercontent.com/372735/131794918-d6602646-4d67-435b-88fe-620a3cc0a3aa.png
---

Virtual Studio Code is a productive programming tool offering extensible flexibility, look-and-feel personalization, and hassle-free accessibility making the text editor an appealing option for Computer Science students and full-time professional developers alike. However, there can be a lot of annoying friction pertaining to managing VSCode, especially when the time comes to start migrating to a new PC.

Fortunately, there are already plenty of solutions you can choose to make these pain points less painful and give you more time to be productive:

<!--more-->

| Solutions                                                                   | Purpose                                                                               |
|-----------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| [Setting Sync](https://code.visualstudio.com/docs/editor/settings-sync)     | Quickly setup your favorite preferences and customizations on new VSCode installs     |
| [Remote Tunnel](https://code.visualstudio.com/docs/remote/tunnels)          | Remotely access VSCode securely through a tunnel                                      |
| [VS Code for the Web](https://code.visualstudio.com/docs/editor/vscode-web) | Remotely access VSCode through the browser                                            |
| [GitHub Codespaces](https://github.com/features/codespaces)                 | Github Integrated prepared VSCode developer environments through the browser          |

Our solution of choice will be to implement a browser-accessible VSCode on a server. In other words, we will use a separate, dedicated machine to host a VSCode instance where opening VSCode is as simple as typing a URL in the browser. To see if this is right for you, check out the implementation's list of pros and cons:

### Pros

1. Practically anything with a browser and internet can open VSCode. This means even an iPad can be used to code from anytime, anywhere.

2. Local-agonistic management of VSCode. Losing your PC won't mean halting progress and losing your files, and all of your preferences and settings are there independent of the browser device. 

3. Tons of flexibility in how you want to host the server, from a dedicated hosting service(I use [DigitalOcean](https://www.digitalocean.com/)) to even a used laptop.

### Cons

1. Internet constrained performance. If you have even an okay amount of bandwidth and good latency, this shouldn't be a problem. But if you're trying to connect to the server located somewhere in Point Nemo, the lag will probably be too much to bear.

2. Considerable learning curve. For those who are inexperienced, you're gonna have to learn to set up a server for the first time, and depending on how hard you screw up you might accidentally give the whole world access to your machine by exposing the page without an required password. If you really don't want to learn how to create your own server, you can achieve roughly the same result by remote tunneling through [vscode.dev](https://www.vscode.dev), allowing you to access your own existing computer's VSCode through the browser.

3. Inaccessible GPUs. This can be a deal-breaker for those applying A.I. models. VSCode's performance is as good as the server hardware can support, so if your development use case needs the "big guns", then it will be both practical and cost-efficient to use whatever local computer you have in hand. 

4. The Desktop and Web versions are not perfectly 1-to-1 due to their inherent target differences. This could mean a specific niche extension may not be available or function weirdly. See [limitations](https://code.visualstudio.com/docs/editor/vscode-web#_limitations) for details.

An excellent example where coding over the web can be especially helpful is web development, as website changes can be ephemerally accessed through the same public IP allowing other collaborators to see these changes in real time. 

## Requirements
* Some basic knowledge of using and accessing a console. 
* A SSH accessible Debian machine with at least 0.5GB RAM

### Variables
These variables will make configuring and installing VSCode much more versatile.
* `${OPENVSCODE_SERVER_VERSION}`: The [openvscode-server](https://github.com/gitpod-io/openvscode-server/releases) version of choice.
* `${OPENVSCODE_SERVER_DIR}`: The directory where VSCode will be installed.


## Steps

From the Debian instance, download and extract [openvscode-server](https://github.com/gitpod-io/openvscode-server), the server software responsible for hosting VSCode.

```bash
wget "https://github.com/gitpod-io/openvscode-server/releases/latest/download/openvscode-server-v${OPENVSCODE_SERVER_VERSION}-linux-x64.tar.gz"
tar -xzf "openvscode-server-v${OPENVSCODE_SERVER_VERSION}-linux-x64.tar.gz" --one-top-level -C "${OPENVSCODE_SERVER_DIR}"
```

Create a systemd file to allow the VSCode server to start on boot.

```bash
sudo tee -a /etc/systemd/system/openvscode-server.service > /dev/null <<-EOF
[Unit]
Description=openvscode-server
After=network.target

[Service]
Type=exec
ExecStart=${OPENVSCODE_SERVER_DIR}/bin/openvscode-server --without-connection-token
Restart=always

[Install]
WantedBy=default.target
EOF
```

Restart the systemd daemon to apply changes and to start the service.
```bash
systemctl daemon-reload
sudo systemctl enable openvscode-server && sudo systemctl start openvscode-server
```

Let us apply a couple of sanity-checks to confirm if the VSCode server is actually running properly. Firstly, let's check to see if systemctl recognizes the recently created openvscode-server service.

```bash
root@debian:~# sudo systemctl status openvscode-server
● openvscode-server.service - openvscode-server
     Loaded: loaded (/etc/systemd/system/openvscode-server.service; enabled; preset: enabled)
     Active: active (running) since Wed 2023-10-18 23:36:06 UTC; 23h ago
   Main PID: 55495 (sh)
      Tasks: 99 (limit: 503)
     Memory: 303.8M
        CPU: 43min 55.363s
     CGroup: /system.slice/openvscode-server.service
             ├─55495 sh /root/openvscode-server/bin/openvscode-server --without-connection-token
             ├─55499 /root/openvscode-server/node /root/openvscode-server/out/server-main.js --without-connection-token
             ├─55603 /root/openvscode-server/node /root/openvscode-server/out/bootstrap-fork --type=ptyHost --logsPath /root/.openvscode-server/data/logs/20231018T233606
             ├─55657 /usr/bin/bash --init-file /root/openvscode-server/out/vs/workbench/contrib/terminal/browser/media/shellIntegration-bash.sh
             ├─55907 /usr/bin/bash --init-file /root/openvscode-server/out/vs/workbench/contrib/terminal/browser/media/shellIntegration-bash.sh
             ├─67198 /root/openvscode-server/node /root/openvscode-server/out/bootstrap-fork --type=fileWatcher
             ├─91469 /root/openvscode-server/node --dns-result-order=ipv4first /root/openvscode-server/out/bootstrap-fork --type=extensionHost --transformURIs --useHostProxy=false
             ├─91481 /root/openvscode-server/node /root/openvscode-server/out/bootstrap-fork --type=fileWatcher

Oct 19 20:31:00 debian openvscode-server[55499]: [20:31:00] [::1][e3747162][ExtensionHostConnection] New connection established.
Oct 19 20:31:00 debian openvscode-server[55499]: [20:31:00] [::1][e3747162][ExtensionHostConnection] <82029> Launched Extension Host Process.
Oct 19 21:59:15 debian openvscode-server[55499]: [21:59:15] [::1][72f0ef36][ManagementConnection] The client has disconnected gracefully, so the connection will be disposed.
Oct 19 21:59:15 debian openvscode-server[55499]: [21:59:15] [::1][e3747162][ExtensionHostConnection] <82029> Extension Host Process exited with code: 0, signal: null.
Oct 19 21:59:19 debian openvscode-server[55499]: File not found: /root/openvscode-server/out/vsda_bg.wasm
Oct 19 21:59:19 debian openvscode-server[55499]: File not found: /root/openvscode-server/out/vsda.js
Oct 19 21:59:20 debian openvscode-server[55499]: [21:59:20] [::1][34ff706a][ManagementConnection] New connection established.
Oct 19 21:59:20 debian openvscode-server[55499]: [21:59:20] [::1][b7c6b9dc][ExtensionHostConnection] New connection established.
Oct 19 21:59:20 debian openvscode-server[55499]: [21:59:20] [::1][b7c6b9dc][ExtensionHostConnection] <91469> Launched Extension Host Process.
```

The openvscode-server appears to be running but is currently only accessible through localhost/127.0.0.1, an IP address internal to each network device, and is designed to loopback like a USB hub that was somehow connected to itself.

![Self-powered power](https://cdn.wccftech.com/wp-content/uploads/2018/12/infpower.png)
*A loopback life exploit where one of the power brick's outlets is used to charge itself for truly infinite power. Courtesy of [wccftech](https://wccftech.com/asus-rog-phone-charges-when-plugged-into-itself/).*

To temporarily visit the code browser page, we will apply a port forward on the code server's current port, which is defaulted to 3000. Let the following command run from your PC to mirror the server's 3000 port to our own local 3000 port: 

```bash
ssh -N -L 3000:localhost:3000 [user]@<instance-ip>
```

Then access `http://localhost:3000/` on any browser to test for expected VSCode functionality. If everything is set up right, you should see the VSCode welcome page with all of its GUI glory.

At this point, you have successfully created a working VSCode server and a means to enter this server securely. However, we have not yet configured this server to allow us to conveniently access the web page without SSH. Those who only need to access this web page through LAN may append the `/etc/systemd/system/openvscode-server.service` ExecStart line with `--host 0.0.0.0` to open the server with its local IP, but doing so on a public IP will result in all data being unencrypted without any authentication. Our next article will address this problem by introducing reverse proxies and how they can help establish a secure and authenticated connection to our website. 

### Addendum

You may have noticed that this guide looks oddly familiar to [code-server](https://github.com/coder/code-server/), an alternative open-source implementation of VSCode for the Web. The reason is primarily because code-server was in fact my cloud developer environment of choice and it was what I have used for a long while until a recent [bug](https://github.com/coder/code-server/issues/4693) prevented me from using code-server ever since version 4, or a year ago. And while code-server in my opinion offers significantly more community support, I feel openvscode-server to be friendlier for HTTP use, a perfectly acceptable situation if the server is only opened to the local network.