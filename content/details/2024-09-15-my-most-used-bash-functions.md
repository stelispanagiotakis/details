---
title: "My most used bash functions"
date: "2024-09-15"
---

There's problems you'd like to solve.  
And problems you like solving.

What you put in the second category is almost uncomfortably revealing of who you are.  
Anything you consider trivial fits right into the first category and is a candidate for automation.

Here are some of these automations I bring with me to all new laptops I work on.

## get a random string of variable length

```bash
function randpw() {
 # e.g randpw 20
  local pwlength="${1}"
  cat /dev/urandom | tr -dc 'a-zA-Z0-9!@#$%^&*()-[]{}:;\|,.<>/?' | fold -w "${pwlength:-50}" | head -n 1
}
```

## full update for ubuntu distros

```bash
function fu() {
    sudo apt update
    sudo apt -y upgrade
    sudo apt -y dist-upgrade
    sudo apt -y autoremove
    sudo ubuntu-drivers install
    brew update
    brew upgrade
}
```

## update all repos in your code directory

```bash
function getallprojects() {
  find "${YOUR_PROJECTS_DIR}" -maxdepth 1 \
            -mindepth 1 \
            -type d \
            -printf '%f\n' \
            | grep -v .vscode
}
function getdefaultbranch() {
  git remote show origin | grep 'HEAD branch' | cut -d' ' -f5
}
function refreshprojects() {

  local LIGHTGREEN="\e[92m"
  local ENDCOLOR="\e[0m"

  echo "---------------"
  for project in "$(getallprojects)";
  do
    echo -e "Refreshing ${LIGHTGREEN}${project}${ENDCOLOR}"
    pushd "${project}" > /dev/null || exit

    git fetch --prune
    git checkout "$(getdefaultbranch)"
    git pull
    popd > /dev/null || exit
    echo "---------------"
  done
}
```

## hack to get a urlencoded string

```bash
function urlenc() {
  local str_to_encode="${1}"
  encoded_str=$(python3 -c "import urllib.parse; print(urllib.parse.quote('''${str_to_encode}'''))")
  echo "${encoded_str}"
  # why did I need it?
  # verify connectivity to rabbitmq, password had special characters
  # rabbitmq-dump-queue -uri="amqps://mq_user:$(urlenc "${mq_password}")@my.mq.endpoint.tld:5671/" -queue=myqueue -max-messages=1 -output-dir=./
}
```

Is it worth the time? [It depends](https://xkcd.com/1205/ "It depends")
