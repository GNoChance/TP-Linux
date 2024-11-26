# TP Avancé : "Mission Ultime : Sauvegarde et Sécurisation"

## Étape 1 : Analyse et nettoyage du serveur
1. **Lister les tâches cron pour détecter des backdoors** :

- Analysez les tâches cron de tous les utilisateurs pour identifier celles qui semblent malveillantes. 

```
[root@localhost ~]# for user in $(cut -f1 -d: /etc/passwd); do echo $user; sudo crontab -u $user -l; done
[...]
attacker
*/10 * * * * /tmp/.hidden_script
```
2. **Identifier et supprimer les fichiers cachés** :
   - Recherchez les fichiers cachés dans les répertoires `/tmp`, `/var/tmp` et `/home`.
   - Supprimez tout fichier suspect ou inconnu.
```
[root@localhost ~]# ls -alh /tmp/
total 56K
[...]
-rwxrwxrwx.  1 attacker attacker   18 Nov 24 18:24 .hidden_file
-rwxrwxrwx.  1 attacker attacker   17 Nov 24 18:11 .hidden_script
-rwxr-xr-x.  1 attacker attacker   23 Nov 24 18:11 malicious.sh
[...]
```
```
[root@localhost ~]# rm /tmp/.hidden_file
[root@localhost ~]# rm /tmp/.hidden_script
[root@localhost ~]# rm /tmp/malicious.sh
```
```
[root@localhost ~]# ls -alh /var/tmp/
total 56K
[...]
-rwxrwxrwx.  1 attacker attacker    7 Nov 24 20:10 .nop
[...]
```
```
[root@localhost ~]# rm /var/tmp/.nop
```
```
[root@localhost ~]# ls -alh /home/
total 16K
[...]
drwx------.  2 attacker attacker 4.0K Nov 24 20:09 attacker
[...]
```
```
[root@localhost /]# ls -alh /home/attacker/
total 28K
[...]
-rw-r--r--. 1 attacker attacker   18 Nov 24 20:09 .hidden_file
```
```
[root@localhost ~]# rm /home/attacker/.hidden_file
```

3. **Analyser les connexions réseau actives** :
   - Listez les connexions actives pour repérer d'éventuelles communications malveillantes.
```
ss
```
Il n'y en a pas.

## Étape 2 : Configuration avancée de LVM

1. **Créer un snapshot de sécurité pour `/mnt/secure_data`** :
   - Prenez un snapshot du volume logique `secure_data`.
```
[root@localhost]# lvcreate -L 500M -s -n rootsnap /dev/vg_secure/secure_data
  Logical volume "rootsnap" created.
```
2. **Tester la restauration du snapshot** :
   - Supprimez un fichier dans `/mnt/secure_data`.
   - Montez le snapshot et restaurez le fichier supprimé.
```
[root@localhost secure_data]# ls
lost+found  sensitive1.txt  sensitive2.txt
[root@localhost secure_data]# rm sensitive1.txt
rm: remove regular file 'sensitive1.txt'? y
[root@localhost secure_data]# mount /dev/vg_secure/rootsnap /mnt/rootsnap
[root@localhost secure_data]# ls
lost+found  sensitive2.txt
[root@localhost secure_data]# cd /mnt/rootsnap/
[root@localhost rootsnap]# ls
lost+found  sensitive1.txt  sensitive2.txt
[root@localhost rootsnap]# cp sensitive1.txt /mnt/secure_data
[root@localhost rootsnap]# cd /mnt/secure_data/
[root@localhost secure_data]# ls
lost+found  sensitive1.txt  sensitive2.txt
```
3. **Optimiser l’espace disque** :
   - Si le volume logique `secure_data` est plein, étendez-le en ajoutant de l’espace à partir du groupe de volumes existant.
   ```
   [root@localhost ~]# umount /mnt/secure_data/
   [root@localhost ~]# umount /mnt/rootsnap/
   [root@localhost ~]# lvchange -an /dev/vg_secure/secure_data
   [root@localhost ~]# sudo lvextend -L +20M /dev/vg_secure/secure_data
   Size of logical volume vg_secure/secure_data changed from 500.00 MiB (125 extents) to 520.00 MiB (130 extents).
   Logical volume vg_secure/secure_data successfully resized.
   ```


## Étape 3 : Automatisation avec un script de sauvegarde

1. **Créer un script `secure_backup.sh`** :
   - Archive le contenu de `/mnt/secure_data` dans `/backup/secure_data_YYYYMMDD.tar.gz`.
   - Exclut les fichiers temporaires (.tmp, .log) et les fichiers cachés.
   ```
   [root@localhost ~]# nano secure_backup.sh
   #!/bin/bash
   mkdir -p /backup/
   tar --exclude='*.tmp' --exclude='*.log' --exclude='.*' -czf /backup/secure_data_$
   (date +%Y-%m-%d).tar.gz /mnt/secure_data
   ```
   ```
   [root@localhost ~]# ./secure_backup.sh
   tar: Removing leading `/' from member names
   tar: Removing leading `/' from hard link targets
   ```

2. **Ajoutez une fonction de rotation des sauvegardes** :
   - Conservez uniquement les 7 dernières sauvegardes pour économiser de l’espace.
   ```
   #!/bin/bash
   mkdir -p /backup/
   tar --exclude='*.tmp' --exclude='*.log' --exclude='.*' -czf /backup/secure_data_$
   (date +%Y-%m-%d-%S).tar.gz /mnt/secure_data
   BACKUP_COUNT=$(ls -1 /backup/secure_data_*.tar.gz 2>/dev/null | wc -l)
   if (( $BACKUP_COUNT > 7 )); then
    ls -1t /backup/secure_data_*.tar.gz | tail -n 1 | xargs rm -f
    fi
   ```

3. **Testez le script** :
   - Exécutez le script manuellement et vérifiez que les archives sont créées correctement.
   ```
   [root@localhost ~]# ./secure_backup.sh
   tar: Removing leading `/' from member names
   [root@localhost ~]# ll /backup/
   total 28
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-35.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-36.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-37.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-39.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-45.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-48.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-52.tar.gz
    [root@localhost ~]# ./secure_backup.sh
    tar: Removing leading `/' from member names
    [root@localhost ~]# ll /backup/
    total 28
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-36.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-37.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-39.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-45.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-48.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-52.tar.gz
    -rw-r--r--. 1 root root 118 Nov 25 12:36 secure_data_2024-11-25-54.tar.gz
    ```

4. **Automatisez avec une tâche cron** :
   - Planifiez le script pour qu’il s’exécute tous les jours à 3h du matin.
   ```
   [root@localhost ~]# crontab -l
    0 3 * * * /root/secure_backup.sh
    ```

## Étape 4 : Surveillance avancée avec `auditd`

1. **Configurer auditd pour surveiller `/etc`** :
   - Ajoutez une règle avec `auditctl` pour surveiller toutes les modifications dans `/etc`.
    ```
    [root@localhost ~]# dnf install audit -y
    Last metadata expiration check: 2:49:22 ago on Mon Nov 25 10:18:31 2024.
    Package audit-3.1.5-1.el9.x86_64 is already installed.
    Dependencies resolved.
    Nothing to do.
    Complete!
    [root@localhost ~]# systemctl enable auditd
    [root@localhost ~]# dnf install audit audit-libs audispd-plugins -y
    [root@localhost ~]# systemctl restart auditd
    ```

2. **Tester la surveillance** :
   - Créez ou modifiez un fichier dans `/etc` et vérifiez que l’événement est enregistré dans les logs d’audit.

    ```
    [root@localhost ~]# ausearch -k etc_monitor
    ----
    time->Mon Nov 25 13:11:49 2024
    type=PROCTITLE msg=audit(1732536709.118:907): proctitle=617564697463746C002D77002F657463002D70007761002D6B006574635F6D6F6E69746F72
    type=SYSCALL msg=audit(1732536709.118:907): arch=c000003e syscall=44 success=yes exit=1072 a0=4 a1=7ffcf83f45b0 a2=430 a3=0 items=0 ppid=4108 pid=4365 auid=0 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=46 comm="auditctl" exe="/usr/sbin/auditctl" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key=(null)
    type=CONFIG_CHANGE msg=audit(1732536709.118:907): auid=0 ses=46 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 op=add_rule key="etc_monitor" list=4 res=1
    ----
    ```

3. **Analyser les événements** :
   - Recherchez les événements associés à la règle configurée et exportez les logs filtrés dans `/var/log/audit_etc.log`.
   ```
   [root@localhost ~]# ausearch -k etc_monitor > /var/log/audit_etc.log
   ```


## Étape 5 : Sécurisation avec Firewalld

1. **Configurer un pare-feu pour SSH et HTTP/HTTPS uniquement** :
   - Autorisez uniquement les ports nécessaires pour SSH et HTTP/HTTPS.
   - Bloquez toutes les autres connexions.
    ```
    [root@localhost ~]# firewall-cmd --permanent --add-service=ssh
    Warning: ALREADY_ENABLED: ssh
    success
    [root@localhost ~]# firewall-cmd --permanent --add-service=http
    firewall-cmd --permanent --add-service=https
    success
    Warning: ALREADY_ENABLED: https
    success
    [root@localhost ~]# firewall-cmd --set-default-zone=block
    success
    [root@localhost ~]# firewall-cmd --reload
    success
    ```

2. **Bloquer des IP suspectes** :
   - À l’aide des logs d’audit et des connexions réseau, bloquez les adresses IP malveillantes identifiées.
   

3. **Restreindre SSH à un sous-réseau spécifique** :
   - Limitez l’accès SSH à votre réseau local uniquement (par exemple, 192.168.x.x)
