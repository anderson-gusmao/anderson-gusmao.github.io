---
layout: post
title:  "Comment l'ordinateur découvre-t-il quoi charger à l'allumage ?"
date:   2026-05-01 18:53:56 -0300
categories: operating-system
lang: fr
permalink: /fr/operating-system/2026/05/01/bootloader/
---
**Dans cet article, nous explorons le processus technique de démarrage d'un ordinateur, en détaillant le chargement du Master Boot Record (MBR), la structure des secteurs d'un disque et l'implémentation d'un bootloader en Assembly avec un focus sur la validation de la signature de boot.**

# Chargement du bootloader

Lors de la mise sous tension, le matériel exécute le `Power-On Self Test (POST)`. Après la validation des composants, le BIOS recherche des périphériques de stockage amorçables. Le premier secteur physique (les **512 premiers octets, connus sous le nom de secteur d'amorçage ou MBR**) est chargé dans la mémoire RAM. Le BIOS valide l'intégrité de ce secteur en recherchant la "**signature magique**" `0xAA55` dans ses deux derniers octets ; si elle est présente, l'exécution est transférée à cette adresse et le processus de démarrage se poursuit.

> ⚠️ **Note technique :** Introduit en 1983, le **MBR (Master Boot Record)** a été le standard dominant pendant des décennies. Actuellement, les systèmes modernes utilisent le **GPT (GUID Partition Table)**, qui surpasse les limitations de partitionnement et de taille de disque du MBR. À des fins didactiques, nous utiliserons le modèle MBR dans cet article.
{: .warning}

![Chargement du secteur d'amorçage](/assets/images/boot-sector-loading-en.png)

# Géométrie du disque et secteurs

Pour comprendre l'organisation des données, considérez la structure d'un disque dur (HDD) conventionnel. Le disque est organisé de manière hiérarchique en secteurs, pistes (tracks) et têtes (heads).

![Secteurs d'un disque dur](/assets/images/disk-sector-tracks-en.png)

Le **MBR (Master Boot Record)** se situe dans le **premier secteur** physique du disque. Ce secteur possède exactement **512 octets** de capacité, l'espace où est défini le **premier stade (Stage 1)** du chargeur d'amorçage du système d'exploitation.

# Implémentation du secteur de boot

Le développement d'un bootloader est typiquement réalisé en **Assembly**, garantissant un contrôle direct sur le matériel et la cartographie de la mémoire. Ci-dessous, nous présentons un exemple d'implémentation :

{% highlight assembly linenos%}
# ***********************************************************
# Exemple de secteur de boot
# ***********************************************************

.code16
.intel_syntax noprefix
.text
.org 0x0                                        

LOAD_SEGMENT = 0x1000                     # Le chargeur de 2e stade sera chargé dans le segment 1000h
FAT_SEGMENT  = 0x0ee0                     # La FAT du disque de boot sera chargée dans le segment 0x0ee0 
                                          # (9*512 octets sous le chargeur de 2e stade)

.global main

main:
    jmp short start                       # Saute au début du code
    nop                                   # Alignement (nop) pour l'en-tête du secteur de boot

.include "bootsector.s"
.include "macros.s"

start:
  mInitSegments                           # Initialise les segments de mémoire
  mResetDiskSystem                        # Réinitialise le sous-système de disque
  mWriteString loadmsg                    # Affiche le message de chargement
  mFindFile filename, LOAD_SEGMENT        # Localise le fichier du 2e stade dans le répertoire racine
  mReadFAT FAT_SEGMENT                    # Charge la table FAT en mémoire
  mReadFile LOAD_SEGMENT, FAT_SEGMENT     # Transfère le 2e stade en mémoire RAM
  mStartSecondStage                       # Transfère le flux d'exécution au 2e stade
 
# Routine de gestion des échecs du processus de boot
bootFailure:
  mWriteString diskerror                  # Affiche un message d'erreur de disque
  mReboot                                 # Sollicite le redémarrage du système
  
.include "functions.s"
    
# Définition des données et constantes
filename:    .asciz "2NDSTAGEBIN"
rebootmsg:   .asciz "Appuyez sur n'importe quelle touche pour redémarrer.\r\n"
diskerror:   .asciz "Erreur de disque. "
loadmsg:     .asciz "Chargement de DevOS...\r\n"

root_strt:   .byte 0,0      # Offset du répertoire racine
root_scts:   .byte 0,0      # Nombre de secteurs du répertoire racine
file_strt:   .byte 0,0      # Offset du bootloader sur le disque

.fill (510-(.-main)), 1, 0  # Padding avec des zéros jusqu'à l'octet 510
BootMagic:  .int 0xAA55     # Signature magique pour la reconnaissance par le BIOS
{% endhighlight %}

# Analyse technique et signature de boot

L'implémentation utilise des directives spécifiques pour garantir la conformité au standard MBR. Le point critique réside dans le remplissage du secteur pour atteindre la taille exacte de 512 octets (lignes 49 et 50).

Comme le volume d'instructions et de données peut varier, il est nécessaire de calculer dynamiquement le remplissage (padding) nécessaire. L'expression utilisée est :
`(510 - (.-main))`

Composants de l'expression :
*   `.` (point) : Représente le compteur de localisation actuel (location counter).
*   `main` : L'adresse du point d'entrée initial.
*   `(.-main)` : Calcule le déplacement (offset) total d'octets générés jusqu'à présent.
*   `510 - (.-main)` : Détermine combien d'octets il reste pour atteindre la marque de 510 octets.

Les deux derniers octets (511 et 512) sont réservés pour la signature **0xAA55**. Sans cette signature, le BIOS ne reconnaîtra pas le périphérique comme amorçable.

# Génération du binaire et chargement en RAM

Après l'assemblage du code source, un fichier binaire est généré. Quand l'ordinateur identifie un périphérique de démarrage, le BIOS lit le premier secteur (512 octets), copie son contenu à l'adresse physique de mémoire **0x7C00** et lance l'exécution des instructions à partir de cette adresse.

![Fichier binaire en hexadécimal](/assets/images/bootfile-en.png)

# Prochaines étapes et conclusion

L'initialisation d'un système d'exploitation est un processus multi-stades. Le code analysé représente le **Stade 1**, dont la fonction primordiale est de localiser et charger le stade subséquent en mémoire RAM, comme démontré dans la logique à partir de la ligne 23.

Dans de futurs articles, nous détaillerons les phases postérieures, incluant la transition vers le Mode Protégé et le chargement du Kernel du système.
