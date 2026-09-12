# Les 2 Secrets hors git

Règle : **aucun Secret n'est commité**. Ces deux-là sont créés à la main, une fois
(et recréés si le cluster est reconstruit). Tout le reste est déclaratif.

## 1. `ghcr-pull` (namespace `ecommerce`, type `docker-registry`)

Les Deployments font `imagePullSecrets: [ghcr-pull]` pour tirer les images
privées `ghcr.io/youssefabidi69/*`.

```bash
export GHCR_USER=<user github> GHCR_READ_TOKEN=<token scope read:packages>
kubectl create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io \
  --docker-username="$GHCR_USER" \
  --docker-password="$GHCR_READ_TOKEN" \
  -n ecommerce --dry-run=client -o yaml | kubectl apply -f -
```

## 2. `pg-app` (namespace `ecommerce`, type `Opaque`, clé `password`)

**Le piège** : `order-service`, `payment-service` et `shipping-service` lisent
`secretKeyRef: {name: pg-app, key: password}` — donc un Secret `pg-app` **dans
leur propre namespace `ecommerce`**. Mais c'est l'opérateur CNPG qui génère ce
mot de passe, et il crée le Secret `pg-app` dans le namespace du cluster, `data`.
Un `secretKeyRef` ne traverse pas les namespaces : sans recopie, les 3 pods
restent en `CreateContainerConfigError: secret "pg-app" not found`.

Fix appliqué (simple, assumé pour le stage) : recopier le Secret à la main.

```bash
kubectl get secret pg-app -n data -o json | python3 -c "
import json, sys
s = json.load(sys.stdin)
s['metadata'] = {k: v for k, v in s['metadata'].items() if k in ('name', 'labels', 'annotations')}
print(json.dumps(s))
" | kubectl apply -n ecommerce -f -
# à rejouer si le mot de passe est régénéré côté CNPG (rare ; sinon restart des pods suffit,
# env injectée au démarrage : kubectl rollout restart deploy/order-service -n ecommerce ...)
```

**Évolution propre (hors stage)** : un opérateur de miroir (`external-secrets`
ou `reflector`) qui synchronise `data/pg-app` → `ecommerce/pg-app` en continu.
Mentionnée dans le rapport comme limite connue, pas implémentée : un opérateur
de plus sur 2×B2s_v2 coûte plus cher que le problème qu'il résout ici.
