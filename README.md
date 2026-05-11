# Évaluation des Risques – Application Mobile "VaultLocker"

## Cartographie de l’entité analysée

| Propriété | Valeur enregistrée |
|-----------|--------------------|
| Identifiant produit | VaultLocker |
| Paquet technique | org.vaultlocker.storage |
| Millésime | 2.4.0 |
| Session d’examen | 2025-12-01 |
| Outil de diagnostic | Drozer 3.2.2 |
| Bac à sable | Android Emulator x86_64 – API 31 (Android 12) |

---

## Bref état des lieux

L’application VaultLocker contient plusieurs défauts de configuration exposant ses composants internes. Un total de huit anomalies a été relevé, dont deux de criticité maximale autorisant l’accès aux mots de passe enregistrés sans aucune forme de contrôle d’identité.

---

## Déroulement de l’investigation

- Test de la chaîne ADB et de la réactivité de l’émulateur
- Dépôt de l’agent Drozer et de l’application cible
- Découverte des composants ouverts (activités, services, récepteurs, fournisseurs)
- Contrôle des permissions et des filtres d’intent
- Recherche des URIs sans restriction
- Évaluation des conséquences et proposition de mesures correctives

---

## Phase préliminaire – Mise en place

<img width="846" height="162" alt="Screenshot 2026-05-11 204944" src="https://github.com/user-attachments/assets/e8c50dcb-6a68-41d9-b9e5-142cac4aad7f" />


    adb install drozer-agent-3.2.2.apk   → retour positif
    adb install vaultlocker.apk          → retour positif
    adb forward tcp:31415 tcp:31415      → redirection établie sur le port 31415

---

## Phase 2 – Connexion à la console d’analyse

<img width="942" height="108" alt="Screenshot 2026-05-11 200031" src="https://github.com/user-attachments/assets/f67d8978-af21-4754-89d3-f76c6d6ff4ad" />


    drozer console connect
    Périphérique retenu : emulator-5554 (Android 12 - API 31)
    Invite de commande Drozer (v3.2.2)
    dz>

---

## Phase 3 – Recensement des composants exposés

<img width="787" height="771" alt="Screenshot 2026-05-11 200145" src="https://github.com/user-attachments/assets/7c17c518-27d2-4536-b91f-38441ad53b6a" />


### Activités sans barrière

    dz> run app.activity.info -a org.vaultlocker.storage

<img width="492" height="175" alt="Screenshot 2026-05-11 200152" src="https://github.com/user-attachments/assets/f746bcb5-9dac-4321-9bc7-5cbc251df67b" />


| Nom de l’activité | Permission exigée |
|------------------|--------------------|
| LoginScreen | ⛔ aucune |
| FileBrowser | ⛔ aucune |
| CredentialsViewer | ⛔ aucune |

### Services ouverts

    dz> run app.service.info -a org.vaultlocker.storage

<img width="492" height="377" alt="Screenshot 2026-05-11 200157" src="https://github.com/user-attachments/assets/ced8d0aa-b83d-4b62-8fd3-d3363c0d92d9" />


| Service | Niveau de protection |
|---------|----------------------|
| AccessManager | ⛔ aucune |
| EncryptionHelper | ⛔ aucune |

### Récepteurs d’événements

    dz> run app.broadcast.info -a org.vaultlocker.storage

<img width="832" height="350" alt="Screenshot 2026-05-11 200206" src="https://github.com/user-attachments/assets/0b6845f2-ba93-44d0-ac13-e37a55a72e05" />


| Nom du récepteur | Droit associé |
|-----------------|---------------|
| ConfigListener | ✅ android.permission.DUMP (accordé) |

### Fournisseurs de données

    dz> run app.provider.info -a org.vaultlocker.storage

<img width="820" height="253" alt="Screenshot 2026-05-11 200215" src="https://github.com/user-attachments/assets/4594033a-2e83-48f2-8a17-d424d9c82bf8" />


| Fournisseur | Permission lecture | Permission écriture |
|-------------|--------------------|---------------------|
| DataCoreProvider | ⛔ rien (sauf /secure) | ⛔ rien (sauf /secure) |
| BackupProvider | ⛔ rien | ⛔ rien |

---

## Phase 4 – Inspection des sécurités actives

### Lecture du fichier manifeste

    dz> run app.package.manifest org.vaultlocker.storage

<img width="827" height="248" alt="Screenshot 2026-05-11 200223" src="https://github.com/user-attachments/assets/27c92ae0-328d-4f12-9013-30f94aa19433" />


**Faiblesses constatées :**
- `debuggable="true"` → autorise l’inspection dynamique en environnement final
- `allowBackup="true"` → possible extraction des préférences et bases de données
- `protectionLevel="dangerous"` sur les constantes READ_CREDENTIALS / WRITE_CREDENTIALS → niveau trop faible

### Chemins de données accessibles

    dz> run scanner.provider.finduris -a org.vaultlocker.storage



**URIs sans contrôle d’accès :**

    content://org.vaultlocker.storage.provider.DataCoreProvider/Passwords
    content://org.vaultlocker.storage.provider.DataCoreProvider/Passwords/
    content://org.vaultlocker.storage.provider.DataCoreProvider/secure/

---

## Phase 5 – Pondération des vulnérabilités

Un score CVSS v3.1 a été attribué à chaque faille pour objectiver la criticité.

| ID | Composant | Description | Sévérité | Score CVSS | Impact concret |
|----|-----------|-------------|----------|------------|----------------|
| A1 | CredentialsViewer | Activité exportée sans restriction | 🔴 Critique | 9.1 | Consultation de tous les identifiants stockés |
| A2 | DataCoreProvider/Passwords | URI non protégée en lecture | 🔴 Critique | 8.6 | Exfiltration complète des mots de passe |
| A3 | FileBrowser | Activité accessible sans raison | 🔴 Élevé | 7.5 | Parcours du système de fichiers |
| A4 | AccessManager | Service exposé sans permission | 🔴 Élevé | 7.8 | Neutralisation du contrôle d’accès |
| A5 | EncryptionHelper | Service exposé sans permission | ⚠️ Moyen | 5.3 | Opérations de chiffrement non autorisées |
| A6 | Application | debuggable true en production | ⚠️ Moyen | 4.9 | Attaches de débogueur possibles |
| A7 | Application | allowBackup true | ⚠️ Moyen | 4.5 | Récupération des données via sauvegarde ADB |
| A8 | DataCoreProvider/secure | URI exposant des clés | ⚠️ Moyen | 5.0 | Divulgation de matériel cryptographique |

---

## Correspondance avec le standard MASVS (OWASP)

| ID | Défaut détecté | Contrôle MASVS | Explication |
|----|----------------|----------------|--------------|
| A1 | CredentialsViewer exportée sans garde | MSTG-PLATFORM-1 | Ne rendre accessible que les composants indispensables |
| A2 | DataCoreProvider mal configuré | MSTG-STORAGE-2 | Les informations sensibles doivent être protégées |
| A4 | AccessManager sans validation | MSTG-PLATFORM-2 | Toute entrée externe doit être vérifiée |
| A3 | Récepteurs sans filtrage | MSTG-PLATFORM-3 | Les intents entrants doivent être contrôlés |
| A5 | Permissions trop permissives | MSTG-AUTH-1 | Renforcer les mécanismes d’authentification |

---

## Préconisations techniques

### Corriger les activités CredentialsViewer et FileBrowser

État original :

    <activity android:name=".CredentialsViewer" android:exported="true" />

État modifié :

    <activity android:name=".CredentialsViewer" android:exported="false" />

### Sécuriser le fournisseur DataCoreProvider

État original :

    <provider android:name=".DataCoreProvider" android:exported="true" />

État corrigé :

    <provider
        android:name=".DataCoreProvider"
        android:exported="true"
        android:readPermission="org.vaultlocker.storage.READ_CREDENTIALS"
        android:writePermission="org.vaultlocker.storage.WRITE_CREDENTIALS" />

### Restreindre les services AccessManager et EncryptionHelper

État original :

    <service android:name=".AccessManager" android:exported="true" />

État corrigé :

    <service
        android:name=".AccessManager"
        android:exported="false" />

### Désactiver les options de débogage et de sauvegarde

État original :

    <application android:debuggable="true" android:allowBackup="true" />

État corrigé :

    <application android:debuggable="false" android:allowBackup="false" />

### Rehausser le niveau des permissions

    <permission
        android:name="org.vaultlocker.storage.READ_CREDENTIALS"
        android:protectionLevel="signature" />

---

## Arborescence des preuves collectées

    investigation/
    ├── rapport_complet.md
    ├── score_cvss.csv
    ├── validation_finale.md
    ├── activites/
    │   └── activites_exportees.txt
    ├── services/
    │   └── services_exportes.txt
    ├── recepteurs/
    │   └── recepteurs_exportes.txt
    └── fournisseurs/
        └── fournisseurs_exportes.txt

---

## Validation des étapes d’audit

- [x] Chaque phase du test a été exécutée
- [x] L’ensemble des composants Android a été passé en revue
- [x] La grille de criticité est entièrement remplie
- [x] Les correctifs proposés sont précis et prêts à être mis en œuvre
- [x] La conformité avec OWASP MASVS est vérifiée
- [x] Aucune information personnelle réelle n’est incluse
- [x] La présentation est cohérente et fluide

