---
title: "Étape 2 · Installer les outils dans WSL2"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, mise en route
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Copie chaque bloc dans le terminal Ubuntu, dans l'ordre. Les versions de Helmfile et de sops sont récupérées automatiquement à la dernière publiée.

## 2.1 · Paquets de base et Ansible

```bash
sudo apt update
sudo apt install -y git make curl gpg jq unzip python3 pipx age openssh-client dnsutils
pipx ensurepath
exec zsh                                   # recharge le PATH
pipx install --include-deps ansible
ansible --version
```

## 2.2 · kubectl

(dépôt officiel, branche 1.34 comme le cluster)

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubectl
kubectl version --client
```

## 2.3 · Helm 3

(script officiel, reste sur la branche 3 : helm-diff a eu des incompatibilités avec Helm 4)

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## 2.4 · Helmfile

```bash
HF=$(curl -fsSL https://api.github.com/repos/helmfile/helmfile/releases/latest | jq -r .tag_name)
curl -fsSL "https://github.com/helmfile/helmfile/releases/download/${HF}/helmfile_${HF#v}_linux_amd64.tar.gz" \
  | sudo tar -xz -C /usr/local/bin helmfile
helmfile --version
```

## 2.5 · sops

```bash
SOPS=$(curl -fsSL https://api.github.com/repos/getsops/sops/releases/latest | jq -r .tag_name)
sudo curl -fsSL -o /usr/local/bin/sops "https://github.com/getsops/sops/releases/download/${SOPS}/sops-${SOPS}.linux.amd64"
sudo chmod +x /usr/local/bin/sops
sops --version
```

## 2.6 · cloudflared

(dépôt APT officiel de Cloudflare)

```bash
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-main.gpg > /dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared
cloudflared --version
```

## 2.7 · Optionnel

`k9s`, une interface terminal pour suivre les pods (binaire sur les releases GitHub du projet).

Les plugins Helm et les collections Ansible s'installent à l'étape 3.4 avec `make deps`, une fois le dépôt en place.

Côté Windows, il faut IntelliJ IDEA **Ultimate** pour l'onglet Database (ou DataGrip, ou DBeaver gratuit). Le client WARP s'installe à l'étape 5.
