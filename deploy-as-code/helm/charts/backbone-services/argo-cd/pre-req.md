# Pre-requisites

Steps to complete before running `helm install`/`helm upgrade` for this chart.

## 1. helm-secrets private keys Secret

The repo server uses [helm-secrets](https://github.com/jkroepke/helm-secrets) (SOPS backend) to decrypt encrypted values files, configured via `repoServer.env`, `repoServer.initContainers`, `repoServer.volumeMounts` and `repoServer.volumes` in `values.yaml`.

The repo server pod expects a Secret named `helm-secrets-private-keys` mounted at `/helm-secrets-private-keys/`, containing the SOPS decryption key.

* **SOPS with age (current default, see `repoServer.env` → `SOPS_AGE_KEY_FILE`):**

  ```bash
  kubectl create secret generic helm-secrets-private-keys \
    --from-file=keys.txt=assets/age/keys.txt \
    -n argocd
  ```

* **SOPS with GPG (alternative — uncomment `HELM_SECRETS_LOAD_GPG_KEYS` in `repoServer.env` if used instead of age):**

  ```bash
  kubectl create secret generic helm-secrets-private-keys \
    --from-file=key.asc=assets/gpg/private2.gpg \
    -n argocd
  ```

Only one of the two (age or GPG) is needed, matching whichever env var is active in `values.yaml`.

## 2. Git repository access

`configs.repositories` registers `hemanth-beehyv/iras-DevOps` (<https://github.com/hemanth-beehyv/iras-DevOps>) as a source repo for Applications. This repo is **public**, so no credentials Secret is required.

If the repo is ever made private, uncomment the `username`/`password` fields under that entry in `values.yaml` and supply a valid GitHub PAT before installing.

## 3. External Redis

This chart is configured to use an existing Redis in the `backbone` namespace instead of deploying its own:

* `redis.enabled: false` and `redisSecretInit.enabled: false` — the chart's bundled Redis and its secret-init hook are disabled.
* `externalRedis.host: redis.backbone.svc.cluster.local`, with empty `username`/`password` — no auth.

Make sure that Redis instance is deployed and reachable in the `backbone` namespace before installing this chart.
