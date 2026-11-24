\# TP 5 Kubernetes : Configuration et données persistantes



\## Environnement

\- Windows, PowerShell, Docker Desktop

\- Cluster kind `tp-k8s` : 1 control-plane + 2 workers (Kubernetes v1.37.0)

\- kubectl v1.34.1, kind v0.33.0



\## Partie A : Namespaces et ConfigMap



Création des namespaces `dev` et `prod`, puis déploiement du même `web.yaml` (Deployment nginx + Service) dans chacun, avec une ConfigMap `web-config` différente (`APP\_ENV` et `index.html`).



\- `dev` affiche `environnement dev` et `APP\_ENV=dev` ; `prod` affiche `environnement prod` et `APP\_ENV=prod`.

\- La variable `APP\_ENV` est injectée par `env.valueFrom.configMapKeyRef`, et `index.html` est monté dans `/usr/share/nginx/html` par un volume de type `configMap`.



\*\*Modification de la ConfigMap de dev (`kubectl edit cm web-config`)\*\* :

\- Le \*\*fichier monté\*\* s'est mis à jour tout seul, sans redémarrer les Pods : le kubelet synchronise les volumes ConfigMap avec un délai d'environ une minute (pas de `subPath`, sinon aucune mise à jour).

\- La \*\*variable d'environnement\*\* est restée à `dev` : elle est fixée au démarrage du conteneur.

\- `kubectl rollout restart deployment web` recrée les Pods progressivement, sans coupure et sans changer l'image. Les nouveaux Pods relisent la ConfigMap : `APP\_ENV` vaut alors `dev-modifie`.



\## Partie B : Secret



Secret `db-cred` créé par la commande `kubectl create secret generic` (clés `POSTGRES\_USER`, `POSTGRES\_PASSWORD`, `POSTGRES\_DB`). \*\*Il n'est pas versionné\*\* : aucun fichier de ce dépôt ne le contient, et le mot de passe n'apparaît pas dans ce compte rendu.



`kubectl get secret db-cred -o yaml` affiche les valeurs en base64, et un simple décodage permet de retrouver le mot de passe : base64 est un encodage, pas un chiffrement.



\## Partie C : PostgreSQL en StatefulSet



\- StorageClass par défaut : `standard`.

\- `postgres.yaml` : Service headless + StatefulSet avec `volumeClaimTemplates` (1 Gi). Le PVC `data-postgres-0` est créé automatiquement et lié à un PV.

\- Création de la table `etudiants` (Amine, Salma), suppression du Pod `postgres-0`, recréation automatique avec le même nom, et le `SELECT` retrouve les 2 lignes.



\*\*Preuve de persistance\*\* (SELECT après redémarrage du Pod) :



!\[SELECT après redémarrage](captures/persistance-select.png)



\## Défi : ResourceQuota



`quota.yaml` limite le namespace `dev` à 4 Pods. `dev` en contenait déjà 3 (2 `web` + `postgres-0`). Après `kubectl scale deployment web --replicas=6`, le ReplicaSet reste à 3 Pods sur 6. `kubectl describe rs` affiche :



> `Error creating: pods "web-7b7d7679bf-..." is forbidden: exceeded quota: pods-quota, requested: pods=1, used: pods=4, limited: pods=4`



Le quota n'agit qu'à la création : les Pods existants ne sont pas touchés.



\## Réponses aux questions



\*\*1. Un Pod de dev peut-il joindre un Service de prod ? Les Namespaces isolent-ils le réseau ?\*\*

Oui, avec le nom DNS `web.prod.svc.cluster.local` (forme `<service>.<namespace>.svc.cluster.local`). Testé depuis un Pod `busybox` de `dev` : la page de prod s'affiche, alors que `http://web` (nom court) pointe vers le Service de `dev`. Les Namespaces séparent les noms et les ressources, mais \*\*pas le réseau\*\* : tout Pod peut joindre tout Service. Pour isoler le trafic, il faut des NetworkPolicies, avec un plugin réseau (CNI) qui les applique.



\*\*2. Pourquoi ne jamais committer un Secret en clair dans Git ? Deux solutions.\*\*

Parce que base64 n'est pas un chiffrement : quiconque lit le dépôt retrouve le mot de passe en une commande. De plus, Git conserve l'historique, donc un secret supprimé reste récupérable dans les anciens commits, et un dépôt public l'expose à tous. Solutions : \*\*Sealed Secrets\*\* (le Secret est chiffré dans Git, seul le contrôleur du cluster peut le déchiffrer) et \*\*External Secrets\*\* (le cluster récupère les valeurs depuis un gestionnaire externe : Vault, AWS Secrets Manager, etc.).



\*\*3. Différences entre Deployment et StatefulSet.\*\*



| | Deployment | StatefulSet |

|---|---|---|

| Nom des Pods | aléatoire (`web-7b7d7679bf-6spws`) | stable et ordonné (`postgres-0`, `-1`, ...) |

| Ordre de démarrage | tous en parallèle | un par un, dans l'ordre |

| Volumes | partagés ou éphémères | un PVC par Pod (`volumeClaimTemplates`), conservé au redémarrage |



\*\*4. Que deviennent le PVC et les données si l'on supprime le StatefulSet ?\*\*

Le PVC et les données \*\*restent\*\*. Vérifié : après `kubectl delete statefulset postgres`, le PVC `data-postgres-0` était toujours `Bound` (plus ancien que le StatefulSet recréé), et après un nouvel `apply`, `postgres-0` a été rattaché au même PVC et le `SELECT` retrouvait Amine et Salma. Les données ne sont perdues que si l'on supprime aussi le PVC.



\## Fichiers du TP

\- `web-config.yaml`, `web-config-prod.yaml`, `web.yaml`

\- `postgres.yaml`

\- `quota.yaml`

\- (le Secret `db-cred` est volontairement exclu)

