### Introduction
Step by step guide how to set up an Indracore node. 
In order to proceed this setup guide there are two ways to get started:

    Setting up a private node, e.g. if you would like to run a validator
    Setting up a public node, e.g. if you want to run connect services or dapps to Selendra ecosystem

For private node you only need to follow steps 0 and 1. And setting up an SSL certificate in steps 2 and 3 for any browser can securely connect to your node. (Most people, including validators, only need to set up a private node.)

### Prerequisites
There are various cloud providers, You may choose one with the familiar with, So that it's won't take so much time to setup.

You may find one of these:

* [Digitalocean](https://www.digitalocean.com)
* [Scaleway](https://www.scaleway.com/en/)
* [Amazon AWS](https://aws.amazon.com) AWS, etc.

Before you begin this guide, you should have a regular, non-root user with sudo privileges configured on your server. You can learn how to configure a regular user account by following the [initial server setup guide for Ubuntu 18.04.](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04).

Provisioning a server the server with following recommendation:

* OS: Linux x64 recommend Ubuntu 18.04 x64.
* RAM: 2GB or above.
* HDD/SSD: 60GB.

When you have an account available, log in as your non-root user to begin.

### Step 0 
For public node, set up DNS from a domain name that you own to point to the server. We will use testnet0.selendra.org. (You don't need to do this if you are setting up a private node.)

SSH into the server.
### 1. Installing Indracore and setting it up as a system service
First, clone the Indracore repo, install any dependencies, and run the required build scripts.
```
apt update
apt install -y gcc libc6-dev
apt install -y cmake pkg-config libssl-dev git clang libclang-dev
```

#### Prefetch SSH publickeys
```
ssh-keyscan -H github.com >> ~/.ssh/known_hosts
```

#### Install rustup
```
curl https://sh.rustup.rs -sSf | sh -s -- -y
source /root/.cargo/env
export PATH=/root/.cargo/bin:$PATH
```

#### Get packages
```
git clone https://github.com/selendra/indracore.git
cd indracore
```

#### Build packages
```
./setup.sh
```

Set up the node as a system service. To do this, navigate into the root directory of the indracore repo and execute the following to create the service configuration file:

```
{
    echo '[Unit]'
    echo 'Description=Indracore'
    echo '[Service]'
    echo 'Type=exec'
    echo 'WorkingDirectory='`pwd`
    echo 'ExecStart='`pwd`'/target/release/node-indracore --chain ./indraSpecRaw.json --port 30333 --ws-port 9944 --rpc-port 9933 --telemetry-url 'wss://telemetry.polkadot.io/submit/ 0' --validator  --rpc-methods=Unsafe --ws-external --rpc-cors "*"'
    echo '[Install]'
    echo 'WantedBy=multi-user.target'
} > /etc/systemd/system/indracore.service
```

_Note: This will create an Indracore server that accepts incoming connections from anyone on the internet. If you are using the node as a validator, you should instead remove the ws-external flag, so Indracore does not accept outside connections._

Double check that the config has been written to /etc/systemd/system/indracore.service correctly. If so, enable the service so it runs on startup, and then try to start it now:

```
systemctl enable indracore
systemctl start indracore
```

Check the status of the service:

```
systemctl status indracore
```

You should see the node connecting to the network and syncing the latest blocks. If you need to tail the latest output, you can use:

```
journalctl -u indracore.service -f
```

2. Configuring an SSL certificate
We will use Certbot to talk to Let's Encrypt. Install Certbot dependencies:

```
apt -y install software-properties-common
add-apt-repository universe
add-apt-repository ppa:certbot/certbot
apt update
```

Install Certbot:

```
apt -y install certbot python-certbot-nginx
```

It will guide you through getting a certificate from Let's Encrypt:

```
certbot certonly --standalone
```

If you already have a web server running (e.g. nginx, Apache, etc.) you will need to stop it, by running e.g. service nginx stop, for this to work.

Certbot will ask you some questions, start its own web server, and talk to Let's Encrypt to issue a certificate. In the end, you should see output that looks like this:
```
root:~/indra-node# certbot certonly --standalone
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator standalone, Installer None
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c'
to cancel): testnet0.selendra.org
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for testnet0.selendra.org
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/testnet0.selendra.org/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/testnet0.selendra.org/privkey.pem
   Your cert will expire on 2019-10-08. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
```
### 3. Configuring a Websockets proxy
First, install nginx:

```
apt -y install nginx
```

Set the intended public address of the server, e.g. testnet0.selendra.org, as an environment variable:

export name=testnet0.selendra.org

Set up an nginx configuration. This will inject the public address you have just defined.

```
{
    echo 'user       www-data;  ## Default: nobody'
    echo 'worker_processes  5;  ## Default: 1'
    echo 'error_log  /var/log/nginx/error.log;'
    echo 'pid        /var/run/nginx.pid;'
    echo 'worker_rlimit_nofile 8192;'
    echo ''
    echo 'events {'
    echo '  worker_connections  4096;  ## Default: 1024'
    echo '}'
    echo ''
    echo 'http {'
    echo '  map $http_upgrade $connection_upgrade {'
    echo '    default upgrade;'
    echo "      \'\' close;"
    echo '  }'
    echo ''
    echo '  server {'
    echo ''
    echo '    server_name '$name';'
    echo ''
    echo '    root /var/www/html;'
    echo '    index index.html;'
    echo ''
    echo '    location / {'
    echo '      try_files $uri $uri/ =404;'
    echo ''
    echo '      proxy_pass http://localhost:9944;'
    echo '      proxy_set_header X-Real-IP $remote_addr;'
    echo '      proxy_set_header Host $host;'
    echo '      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;'
    echo ''
    echo '      proxy_http_version 1.1;'
    echo '      proxy_set_header Upgrade $http_upgrade;'
    echo '      proxy_set_header Connection "upgrade";'
    echo '    }'
    echo ''
    echo '    listen [::]:443 ssl ipv6only=on;'
    echo '    listen 443 ssl;'
    echo '    ssl_certificate /etc/letsencrypt/live/'$name'/fullchain.pem; # managed by Certbot'
    echo '    ssl_certificate_key /etc/letsencrypt/live/'$name'/privkey.pem; # managed by Certbot'
    echo ''
    echo '    ssl_session_cache shared:cache_nginx_SSL:1m;'
    echo '    ssl_session_timeout 1440m;'
    echo ''
    echo '    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;'
    echo '    ssl_prefer_server_ciphers on;'
    echo ''
    echo '    ssl_ciphers "ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS";'
    echo ''
    echo '    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;'
    echo ''
    echo '  }'
    echo '}'
} > /etc/nginx/nginx.conf
```
Make sure that the paths of ssl_certificate and ssl_certificate_key match what Let's Encrypt produced earlier. Check that the configuration file has been created correctly.

```
cat /etc/nginx/nginx.conf
nginx -t
```
_If there is an error, nginx -t should tell you where it is. Note that there may be subtle variations in how different systems are configured, e.g. some boxes may have different login users or locations for log files. It is up to you to reconcile these differences._

Start the server:
```
service nginx restart
```
You can now try to connect to your new node from [polkadot.js/apps](https://polkadot.js.org/apps/#/settings), or by making a curl request that emulates opening a secure WebSockets connection:

```
curl --include --no-buffer --header "Connection: Upgrade" --header "Upgrade: websocket" --header "Host: $name:80" --header "Origin: http://$name:80" --header "Sec-WebSocket-Key: SGVsbG8sIHdvcmxkIQ==" --header "Sec-WebSocket-Version: 13" http://$name:9944/
```
### 4. Connecting to your node
Congratulations on your new node! If you set up public DNS and a SSL certificate in steps 2 and 3, you should be able to connect to it now from polkadot-js/apps:

_To be updated with iamges

In general, you should use these URLs to connect to your node:

    ws://testnet0.selendra.org:9944 if you set it up as a public node with --ws-external in step 1
    wss://testnet0.selendra.org if you set it up as a public node and also followed steps 2 and 3

### Conclusion
Your node will automatically restart when the system reboots, but it may not be able to recover from other failures. To handle those, consider following our guide to Setting up monitoring.

You may also wish to proceed to Validating on Indracore.
