Ubuntu Server
============

1. [SSH to server](#ssh-to-server)
2. [Setup a new project](#setup-a-new-project)
    1. [Nginx setup](#nginx-setup)
    2. [DNS setup](#dns-setup)
    3. [Git server setup](#git-server-setup)
    4. [Git client setup](#git-client-setup)

SSH to server
---------------
`$ ssh user@domain`

Setup a new project
---------------

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

### Git server setup
#### Setup bare repo
```sh
$ su git && cd ~
$ git init --bare repos/{PROJECT}.git
$ mkdir www/{PROJECT}
```

#### Add ssh keys
`$ vim /home/git/.ssh/authorized_keys`

```
# ssh-rsa ABC123 user@domain.com
# ssh-rsa ...
```

#### Add post-receive hook
`$ cd repos/{PROJECT} && vim hooks/post-receive`

```bash
#!/bin/bash 
set -eu # exit script on errors

WORK_TREE="/home/git/www/drinkit"
GIT_DIR="/home/git/repos/drinkit.git"
BRANCH="master"

while read oldrev newrev ref
do
  echo "Ref $ref received."

  if [[ $ref = refs/heads/"$BRANCH" ]];
  then
    echo "Deploying ${BRANCH} branch..."

    echo "> git checkout..."
    git --work-tree="$WORK_TREE" --git-dir="$GIT_DIR" checkout -f

    echo "> npm install..."
    cd "$WORK_TREE"
    npm install

    echo "> forever restart main..."
    MAIN=$(cat package.json | jq -r ".main")
    forever columns set uid script forever pid uptime > /dev/null
    forever restart $MAIN || (forever start $MAIN && forever list)
    forever columns reset > /dev/null
    echo "Deployment ${BRANCH} branch complete."

  else
    echo "No deployment done."
    echo "Only the ${BRANCH} branch may be deployed."
  fi
done
```

`$ chmod +x hooks/post-receive`

### Git client setup
#### Clone repo
`$ git clone git@domain:/home/git/repos/{PROJECT}.git`

#### Init repo
```sh
$ cd {PROJECT}
# Fetch or create a .gitignore
$ curl -o .gitignore https://raw.githubusercontent.com/github/gitignore/master/Node.gitignore
$ npm init
$ npm install express --save
$ git add .
$ git commit -m "initialize commit"
$ git push
```
