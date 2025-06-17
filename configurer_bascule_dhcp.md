# **Prérequis : S'assurer que la plage DHCP n'est pas la même entre la configuration Windows Server et Linux Server**
Si la configuration est identique, modifier la plage (pool) DHCP sur Linux dans le fichier :

```bash
sudo nano /etc/kea/kea-dhcp4.conf
```
En effet, si les deux serveurs sont accidentellement actifs en même temps (même quelques secondes), ils peuvent attribuer la même IP à deux clients différents.

# **Étape 1 : Désactiver le démarrage automatique du serveur DHCP Linux**
```bash
sudo systemctl disable kea-dhcp4-server
```
Pour contrôler que le démarrage automatique est bien désactivé :
```bash
sudo systemctl status kea-dhcp4-server
```
# **Étape 2 : Permettre au DHCP Linux de prendre le relais lorsque le DHCP Windows Server ne répond pas**

1. **Installer l'outil dhcping :**
```bash
sudo apt update && sudo apt install dhcping -y
```

2. **Créer un script qui permet la bascule :**
```bash
cd /usr/local/bin
```

```bash
sudo nano dhcp-bascule.sh
```
"IP_WS" est à adapter :
```bash
   #!/bin/bash

   # Définit le chemin pour trouver les commandes
   PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

   # IP du serveur DHCP principal (Windows)
   IP_WS="10.0.1.120"
   # Création d'un fichier log pour surveiller le bon fonctionnement de la bascule
   LOG="/var/log/dhcp-bascule.log"
   # Date et heure actuelles pour plus d'information dans les logs
   DATE=$(date)
   # Ping du DHCP Windows Server pour savoir s'il est actif ou non
   if /usr/sbin/dhcping -s $IP_WS -c 1 -t 2 >> $LOG 2>&1; then
    echo "$DATE: Windows Server DHCP UP - arrêt du DHCP Linux" >> $LOG
    systemctl stop kea-dhcp4-server
   else
    echo "$DATE: Windows Server DHCP DOWN - démarrage du DHCP Linux et attribution de nouvelles adresses IP" >> $LOG
    systemctl start kea-dhcp4-server
   fi
```
```bash
sudo chmod +x dhcp-bascule.sh
```
Un dhcping est réalisé à destination de Windows Server. Si le ping répond, cela signifie que le DHCP WS est actif, donc le DHCP Linux reste arrêté.  
Dans le cas contraire, si le ping n'aboutit pas, le DHCP Linux prend le relais et attribue une adresse IP au client conformement à la plage IP définie dans /etc/kea/kea-dhcp4.conf

3. **Mettre en place un cron pour exécuter le script à des minutes données**
  
Ouvrir la configuration du crontab :
```bash
sudo crontab -e
```
  
Ajouter la ligne. Le cron va éxecuter le script toutes les 2 minutes pour vérifier l'état du DHCP WS :
```bash
*/2 * * * * /usr/local/bin/dhcp-bascule.sh
```
  
S'assurer que le service cron est bien actif, l'activer si non actif :
```bash
sudo systemctl status cron
```
  
4. **Suivre les logs en direct :**
```bash
tail -f /var/log/dhcp-bascule.log
```
Par exemple, ce que nous pouvons y trouver :
```bash
mar. 17 juin 2025 20:36:01 CEST: Windows Server DHCP UP - arrêt du DHCP Linux
mar. 17 juin 2025 20:38:01 CEST: Windows Server DHCP UP - arrêt du DHCP Linux
mar. 17 juin 2025 20:40:01 CEST: Windows Server DHCP DOWN - démarrage du DHCP Linux et attribution de nouvelles adresses IP
```

5. **(optionnel) Tester manuellement le ping vers le DHCP Windows Server (adapter l'adresse IP Windows Server)** :
```bash
sudo dhcping -s 10.0.1.120 -c 1 -t 3 -v
```
Réponses possibles :
Got answer from: 10.0.1.120  
ou
no answer
