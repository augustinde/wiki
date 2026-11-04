---
title: "Étape 6 · Contrôle final"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, mise en route
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Quand la première liste est cochée pour le projet, passe à la [Procédure](/k3s-lab/procedure), à partir de la réinstallation du serveur. Reviens ensuite ici pour la seconde liste. Variables chargées dans le dossier du projet : `eval "$(make -s vars)"`.

**Avant le cluster**

- [ ] `make doctor` affiche « ok » partout, clé age comprise, et `keys.txt` est sauvegardé hors du poste
- [ ] Le dépôt du projet est créé depuis le modèle, privé, cloné dans `~/<projet>` (3.2)
- [ ] `make config-check` répond `Configuration cohérente.` (valeurs, IP, tunnel, route, bases, secrets chiffrés)
- [ ] OIDC : chaque dépôt d'application est déclaré sous `tenants.<ns>.githubOidc` (3.5)
- [ ] Secrets remplis : identifiants du tunnel, mots de passe Grafana et Rancher (3.6)
- [ ] Connexion par GitHub : application OAuth, fournisseur d'identité testé, politiques sur l'équipe GitHub
- [ ] Domaine actif chez Cloudflare (`dig NS $ZONE +short` montre les serveurs Cloudflare)
- [ ] `cloudflared tunnel route ip show` liste `$ROUTE`, associée au tunnel `$CLUSTER`
- [ ] DNS : joker `*` proxifié vers le tunnel, ou un nom par service pour un projet suivant (4.4)
- [ ] Applications Access du projet ; Rancher et Grafana avec `Équipe + WARP`, API avec `GitHub deploy` (4.7)
- [ ] WARP connecté à ton organisation Zero Trust, la route du projet hors des exclusions (5.3)
- [ ] Dépôts d'applications prêts côté GitHub (procédure, étapes 0.1 à 0.3)

**Après le déploiement du cluster**

- [ ] `cloudflared tunnel info $CLUSTER` liste des connecteurs, et le tableau de bord affiche le tunnel Healthy
- [ ] `https://whoami<suffixe>.<zone>` affiche la page whoami, avec les en-têtes `Cf-Connecting-Ip` et `X-Forwarded-Proto: https`
- [ ] `https://rancher<suffixe>.<zone>` et `https://grafana<suffixe>.<zone>` affichent leur page de connexion (WARP connecté), et sont refusés WARP déconnecté
- [ ] Dans PowerShell, WARP connecté : `Test-NetConnection <ip de la base> -Port 5432` renvoie `TcpTestSucceeded : True` (IP : `make db-creds DB=app`)
- [ ] IntelliJ se connecte à cette IP, port 5432, avec les identifiants de `make db-creds DB=app`
- [ ] Depuis l'extérieur, seul le port 22 du serveur répond (`nc -zv -w 3 $IP 80`, `443` et `6443` échouent)

Si `Test-NetConnection` échoue, vérifie dans cet ordre : la route hors des exclusions WARP (5.3, 10 minutes de propagation), le proxy TCP (5.2), la route (`cloudflared tunnel route ip show`), puis le service dans le cluster (`kubectl -n $NS get svc app-admin` doit montrer l'IP de `databases/app.yaml`).

## Dépannage

| Symptôme | Cause probable | Correctif |
| --- | --- | --- |
| `shopt: command not found`, prompt illisible après `source ~/.bashrc` | `.bashrc` chargé dans zsh | `exec zsh` ; la configuration zsh est dans `~/.zshrc` |
| Ansible : « world writable directory, ignoring ansible.cfg » | dépôt copié depuis Windows | cloner le dépôt dans WSL (3.2), ou `find . -type d -exec chmod 755 {} +` et `find . -type f -exec chmod 644 {} +`, puis `chmod +x scripts/* ci/*.sh` |
| `\r: command not found` dans un script | fins de ligne Windows (CRLF) | `grep -rlI $'\r' . \| xargs -r sed -i 's/\r$//'` dans le dossier du projet |
| SSH : « UNPROTECTED PRIVATE KEY FILE » | clé lue depuis `/mnt/c` | la copier dans `~/.ssh`, puis `chmod 600` (étape 3.1) |
| `cloudflared tunnel login` n'ouvre pas de navigateur | WSL n'a pas de navigateur | copier l'URL affichée dans le navigateur Windows |
| sops : « failed to get the data key » | clé age absente, ou secrets chiffrés pour une autre clé | vérifier `~/.config/sops/age/keys.txt` et la clé publique de `.sops.yaml` (`age-keygen -y ~/.config/sops/age/keys.txt`) |
| `make secrets-edit` ouvre vi | `EDITOR` absent | `export EDITOR=nano` dans `~/.zshrc`, puis `exec zsh` |
| Le domaine ne répond plus après le changement de serveurs DNS | DNSSEC encore actif chez OVH | désactiver le DNSSEC chez OVH, ou remettre temporairement les serveurs OVH |
| Emails coupés après la bascule | MX, SPF ou DKIM non importés | recopier ces enregistrements depuis la zone OVH, en DNS only |
| Erreur 1033 sur un nom du projet | cloudflared non connecté (normal avant le cluster) | après le déploiement : `kubectl -n cloudflared logs deploy/cloudflared` |
| « Inscription refusée » dans le client WARP | compte hors de l'équipe, ou Device enrollment pas sur GitHub | vérifier l'étape 5.1 |
| Le nom d'équipe est refusé | faute de frappe | Zero Trust, Settings, « Team domain » |
| Connexion GitHub refusée (« That account does not have access ») | compte hors de l'équipe, ou organisation non accordée à l'application | [Connexion par GitHub](/k3s-lab/acces/connexion-github), dépannage |
| WSL perd Internet quand WARP est connecté | interaction entre WARP et le réseau NAT de WSL2 | dans `C:\Users\<toi>\.wslconfig`, section `[wsl2]` : `networkingMode=mirrored` et `dnsTunneling=true`, puis `wsl --shutdown` |
| IP d'une base injoignable, WARP connecté | route du projet encore exclue, proxy inactif, ou route absente | étapes 5.3, 5.2 et 4.3 ; attendre 10 minutes |
| Rancher accessible sans WARP | politique `Équipe` encore appliquée | remplacer par `Équipe + WARP` (5.4) |
| Rancher refusé alors que WARP est connecté | client en mode DNS seul, ou contrôle Gateway absent | Service mode *Traffic and DNS* (5.3) ; vérifier 5.4 |
| `route ip add` : chevauchement | plage déjà prise par un autre projet | autre `/28` dans `values.yaml` et `databases/*.yaml` (4.3) |
| Erreur 1033 sur un nom d'un deuxième projet | le joker `*` vise le tunnel du premier | `cloudflared tunnel route dns` pour chaque nom (4.4) |
| Conflit avec le VPN d'un client | ce VPN utilise des adresses en `10.x` | liste précise de l'étape 5.3, ou couper WARP le temps de la session |
