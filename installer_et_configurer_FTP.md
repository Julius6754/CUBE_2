# **Étape 1 : Installation du serveur FTP**

1. **Mettre à jour la liste des paquets** :

Mettre à jour la liste des paquets et mettre à jour les paquets déjà installés.  
L'argument -y = "yes"
```bash
   sudo apt update && sudo apt upgrade -y
```

2. **Installer vsftpd** :

Installer le serveur FTP en ligne de commande :
   ```bash
     sudo apt install vsftpd -y
   ```

Voir la version de vsftpd installée :
   ```bash
     sudo vsftpd -version
   ```
Si la commande renvoie une version, l'installation s'est bien déroulée. 

Lancer le service vsftpd :
   ```bash
     sudo systemctl start vsftpd
   ```

Activer et lancer le service vsftpd à chaque démarrage du serveur :
   ```bash
     sudo systemctl enable vsftpd
   ```

Contrôler le bon fonctionnement du service vsftpd :
   ```bash
     sudo systemctl status vsftpd
   ```
Nous devrions retrouver "enabled" et "active (running)" dans le retour de la commande.

Le serveur FTP est installé. Il est prêt à être configuré !
     

# **Étape 2 : Configuration du serveur FTP**

1. **Installer l'utilisateur de gestion de pare-feu ufw**  :

   ```bash
     sudo apt install ufw -y
   ```

2. **Créer des règles dans le pare-feu** :

   ```bash
     sudo ufw allow 20 && sudo ufw allow 21
   ```
Le port 21 est utilisé pour établir la connexion entre les 2 ordinateurs et le port 20 pour transférer des données.  

3. **Editer le fichier de configuration de fsftpd** :

   ```bash
     sudo nano /etc/vsftpd.conf
   ```

4. **Modifier les lignes** :

listen=YES  
anonymous_enable=NO  
local_enable=YES  
write_enable=YES (décommenter)  
xferlog_enable=YES (pour consulter les logs)  
chroot_local_user=YES (décommenter)  
pam_service_name=vsftpd  

# **Étape 3 : Création et droits des utilisateurs**
#### **Les sous-étapes 1, 2, 3 et 4 sont automatisées dans un script. Voir Annexe 1 en bas de page**

1. **Créer les utilisateurs, à faire pour chaque utilisateur** :

   ```bash
     sudo adduser nom_enseignant
   ```
Mot de passe : azerty-123  
Passer les questions suivantes (touche entrée)

2. **Ajouter les utilisateurs dans la liste d'autorisation vsftpd, à faire pour chaque utilisateur** :
 
   ```bash
     echo nom_enseignant | sudo tee -a /etc/vsftpd.userlist
   ```
3. **Création du répertoire /home de l'utilisateur**
   
   ```bash
      mkdir /home/nom_enseignant
   ```
   
4. **Maintenant, nous voulons restreindre l'accès aux utilisateurs pour que l'enseignant ne puisse accéder qu'à son propre répertoire** :

   ```bash
     sudo chown -R nom_enseignant:nom_enseignant /home/nom_enseignant/
   ```
   ```bash
     sudo chmod 700 /home/nom_enseignant/
   ```
chmod 700 = droit d'écriture, de lecture et d'éxecution uniquement pour le propriétaire

5. **Contrôler et modifier de nouveau le fichier de configuration etc/vsftpd.conf (des lignes sont à ajouter)**
 
   ```bash
     sudo nano /etc/vsftpd.conf
   ```
local_enable=YES  
write_enable=YES  
chroot_local_user=YES  
allow_writeable_chroot=YES  
local_umask=022  
user_sub_token=$USER  
local_root=/home/$USER  

6. **Redémarrer le service vsftpd** :
   
   ```bash
     sudo systemctl restart vsftpd
   ```

# **Étape 4 : Téléchargement d'un client FTP et test !**

1. **Sur le PC client, télécharger FileZilla pour se connecter en FTP** :

   ```bash
     https://filezilla-project.org/download.php?type=client
   ```

2. **Pour se connecter en ftp, remplir les champs** :

Hôte = nom_ecole.local  
Nom utilisateur = nom_enseignant  
Mot de passe = Celui définit à la création de l'utilisateur  
Port : Laisser vide (par défaut il s'agit du port 21)  

3. **Pour tester** :

Créer un fichier .txt sur le bureau du client windows. 
Glisser le fichier dans le repertoire / sur FileZilla.  
Ensuite, double cliquer sur le fichier créé dans FileZilla.  
Par défaut, il est téléchargé puis déplacé dans C:\Users\nom_session 

# **Annexe 1 : ajout_utilisateur.sh**

   ```bash
      cd /tmp
   ```

   ```bash
      sudo nano ajout_utilisateurs.sh
   ```

   ```bash
      sudo chmod +x ajout_utilisateurs.sh
   ```

   ```bash
   #!/bin/bash

# Mot de passe par défaut pour tout le monde
mot_de_passe="azerty-123"

while true
do
  read -p "Veux tu créer un user (o/n)" reponse
  if [[ $reponse == "o" ]]
  then
    echo "On va créer un nouvel utilisateur"
    read -p "Quel est le nom de l'utilisateur à créer ?" utilisateur
    if id "$utilisateur" &>/dev/null; then
       echo "Utilisateur $utilisateur existe déjà, on ignore."
    else
       echo "Ajout de l'utilisateur $utilisateur..."
       # Création de l'utilisateur sans interaction
       sudo adduser --gecos "" --disabled-password "$utilisateur"
       # Attribution du mot de passe
       echo "$utilisateur:$mot_de_passe" | sudo chpasswd
       echo "Utilisateur $utilisateur ajouté avec mot de passe définit."
    fi
    # Ajout de l'utilisateur dans la liste d autorisation vsftpd
    echo $utilisateur | sudo tee -a /etc/vsftpd.userlist
    # Création du repertoire /home de l'utilisateur
    mkdir /home/$utilisateur
    # Restriction des accès. L'utilisateur ne peut accéder qu'à son propre répertoire
    sudo chown -R $utilisateur:$utilisateur /home/$utilisateur/
    # chmod 700 = droit d écriture, de lecture et d éxecution uniquement pour le propriétaire
    sudo chmod 700 /home/$utilisateur/
  else
    echo "Pas de création, fin d'ajout d'utilisateur"
    exit 0
  fi
done
   ```
