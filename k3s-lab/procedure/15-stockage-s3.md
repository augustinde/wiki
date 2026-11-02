---
title: "Étape 15 · Fichiers publics sur S3 OVH"
description: "Bucket OVH Object Storage pour les images publiques, servi par le domaine du projet avec le cache Cloudflare"
published: true
date: 2026-11-02T12:00:00.000Z
tags: k3s-lab, procédure
editor: markdown
dateCreated: 2026-11-02T12:00:00.000Z
---

Les fichiers publics des applications (photos de produits, images d'articles, avatars) sont stockés chez OVH (Object Storage, compatible S3), hors du cluster : ils survivent à `make rebuild` et à une réinstallation du serveur. Le trafic sortant d'OVH n'est pas facturé ; seul le volume stocké l'est.

```text
Envoi    navigateur ─► API (Spring Boot) ─► SDK S3 ─► bucket OVH (objet en lecture publique)
Lecture  <img src="https://files<suffixe>.<zone>/produits/<uuid>.webp">
         navigateur ─► Cloudflare ── en cache ─► réponse immédiate
                                  └─ sinon ─► tunnel ─► proxy s3 (nginx) ─► bucket OVH
```

Le proxy (`helmfile/charts/s3-proxy`, une vingtaine de Mo de mémoire) sert uniquement en lecture : il relaie vers le bucket avec le bon nom d'hôte, ce que Cloudflare gratuit ne sait pas faire, et ajoute un `Cache-Control` de 30 jours pour que Cloudflare garde chaque fichier en cache. Il supprime les paramètres de requête, n'expose jamais la liste du bucket et renvoie 404 pour un fichier absent ou non public. Les URL enregistrées en base restent celles du domaine : changer de fournisseur ne change que la configuration du proxy.

Un bucket et un utilisateur S3 par environnement applicatif : `$CLUSTER-$NS` pour le test, `$CLUSTER-$NS-prod` pour la prod. Variables chargées : `eval "$(make -s vars)"`.

## 15.1 · Reporter le modèle

Pour un projet créé avant l'ajout du stockage S3 au modèle (k3s-test, par exemple) :

```bash
ls helmfile/charts/s3-proxy 2>/dev/null || {
  git remote get-url modele 2>/dev/null || git remote add modele https://github.com/augustinde/k3s-lab.git
  git fetch modele
  git checkout modele/main -- Makefile ansible helmfile profiles scripts ci renovate.json
}
make config-check
```

> **Vérifier** · `ls helmfile/charts/s3-proxy` liste `Chart.yaml`, `templates` et `values.yaml`, et `config-check` répond `Configuration cohérente.` Commite le report avec `git commit -am "Mise à jour depuis le modèle"`.
{.is-success}

## 15.2 · Créer le bucket

*Mot-clé de recherche dans l'espace client : « Object Storage ».* Espace client OVH, **Public Cloud**. Sans projet Public Cloud, crée-le d'abord (**Créer un projet**, moyen de paiement demandé). Puis, dans le projet, **Object Storage**, **Créer un conteneur d'objets** :

| Champ | Valeur |
| --- | --- |
| Offre | API S3, classe **Standard** |
| Déploiement | 1-AZ |
| Région | au choix : la plus proche du serveur ou des visiteurs ; note son code en minuscules, il sert ensuite partout (`<region>`) |
| Nom | `<cluster>-<ns>` (`echo $CLUSTER-$NS`) ; s'il est déjà pris, ajoute un suffixe |
| Gestion des versions, Object Lock | désactivés |
| Utilisateur | **Créer un nouvel utilisateur**, description `<cluster>-<ns>` |
| Chiffrement | facultatif (SSE-OMK) |

Le bucket reste **privé** : seuls les objets envoyés en lecture publique seront lisibles, et sa liste n'est jamais exposée.

Note la clé d'accès et la clé secrète de l'utilisateur dans ton gestionnaire de mots de passe : onglet **Utilisateurs Object Storage**, **Voir les credentials**.

> **Vérifier** · L'onglet **Mes conteneurs** liste le bucket dans la région choisie, et l'onglet **Utilisateurs Object Storage** l'utilisateur `<cluster>-<ns>`.
{.is-success}

## 15.3 · Limiter l'utilisateur à son bucket

Par défaut, un utilisateur S3 voit tous les buckets du projet. Onglet **Utilisateurs de stratégies Object Storage** (ou **Utilisateurs Object Storage**), **…** à droite de l'utilisateur, **Importer un fichier JSON**. Prépare le fichier dans WSL :

```bash
BUCKET=$CLUSTER-$NS          # nom exact du bucket créé en 15.2
cat > ~/s3-policy-$BUCKET.json <<EOF
{
  "Statement": [
    {
      "Sid": "LectureEcritureDuBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject", "s3:PutObject", "s3:PutObjectAcl", "s3:DeleteObject",
        "s3:AbortMultipartUpload", "s3:ListMultipartUploadParts",
        "s3:ListBucket", "s3:ListBucketMultipartUploads", "s3:GetBucketLocation"
      ],
      "Resource": ["arn:aws:s3:::$BUCKET", "arn:aws:s3:::$BUCKET/*"]
    },
    {
      "Sid": "PasLesAutresBuckets",
      "Effect": "Deny",
      "Action": ["s3:ListAllMyBuckets"],
      "Resource": ["arn:aws:s3:::*"]
    }
  ]
}
EOF
explorer.exe "$(wslpath -w ~)"     # ouvre le dossier dans l'explorateur Windows, pour l'import
```

`s3:PutObjectAcl` est nécessaire : l'application envoie chaque objet en lecture publique.

> **Vérifier** · Après l'import, **Télécharger le fichier JSON** de l'utilisateur renvoie bien le nom de ton bucket dans `Resource`. Le contrôle réel est fait à l'étape suivante.
{.is-success}

## 15.4 · Tester avec l'AWS CLI

Une fois par poste :

```bash
pipx install awscli
```

Puis, pour le bucket :

```bash
S3_REGION=<region>                    # code de la région du bucket, en minuscules
mkdir -p ~/.aws
cat >> ~/.aws/config <<EOF

[profile ovh-$BUCKET]
region = $S3_REGION
endpoint_url = https://s3.$S3_REGION.io.cloud.ovh.net
request_checksum_calculation = when_required
response_checksum_validation = when_required
EOF
aws configure --profile ovh-$BUCKET        # clé d'accès, clé secrète ; Entrée pour les deux dernières questions
```

Les deux réglages `checksum` évitent un refus d'OVH avec les versions récentes des SDK AWS, qui ajoutent par défaut des sommes de contrôle que tous les S3 compatibles ne gèrent pas.

```bash
base64 -d > /tmp/pixel.png <<< iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=
aws --profile ovh-$BUCKET s3 cp /tmp/pixel.png s3://$BUCKET/test/pixel.png --acl public-read --content-type image/png
aws --profile ovh-$BUCKET s3 ls s3://$BUCKET/test/
aws --profile ovh-$BUCKET s3 ls
curl -s -o /dev/null -w '%{http_code}\n' https://$BUCKET.s3.$S3_REGION.io.cloud.ovh.net/test/pixel.png
curl -s -o /dev/null -w '%{http_code}\n' https://$BUCKET.s3.$S3_REGION.io.cloud.ovh.net/
```

> **Vérifier**
>
> | Commande | Résultat attendu |
> | --- | --- |
> | `s3 cp` | `upload: /tmp/pixel.png to s3://<bucket>/test/pixel.png` |
> | `s3 ls s3://…/test/` | une ligne `pixel.png` |
> | `s3 ls` (tous les buckets) | `AccessDenied`, ou ce seul bucket : l'utilisateur ne voit pas les autres |
> | `curl` sur l'objet | `200` : il est public |
> | `curl` sur la racine | `403` : la liste du bucket reste privée |
>
> `SignatureDoesNotMatch` : clé secrète mal copiée. `AccessDenied` sur `cp` : la politique de 15.3 ne nomme pas ce bucket, ou il manque `s3:PutObjectAcl`.
{.is-success}

## 15.5 · Servir le bucket sur le domaine

Dans `$ENV_DIR/values.yaml` :

```yaml
publicBuckets:
  files:                       # nom du proxy, et sous-domaine : files<suffixe>.<zone>
    namespace: <ns>
    bucket: <cluster>-<ns>
    region: <region>
```

Puis :

```bash
make config-check
make apps
make urls | grep files
```

Projet suivant sur la même zone, sans joker DNS : `cloudflared tunnel route dns $CLUSTER files$SUFFIX.$ZONE` (Mise en route, 4.4).

> **Vérifier**
>
> ```bash
> kubectl -n $NS get deploy,ingress files
> for i in 1 2; do curl -sI https://files$SUFFIX.$ZONE/test/pixel.png | grep -iE '^(HTTP|cache-control|cf-cache-status)'; done
> curl -s -o /dev/null -w '%{http_code}\n' https://files$SUFFIX.$ZONE/
> curl -s -o /dev/null -w '%{http_code}\n' https://files$SUFFIX.$ZONE/test/absent.png
> curl -s -o /dev/null -w '%{http_code}\n' -X PUT -d x https://files$SUFFIX.$ZONE/test/pixel.png
> ```
>
> | Contrôle | Résultat attendu |
> | --- | --- |
> | Proxy | déploiement `files` en `1/1`, ingress sur `files<suffixe>.<zone>` |
> | Premier appel | `200`, `cache-control: public, max-age=2592000`, `cf-cache-status: MISS` |
> | Second appel | `cf-cache-status: HIT` : servi par Cloudflare, sans passer par le serveur |
> | Racine, fichier absent, `PUT` | `404` |
>
> `cf-cache-status` absent ou `DYNAMIC` : le nom n'a pas d'extension mise en cache par défaut (voir la règle des clés en 15.7).
{.is-success}

## 15.6 · Donner les identifiants à l'application

```bash
make secrets-edit
```

Sous le namespace de l'application, dans `tenants.<ns>.appSecrets`, ajoute :

```yaml
tenants:
  <ns>:
    appSecrets:
      s3-env:
        S3_ENDPOINT: https://s3.<region>.io.cloud.ovh.net
        S3_REGION: <region>
        S3_BUCKET: <cluster>-<ns>
        S3_PUBLIC_URL: https://files<suffixe>.<zone>
        AWS_ACCESS_KEY_ID: <clé d'accès>
        AWS_SECRET_ACCESS_KEY: <clé secrète>
```

```bash
make platform
git add $ENV_DIR && git commit -m "Stockage S3 : bucket public et identifiants" && git push
```

Dans le dépôt de l'application, `deploy/values-test.yaml` :

```yaml
envFromSecrets:
  - s3-env
```

> **Vérifier** · `kubectl -n $NS get secret s3-env -o jsonpath='{.data}' | jq -r 'keys[]'` liste les six variables. Après le prochain déploiement de l'API, `kubectl -n $NS exec deploy/<api> -- printenv S3_BUCKET` affiche le nom du bucket.
{.is-success}

## 15.7 · Côté Spring Boot

Dépendance Maven, version 2.30 ou plus récente du SDK AWS (Renovate la tient ensuite à jour) :

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>software.amazon.awssdk</groupId>
      <artifactId>bom</artifactId>
      <version>${aws-sdk.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
  </dependency>
</dependencies>
```

`application.yml` :

```yaml
s3:
  endpoint: ${S3_ENDPOINT}
  region: ${S3_REGION}
  bucket: ${S3_BUCKET}
  public-url: ${S3_PUBLIC_URL}

spring:
  servlet:
    multipart:
      max-file-size: 15MB
      max-request-size: 20MB
```

Configuration et service de stockage :

```java
package fr.augustinde.demo.storage;

import java.net.URI;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import software.amazon.awssdk.auth.credentials.EnvironmentVariableCredentialsProvider;
import software.amazon.awssdk.core.checksums.RequestChecksumCalculation;
import software.amazon.awssdk.core.checksums.ResponseChecksumValidation;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;

@Configuration
@EnableConfigurationProperties(S3Config.S3Properties.class)
public class S3Config {

    @ConfigurationProperties(prefix = "s3")
    public record S3Properties(String endpoint, String region, String bucket, String publicUrl) {}

    @Bean
    S3Client s3Client(S3Properties props) {
        return S3Client.builder()
                .endpointOverride(URI.create(props.endpoint()))
                .region(Region.of(props.region()))
                // AWS_ACCESS_KEY_ID et AWS_SECRET_ACCESS_KEY, injectés par le secret s3-env
                .credentialsProvider(EnvironmentVariableCredentialsProvider.create())
                // Sommes de contrôle seulement quand l'API S3 les exige : compatibilité OVH
                .requestChecksumCalculation(RequestChecksumCalculation.WHEN_REQUIRED)
                .responseChecksumValidation(ResponseChecksumValidation.WHEN_REQUIRED)
                .build();
    }
}
```

```java
package fr.augustinde.demo.storage;

import java.io.IOException;
import java.io.InputStream;
import java.util.Map;
import java.util.UUID;

import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.DeleteObjectRequest;
import software.amazon.awssdk.services.s3.model.ObjectCannedACL;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;

/**
 * Fichiers publics : chaque envoi crée une clé unique (dossier/uuid.extension), lisible publiquement.
 * On enregistre la clé en base ; l'URL publique se reconstruit avec publicUrl(key).
 */
@Service
public class PublicFileStorage {

    // SVG exclu : un SVG envoyé par un utilisateur et servi depuis ton domaine peut contenir du script
    private static final Map<String, String> EXTENSIONS = Map.of(
            "image/jpeg", "jpg",
            "image/png", "png",
            "image/webp", "webp",
            "image/avif", "avif",
            "image/gif", "gif");

    private final S3Client s3;
    private final S3Config.S3Properties props;

    public PublicFileStorage(S3Client s3, S3Config.S3Properties props) {
        this.s3 = s3;
        this.props = props;
    }

    /** Envoie le fichier dans le dossier donné (ex. "produits", "avatars") et renvoie sa clé. */
    public String store(String folder, MultipartFile file) throws IOException {
        String contentType = file.getContentType();
        String extension = contentType == null ? null : EXTENSIONS.get(contentType);
        if (extension == null) {
            throw new IllegalArgumentException("Type de fichier refusé : " + contentType);
        }
        String key = folder + "/" + UUID.randomUUID() + "." + extension;
        PutObjectRequest request = PutObjectRequest.builder()
                .bucket(props.bucket())
                .key(key)
                .contentType(contentType)
                .acl(ObjectCannedACL.PUBLIC_READ)
                .build();
        try (InputStream in = file.getInputStream()) {
            s3.putObject(request, RequestBody.fromInputStream(in, file.getSize()));
        }
        return key;
    }

    /** URL servie par le proxy et le cache Cloudflare. */
    public String publicUrl(String key) {
        return props.publicUrl() + "/" + key;
    }

    public void delete(String key) {
        s3.deleteObject(DeleteObjectRequest.builder().bucket(props.bucket()).key(key).build());
    }
}
```

Trois règles pour les clés :

- **Une clé = un contenu.** Cloudflare garde un fichier 30 jours : pour remplacer une image (nouvel avatar), envoie-la sous une nouvelle clé et supprime l'ancienne, ne la réécris pas.
- **Garder l'extension** (`.jpg`, `.webp`…) : Cloudflare ne met en cache par défaut que les fichiers dont l'extension est connue.
- **En base, la clé, pas l'URL** : `produits/3f2c….webp`. L'URL publique se reconstruit avec `publicUrl(key)`.

> **Vérifier** · Envoie une image par l'API en test, puis ouvre l'URL renvoyée par `publicUrl` : l'image s'affiche, et le second chargement répond `cf-cache-status: HIT`.
{.is-success}

## 15.8 · Prod

Même chose avec un bucket et un utilisateur séparés : la prod ne doit pas pouvoir écrire dans le bucket de test, ni l'inverse.

1. 15.2 et 15.3 avec le nom `<cluster>-<ns>-prod` (`BUCKET=$CLUSTER-$NS-prod`), puis 15.4.
2. Dans `values.yaml`, un second proxy avec le nom de prod :

```yaml
publicBuckets:
  files-prod:
    namespace: <ns>-prod
    bucket: <cluster>-<ns>-prod
    region: <region>
    host: files.<zone>
```

3. `make apps`, puis le secret `s3-env` sous `tenants.<ns>-prod.appSecrets`, avec `S3_PUBLIC_URL: https://files.<zone>`, et `make platform`.
4. `envFromSecrets: [s3-env]` dans `deploy/values-prod.yaml`.

> **Vérifier** · Les contrôles de 15.5 sur `https://files.<zone>/test/pixel.png`, et `aws --profile ovh-<cluster>-<ns>-prod s3 ls s3://<cluster>-<ns>/` (bucket de test avec l'utilisateur de prod) répond `AccessDenied`.
{.is-success}

## Si ça ne marche pas

| Symptôme | Cause probable | Correctif |
| --- | --- | --- |
| `404` sur une image qui existe | objet envoyé sans `--acl public-read` (ou `ObjectCannedACL.PUBLIC_READ`) | `aws … s3api put-object-acl --bucket <bucket> --key <clé> --acl public-read` |
| `502` sur `files<suffixe>.<zone>` | proxy qui ne joint pas OVH (nom de bucket ou région faux) | `kubectl -n $NS logs deploy/files` ; corriger `bucket` ou `region`, puis `make apps` |
| Erreur 1033 sur `files<suffixe>.<zone>` | nom absent du DNS (projet sans joker) | `cloudflared tunnel route dns $CLUSTER files$SUFFIX.$ZONE` |
| `cf-cache-status: DYNAMIC` | clé sans extension, ou extension hors de la liste Cloudflare | clés avec extension (15.7) |
| Ancienne image toujours affichée | même clé réécrite, encore en cache | nouvelle clé ; en urgence, Cloudflare, Caching, **Purge Cache** sur l'URL |
| Envoi refusé par l'API au-delà d'1 Mo | limite par défaut de Spring Boot | `spring.servlet.multipart` (15.7) |
| `InvalidRequest` ou erreur de checksum à l'envoi | SDK AWS récent sans le réglage `WHEN_REQUIRED` | 15.7, et `request_checksum_calculation` pour la CLI (15.4) |
