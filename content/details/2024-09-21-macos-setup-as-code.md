---
title: "macos setup as code"
date: 2024-09-21
---

I try to keep track of everything that goes into my work machine.  
I, of course, start missing things after a couple of months.  
But the value is in trying, right? Let's hope so.

In any case, I recently setup a macos machine from scratch. It was intended for devops/infra work ( k8s, terraform, aws etc).  
It also includes java + node as I'll occasionally need to build the apps before running them.

Here is how everything went. Apart from a couple of tokens and ssh keys that are org specific, this is the full setup.

While it was macos + zsh in this instance, I'm pretty sure most of this is portable to Linux and WSL (if you opt for `brew` for package mgmt and `zsh` for a shell). YMMV, of course.

## install everything

```bash
# install brew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
(echo; echo 'eval "$(/opt/homebrew/bin/brew shellenv)"') >> "/Users/${USER}/.zprofile"
eval "$(/opt/homebrew/bin/brew shellenv)"

# install gui apps
brew install --cask rectangle # split screen to 2,4 windows w/ kb shortcuts
brew install --cask google-chrome
brew install --cask firefox
brew install --cask chromium
brew install --cask microsoft-edge
brew install --cask visual-studio-code
brew install --cask dbeaver-community
brew install --cask postman
brew install --cask microsoft-teams
brew install slack

# install cli apps
brew install awscli
brew install wget
brew install zsh-completions
brew install watch
brew install fzf
brew install libpq && brew link --force libpq # postgres tools

# docker
brew install colima
brew services start colima
colima start
brew install docker
brew install docker-compose
brew install docker-credential-helper

# k8s
brew install kubectl
brew install kube-ps1
brew install kubectx
# krew
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
kubectl krew install resource-capacity

# terraform
brew install tfenv
tfenv use x.y.z # select your version here
brew install tflint
terraform -install-autocomplete

#### dev setup

# java
brew tap sdkman/tap
brew install sdkman-cli
export SDKMAN_DIR=$(brew --prefix sdkman-cli)/libexec
[[ -s "${SDKMAN_DIR}/bin/sdkman-init.sh" ]] && source "${SDKMAN_DIR}/bin/sdkman-init.sh"
sdk install java x.y.z-distro # select your version here
java --version

# node, yarn
brew install nvm
mkdir -p ~/.nvm
export NVM_DIR="$HOME/.nvm"
[ -s "/opt/homebrew/opt/nvm/nvm.sh" ] && \. "/opt/homebrew/opt/nvm/nvm.sh"
nvm install  x.y.z # select your version here
npm install --global yarn
node --version
yarn --version

# configure git
git config --global user.name "Firstname Lastname"
git config --global user.email "username@email.tld"
```

## configure everything

```bash
#/.zshrc
source "/opt/homebrew/opt/kube-ps1/share/kube-ps1.sh"
source /opt/homebrew/share/zsh-autocomplete/zsh-autocomplete.plugin.zsh

export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
export PATH="/opt/homebrew/opt/libpq/bin:$PATH"

source <(fzf --zsh)
PS1='$(kube_ps1)'$PS1

export KUBE_EDITOR='nano'
[[ $commands[kubectl] ]] && source <(kubectl completion zsh)

# java
export SDKMAN_DIR=$(brew --prefix sdkman-cli)/libexec
[[ -s "${SDKMAN_DIR}/bin/sdkman-init.sh" ]] && source "${SDKMAN_DIR}/bin/sdkman-init.sh"

# node
mkdir -p ~/.nvm
export NVM_DIR="$HOME/.nvm"
[ -s "/opt/homebrew/opt/nvm/nvm.sh" ] && \. "/opt/homebrew/opt/nvm/nvm.sh"  # This loads nvm
[ -s "/opt/homebrew/opt/nvm/etc/bash_completion.d/nvm" ] && \. "/opt/homebrew/opt/nvm/etc/bash_completion.d/nvm"

chmod go-w '/opt/homebrew/share'
chmod -R go-w '/opt/homebrew/share/zsh'
complete -o nospace -C /opt/homebrew/Cellar/tfenv/3.0.0/versions/1.2.6/terraform terraform
complete -C '/opt/homebrew/bin/aws_completer' aws
eval "$(op completion zsh)"; compdef _op op

if type brew &>/dev/null; then
  FPATH=$(brew --prefix)/share/zsh-completions:$(brew --prefix)/share/zsh/site-functions:$FPATH

  autoload -Uz compinit
  compinit
fi

```
