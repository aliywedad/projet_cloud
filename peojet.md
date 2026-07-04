# Projets d'étudiants : Architecture Cloud Avancée (IaaS, PaaS) 2026

Ce document présente cinq idées de projets pour les étudiants inscrits au cours "Architecture Cloud Avancée". Chaque projet est conçu pour renforcer les concepts fondamentaux de l'infrastructure et de la plateforme AWS en utilisant AWS CloudFormation, AWS CDK.

---

## 📁 Projet 1 : Infrastructure d'application Web évolutive (PROJET-1)

### Description :

Concevoir et déployer une infrastructure d'application web multi-niveaux et hautement disponible en utilisant **AWS CloudFormation**. L'architecture doit être adaptée à l'hébergement d'une application de niveau production.

### Composants de base :

- Outil IaC : AWS CloudFormation
- VPC (avec des sous-réseaux privés à travers plusieurs AZ)
- Équilibreur de charge d'application (ALB)
- EC2 Auto Scaling Group (groupe de mise à l'échelle automatique) (type d’instance : `t3.micro`, ceci est une exigence).
- RDS (MySQL/PostgreSQL) dans le sous-réseau privé (type d’instance: `db.t4g.micro` , ceci est tune exigence).
- Journaux et alarmes CloudWatch
- Rôles IAM et groupes de sécurité

### Critères d'acceptation :

- Toutes les ressources doivent être étiquetées de manière appropriée
- L'application doit être accessible via le point de terminaison ALB
- L'instance RDS doit être privée et sécurisée
- La mise à l'échelle automatique doit se déclencher en fonction de l'utilisation de l'unité centrale.
- Les logs sont visibles dans CloudWatch
- Code du projet poussé sur GitHub
- L'étudiant doit expliquer l'architecture et la configuration du VPC

---
    