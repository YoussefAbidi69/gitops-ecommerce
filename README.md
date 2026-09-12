# gitops-ecommerce — état désiré du cluster (source de vérité)

Ce repo est lu en continu par **ArgoCD** : tout commit sur `main` est réconcilié
automatiquement vers le cluster. On ne fait plus `kubectl apply` à la main —
sauf le tout premier : `kubectl apply -n argocd -f argocd/root.yaml`.

Importé depuis `k8s/` du repo `ecommerce-microservices`, puis adapté GitOps :
ajout des `kustomization.yaml`, des **sync-waves** ArgoCD et des Applications.

```
gitops-ecommerce/
├── platform/    namespaces, StorageClasses, ClusterIssuers (vague 0)
├── data/        CNPG (cluster/databases/pooler), Kafka (+topics), Mongo (vague 1)
├── app/         ConfigMap, 8 microservices, UI, Ingress (vague 2)
├── argocd/      root (App of Apps) + 3 Applications enfants
├── bootstrap/   scripts one-shot : Argo, GitLab, runner, secrets (PAS gérés par Argo)
└── examples/    pipeline GitLab CI d'exemple + script de création de ce repo
```

## Ordre de synchronisation

| App Argo | Contenu | Vagues internes |
|---|---|---|
| `platform` (0) | namespaces → storageclasses → issuers | 0, 1, 2 |
| `data` (1) | cluster/kafka/mongo → databases+pooler → topics | 0, 1, 2 |
| `app` (2) | configmap → registry → services+UI → ingress | 0, 1, 2, 3 |

`platform` est en sync auto **sans prune** (on ne supprime jamais un Namespace ou
une StorageClass par accident) ; `data` et `app` sont en auto + prune + selfHeal.

## Ce qui n'est PAS dans git (volontaire)

Deux Secrets, créés à la main (voir `app/secrets.README.md`) :

1. `ghcr-pull` (namespace `ecommerce`) — pull des images privées GHCR.
2. `pg-app` (namespace `ecommerce`) — mot de passe Postgres, recopié depuis
   `data` où l'opérateur CNPG le génère (`secretKeyRef` ne traverse pas les namespaces).

## Ordre d'installation (détail : guide dans le repo source, `docs/GITOPS-SETUP.md`)

L'installation se fait **commande par commande** depuis le guide — aucun script,
chaque étape est expliquée avant d'être exécutée. Depuis `gitops-ecommerce/` :

- partie A : GitLab auto-hébergé (externes + chart 10.x) → étapes A1 à A8
- partie B : ArgoCD + App of Apps → étapes B1 à B4
- partie C : runner DinD + boucle CI complète → étapes C1 à C4

Prérequis : opérateurs **CNPG** et **Strimzi** déjà installés (leurs CRDs sont
indispensables au sync `data`), Traefik + cert-manager + `letsencrypt-prod` en place.

## Boucle CI/CD

```
git push (code) -> GitLab CI : build DinD -> push GHCR -> bump tag ici (kustomize edit set image)
                                                                          |
                                                                          v
                                                            ArgoCD sync -> rollout
```

Exemple de pipeline : `examples/.gitlab-ci.yml` (à adapter dans le repo source).

## Repo public ou privé ?

Public = ArgoCD le lit sans credential (recommandé ici : les manifests ne
contiennent aucun secret, juste des noms d'hôtes). En privé, déclarer le repo
dans ArgoCD (UI : Settings → Repositories → Connect, token GitHub `repo`).
