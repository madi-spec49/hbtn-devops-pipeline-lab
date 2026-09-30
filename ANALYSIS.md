# ANALYSIS.md — Portée et limites du pipeline

## Ce qu'un pipeline vert garantit

Un run vert (`test → build → deploy` tous réussis) établit les faits suivants :

### Reproductibilité de l'installation
`npm ci` + `package-lock.json` commité → l'arbre de dépendances installé en CI est identique à celui revu par un humain. Pas de dérive de version.

### Conformité de style
Le step `Lint` (`npm run lint`) détecte erreurs ESLint, variables non utilisées, patterns problématiques — avant toute construction.

### Non-régression fonctionnelle
Les 11 tests (8 unitaires + 3 d'intégration) passent :
- les fonctions testées se comportent comme prévu ;
- l'API répond correctement sur les routes couvertes ;
- les interactions PostgreSQL fonctionnent ;
- les migrations et seeds s'appliquent.

### Intégrité de l'artefact
Le job `build` produit une image Docker **immuable**, taguée par le SHA du commit. Deux tags coexistent :
- `ghcr.io/...:<SHA>` — trace exacte, utilisable pour rollback ;
- `ghcr.io/...:latest` — commodité, mouvant.

### Déployabilité en environnement réel
Le job `deploy` redéploie sur un environnement distinct (Render + PostgreSQL Neon) et vérifie :
- `/health` → **200** = processus vivant ;
- `/items` → **200** = l'API interroge la base avec succès.

Un vert sur ces deux endpoints prouve que l'app démarre et fonctionne sur une plateforme qui n'est pas celle du runner CI.

### Blocage des chaînes en aval
`needs:` empêche `build` de démarrer si `test` échoue, et `deploy` si `build` échoue. Aucune image n'est publiée, aucun staging modifié, tant que les conditions préalables ne sont pas remplies.

### Préservation des preuves
`Upload test reports` avec `if: always()` garantit qu'un échec laisse une trace exploitable. Le rapport JUnit est mis à disposition comme artefact (rétention 7 jours).

### Absence de credential en dur
Usage exclusif de `${{ secrets.* }}` et `${{ vars.* }}`. Aucune clé API, aucun mot de passe, aucun token n'est commité, loggé ou exposé.

---

## Ce qu'un pipeline vert ne garantit PAS

### Absence de bug
Les tests couvrent uniquement les cas écrits par les développeurs. Les cas limites, inputs malformés et chemins non testés restent hors couverture. *Un test qui passe prouve la non-régression sur ce qui est testé, pas l'absence de bug.*

### Sécurité
Aucun scan de vulnérabilités :
- pas de `npm audit` / Dependabot ;
- pas de scan d'image (Trivy, Grype) ;
- pas de SBOM ni de signature (cosign) ;
- pas de policy-as-code.

Une CVE dans une dépendance ou dans l'image de base passerait inaperçue.

### Performance
Aucun test de charge, aucun benchmark, aucun profiling. Un test vert ne dit rien sur la latence, l'usage mémoire ou la tenue sous charge.

### Comportement en production
Le staging est simplifié : une seule instance, une seule base, peu ou pas de trafic réel, pas de dépendances externes (CDN, queues, cache distribué). Un bug qui ne se manifeste que sous charge ou avec une vraie base multi-tenant peut passer.

### Rollback automatique
Le pipeline publie une image immuable mais ne rollback pas automatiquement. Le rollback est une action manuelle documentée dans `DEPLOY.md`.

### Observabilité
Aucune métrique runtime (Prometheus, Grafana), aucune trace distribuée, aucun alerting. Un vert ne dit pas si l'app se porte bien en continu.

### Conformité réglementaire
Aucune gestion RGPD, SOC 2, PCI DSS, journalisation d'audit, politiques de rétention.

---

## Synthèse

**Un pipeline vert établit :**
> « Le code revu a passé des vérifications reproductibles, une image immuable a été publiée, et un staging a été mis à jour et confirmé fonctionnel. »

**Un pipeline vert n'établit PAS :**
> « Le logiciel est exempt de bugs, sûr, performant, conforme, et prêt pour la production. »

**Règle pratique :**
> Un pipeline vert est une **condition nécessaire**, jamais une **condition suffisante**. Il élimine les erreurs qu'il sait détecter — pas celles qu'il ignore.

---

## Application à ce projet

Le pipeline a prouvé que :
- les 11 tests définis passent ;
- l'image Docker se construit et se publie sur GHCR ;
- l'app démarre sur Render et interroge PostgreSQL (Neon) avec succès.

Le pipeline n'a pas prouvé que :
- l'API résiste à 1000 req/s ;
- les dépendances npm sont exemptes de CVE ;
- aucun cas limite non testé ne provoquera un crash ;
- l'app se remettra d'une panne de la base Neon.

Ces garanties exigent des outils supplémentaires — scans de sécurité, tests de charge, observabilité, IaC. Chacun étend la **couverture de confiance**, aucun ne l'atteint complètement.