# Faire de la place pour passer en Wheezy

* [Avez-vous de la place dans la partition `/` ?](#avez-vous-de-la-place-dans-la-partition--)
* [1ère solution : Ajouter un disque physique pour déplacer `/tftpboot` et son contenu](#ajouter-un-disque-physique-pour-déplacer-tftpboot-et-son-contenu)
    * [Repérer le nouveau disque](#repérer-le-nouveau-disque)
    * [Créer une partition sur l'ensemble du disque](#créer-une-partition-sur-lensemble-du-disque)
* [Modifier le partitionnement `LVM`](#modifier-le-partitionnement-lvm)
    * [Déplacer le contenu de `/tftpboot` dans le disque](#déplacer-le-contenu-de-tftpboot-dans-le-disque)
    * [Monter le disque sur `/tftpboot`](#monter-le-disque-sur-tftpboot)
* [Modifier le partitionnement `LVM`](#modifier-le-partitionnement-lvm)
    * [Déplacer le contenu de `/tftpboot` dans le disque](#déplacer-le-contenu-de-tftpboot-dans-le-disque)
    * [Monter le disque sur `/tftpboot`](#monter-le-disque-sur-tftpboot)
* [2ème solution : Modifier le partitionnement `LVM`](#modifier-le-partitionnement-lvm)
    * [Vue d'ensemble](#vue-densemble)
    * [Effectuer un `dump`](#effectuer-un-dump)
    * [Redimensionner un volume `LVM` pour disposer d'espace libre](#redimensionner-un-volume-lvm-pour-disposer-despace-libre)
    * [Ajouter une partition `/tftpboot`](#ajouter-une-partition-tftpboot)


## Avez-vous de la place dans la partition `/` ?

Si vous avez conservé les paramètres par défaut lors de l'installation en Squeeze, vous avez une partition racine `/` de 3 Go.

Avec le développement de `se3-clients-linux`, les outils d'installation s'accumulent dans le répertoire `/tftpboot`, et la partition racine `/` devient trop petite pour envisager une migration en toute sérénité.

La commande `df -h` ou l'affichage de l'espace disque dans l'interface web pourra vous aider à voir où vous en êtes…

Plusieurs solutions peuvent être envisagées. Nous en présentons 2.

Si votre installation utilise `LVM`, il est possible de modifier la répartition des différents volumes pour augmenter la taille de la partition racine `/`, ou plutôt pour ajouter une partition spécifique distincte pour `/tftpboot`.

Si votre installation n'utilise pas `LVM`, ou si vous trouvez cela plus simple dans le cas où votre installation utilise`LVM`, l'autre solution est d'ajouter un disque physique, et de l'utiliser pour y placer le répertoire `/tftpboot`.


## Ajouter un disque physique pour déplacer `/tftpboot` et son contenu

Cette procédure nécessite à priori d'éteindre le serveur, d'y ajouter un disque physique, puis de redémarrer le serveur.

Ce disque n'a pas besoin d'être d'un volume important. À la limite, une clé `USB` branchée sur un port interne pourrait suffire !


### Repérer le nouveau disque

La commande suivante devrait vous indiquer le nom du disque ajouté.
```sh
lsblk
```

À priori, si vous n'aviez qu'un seul disque (repéré par `sda`), cela devrait être `sdb` pour le nouveau disque. Repérez bien cette référence pour adapter les commandes ci-dessous.

**Remarque :** selon les versions de Debian, il se peut que cette commande soit indisponible. Il faudra alors utiliser la commande `fdisk -l`. Dans cette commande, les disques sont repérés par `/dev/sda`, `/dev/sdb`,…


### Créer une partition sur l'ensemble du disque

Si ce n'est pas déjà le cas, créer une partition primaire occupant la totalité du disque : 
```sh
fdisk /dev/sdb
```

Répondre aux questions…

…puis formater cette partition en `ext3` :
```sh
mkfs.ext3 /dev/sdb1
```


### Déplacer le contenu de `/tftpboot` dans le disque

Monter ce disque provisoirement dans `/mnt/disque` :
```sh
mkdir /mnt/disque
mount /dev/sdb1 /mnt/disque
```

Déplacer le contenu de `/tftpboot` sur ce nouveau disque :
```sh
mv /tftpboot/* /mnt/disque
```

Vérifier que `/tftpboot` est vide :
```sh
ls -alh /tftpboot
```

Démonter le disque :
```sh
umount /mnt/disque
rmdir /mnt/disque
```


### Monter le disque sur `/tftpboot`

Modifier le fichier `/etc/fstab` en ajoutant la ligne suivante à la fin du fichier :
```sh
/dev/sdb1 /tftpboot     ext3    defaults        0       2
UUID=xxx /tftpboot ext3 defaults 0 2
```
**Remarque :** Dans cette ligne, vous remplacerez xxx par la valeur de l’UUID du disque dur. Pour connaître la valeur de l'UUID, vous pouvez utiliser la commande suivante :
```sh
blkid | grep /dev/sdb1
```

Effectuer les montages contenus dans `/etc/fstab` avec la commande :
```sh
mount -a
```

Vérifier que le contenu de `/tftpboot` est bien là :
```sh
ls -alh /tftpboot
```

**Et voilà !** Un petit coup de la commande suivante (ou bien une petite visite de l'interface web)…
```sh
df -h
```
…pour vérifier que `/` a plus de place, et **vous pouvez respirer !**


## Modifier le partitionnement `LVM`

### Vue d'ensemble

Si vous utilisez `LVM`, il est possible de modifier la répartition de l'espace entre les différentes partitions faisant partie du `groupe de volume`. Pour bien comprendre de quoi il s'agit, il est vivement conseillé de bien connaître [ce qu'est et ce que permet `LVM`](https://doc.ubuntu-fr.org/lvm).

Il demeure **un problème de taille** toutefois. En effet, si la réduction de la taille d'un systeme de fichier `ext3` est possible, seule l'augmentation de la taille d'un système de fichier `xfs` est possible. Or, il y a de fortes chances que vous souhaitiez rogner sur votre partition `/home` ou `/var/se3`, qui sont en `xfs`, pour augmenter la racine ou créer une partition dédiée à `/tftpboot`.

Si c'est à `/var` que vous voulez prendre de l'espace, alors ça devrait être plus simple, car il est en ext3. Mais il faudrait le démonter, et cela pourrait poser problème sur un serveur en fonctionnement. Cette solution réalisable en démarrant le serveur sur un liveCD ne sera pas abordée.

La solution concernant les partitions en xfs consistera à :

* créer un `dump` de la partition `/home` ou `/var/se3` sur un disque externe d'un volume suffisant,
* réduire la taille du volume contenant `/home` ou `/var/se3`,
* reformater le volume réduit en `xfs`,
* restaurer le `dump`,
* utiliser l'espace libéré comme on le souhaite.

Dans l'exemple qui suit, on supposera que l'on dispose d'une partition `/var/se3` de 50 Go utilisée pour moitié (25 Go), qui dispose donc de 25 Go d'espace libre, et dont on souhaite réduire la taille à 45 Go : l'espace libre ne sera plus que de 20 Go.

Il va sans dire qu'il n'est pas possible dans notre cas de réduire la taille de la partition à moins de 25 Go, et on préfèrera, par sécurité, lui laisser plus d'espace que celui réellement utilisé.

D'autre part, la procédure nécessite l'utilisation d'un support externe (disque `USB` ou `NAS`) disposant de l'espace équivalent à l'espace utilisé dans `/var/se3` pour y stocker provisoirement les données. On pourra donc monter provisoirement un disque `USB`, ou comme dans cet exemple, utiliser le montage `/sauveserveur` s'il dispose de l'espace nécessaire.

Enfin, il est vivement conseillé d'effectuer ces opérations lorsque personne n'utilise le réseau ;-)

Ah, et aussi d'avoir [une "vraie" sauvegarde](http://www.samba-edu.ac-versailles.fr/Sauvegarde-et-restauration-SE3) sous le coude !


### Effectuer un `dump`

On vérifie que personne n'utilise `/var/se3` : la commande suivante ne doit rien renvoyer.
```sh
lsof | grep var/se3
```

Pour clore les connexions utilisant éventuellement le disque, on pourra arrêter le service smbd :
```
service smbd stop
```

On démonte `/var/se3` :
```sh
umount /var/se3
```

On vérifie le système de fichier :
```sh
xfs_check /dev/mapper/vol0-lv_var_se3
```

On remonte `/var/se3` :
```sh
mount -a
```

On réalise le `dump` :
```sh
xfsdump -f /sauveserveur/varse3.dump /var/se3
```

ou

```sh
xfsdump -f /sauveserveur/varse3.dump /dev/vol0/lv_var_se3
```

On vous demandera de saisir un "session label" et un "media label". Peu importe ce que vous saisissez, les deux peuvent rester vides.

### Redimensionner un volume `LVM` pour disposer d'espace libre

On peut afficher la liste et le détail des volumes du `LVM` avec les  2 commandes :
```sh
lvs
lvdisplay
```

On démonte `/var/se3` :
```sh
umount /var/se3
```

On réduit la taille du volume logique à 45 Go (adapter la commande à votre cas) :
```sh
lvreduce --size 45g /dev/vol0/lv_var_se3
```

On reformate le volume logique en `xfs` :
```sh
mkfs.xfs -f /dev/vol0/lv_var_se3
```

On remonte le volume tel qu'indiqué dans le fichier `/etc/fstab` :
```sh
mount -a
```

On restaure les données :
```sh
xfsrestore -f /sauveserveur/varse3.dump /var/se3/
```

Et redémarrer le service smbd si nécessaire :
```
service smbd start
```

Nous avons donc maintenant (dans cet exemple) 5Go disponible pour notre groupe de volume.


### Ajouter une partition `/tftpboot`

Il n'est pas nécessaire d'utiliser la totalité de l'espace libéré pour `/tftpboot`, et on peut garder quelques Go sous le coude en cas de coup dur ;-)

Pour connaitre l'espace disponible :
```sh
vgdisplay
```

Dans cet exemple, nous allons créer un nouveau volume de 2 Go :
```sh
lvcreate -n lv_tftpboot -L 2g vol0
```

Si vous souhaitez utiliser tout l'espace rendu disponible :
```sh
lvcreate -n lv_tftpboot -l 100%FREE vol0
```

Que l'on formate en `ext3` (comme `/`) :
```sh
mkfs.ext3 /dev/mapper/vol0-lv_tftpboot
```

On monte provisoirement ce volume :
```sh
mkdir /mnt/tftpboot
mount /dev/vol0/lv_tftpboot /mnt/tftpboot
```

On déplace les données :
```sh
mv /tftpboot/* /mnt/tftpboot/
```

On démonte le volume contenant `/tftpboot` :
```sh
umount /mnt/tftpboot
```

On ajoute la ligne suivante au fichier `/etc/fstab` :
```sh
/dev/mapper/vol0-lv_tftpboot /tftpboot     ext3    defaults        0       2
```

On remonte le tout tel qu'indiqué dans le fichier `/etc/fstab` :
```sh
mount -a
```

On vérifie tout ça et **on se détend ;-)**
```sh
df -h
```

Par la suite, ne pas oublier de supprimer le `dump` qui prend de la place inutilement :
```sh
rm /sauveserveur/varse3.dump
```