# SMS_RDV_PMI

## Contexte

L'application HORUS utilisée par la PMI gère des rendez vous avec les personnes en charge de l'enfant (parents, assfam...).

Le programme Talend _SMS_RDV_PMI_ envoie des rappels de ces rendez vous aux familles afin d'augmenter la proportion de rdv honorés.

## Lecture des données depuis la base HORUS

### Vue _V_LST_RDV_ dans le le schéma _ASTAS_

Cette vue, rafraichie à chaque exécution du traitement Talend, nous donne accès aux informations de rendez vous de l'application.
C'est l'éditeur de HORUS qui la fournit et la maintient.

```sql
-- Rafraichir la vue en base de données
CALL DBMS_MVIEW.REFRESH('ASTAS.V_LST_RDV','')
-- retrouver les rdv pour une date précise et une tranche de prénoms précises (pas de notion de lieu de rdv comme dans le traitement)
SELECT *
FROM ASTAS.V_LST_RDV
WHERE SUBSTR(TRIM(PRENOM_BENEFICIAIRE), 1,1) BETWEEN 'A' AND 'G'
AND  DATE_RDV BETWEEN TO_DATE('2022-05-27 00:00:00','YYYY-MM-DD HH24:MI:SS') AND TO_DATE('2022-07-27 00:00:00','YYYY-MM-DD HH24:MI:SS')
AND ENVOI_SMS_CONTACT ='1'
```

## Données et services externes utilisés

### Services externes

- Service d'envoi de SMS contact everyone d'orange : <https://contact-everyone.orange-business.com/>

### Base de donnée

- Base de donnée du logiciel Horus

Informations dans la BDD Horus qui alimentent la vue V_LST_RDV

| Nom de table + Champ utile         | Contenu                                                                               |
| --------------------------------- | ------------------------------------------------------------------------------------- |
| ASTAS.IDENTSITE LIBELLE           | contient les libellés des lieux                                                       |
| ASTAS.PINSTACTION                 | liste les rendez vous si libellé est celui d'un rdv                                   |
| ASTAS.PINSTACTION CONTACT         | contient les contacts du rdv qui ne sont pas des tiers en lien avec l'enfant          |
| ASTAS.LIENSENTITES                | liste les tiers en lien avec l'enfant                                                 |
| ASTAS.H_IDENTMERE TELPORTABLE     | information des tiers de type mère, numero de mobile                                  |
| ASTAS.H_IDENTMERE INDENVOISMS     | information des tiers de type mère, autorisation envoi SMS                            |
| ASTAS.H_IDENTTIERS MOBILE         | information des tiers autres que mère, numero de mobile                               |
| ASTAS.H_IDENTTIERS INDENVOISMS    | information des tiers autres que mère, autorisation envoi SMS                         |
| ASTAS.V_LST_RDV                   | Materialized view agrégeant les données ci dessus, rafraichissement manuel            |

Ces tables pour information/analyse/MCO uniquement.

C'est la table _ASTAS.V_LST_RDV_ uniquement qui est utilisée pour alimenter les données de RDV issues de Horus dans le traitement talend.

Dans _ASTAS.V_LST_RDV_

| Nom du champ                      | Contenu                                                                               |
| --------------------------------- | ------------------------------------------------------------------------------------- |
| REFERENCE_RDV                     | Identifiant du rendez vous, agrège plusieurs infos dont la référence à l'enfant       |
| REFENTITE_CONTACT                 | Identifiant du tiers convoqué au rdv, plusieurs types possibles, une ligne par tiers  |
| DATE_RDV                          | Date JJ/MM/AAAA du rdv, l'horodatage n'est pas utilisé                                |
| DELAIS_RDV                        | Non utilisé                                                                           |
| HEURE_RDV                         | Horaire rdv au format 'HH'h'MM'                                                       |
| DUREE_RDV                         | Non utilisé                                                                           |
| PRECISION_LIEU_RDV                | Non utilisé                                                                           |
| LIEU_RDV                          | Indique le type de rdv                                                                |
| LIBELLE_LIEU_RDV                  | Non utilisé                                                                           |
| LIBELLE_AUTRE                     | Non utilisé                                                                           |
| ADRESSE_LIEU_RDV                  | Non utilisé                                                                           |
| ADR_LIEU_RDV_L1                   | Non utilisé                                                                           |
| ADR_LIEU_RDV_L2                   | Non utilisé                                                                           |
| ADR_LIEU_RDV_L3                   | Non utilisé                                                                           |
| ADR_LIEU_RDV_L4                   | Non utilisé                                                                           |
| ADR_LIEU_RDV_L5                   | Non utilisé                                                                           |
| ADR_LIEU_RDV_L6                   | Non utilisé                                                                           |
| TELEPHONE_RDV                     | Téléphone du secrétariat pour modification RDV                                        |
| NOM_BENEFICIAIRE                  | Nom de l'enfant                                                                       |
| PRENOM_BENEFICIAIRE               | Prénom de l'enfant                                                                    |
| AGE_BENEFICIAIRE_RDV              | Non utilisé                                                                           |
| TYPE_CONSULTATION                 | Non utilisé                                                                           |
| LIBELLE_CONSULTATION              | Type de consultation                                                                  |
| TITRE_CONTACT                     | Non utilisé                                                                           |
| NOM_CONTACT                       | Nom de l'adulte référent du RDV                                                       |
| PRENOM_CONTACT                    | Prénom de l'adulte rédérent du RDV                                                    |
| TELEPHONE_FIXE_CONTACT            | Non utilisé                                                                           |
| TELEPHONE_PORTABLE_CONTACT        | Téléphone de l'adulte référent du RDV                                                 |
| ENVOI_SMS_CONTACT                 | Indicateur de l'autorisation d'envoi de SMS pour le RDV                               |
| EMAIL_CONTACT                     | Non utilisé                                                                           |
| ADRESSE_CONTACT                   | Non utilisé                                                                           |
| ADR_CONTACT_L1                    | Non utilisé                                                                           |
| ADR_CONTACT_L2                    | Non utilisé                                                                           |
| ADR_CONTACT_L3                    | Non utilisé                                                                           |
| ADR_CONTACT_L4                    | Non utilisé                                                                           |
| ADR_CONTACT_L5                    | Non utilisé                                                                           |
| ADR_CONTACT_L6                    | Non utilisé                                                                           |

## Détails du traitement

### Règles de gestion

principalement gérées dans l'appel SQL sur la base

```sql
--SELECTION TRANCHE PRENOMS
WHERE SUBSTR(TRIM(PRENOM_BENEFICIAIRE),1,1) BETWEEN '"+context.Lettre_Debut+"' AND '"+context.Lettre_Fin+"'
--SELECTION DATE RDV
AND DATE_RDV = TO_DATE('" + globalMap.get("Str_date_rappel") + "','YYYY-MM-DD HH24:MI:SS')
--SELECTION TYPES RDV POUR RAPPELS
AND LIBELLE_CONSULTATION IN ("+context.SMS_Types_rdv+")
--SELECTION TERRITOIRES RDV POUR RAPPELS
AND LIBELLE_LIEU_RDV IN ("+context.SMS_Lieux+")
--EXCLUSION DES RDV A DOMICILE ET AUTRE, NECESSITERA DEV SPECIFIQUE SI BESOIN
AND  LIEU_RDV NOT IN ('A domicile','Autre')
--EXCLUSION DES CONTACTS QUI N ONT PAS DONNE D ACCORD POUR RECEVOIR DES SMS
AND ENVOI_SMS_CONTACT = '1'
```

On signale via un mailing si un centre de soins n'a pas de téléphone correct paramétré :

- longueur différente de 10
- valeur Null

On signale via un mailing si un contact de rdv n'a pas de téléphone correct paramétré :

- longueur différente de 10
- valeur Null

### Étapes du traitement

1. Initialisation des variables du traitement, gestion des dates
2. Connexion à la base HORUS
3. Connexion à l'API Orange
4. Récupération d'un token oauth et du groupe de distribution via l'API
5. Rafraichissement en base de la vue V_LST_RDV
6. Sélection des RDV selon critères
7. Formatage des données du message
8. Formatage JSON des informations de SMS pour chaque appel
9. Itération des appels à l'API
10. Envoi d'un mail pour les dossiers avec téléphone EDS invalide ou téléphone contact vide
11. Déconnexion de la base HORUS

### Elements spécifiques du context.properties

| Var env           | Description                                               |
|-------------------|-----------------------------------------------------------|
|# Connexion HORUS    ||
|HORUS_CNX_Server=  | serveur où est localisé la base                           |
|HORUS_CNX_Port=    | port de connexion à la base                               |
|HORUS_CNX_Sid=     | nom de la base de données                                 |
|HORUS_CNX_Login=   | login de connexion à la base                              |
|HORUS_CNX_Password=| mot de passe de connexion à la base  
|HORUS_CNX_Schema=  | schéma de la base de données                              |
|||
|# Variables Metier   ||
|Lettre_Debut=      | Lettre du prénom de l'enfant du début du batch            |
|Lettre_Fin=        | Lettre du prénom de l'enfant de fin du batch              |
|SMS_Date_Rappel=   | Date des RDVs pour lesquels on envoie des rappels         |
|SMS_debug=         | Active un log détaillé si mis à OUI                       |
|||
|# Parametrage SMS    ||
|SMS_server_url=    | URL API envoi SMS                                         |
|SMS_login=         | Login sur API                                             |
|SMS_password=      | Mot de passe sur API                                      |
|SMS_custId=        | Code client                                               |
|SMS_authent=       | Version/arborescence API de l'envoi de SMS                |
|SMS_Message=       | Contenu du message SMS (canvas fusionné ensuite avec les infos de RDV)|
|SMS_Types_rdv=     | Liste les types de rdv qui génèrent des rappels SMS       |
|SMS_Lieux=         | Liste les territoires sur lesquels on envoie des rappels SMS|

On a exclu les rdv ('A domicile','Autre') en dur dans le code car leur gestion nécessitera une nouvelle analyse et de nouveaux devs
(table.champ _ASTAS.V_LST_RDV.LIEU_RDV_)

Le métier reviendra avec une expression de besoin sur ce sujet spécifique.

## Mise en exploitation

### Plannification via VTOM

Le script VTOM HORUS_Rappels_SMS_PMI exécute le traitement Talend en plusieurs batchs quotidiens.

Dans VTOM création d'un objet "date jour+2" basé sur le calendrier des jours ouvrés, c'est donc VTOM qui gère la notion de jour ouvré.

(Dates VTOM format AAAA-MM-JJ exécution pour les rdv à date du jour ouvré courant + 2 jours ouvrés)

On passe en paramètre la date de rdv ainsi calculée ainsi que deux lettres délimitant une tranche de premières lettres des prénoms des enfants.

Exemple de commande :

```bash
home/users/phorus/talend/jobs/Rappels_RDV_PMI-1.3/Rappels_RDV_PMI/Rappels_RDV_PMI_run.sh -Xms256M -Xmx1024M -Duser.country=us -Duser.language=en --context_param Fichier_Contexte="/home/users/phorus/talend/jobs/../config/context.properties" --context_param SMS_Date_Rappel=2022-09-19 --context_param Lettre_Debut=A --context_param Lettre_Fin=H
```

### Encodage des dates

Pour une bonne exécution du rafraichissement de la vue _V_LST_RDV_ il faut que la locale du compte qui exécute le talend soit en dates US
(sinon erreur ORA-12008 et ORA-01843: not a valid month)

### Rapports d'exécution

Le service Orange contact everyone est paramétré pour envoyer par mail des rapports d'exécution à chaque envoi de sms.

Les logs VTOM listent également l'ensemble des appels à l'API effectués

### Périodicité

Afin que les secrétariats n'aient pas tous les appels au même moment, les rappels sont répartis dans la journée par tranche de prénoms des enfants.

Quatre éxécutions par jour :

- A 9h, Lettres A à H
- A 11h, Lettres I à M
- A 13h, Lettres N à T
- A 15h, Lettres U à Z
