# Exercice 1 : Manipulations pratiques sur VM Windows (temps estimé : 1h30)
Pour cet exercice tu as besoin de la VM SRVWIN01.

## Partie 1 : Gestion des utilisateurs
L'utilisateur Kelly Rhameur a quitté l'entreprise.
Elle est remplacée par Lionel Lemarchand

### Q.1.1.1 Créer l'utilisateur Lionel Lemarchand avec les même attribut de société que Kelly Rhameur.

- Créez un nouvel utilisateur **Lionel Lemarchand** dans Active Directory avec les mêmes attributs que **Kelly Rhameur** (nom complet, identifiant, groupes, etc.).

- Ajouter l'utilisateur : **Kelly Rhameur**
    - Ouvrez le Gestionnaire de serveur sur un serveur Windows et allez dans Outils > Utilisateurs et ordinateurs Active Directory.
    - Utilisateurs >  Nouveau > Utilisateur.
    - Allez dans l'onglet Membre de, puis cliquez sur Ajouter pour ajouter Kelly Rhameur à des groupes comme Utilisateurs.

#### Récupérer les attributs de Kelly Rhameur en utilisant le SamAccountName exact
- code en powershel attributs.sh
```powershell
$kelly = Get-ADUser -Identity "Kelly.Rhameur" -Properties *

# Vérifiez si l'utilisateur Kelly a été récupéré correctement
if ($kelly) {
    # Créer l'utilisateur Lionel Lemarchand avec les mêmes attributs
    New-ADUser -SamAccountName "LionelLemarchand" `
        -UserPrincipalName "$($kelly.UserPrincipalName.Replace('Kelly.Rhameur', 'Lionel.Lemarchand'))" `
        -GivenName "Lionel" `
        -Surname "Lemarchand" `
        -Name "Lionel Lemarchand" `
        -DisplayName "Lionel Lemarchand" `
        -Department $kelly.Department `
        -Title $kelly.Title `
        -EmailAddress $(if ($kelly.EmailAddress) { $kelly.EmailAddress.Replace('Kelly.Rhameur', 'Lionel.Lemarchand') } else { "lionel.lemarchand@ranka.fr" }) `
        -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) `
        -Enabled $true `
        -PassThru

    # Ajouter l'utilisateur au groupe "Utilisateurs"
    Add-ADGroupMember -Identity "Utilisateurs" -Members "LionelLemarchand"
} else {
    Write-Host "L'utilisateur Kelly Rhameur n'a pas été trouvé."
}

```

![users](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/users.png)


### Q.1.1.2 Créer une OU DeactivatedUsers et déplace le compte désactivé de Kelly Rhameur dedans.

- Désactive le compte de Kelly Rhameur.
- Créez une nouvelle **OU** (Organizational Unit) nommée **DeactivatedUsers**.

1. Dans le volet gauche, **cliquez droit** sur le domaine.
2. Sélectionnez **Nouveau > Unité organisationnelle**.
3. Dans la fenêtre **Nouvel objet – Unité organisationnelle**, entrez **DeactivatedUsers** comme nom de l'OU.
4. Cliquez sur **OK** pour créer l'OU.

```powershell
New-ADOrganizationalUnit -Name "DeactivatedUsers" -Path "OU=ranka,DC=ranka,DC=fr"
```

- Déplacez le compte désactivé **Kelly Rhameur** dans cette nouvelle **OU**.
    
1. Faites un **clic droit** sur le compte **Kelly Rhameur**, puis sélectionnez **Désactiver le compte**.
2. Faites un **clic droit** sur le compte **Kelly Rhameur**, puis sélectionnez **Déplacer**.
3. Dans la fenêtre **Déplacer**, sélectionnez l'OU **DeactivatedUsers** que vous venez de créer.
4. Cliquez sur **OK** pour déplacer le compte dans cette nouvelle OU.

```powershell
# Désactiver le compte de Kelly Rhameur
Disable-ADAccount -Identity "Kelly.Rhameur"

# Déplacer le compte de Kelly Rhameur dans l'OU DeactivatedUsers
Move-ADObject -Identity "CN=Kelly Rhameur,CN=Users,DC=ranka,DC=fr" -TargetPath "OU=DeactivatedUsers,DC=ranka,DC=fr"

```

![oudesactive](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/oudesactive.png)

### Q.1.1.3 Modifier le groupe de l'OU dans laquelle était Kelly Rhameur en conséquence.
- Modifiez les groupes associés à l'ancienne **OU** de Kelly Rhameur pour mettre à jour les membres ou groupes de sécurité.
- Créer un nouveau groupe **utilisateursDésactivé*

1. **Faites un clic droit** sur le compte **Kelly Rhameur**, puis sélectionnez **Propriétés**.
2. Allez dans l'onglet **Membres de** 
3. Cliquez sur **Ajouter** pour ajouter **Kelly Rhameur** à un nouveau groupe **utilisateursDésactivé**.
4. Pour supprimer **Kelly Rhameur** d’un groupe, sélectionnez le groupe et cliquez sur **Supprimer**.
5. Cliquez sur **OK** pour enregistrer les modifications.


### Q.1.1.4 Créer le dossier Individuel du nouvel utilisateur et archive celui de Kelly Rhameur en le suffixant par -ARCHIVE.

- Archivez le dossier personnel de **Kelly Rhameur**.
    ```powershell
    Compress-Archive -Path "C:\Users\Kelly.Rhameur" -DestinationPath "C:\Users\Kelly.Rhameur-ARCHIVE.zip"
    ```

## Partie 2 : Restriction utilisateurs
### Q.1.2.1 Faire en sorte que l'utilisateur Gabriel Ghul ne puisse se connecter que du lundi au vendredi, de 7h à 17h.

- Étapes :

1. **Ouvrir** Active Directory Users and Computers.
2. **Rechercher** l'utilisateur **Gabriel Ghul** dans l'OU où il se trouve.
3. **Faire un clic droit** sur le compte de **Gabriel Ghul**, puis sélectionnez **Propriétés**.
4. Dans l'onglet **Compte**, cliquez sur **Heures de connexion**.
5. Dans la fenêtre **Heures de connexion**, vous pouvez définir les jours et les heures auxquels **Gabriel Ghul** peut se connecter.
6. Sélectionner les jours **lundi à vendredi**.
7. Sélectionner l'intervalle horaire de **7h00 à 17h00**.
8. **Appliquer** et **OK** pour enregistrer les modifications.

![Restriction](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/Restriction.png)

### Q.1.2.2 De même, bloquer sa connexion au seul ordinateur CLIENT01.

- Configurez une restriction pour bloquer la connexion de **Gabriel Ghul** uniquement à l'ordinateur **CLIENT01**.
    ```powershell
    Set-ADUser -Identity "GabrielGhul" -LogonWorkstations "CLIENT01"
    ```

- Étapes 

1. **Ouvrir Active Directory Users and Computers**.  
2. **Rechercher** l'utilisateur **Gabriel Ghul**.  
3. **Faire un clic droit** sur son compte et sélectionner **Propriétés**.  
4. **Aller dans l'onglet** **Compte**.  
5. **Cliquer sur** **Postes de travail** (ou **Workstations** selon la langue de votre serveur).  
6. **Dans cette section**, entrer le nom de l'ordinateur : `CLIENT01` (le nom du poste auquel Gabriel Ghul est autorisé à se connecter).  
7. **Cliquer sur OK** pour appliquer les paramètres.  

![client01](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/client01.png)

### Q.1.2.3 Mettre en place une stratégie de mot de passe pour durcir les comptes des utilisateurs de l'OU LabUsers.

#### Ouvrir la console "Gestion des stratégies de groupe" (Group Policy Management - GPMC) :

- Sur le contrôleur de domaine, ouvrez **Exécuter** (Windows + R), tapez `gpmc.msc` et appuyez sur **Entrée**.

#### Créer une nouvelle GPO :

1. Dans la console **GPMC**, accédez à :
   - **Forêts** > **Domaine** > `[ranka.fr]` > **Group Policy Objects**.
2. Faites un **clic droit** sur **Group Policy Objects** et sélectionnez **Nouvelle**.
3. Donnez un nom explicite à la GPO, par exemple :  
   **"Durcissement mot de passe - LabUsers"**.

#### Modifier la GPO :

- Faites un **clic droit** sur la GPO créée et cliquez sur **Modifier**.


#### Configuration des règles de mot de passe Dans l'éditeur de stratégie de groupe :

- Accédez à :  
   **Configuration de l'ordinateur** > **Stratégies** > **Paramètres Windows** > **Paramètres de sécurité** > **Stratégies de compte** > **Stratégie de mot de passe**.

#### Configurer les paramètres suivants :

| Paramètre | Valeur recommandée |
|-----------|--------------------|
| Exiger la complexité du mot de passe | **Activé** |
| Longueur minimale du mot de passe | **12 caractères** |
| Durée de vie maximale du mot de passe | **30 jours** |
| Durée de vie minimale du mot de passe | **1 jour** |
| Historique des mots de passe | **10 anciens mots de passe** |
| Stockage des mots de passe réversibles | **Désactivé** |
| Seuil de verrouillage du compte | **5 tentatives échouées** |
| Durée du verrouillage du compte | **15 minutes** |
| Réinitialisation du compteur de verrouillage | **15 minutes** |

![password](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/password.png)

## Partie 3 : Lecteurs réseaux
### Q.1.3.1 Créer une GPO Drive-Mount qui monte les lecteurs E: et F: sur les clients.
- Créez une **GPO (Group Policy Object)** nommée **Drive-Mount** pour monter les lecteurs **E:** et **F:** sur les postes clients via la stratégie de groupe.

- Créer une Nouvelle GPO  et Nommez-la "Drive-Mount" et cliquez sur OK.

- Configurer le Mappage des Lecteurs Réseaux
    - Faites un clic droit sur "Drive-Mount" et sélectionnez Modifier.
    - Accédez à : Configuration utilisateur > Préférences > Paramètres Windows > Mappage de lecteurs.
    - Faites un clic droit dans la fenêtre de droite et sélectionnez Nouveau > Lecteur mappé.

- Ajouter le Lecteur E:
    - Action : Créer
    - Emplacement : \\WINSERV1\PartageE (remplacez avec le chemin du partage réseau)
    - Attribuer la lettre : E:
    - Reconnecter : Cochez cette option
    - Exécuter dans le contexte de sécurité de l’utilisateur : Cochez cette option
    - Cliquez sur OK.

- Ajouter le Lecteur F:
    - épétez la même procédure en choisissant la lettre F: et en mettant le bon chemin de partage.

- Appliquer la GPO aux Utilisateurs :
    - ermez l'éditeur de gestion des stratégies de groupe.
    - Assurez-vous que la GPO est bien liée à l’OU contenant les utilisateurs ou les groupes qui doivent recevoir ces lecteurs.

- Forcer la Mise à Jour et Vérifier
    ```bash
    gpupdate /force
    ```
