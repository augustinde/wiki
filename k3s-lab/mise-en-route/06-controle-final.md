---
title: "Étape 6 · Contrôle final"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, mise en route
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Quand la première liste est cochée, passe à la procédure de mise en place du cluster dans la [Procédure](/k3s-lab/procedure), à partir de la réinstallation du VPS. Reviens ensuite ici pour la seconde liste.

**Avant le cluster**

- [ ] `make doctor` affiche « ok » partout, clé age comprise
- [ ] `.sops.yaml` contient ta clé publique, et `keys.txt` est sauvegardé hors du poste
- [ ] Le dépôt est dans `~/k3s-lab`, permissions corrigées, sur GitHub en privé sans `.kube/`
- [ ] `test.yaml` : zone, ID du tunnel, `apiHostname`
- [ ] OIDC : `github_oidc_repositories` et `tenants.demo.githubOidc` listent api-demo et front-demo (étape 3.5)
- [ ] Secrets remplis : identifiants du tunnel, mots de passe Grafana et Rancher
- [ ] Domaine actif chez Cloudflare (`dig NS` montre les serveurs Cloudflare)
- [ ] `cloudflared tunnel route ip show` liste `10.43.0.96/28`
- [ ] CNAME `*` proxifié vers le tunnel
- [ ] Trois applications Access ; Rancher et Grafana avec `Augustin + WARP`
- [ ] Jeton de service `github-deploy` créé (4.6), et politique `GitHub deploy` sur k3s API test (4.7)
- [ ] WARP connecté à ton organisation Zero Trust, `10.0.0.0/8` retiré des exclusions
- [ ] Dépôts api-demo et front-demo poussés, environnement `test` et secrets Cloudflare configurés (procédure k3s, étapes 0.1 à 0.3)

**Après le déploiement du cluster**

- [ ] `cloudflared tunnel list` : le tunnel a des connexions actives, et le tableau de bord l'affiche Healthy
- [ ] `https://whoami-test.tondomaine.fr` affiche la page whoami, avec les en-têtes `Cf-Connecting-Ip` et `X-Forwarded-Proto: https`
- [ ] `https://rancher-test.tondomaine.fr` et `https://grafana-test.tondomaine.fr` affichent leur page de connexion (WARP connecté), et sont refusés WARP déconnecté
- [ ] Dans PowerShell, WARP connecté : `Test-NetConnection 10.43.0.100 -Port 5432` renvoie `TcpTestSucceeded : True`
- [ ] IntelliJ se connecte à `10.43.0.100:5432` avec les identifiants de `make db-creds DB=app`
- [ ] Depuis l'extérieur, seul le port 22 du VPS répond (`nc -zv -w 3 <IP> 80`, `443` et `6443` échouent)

Si `Test-NetConnection` échoue, vérifie dans cet ordre : la ligne `10.0.0.0/8` retirée (5.3, 10 minutes de propagation), le proxy TCP (5.2), la route (`cloudflared tunnel route ip show`), puis le service dans le cluster (`kubectl -n demo get svc app-admin` doit montrer `10.43.0.100`).

## Dépannage

| Symptôme | Cause probable | Correctif |
| --- | --- | --- |
| `shopt: command not found`, prompt illisible après `source ~/.bashrc` | `.bashrc` chargé dans zsh | `exec zsh` ; la configuration zsh est dans `~/.zshrc` |
| `cp: -r not specified; omitting directory` | copie d'un dossier sans `-r` | `cp -r /mnt/f/Projects/k3s-lab ~/` |
| Ansible : « world writable directory, ignoring ansible.cfg » | permissions héritées de Windows | les commandes `find … chmod` de l'étape 3.2 |
| `\r: command not found` dans un script | fins de ligne Windows (CRLF) | `grep -rlI $'\r' ~/k3s-lab \| xargs -r sed -i 's/\r$//'` |
| SSH : « UNPROTECTED PRIVATE KEY FILE » | clé lue depuis `/mnt/c` | la copier dans `~/.ssh`, puis `chmod 600` (étape 3.1) |
| `cloudflared tunnel login` n'ouvre pas de navigateur | WSL n'a pas de navigateur | copier l'URL affichée dans le navigateur Windows |
| sops : « failed to get the data key » | clé age absente, ou `.sops.yaml` avec une autre clé | vérifier `~/.config/sops/age/keys.txt` et la clé publique de `.sops.yaml` |
| `make secrets-edit` ouvre vi | `EDITOR` absent | `export EDITOR=nano` dans `~/.zshrc`, puis `exec zsh` |
| Le domaine ne répond plus après le changement de serveurs DNS | DNSSEC encore actif chez OVH | désactiver le DNSSEC chez OVH, ou remettre temporairement les serveurs OVH |
| Emails coupés après la bascule | MX, SPF ou DKIM non importés | recopier ces enregistrements depuis la zone OVH, en DNS only |
| Erreur 1033 sur un nom de test | cloudflared non connecté (normal avant le cluster) | après le déploiement : `kubectl -n cloudflared logs deploy/cloudflared` |
| « Inscription refusée » dans le client WARP | email absent de Device enrollment | vérifier l'étape 5.1 |
| Le nom d'équipe est refusé | faute de frappe | Zero Trust, Settings, « Team domain » |
| Le code email n'arrive pas | One-time PIN non activé | Settings, Authentication, Login methods |
| WSL perd Internet quand WARP est connecté | interaction entre WARP et le réseau NAT de WSL2 | dans `C:\Users\<toi>\.wslconfig`, section `[wsl2]` : `networkingMode=mirrored` et `dnsTunneling=true`, puis `wsl --shutdown` |
| `10.43.0.100` injoignable, WARP connecté | `10.0.0.0/8` encore exclu, proxy inactif, ou route absente | étapes 5.3, 5.2 et 4.3 ; attendre 10 minutes |
| Rancher accessible sans WARP | politique `Augustin uniquement` encore appliquée | remplacer par `Augustin + WARP` (5.4) |
| Rancher refusé alors que WARP est connecté | client en mode DNS seul, ou contrôle Gateway absent | Service mode *Traffic and DNS* (5.3) ; vérifier 5.4 |
| Conflit avec le VPN d'un client | ce VPN utilise des adresses en `10.x` | liste précise de l'étape 5.3, ou couper WARP le temps de la session |
