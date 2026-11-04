---
title: "Étape 05 · Piloter le cluster depuis ton poste"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

L'API k3s n'écoute pas sur Internet : ton poste l'atteint par un tunnel SSH, sur `127.0.0.1:6443` (ou le port `localApiPort` du projet).

## 5.1 · Ouvrir le tunnel

```bash
make tunnel
```

> **Vérifier** · Le message `Tunnel ouvert vers <cluster>.` s'affiche, et `pgrep -af 127.0.0.1:6443` montre un processus `ssh -fNT`.
>
> Le tunnel se ferme quand WSL s'arrête ou que le PC se met en veille. Si une commande `make` répond « API de <cluster> injoignable », relance `make tunnel`.
{.is-success}

## 5.2 · Rendre `kubectl` utilisable dans ton terminal

Les cibles `make` trouvent le kubeconfig toutes seules ; `kubectl` et `k9s`, eux, ont besoin de la variable `KUBECONFIG`. Une petite fonction, ajoutée une fois à `~/.zshrc`, ouvre un projet : dossier, variables et kubeconfig.

```bash
cat >> ~/.zshrc <<'EOF'

# Ouvre un projet k3s : projet <dossier> [env]
projet() {
  cd ~/"$1" || return
  unset ENV NS
  [ -n "$2" ] && export ENV="$2"
  eval "$(make -s vars)" && export KUBECONFIG="$PWD/.kube/$CLUSTER.yaml"
  # Certificat cloudflared du compte Cloudflare du projet, s'il a le sien (plusieurs comptes)
  if [ -f ~/.cloudflared/$CLUSTER/cert.pem ]; then
    export TUNNEL_ORIGIN_CERT=~/.cloudflared/$CLUSTER/cert.pem
  else
    unset TUNNEL_ORIGIN_CERT
  fi
  echo "$CLUSTER ($ENV) : $IP"
}
EOF
exec zsh
projet <projet>
```

Supprime de `~/.zshrc` une ancienne ligne `export KUBECONFIG=…` s'il y en a une. La fonction existe déjà ? Remplace-la par celle-ci dans `~/.zshrc` : elle choisit aussi le certificat `cloudflared` du projet quand tes projets sont sur plusieurs comptes Cloudflare.

> **Vérifier**
>
> ```bash
> kubectl config current-context
> kubectl get nodes
> kubectl get pods -A
> kubectl -n kube-system get svc traefik
> ```
>
> | Commande | Résultat attendu |
> | --- | --- |
> | contexte | `<cluster>` |
> | nœuds | `<nœud>   Ready` |
> | pods | `coredns`, `local-path-provisioner`, `metrics-server` et `traefik` en `Running` ; `helm-install-traefik-…` en `Completed` ; aucun pod `svclb-…` |
> | Traefik | `TYPE ClusterIP`, `EXTERNAL-IP <none>` : rien n'est publié sur l'IP du serveur |
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}
