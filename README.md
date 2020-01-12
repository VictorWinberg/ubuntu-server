Ubuntu Server
============

1. [Setup wireless network](#setup-wireless-network)
2. [SSH to server](#ssh-to-server)
3. [Setup a new project](#setup-a-new-project)
    1. [Nginx setup](#nginx-setup)
    2. [DNS setup](#dns-setup)
    3. [Git server setup](#git-server-setup)
    4. [Git client setup](#git-client-setup)
4. [Run on Startup](#run-on-startup)
5. [Update Certificates](#update-certificates)
6. [External IP](#external-ip)
7. [Nextcloud](#nextcloud)
8. [Home Assistant](#home-assistant)

Setup wireless network
----------------------
### Preparation
Wireless networking configuration, usually named `wlan0` or `eth0`
```
$ iwconfig
wlan0  IEEE 802.11...
$ ip link show wlan0             # check if wireless interface is up
$ ip link set wlan0 up           # bring wireless interface up
$ iwlist wlan0 scan | grep ESSID # scan wireless access points
```

### Generate psk key
`wpa_passphrase [ssid] [passphrase]` eg.

```
$ wpa_passphrase "My WiFi" "password"
network={
    ssid="My WiFi"
    #psk="password"
    psk=59e0...
}
```

### Configure the wireless network interfaces
```
$ cat >> /etc/network/interfaces
# The wireless network interface
auto wlan0
iface wlan0 inet static # can be dhcp with router setup
  address 192.168.0.80 # which ip the server should be at
  netmask 255.255.255.0
  gateway 192.168.0.1
  broadcast 192.168.0.255 # needed?
  network 192.168.0.0 # needed?  
  dns-nameservers 8.8.8.8 8.8.4.4 # needed?

  wpa-essid "My WiFi"
  wpa-psk 59e0... # from wpa_passphrase
```

### *Alternative*

Instead of `wpa_essid` and `wpa-psk` in `/etc/network/interfaces` use 
```
  wpa-driver wext
  wpa-conf /etc/wpa_supplicant.conf
```
where /etc/wpa_supplicant.conf examples could be found [here](http://www.lsi.upc.edu/lclsi/Manuales/wireless/files/wpa_supplicant.conf)

Could be tested as root with 
```
$ killall -q wpa_supplicant
$ wpa_supplicant -B -D wext -i wlan0 -c /etc/wpa_supplicant.conf -d
```

SSH to server
-------------
`$ ssh user@domain`

Setup a new project
-------------------

### Nginx setup
`$ sudo vim /etc/nginx/sites-available/default`

```nginx
# ...
server {
  listen 3000;
  server_name {PROJECT}.domain;
  
  location / {
    # Use the PORT that the PROJECT will run on
    proxy_pass http://localhost:{PORT};
  }
}
```

`$ sudo /etc/init.d/nginx reload (start/restart)`

### DNS setup
| Host      | Type | TTL   | Target      |
| --------- | ---- | ----- | ----------- |
|           | A    | 86400 | {SERVER_IP} |
| www       | A    | 86400 | {SERVER_IP} |
| {PROJECT} | A    | 86400 | {SERVER_IP} |
| ...

##### Alt with wildcard domain name support
| Host      | Type | TTL   | Target      |
| --------- | ---- | ----- | ----------- |
|           | A    | 86400 | {SERVER_IP} |
| *         | A    | 86400 | {SERVER_IP} |

### Git server setup
#### Setup bare repo
```sh
$ su git && cd ~
$ git init --bare repos/{PROJECT}.git
$ mkdir www/{PROJECT}
```

#### Add ssh keys
`$ cat >> /home/git/.ssh/authorized_keys`

```
ssh-rsa ABC123 user@domain.com
```
`[ENTER] + Ctrl+D`

#### Add post-receive hook
*Requires: npm install forever --global && apt-get install jq*

`$ cd repos/{PROJECT}.git && cat > hooks/post-receive`

```bash
#!/bin/bash 
set -eu # exit script on errors

WORK_TREE="/home/git/www/{PROJECT}"
GIT_DIR="/home/git/repos/{PROJECT}.git"
BRANCH="master"

while read oldrev newrev ref
do
  echo "Ref $ref received."

  if [[ $ref = refs/heads/"$BRANCH" ]];
  then
    echo "Deploying ${BRANCH} branch..."

    echo "> git checkout..."
    git --work-tree="$WORK_TREE" --git-dir="$GIT_DIR" checkout -f
    cd "$WORK_TREE"

    echo "> npm install..."
    npm install

    echo "> npm run build..."
    npm run build

    echo "> forever restart server..."
    forever columns set uid script forever pid uptime > /dev/null
    forever restart server || (forever start server && forever list)
    forever columns reset > /dev/null
    echo "Deployment ${BRANCH} branch complete."

  else
    echo "No deployment done."
    echo "Only the ${BRANCH} branch may be deployed."
  fi
done
```
`[ENTER] + Ctrl+D`

`$ chmod +x hooks/post-receive`

#### Run post-receive command on all repos (reset)
```bash
for repo in $(find /home/git/repos -name *.git); do
  echo "oldrev newrev refs/heads/master" | "$repo/hooks/post-receive"
done
```

### Git client setup
#### Clone repo
`$ git clone git@domain:repos/{PROJECT}.git`

#### Add remote
`$ git remote add server git@domain:repos/{PROJECT}.git`

#### Init repo
```sh
$ cd {PROJECT}
# Fetch or create a .gitignore
$ curl -o .gitignore https://raw.githubusercontent.com/github/gitignore/master/Node.gitignore
$ npm init
$ npm install express --save
# MAIN from npm init or package.json (default: index.js)
$ atom {MAIN}.js
```
```js
// {MAIN}.js
const express = require('express')
const app = express()

app.get('/', (req, res) => res.send('Hello World!'))

app.listen({PORT}, () => console.log('Example app listening on port {PORT}!'))
```
```sh
$ git add .
$ git commit -m "initialize commit"
$ git push
```

**Done!**

Run on Startup
--------------
`$ sudo crontab -e`

```
# Edit this file to introduce tasks to be run by cron.
@reboot /bin/sh /home/git/reset.sh # Run as root
@reboot /bin/su -c '/home/git/reset.sh' -s /bin/sh - git # Run as user "git"
```

Update Certificates
-------------------
```
$ sudo certbot --authenticator webroot --installer nginx
Input webroot: /usr/share/nginx/html
$ sudo vim /etc/nginx/sites-available/default
$ sudo /etc/init.d/nginx reload (start/restart)
```

External IP
----------
```
$ curl https://ipinfo.io/ip
```

Nextcloud
----------
#### Reset file indexing
```
sudo -u www-data php occ maintenance:mode --on
sudo -u www-data php occ files:scan --all
sudo -u www-data php occ maintenance:mode --off
```
**Might need to reset file locks**
```
sudo -u postgres psql
\c nextcloud
DELETE FROM oc_file_locks WHERE 1 = 1;
```

Home Assistant
----------
**Change user**
`sudo -u homeassistant -H -s`
