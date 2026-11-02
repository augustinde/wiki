---
title: "Étape 5 · Piloter le cluster depuis ton poste"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

L'API k3s n'écoute pas sur Internet : ton poste l'atteint par un tunnel SSH, sur `127.0.0.1:6443`.

## 5.1 · Ouvrir le tunnel

```bash
make tunnel
```

> **Vérifier** · Le message `Tunnel ouvert.` s'affiche, et `pgrep -af 6443:127.0.0.1:6443` montre un processus `ssh -fNT`.
>
> Le tunnel se ferme quand WSL s'arrête ou que le PC se met en veille. Si une commande `make` répond « API k3s injoignable », relance `make tunnel`.
{.is-success}

## 5.2 · Rendre `kubectl` utilisable dans ton terminal

Les cibles `make` trouvent le kubeconfig toutes seules ; `kubectl` et `k9s`, eux, ont besoin de la variable `KUBECONFIG`.

```bash
echo 'export KUBECONFIG=$HOME/k3s-lab/.kube/k3s-test.yaml' >> ~/.zshrc
exec zsh
```

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
> | contexte | `k3s-test` |
> | nœuds | `vps-test   Ready` |
> | pods | `coredns`, `local-path-provisioner`, `metrics-server` et `traefik` en `Running` ; `helm-install-traefik-…` en `Completed` ; aucun pod `svclb-…` |
> | Traefik | `TYPE ClusterIP`, `EXTERNAL-IP <none>` : rien n'est publié sur l'IP du VPS |
>
> *Si le résultat diffère, ne passe pas à la suite.*
{.is-success}
