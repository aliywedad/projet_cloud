# Guide pas-à-pas : PROJET-1 — Infrastructure Web évolutive (AWS CloudFormation)

Ce guide t'explique, étape par étape, comment réaliser le projet décrit dans
[peojet.md](peojet.md). Il est écrit pour quelqu'un qui n'a **jamais utilisé AWS**.

Adapté à ta situation :
- Tu as **déjà un compte AWS avec accès admin** → pas besoin de créer de compte/utilisateur IAM.
- Tu utiliseras une **appli Flask CRUD** comme application web (parfait, elle utilisera RDS
  pour le C-R-U-D → ça justifie bien la base de données).
- Tu veux tout faire via la **console web AWS (interface graphique)**, pas en ligne de commande.
  Tous les déploiements ci-dessous se font donc en cliquant dans la console CloudFormation
  (pas de `aws cloudformation deploy`).

Suis les étapes dans l'ordre — ne saute rien.

---

## 0. Comprendre ce que tu vas construire

```
                         Internet
                            │
                      ┌─────▼─────┐
                      │    ALB    │  (Load Balancer public)
                      └─────┬─────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
     ┌────▼────┐      ┌────▼────┐       ┌────▼────┐
     │ EC2 #1  │      │ EC2 #2  │  ...  │ EC2 #N  │   (Auto Scaling Group,
     │ Flask   │      │ Flask   │       │ Flask   │    instances t3.micro,
     │ (privé) │      │ (privé) │       │ (privé) │    dans 2+ zones)
     └────┬────┘      └────┬────┘       └────┬────┘
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                      ┌─────▼─────┐
                      │    RDS    │  (MySQL/PostgreSQL, privée,
                      │(db.t4g.micro)  utilisée par le CRUD Flask)
                      └───────────┘
```

Vocabulaire de base :

| Terme | Explication simple |
|---|---|
| **VPC** | Ton propre réseau privé virtuel dans AWS. Tout le reste vit à l'intérieur. |
| **Subnet (sous-réseau)** | Une subdivision du VPC. "Public" = accessible depuis Internet. "Privé" = non accessible directement. |
| **AZ (Availability Zone)** | Un datacenter physique distinct. On répartit les ressources sur 2+ AZ pour la haute disponibilité. |
| **ALB** | Répartit le trafic web entrant vers plusieurs instances EC2. |
| **EC2** | Une machine virtuelle qui fait tourner ton application Flask. |
| **Auto Scaling Group (ASG)** | Ajoute/enlève automatiquement des EC2 selon la charge (CPU). |
| **RDS** | Base de données managée par AWS (MySQL/PostgreSQL) où ton CRUD stocke ses données. |
| **Security Group** | Un pare-feu virtuel attaché à une ressource. |
| **CloudFormation** | Outil qui crée toute ton infrastructure à partir d'un fichier texte (YAML) — "Infrastructure as Code". Tu vas écrire les fichiers YAML, puis les **déployer via la console web** (pas en CLI). |

---

## 1. Préparer ton projet

### 1.1 Ton appli Flask CRUD
Tu as besoin d'un dépôt Git avec ton appli Flask CRUD (ou tu peux l'écrire pendant ce projet).
Points importants pour qu'elle marche bien avec RDS :
- Utilise une variable d'environnement pour la connexion DB (ne mets jamais d'identifiants en dur) :
  ```python
  import os
  DB_HOST = os.environ["DB_HOST"]
  DB_USER = os.environ["DB_USER"]
  DB_PASSWORD = os.environ["DB_PASSWORD"]
  DB_NAME = os.environ["DB_NAME"]
  ```
- Fais en sorte que Flask écoute sur `0.0.0.0` (pas juste `127.0.0.1`), sinon l'ALB/health check ne pourra pas l'atteindre :
  ```python
  if __name__ == "__main__":
      app.run(host="0.0.0.0", port=5000)
  ```
- Ajoute une route simple type `/health` qui répond `200 OK` — elle servira de health check à l'ALB.
- Utilise `Flask-SQLAlchemy` (ou `PyMySQL`/`psycopg2`) pour parler à RDS, et appelle
  `db.create_all()` au démarrage pour créer les tables automatiquement (simple pour un projet étudiant).

### 1.2 Créer ton dépôt GitHub (via l'interface web)
1. Va sur https://github.com/new
2. Nom du repo : `projet1-infra-web`
3. Public ou privé, avec un README
4. Clique **Create repository**
5. Ajoute ton code Flask et les fichiers `templates/*.yaml` de CloudFormation dedans (tu peux
   glisser-déposer les fichiers directement depuis l'interface GitHub, ou utiliser Git en local).

Structure de dossier recommandée :
```
projet1-infra-web/
├── templates/                 # tes fichiers CloudFormation (YAML)
│   ├── vpc.yaml
│   ├── security.yaml
│   ├── alb-asg.yaml
│   ├── rds.yaml
│   └── monitoring.yaml
├── app/                        # ton code Flask CRUD
│   ├── app.py
│   ├── requirements.txt
│   └── templates/ (HTML Flask, pas CloudFormation)
├── README.md
└── GUIDE.md
```

---

## 2. Construire l'infrastructure, étape par étape

**Stratégie pour débutant : ne fais pas un seul énorme template.** Écris et déploie un
morceau à la fois, dans cet ordre, **via la console CloudFormation**. Chaque étape doit
fonctionner avant de passer à la suivante.

### Comment déployer un template via la console (à répéter à chaque étape)
1. Console AWS → cherche **CloudFormation** → clique dessus.
2. Clique **Create stack** → **With new resources (standard)**.
3. Dans "Prerequisite - prepare template", choisis **Template is ready**.
4. Dans "Specify template", choisis **Upload a template file** → clique **Choose file** →
   sélectionne ton fichier YAML local (ex. `templates/vpc.yaml`).
5. Clique **Next**.
6. Donne un **Stack name** (ex. `projet1-vpc`), remplis les paramètres si le template en demande.
7. Clique **Next**, **Next** à nouveau (options par défaut).
8. En bas, coche la case **"I acknowledge that AWS CloudFormation might create IAM resources"**
   si elle apparaît.
9. Clique **Submit**.
10. Attends que le statut passe de `CREATE_IN_PROGRESS` à `CREATE_COMPLETE` (actualise la page,
    ou regarde l'onglet **Events** pour voir la progression / les erreurs).
11. Va dans l'onglet **Outputs** du stack : c'est là que tu récupères les valeurs (VPC ID,
    subnet IDs, URL de l'ALB, endpoint RDS...) dont tu auras besoin pour l'étape suivante.

### Étape 2.1 — Le VPC et les sous-réseaux (`templates/vpc.yaml`)
Doit définir :
- 1 VPC (ex. `10.0.0.0/16`)
- 2 sous-réseaux **publics** (un par AZ) — pour l'ALB
- 2 sous-réseaux **privés** (un par AZ) — pour EC2 et RDS
- 1 Internet Gateway attachée au VPC
- 1 NAT Gateway (pour que les instances privées puissent sortir vers Internet : mises à jour, `pip install`...)
- Tables de routage associées
- **Outputs** : VpcId, les 4 SubnetIds (exportés avec `Export`, pour les templates suivants)

Déploie-le via la console (voir procédure ci-dessus, stack name `projet1-vpc`).
Puis vérifie visuellement : Console → VPC → tes sous-réseaux et AZ doivent apparaître.

### Étape 2.2 — Security Groups (`templates/security.yaml`)
- SG pour l'ALB : autorise le port 80 depuis `0.0.0.0/0` (Internet).
- SG pour les EC2 : autorise le port 5000 (ou 80 si tu mets Nginx devant Flask) **uniquement**
  depuis le SG de l'ALB (pas depuis Internet directement).
- SG pour RDS : autorise le port 3306 (MySQL) ou 5432 (PostgreSQL) **uniquement** depuis le SG des EC2.

Déploie via la console (stack name `projet1-security`), en important le VpcId du stack précédent
(`Fn::ImportValue`).

### Étape 2.3 — ALB + Auto Scaling Group EC2 (`templates/alb-asg.yaml`)
- Application Load Balancer, placé dans les sous-réseaux **publics**.
- Target Group avec health check sur `/health` (celui que tu as ajouté dans Flask).
- Launch Template : `InstanceType: t3.micro` (⚠️ exigence obligatoire), avec un `UserData` qui :
  1. Installe Python3, pip, git
  2. Clone ton repo GitHub (le dossier `app/`)
  3. Installe les dépendances (`pip install -r requirements.txt`)
  4. Exporte les variables d'environnement DB (host = endpoint RDS, user, password, nom DB)
  5. Lance Flask (idéalement avec `gunicorn` en arrière-plan, via un service systemd, pour que
     l'appli redémarre si l'instance reboote)
- Auto Scaling Group : instances dans les sous-réseaux **privés**, min 2 / max 4, attaché au
  Target Group.
- Policy de scaling **basée sur le CPU** (target tracking, ex. 60%) → répond à l'exigence
  "la mise à l'échelle automatique doit se déclencher en fonction de l'utilisation CPU".

Déploie via la console (stack name `projet1-alb-asg`).

**Test immédiat :** onglet **Outputs** du stack → copie l'URL/DNS de l'ALB → colle-la dans
ton navigateur. Tu dois voir ton appli Flask CRUD. ✅ Ça valide le critère "L'application doit
être accessible via le point de terminaison ALB".

### Étape 2.4 — RDS (`templates/rds.yaml`)
- DB Subnet Group utilisant les sous-réseaux **privés**.
- Instance RDS : `DBInstanceClass: db.t4g.micro` (⚠️ exigence obligatoire), moteur MySQL ou PostgreSQL.
- `PubliclyAccessible: false` (garantit qu'elle est privée).
- Utilise le Security Group RDS créé à l'étape 2.2.
- Pour le mot de passe, le plus simple pour un projet étudiant : un paramètre CloudFormation
  de type `NoEcho: true` que tu saisis dans la console au moment du déploiement (évite de
  l'écrire en dur dans le fichier).

Déploie via la console (stack name `projet1-rds`). Récupère l'**endpoint RDS** dans les Outputs
→ c'est le `DB_HOST` que ton Auto Scaling Group EC2 doit utiliser (étape 2.3). Si tu déploies
RDS après l'ASG, tu devras mettre à jour le stack `projet1-alb-asg` (bouton **Update stack**
dans la console) une fois que tu as l'endpoint.

### Étape 2.5 — CloudWatch (`templates/monitoring.yaml`)
- Log Group CloudWatch pour les logs de ton appli Flask (l'agent CloudWatch envoie les logs
  depuis chaque EC2 — installe/configure `amazon-cloudwatch-agent` dans le `UserData`).
- Alarme CloudWatch sur le CPU moyen de l'ASG.

### Étape 2.6 — Tagger toutes les ressources
Chaque ressource de chaque template doit avoir des tags :
```yaml
Tags:
  - Key: Project
    Value: PROJET-1
  - Key: Environment
    Value: dev
  - Key: Owner
    Value: TonNom
```
C'est un critère d'acceptation explicite — ne l'oublie pas dans **chaque** ressource.

---

## 3. Vérifier les critères d'acceptation (checklist finale)

- [ ] Toutes les ressources sont taguées → Console → **Resource Groups & Tag Editor**
- [ ] L'app Flask CRUD est accessible via l'URL de l'ALB → ouvre-la dans un navigateur, teste Create/Read/Update/Delete
- [ ] RDS est privée et sécurisée → Console RDS → "Publicly Accessible" = No
- [ ] L'auto scaling réagit au CPU → connecte-toi à une instance (Session Manager) et lance
      une charge CPU artificielle (ex. `stress --cpu 2`), observe une nouvelle instance
      apparaître dans l'ASG (Console → EC2 → Auto Scaling Groups → Activity)
- [ ] Les logs sont visibles dans CloudWatch → Console CloudWatch → Log groups
- [ ] Code (Flask + templates CloudFormation) poussé sur GitHub
- [ ] Tu peux expliquer l'architecture et le VPC → prépare 5–10 min d'explication

---

## 4. Mettre en pause / tout détruire facilement (sans changer d'outil)

Tu voulais pouvoir "pauser" ou détruire toute l'infra en une seule commande, comme avec
Terraform. CloudFormation le permet aussi très bien — pas besoin de changer d'outil :

### Option A — Mettre en pause (sans supprimer, pour ne pas repartir de zéro)
Utile entre deux sessions de travail pour économiser des coûts, sans perdre ta config.
1. **ASG → 0 instance** : Console EC2 → Auto Scaling Groups → sélectionne ton ASG →
   **Edit** → mets Desired/Min à `0` → **Update**. Toutes les instances EC2 s'arrêtent,
   mais l'ASG, l'ALB, les Target Groups restent configurés.
2. **RDS → Stop** : Console RDS → sélectionne ta base → **Actions** → **Stop temporarily**.
   (AWS la redémarre automatiquement après 7 jours si tu ne l'as pas relancée avant — pense
   à vérifier si ta session de travail dépasse cette durée.)
3. Pour reprendre : remets l'ASG à Desired ≥ 2 et redémarre RDS (**Actions → Start**).

### Option B — Tout détruire en une seule action (comme `terraform destroy`)
En CloudFormation, l'équivalent exact est de supprimer les stacks. Tu peux le faire :
- **En un clic par stack** dans la console : CloudFormation → sélectionne le stack → **Delete**.
- **En une seule commande** si tu veux automatiser, via l'AWS CLI (optionnel, tu n'es pas
  obligé de l'utiliser, mais ça répond exactement à ton besoin de "1 commande") :
  ```bash
  for s in projet1-monitoring projet1-rds projet1-alb-asg projet1-security projet1-vpc; do
    aws cloudformation delete-stack --stack-name "$s"
    aws cloudformation wait stack-delete-complete --stack-name "$s"
  done
  ```
  Ce script attend que chaque stack soit bien supprimé avant de passer au suivant (l'ordre
  compte : on détruit dans l'ordre inverse de la création, sinon ça échoue à cause des
  dépendances comme le VPC).

Une fois le projet noté/terminé, utilise l'Option B pour tout supprimer. Ordre :
`projet1-monitoring` → `projet1-rds` → `projet1-alb-asg` → `projet1-security` → `projet1-vpc`

Vérifie ensuite dans EC2/VPC qu'il ne reste aucune ressource orpheline (Elastic IP, NAT Gateway
— ce sont les plus coûteuses si oubliées).

---

## 5. Préparer ton explication orale

Sois capable de répondre à :
- Pourquoi 2 AZ minimum ? (haute disponibilité, tolérance de panne)
- Pourquoi RDS et EC2 sont en sous-réseau privé et pas l'ALB ?
- Comment le trafic circule de l'utilisateur jusqu'à la base de données (ALB → EC2 Flask → RDS) ?
- Comment fonctionne le scaling automatique (quelle métrique CPU, quel seuil) ?
- Que fait le NAT Gateway et pourquoi les instances privées en ont besoin ?
- Comment ton appli Flask CRUD utilise RDS (quelles opérations Create/Read/Update/Delete, quelle table) ?

---

**Prochaine étape suggérée :** commence par l'étape 2.1 (VPC) — écris `templates/vpc.yaml`
et déploie-le via la console. Dis-moi quand c'est fait ou si tu bloques, on avancera brique
par brique.
