Ubuntu Server
============

1. [SSH to server](#ssh-to-server)
2. [Setup a new project](#setup-a-new-project)
    1. [Nginx setup](#nginx-setup)
    2. [DNS setup](#dns-setup)
    3. [Git setup](#git-setup)
    4. [Git hooks](#git-hooks)

SSH to server
---------------
`$ ssh user@domain`

Setup a new project
---------------

### Nginx setup
`$ sudo vim /etc/nginx/sites-available/default`

```
# ...
server {
  listen 3000;
  server_name {PROJECT}.domain;
  
  location / {
    proxy_pass http://localhost:{PORT};
  }
}
```

`$ sudo /etc/init.d/nginx reload (start/restart)`

### DNS setup
| Host      | Type | TTL | Target      |
| --------- | ---- | --- | ----------- |
|           | A    | 300 | {SERVER_IP} |
| www       | A    | 300 | {SERVER_IP} |
| {PROJECT} | A    | 300 | {SERVER_IP} |
| ...

##### Alt with wildcard domain name support
| Host      | Type | TTL | Target      |
| --------- | ---- | --- | ----------- |
|           | A    | 300 | {SERVER_IP} |
| *         | A    | 300 | {SERVER_IP} |

### Git setup
#### Setup bare repo
```
$ mkdir repos/{PROJECT}.git www/{PROJECT}
$ cd repos/{PROJECT}.git
$ git init --bare
```

#### Add ssh keys
`$ vim /home/git/.ssh/authorized_keys`

```
# ssh-rsa ABC123 user@domain.com
# ssh-rsa ...
```

#### Clone repo
`$ git clone git@domain:/home/git/repos/{PROJECT}.git`

### Git hooks
#### Add post-receive hook
`$ vim hooks/post-receive`

```
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

    echo "--> Checking out..."
    git --work-tree="$WORK_TREE" --git-dir="$GIT_DIR" checkout -f

    echo "--> NPM install..."
    cd "$WORK_TREE"
    npm install

    echo "--> NPM restart..."
    npm restart

    echo "Deployment ${BRANCH} branch complete."

  else
    echo "No deployment done."
    echo "Only the ${BRANCH} branch may be deployed."
  fi
done
```

`$ chmod +x hooks/post-receive`
