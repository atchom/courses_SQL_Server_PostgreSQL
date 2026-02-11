# courses_SQL_Server_PostgreSQL
markdown

# 📋 RAPPORT D'AUDIT D'INTÉGRATION : CacaoLogistiqueDB (PostgreSQL) ↔ CacaoProductionDB (SQL Server)

**Date de l'audit** : 11 Février 2026  
**Auditeur** : Cabinet Conseil en Systèmes d'Information  
**Objet** : Analyse des ruptures d'intégration entre les bases logistique et production

---

## 🔴 A. INTÉGRRATION MANQUANTE ENTRE LES BASES

### 📌 Problème n°1 : Absence totale de clé de jointure métier

| Constat | Impact | Niveau de criticité |
|---------|--------|---------------------|
| Aucun champ commun ne permet de relier un enregistrement de inventaire_logistique (PG) à une Plantation ou une Recolte (SS) | Impossibilité de tracer l'utilisation réelle des intrants et équipements sur le terrain | 🔴 CRITIQUE |

**Manifestation concrète :**

- On sait que des sacs jute ont été achetés (PG : CMD-2025-003, 5000 unités)
- On sait que des récoltes ont eu lieu (SS : 25 récoltes en 2024)
- On ne peut PAS prouver que ces sacs ont servi à conditionner ces récoltes

**Recommandation :**

```sql
-- Créer une table de liaison `EmballageUtilise` dans SQL Server avec :
-- - RecolteID (FK)
-- - ProduitCode_PG (VARCHAR(50)) → référence vers inventaire_logistique
-- - QuantiteUtilisee
-- - DateUtilisation

📌 Problème n°2 : Rupture de traçabilité Fournisseurs ↔ Clients
Constat	Impact	Niveau de criticité
Les fournisseurs (PG) et les Clients (SS) sont deux entités totalement déconnectées	Impossible d'analyser le cycle complet : achat d'intrants → production → vente à l'export	🔴 CRITIQUE

Scénario métier non traçable :

    J'achète de l'engrais NPK au fournisseur 'Agri-Intrants Cameroun' (PG)

    Je distribue cet engrais aux agriculteurs de la coopérative de Muyuka (SS)

    Je récolte du cacao (SS)

    Je vends ce cacao à 'Chocolate World Ltd' (SS)
    ❌ Aucun lien entre l'étape 1 et les étapes 2-3-4

Recommandation :
sql

-- Ajouter dans la table `UtilisationIntrants` (à créer) :
-- - FournisseurID_PG (INT) → référence vers fournisseurs PostgreSQL
-- Permettant de répondre : "Quel fournisseur a vendu les intrants utilisés sur cette plantation ?"

📌 Problème n°3 : Stock de fèves inexistant côté SQL Server
Constat	Impact	Niveau de criticité
Les fèves récoltées (Recoltes.PoidsFevesFraiches) disparaissent après l'enregistrement	Aucune gestion de stock, aucune liaison avec les Exportations	🔴 CRITIQUE

Incohérence manifeste :
sql

-- SQL Server : on enregistre des tonnes de fèves
SELECT SUM(PoidsFevesFraiches) FROM Recoltes WHERE Saison = 'Grande' 
-- Résultat : ~8,5 tonnes

-- SQL Server : on exporte des tonnes de cacao
SELECT SUM(QuantiteTonnes) * 1000 FROM Exportations 
-- Résultat : ~250 tonnes (incohérence totale avec la production)

Recommandation :
sql

-- Créer la table `StockFeves` avec :
-- - StockFevesID (PK)
-- - RecolteID (FK)
-- - DateEntree
-- - QuantiteKG
-- - LotID (nouveau champ à créer dans Exportations)
-- - DateSortie
-- Permettant de tracer : Récolte → Stockage → Affectation à un contrat d'export

📌 Problème n°4 : Maintenance des équipements sans lien avec les plantations
Constat	Impact	Niveau de criticité
maintenance_equipements (PG) enregistre des interventions sur du matériel	On ne sait pas où se trouve ce matériel ni qui l'utilise	🟠 ÉLEVÉ

Exemple concret :

    PG : Égreneuse manuelle EGM-200 (inventaire_id = 1)

    PG : Maintenance corrective le 2025-06-15 (courroie remplacée)

    SS : 25 plantations utilisent des égreneuses
    ❌ On ne peut pas lier la maintenance à une plantation spécifique

Recommandation :
sql

-- Créer la table `AffectationEquipement` dans SQL Server :
-- - AffectationID (PK)
-- - EquipementID_PG (INT)
-- - PlantationID (FK)
-- - DateDebut
-- - DateFin (NULL si toujours affecté)
-- - Responsable

📌 Problème n°5 : Certifications et traçabilité qualité
Constat	Impact	Niveau de criticité
Exportations.Certificats (SS) mentionne 'BIO, Fairtrade, UTZ'	Aucune preuve traçable que ces certifications sont respectées	🟠 ÉLEVÉ

Besoins métier non couverts :

    Quels intrants bio ont été utilisés pour produire ce lot certifié ?

    Quel équipement certifié a transformé ces fèves ?

    Quels contrôles labo ont validé la conformité ?

Recommandation :
sql

-- Ajouter dans SQL Server :
-- - Table `ControleQualite` liée à `Recoltes` et à `maintenance_equipements` (PG)
-- - Champ `LotCertification` dans `Exportations` pour traçabilité descendante
-- - Vue matérialisée côté PostgreSQL des lots certifiés

📌 Problème n°6 : Vue PostgreSQL sous-optimisée
Constat	Impact	Niveau de criticité
La vue vw_CommandesFournisseurs_PostgreSQL n'inclut pas les données stratégiques	Les utilisateurs SQL Server n'ont pas accès à l'état réel des stocks et équipements	🟡 MOYEN

Éléments manquants dans la vue actuelle :
sql

-- Manque :
-- - i.quantite_stock (état du stock)
-- - i.seuil_min / seuil_max (alertes réapprovisionnement)
-- - m.type_mouvement (traçabilité des entrées/sorties)
-- - me.date_maintenance, me.prochaine_maintenance (planification maintenance)

Recommandation :
sql

CREATE OR ALTER VIEW vw_SuiviLogistiqueComplet_PostgreSQL AS
SELECT 
    cf.commande_id,
    cf.date_commande,
    f.nom_fournisseur,
    i.produit_nom,
    i.quantite_stock,
    i.seuil_min,
    i.valeur_stock,
    cf.statut as statut_commande,
    i.statut as statut_stock,
    me.date_maintenance,
    me.prochaine_maintenance,
    me.type_maintenance
FROM OPENQUERY(POSTGRES_LINKED_SERVER, '
    SELECT cf.*, f.nom_fournisseur, i.produit_nom, i.quantite_stock, 
           i.seuil_min, i.valeur_stock, i.statut,
           me.date_maintenance, me.prochaine_maintenance, me.type_maintenance
    FROM commandes_fournisseurs cf
    JOIN fournisseurs f ON cf.fournisseur_id = f.fournisseur_id
    JOIN inventaire_logistique i ON cf.commande_id = i.commande_id
    LEFT JOIN maintenance_equipements me ON i.inventaire_id = me.equipement_id
');

📊 SYNTHÈSE DES RUPTURES D'INTÉGRATION
ID	Problème	Tables concernées	Solution	Criticité
P1	Absence de clé commune	inventaire_logistique ↔ Plantations	Table EmballageUtilise + UtilisationIntrants	🔴
P2	Fournisseurs ↔ Clients déconnectés	fournisseurs ↔ Clients	Ajout FournisseurID_PG dans Integration.LogistiqueMapping	🔴
P3	Stock fèves inexistant	Recoltes ↔ Exportations	Création StockFeves avec LotID	🔴
P4	Maintenance sans localisation	maintenance_equipements ↔ Plantations	Création AffectationEquipement	🟠
P5	Certifications non traçables	Exportations.Certificats ↔ inventaire_logistique	Table ControleQualite + traçabilité lot	🟠
P6	Vue PostgreSQL incomplète	vw_CommandesFournisseurs_PostgreSQL	Refonte de la vue avec stocks et maintenance	🟡
🎯 PLAN D'ACTION PRIORISÉ
🔴 Priorité 1 - URGENT (Sprint 1)

    Créer la table StockFeves dans SQL Server

    Alimenter rétroactivement avec les données des récoltes 2024

    Lier les exportations existantes à des lots de stock

🔴 Priorité 2 - URGENT (Sprint 1)

    Créer les tables UtilisationIntrants et EmballageUtilise

    Établir le mapping avec inventaire_logistique via Integration.LogistiqueMapping

🟠 Priorité 3 - IMPORTANT (Sprint 2)

    Créer la table AffectationEquipement

    Refondre la vue PostgreSQL pour inclure les données de maintenance et stocks

🟡 Priorité 4 - SOUHAITABLE (Sprint 3)

    Mettre en place la traçabilité des certifications

    Créer des rapports croisés (Power BI / SSRS) exploitant les deux bases

✅ LIVRABLES IMMÉDIATS PROPOSÉS

    Scripts de création des tables manquantes (SQL Server)

    Script ETL pour synchroniser les données PostgreSQL → SQL Server

    Vue unifiée côté SQL Server combinant production et logistique

    Jeu de tests avec 10 scénarios métier traçables de bout en bout
