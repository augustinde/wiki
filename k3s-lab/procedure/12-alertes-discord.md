---
title: "Étape 12 · Alertes Discord"
description: ""
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Alertmanager envoie les alertes de Prometheus dans un salon Discord. Le titre indique l'origine, d'après le namespace concerné, puis le cluster : plusieurs projets peuvent partager un salon.

| Préfixe | Namespaces | Exemple |
| --- | --- | --- |
| 🔴 PROD | se terminant par `-prod` | 🔴 PROD · k3s-test · KubePodCrashLooping |
| 🟡 TEST | les autres namespaces applicatifs | 🟡 TEST · k3s-test · KubeDeploymentReplicasMismatch |
| ⚙️ INFRA | nœud et plateforme (`monitoring`, `cloudflared`, `kube-system`…) | ⚙️ INFRA · k3s-test · MemoireNoeudSaturee |

Seules les alertes `warning` et `critical` partent, regroupées par nom et namespace, avec un message « ✅ Résolu » à la fin. Les titres et le cadre sont en français ; le texte des règles fournies par kube-prometheus-stack reste en anglais. Trois règles sont ajoutées, en français : tunnel sans connecteur, PostgreSQL arrêté, mémoire du nœud au-delà de 90 %. Gabarits et règles : `helmfile/values/alerting.yaml`.

## 12.1 · Créer le webhook Discord

Dans Discord, sur le salon choisi : **Modifier le salon**, **Intégrations**, **Webhooks**, **Nouveau webhook**. Nomme-le `k3s-lab`, puis **Copier l'URL du webhook**.

> **Vérifier** · Sans laisser l'URL dans l'historique du terminal :
>
> ```bash
> read -rs HOOK      # colle l'URL, puis Entrée (rien ne s'affiche)
> curl -s -o /dev/null -w '%{http_code}\n' -H 'Content-Type: application/json' \
>   -d '{"content":"Test k3s-lab ✅"}' "$HOOK"
> ```
>
> La commande affiche `204`, et le message apparaît dans le salon.
{.is-success}

## 12.2 · Ajouter le webhook aux secrets

```bash
read -rs HOOK      # colle l'URL du webhook, puis Entrée (rien ne s'affiche)
sops set $ENV_DIR/secrets.yaml '["alerting"]' "{\"discordWebhookUrl\": \"$HOOK\"}"
unset HOOK
```

`sops set` écrit la valeur sans ouvrir d'éditeur. Avec `make secrets-edit`, l'erreur `File has not changed` (code 200) signifie que l'éditeur a été fermé sans enregistrer, ou que `EDITOR` vaut `code` sans `--wait`.

> **Vérifier**
>
> ```bash
> sops -d --extract '["alerting"]["discordWebhookUrl"]' $ENV_DIR/secrets.yaml | cut -c1-34; echo
> git commit -am "Webhook Discord des alertes" && git push
> ```
>
> La première commande affiche `https://discord.com/api/webhooks/`. Le fichier reste chiffré dans le dépôt.
{.is-success}

## 12.3 · Activer Alertmanager

```bash
make platform
```

> **Vérifier**
>
> ```bash
> kubectl -n monitoring get pods | grep alertmanager
> kubectl -n monitoring get prometheusrules | grep k3s-lab
> ```
>
> Un pod `alertmanager-monitoring-kube-prometheus-alertmanager-0` en `2/2 Running`, et une ressource de règles contenant `k3s-lab`. Dans Grafana, Alerting, Alert rules : les règles `TunnelCloudflareSansConnecteur`, `BasePostgresArretee` et `MemoireNoeudSaturee` apparaissent.
{.is-success}

## 12.4 · Envoyer une alerte de test

On injecte une fausse alerte directement dans Alertmanager, pour chaque préfixe :

```bash
kubectl -n monitoring port-forward svc/monitoring-kube-prometheus-alertmanager 9093 > /dev/null &
sleep 3
for ns in $NS-prod $NS monitoring; do
  curl -s -XPOST localhost:9093/api/v2/alerts -H 'Content-Type: application/json' -d "[{
    \"labels\": {\"alertname\": \"AlerteDeTest\", \"namespace\": \"$ns\", \"severity\": \"warning\"},
    \"annotations\": {\"summary\": \"Alerte envoyée à la main depuis WSL\"}}]"
done
kill %1
```

> **Vérifier** · En moins d'une minute, le salon reçoit trois messages, 🔴 PROD, 🟡 TEST et ⚙️ INFRA · AlerteDeTest. Une alerte injectée à la main ne porte pas le nom du cluster : seules celles de Prometheus l'ont. Environ 5 minutes plus tard arrivent les trois messages « ✅ Résolu ».
>
> Rien n'arrive : `kubectl -n monitoring logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -c alertmanager | grep -i discord` montre l'erreur. La cause la plus courante est une URL de webhook incomplète dans les secrets.
{.is-success}
