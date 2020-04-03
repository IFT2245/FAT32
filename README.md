# FAT32

Tour d'horizon sur le système de fichier FAT32.


## Qu'est-ce qu'un système de fichier?

Un système de fichier est composée de plusieurs parties qui permettent d'organiser le stockage des données sur un médium de stockage d'information. Le choix du système de fichier à une influence sur la rapidité de l'accès aux fichier, la sécurité des données sur le médium (options de redondances de l'information) ainsi que le contrôle d'accès aux données au travers des systèmes de permissions.  

Parmis les systèmes de fichiers, on retrouve

- FAT[8|12|16|32]
- exFAT
- NTFS
- ext4

Et bien d'autres.

Un système de fichier peut être décomposée en plusieurs niveau d'abstraction: on peut parler du système de fichier logique quant on se concentre sur les opérations haut niveau qu'il offre. Un système de fichier virtuel est une abstraction du matériel en composantes d'un système de fichier. C'est ce qu'on retrouve dans linux, quand on ouvre des "streams" comme des fichiers, par exemple. On se concentre ici sur la partie phyisque du système de fichier FAT32: on verra comment les données sont organisées sur le disque, et les opérations à faire pour retrouver cette information.

Certains systèmes de fichiers sont créées avec le médium de stockage en tête. Un système de fichier qui à pour objectif de contenir de l'information sur un disque dur n'aura peut-être pas la même organisation qu'un système de fichier qui stocke de l'information sur un ruban. Le système de fichier FAT32 est un système qui à évolué par rapport à FAT12, et donc il à originallement été conçu pour les disquettes (floppy). Des adaptations l'ont rendu approprié pour être exécuté sur des disques durs et il a longtemps été utilisé, avant d'être remplacé par NTFS comme système de fichier pour le système Windows. Il est encore utilisé de nos jours pour des clés USBs ainsi que des appareils de photographies, qui utilisent le système de fichier FAT comme standard. 

Afin d'être capable de lire des données sur un disque formatté en FAT32, parlons d'abord de l'organisation d'un disque dur.


## Les secteurs

Un disque dur, comme plusieurs médium de stockage, ne permet pas de lire un byte particulier directement. En effet, la granularité est limité à ce que l'on appel un secteur. Un secteur à une interprétation physique, mais nous ne soucierions pas de celle-ci dans ce document. Un secteur est donc, pour nous, une zone de byte contigüe. Sur un disque dur, un secteur est traditionellement de 512 bytes. Le système FAT tiens compte de cette abstraction, mais ne force pas un secteur à être de taille de 512 bytes: le système de fichier peut être construit selon une taille de secteur variable. Il s'agit d'un détail important à tenir en compte.

## Les clusters

Un cluster est un groupement de secteurs. Encore une fois, FAT32 tiens compte de cette structure de donnée. Afin d'éviter du vas et viens inutile sur un disque, le système de fichier FAT32 ne permet **pas** d'allouer une structure plus petite qu'un cluster. C'est la première influence de la considération de la rapidité d'accès que l'on retrouve dans FAT32. Vous l'aurez compris, un fichier occupe donc toujours un minimum d'un cluster comme taille physique sur un disque. La taille logique peut être cependant plus petite.

FAT32 permet de s'adapter à des tailles de cluster variables. Encore une fois, il faut tenir cela en compte lorsqu'on travaille avec le système de fichier. Un cluster pourrait être un seul secteur ou plus. Cependant, un cluster est toujours une puissance de 2. Ainsi, il est possible d'utiliser le logarithme en base 2 pour stocker le nombre de *secteurs par cluster*, ce qui permet de faire des opérations de multiplication efficaces en utiliser des bitshifts.

FAT32 utilisent la terminologie cluster de la même façon que l'on parle de cluster sur un disque, mais avec la particularité que le premier cluster ne se trouve pas au début du disque, mais bien au début de la zone de données. Plus de détails sur cela suivront.


## L'accès à l'information sur un disque dur

Traditionnelement, l'accès à l'information sur un disque dur se fait par un modèle que l'on appel *CHS* (cylinder, head, sector). Ce modèle est une représentée des coordonnées de l'information sur le disque en 3D. Nul besoin de le dire, ce système d'addressage est très pénible à utiliser et a rapidement montré ses limites. 

Il a été remplacé par le *LBA*, le logical block adressing que nous utiliserons ici. Il est beaucoup plus simple: on identifie un secteur par son numéro logique. Ainsi, on parlera du premier secteur (LBA 0), du deuxième (LBA 1), du n-ième (LBA n-1)...

## Comment avoir toute cette information pour un disque donné?

Traditionellement, les disques durs contiennent un *Master boot record*, qui est un secteur spécial qui contient de l'information précise quant au dique que l'on essai d'utiliser. Le MBR peut contenir plusieurs types d'informations dont:

- Organisation physique du disque, information quant au système
- Table de partition
- Code de démarrage de l'OS

Le MBR se trouve au début du disque, au premier secteur (LBA) et donc LBA 0. Il contient un mélange de code et d'information utile pour nous permettre de lire le disque. Voici un exemplle de MBR que j'ai écrit en assembleur, pour vous donner une idée de la structure:

```assembly
jmp after_header  # jump after the header block
.byte 0x00 # pad for correct length
oem_name:
  .ascii "MIMOSA"
  .byte 0
  .byte 0 # OEM name 8 characters (two spaces to get to 8)
nb_bytes_per_sector:
  .word 512    # bytes per sector (512 bytes)
nb_sectors_per_cluster:
  .byte 0x01      # sector per allocation unit -> sector/cluster
nb_reserved_sectors:  
  .word 32      # reserved sectors for booting (32 to get an extended boot sector (also FAT 32 compatible))
nb_of_fats:
  .byte 0x02      # number of FATs (usually 2)
nb_root_dir_entries:
  .word 0x00      # number of root dir entries (0 for FAT 32)
old_nb_logical_sectors:
  .word 0x00    # number of logical sectors on old devices
media_descriptor:
  .byte 0xF8      # media descriptor byte (f0h: floppy, f8h: disk drive)
old_nb_sectors_per_fat: 
  .word 0x00    # sectors per fat (0 For FAT32)
nb_sectors_per_track:
  .word 0x012     # sectors per track (cylinder)
nb_heads:
  .word 64 # number of heads
nb_hidden_sectors:
  .long 0x00 # number of hidden sectors
# --------------------------------------------------------------------------
# FAT 32 EBP
# --------------------------------------------------------------------------
nb_logical_sectors:
  .long 102400 
nb_sectors_per_fat:
  .long 788 
mirror_flags:
  .word 0x00                                                                   # TODO
fs_version:
  .word 0x00
first_cluster_root_dir:
  .long 0x02
fat_32_fs_region:
  .word 0x02      # We use 0x02 because 0x01 is the extended bootsector
backup_bootsec:
  .word 0xFFFF    # (none)
reserved_data:
  .long 0x00
  .long 0x00
  .long 0x00
drive:            # Extended block, supposed to be only for FAT 16
  .byte 0x80      # logical drive number
clean_fs:
  .byte 0x01      # reserved
extended_bpb_sig:
  .byte 0x29      # extended signature
serial_num:
  .byte 0xFF,0xFF,0xFF,0xFF # serial number
drive_lbl:
  .ascii "MIMOSA OS"
  .byte 0
  .byte 0 # drive label (11 char)
fs_label:
  .ascii "FAT32   " # file system type (FAT 12 here, 8 chars long)
# ------------------------------------------------------------------------------
after_header:

# Setup segments.
  cli
  xorw  %cx,%cx
  movw  %cx,%ds
  movw  %cx,%ss
  movw  $(STACK_TOP & 0xffff),%sp
  sti

  movb %dl, drive

  pushl %eax
  pushl %ebx

  movw $oem_name, %si
  call print_string

  movw $new_line, %si
  call print_string

  popl %ebx
  popl %eax
  
# Get drive geometry
  movb $0x08, %ah
  int $0x13
  jc cannot_load

  incb %dh                # dh is maximum head index, we increment by one to get head count
  shrw $8, %dx            # place dh into dl (effictively, also keeps dh to 0x00)
  movw %dx, nb_heads      # put the number of heads in the header (max number + 1)
  andb $0x3f,%cl          # cl: 00s5......s1 (max sector)
  
  xorb %ch, %ch
  movw %cx, nb_sectors_per_track # the number of cylinder is useless for the LDA to CHS conversion
  
  # ------------------------------------------------------------------------------
  # Load the extended bootsector into the RAM
  # In order to use the extended bootsector correctly, the extended boot sector is 
  # loaded directly after the normal bootsector. This allows relative jumps in this
  # file to work correctly.

  movl $0x01, %eax # first sector of drive
  movl $EXTENDED_MEM, %ebx
  call read_sector

  # ------------------------------------------------------------------------------
  # Beyond this point, code in the extended bootsector can be correctly called
  # ------------------------------------------------------------------------------

  movw $extended_bootsector_loaded, %si
  call print_string

  # Prepare values for stage two loading

  movw $ROOT_DIR_ENTRY_SIZE, %ax # size in bytes of an entry in the root table
  xor %dx, %dx 
  mulw nb_root_dir_entries  # DX contains the high part, AX contains the low part of (number of entries * size of entries)
  # = directory size
  # Now we want the number of sectors occupied by the table
  divw nb_bytes_per_sector # ax now contains the number of sectors taken up by the table  
                           # dx contains the number of bytes extra
  movw %ax, nb_root_sectors

  # ------------------------------------------------------------------------------
  # Leave the common bootsector and jump to the extended bootsector
  # ------------------------------------------------------------------------------
  jmp extended_code_start


read_sector:

# Read one sector from relative sector offset %eax (with bootsector =
# 0) to %ebx.
# CF = 1 if an error reading the sector occured
# This function MUST be in the first sector of the bootsector(s)
  pusha

  movl  %eax,%edx               # edx contains LDA
  shrl  $16,%edx                # dx contains LDA
  divw  nb_sectors_per_track    # %ax = cylinder, %dx = sector in track
  incw  %dx                     # increment sector
  movb  %dl,%cl                 # move the sector per track to cl
  xorw  %dx,%dx                 # erase dx
  divw  nb_heads                # %ax = track, %dx = head
  shlb  $6,%ah                  # keep the top 2 bits of track in ah
  orb   %ah,%cl                 # cl is top two bits of ah and sectors per track
  movb  %al,%ch                 # ch is the bottom part of the track
  movb  %dl,%dh                 # head is now in dh

  movb  drive,%dl               # dl is now the drive
  movl  %ebx,%eax               # put the target address in eax
  shrl  $4,%eax                 # div the address by 2^4
  movw  %ax,%es                 # insert the address in es
  andw  $0x0f,%bx
  movw  $0x0201,%ax             # in AH, write 0x02 (command read) and in AL write 0x01 (1 sector to read)

  int   $0x13                   # Call the read
  jc cannot_load

  popa

  ret

# ------------------------------------------------------------------------------
# Routines and functions
# ------------------------------------------------------------------------------

cannot_load:
  
  pushl %eax
  pushl %ebx

  movw $load_error_message, %si
  call print_string

failure_routine:
  movw $any_key_message, %si
  call print_string
  
  popl %ebx
  popl %eax

  xorb %ah, %ah          # Make sure it's on read key
  int $0x16
  
  ljmp  $0xf000,$0xfff0  # jump to 0xffff0 (the CPU starts there when reset)

nb_root_sectors:
  .long 0x00000 # number of sectors in the root directory
cluster_begin_lba:
  .long 0x00    # default value on floppy is 19; should be read correctly

any_key_reboot_msg:
  .ascii "\n\rPress any key to reboot"
  .byte 0

new_line:
  .ascii "\n\r"
  .byte 0

load_error_message:
  .ascii "IO Error. The system failed to load. Please reboot."
  .byte 0

any_key_message:
  .ascii "Press any key to reboot..."
  .byte 0

kernel_name:
  .ascii "BOOT    SYS" # the extension is implicit, spaces mark blanks

# Print string utility.
print_string_loop:
  movb  $INT10_TTY_OUTPUT_FN,%ah
  movw  $(0<<8)+7,%bx   # page = 0, foreground color = 7
  int   $0x10
  incw  %si
print_string:
  movb  (%si),%al
  test  %al,%al
  jnz   print_string_loop
  ret
# ------------------------------------------------------------------------------
# MBR data
# ------------------------------------------------------------------------------
code_end:
  .space (1<<9)-(2 + 16 * 4)-(code_end-code_start)  # Skip to the end (minus 2 for the signature)
# Partition table (only one)

# partition 1
.byte 0x80                   # boot flag (0x00: inactive, 0x80: active)
.byte 0x00, 0x00, 0x00       # Start of partition address
.byte 0x0C                   # system flag (xFAT32, LBA access)
.byte 0x00, 0x00, 0x00       # End of partition address CHS : 79 1 18
.long 0x00                   # Start sector relative to disk
.long 102400 # number of sectors in partition

# partition 2
.byte 0x00                   # boot flag (0x00: inactive, 0x80: active)
.byte 0x00, 0x00, 0x00       # Start of partition address
.byte 0x00                   # system flag
.byte 0x00, 0x00, 0x00       # End of partition address
.long 0x00                   # Start sector relative to disk
.long 0x00 # number of sectors in partition

# partition 3
.byte 0x00                   # boot flag (0x00: inactive, 0x80: active)
.byte 0x00, 0x00, 0x00       # Start of partition address
.byte 0x00                   # system flag
.byte 0x00, 0x00, 0x00       # End of partition address
.long 0x00                   # Start sector relative to disk
.long 0x00 # number of sectors in partition

# partition 4
.byte 0x00                   # boot flag (0x00: inactive, 0x80: active)
.byte 0x00, 0x00, 0x00       # Start of partition address
.byte 0x00                   # system flag
.byte 0x00, 0x00, 0x00       # End of partition address
.long 0x00                   # Start sector relative to disk
.long 0x00 # number of sectors in partition

# Signature
.byte 0x55
.byte 0xAA
```

Tout ce code fait exactement 512 bytes. On peut voir qu'il termine par une signature: les bytes
`0x55 0xAA` identifient la fin MBR, au position 510 et 511 obligatoirement. Le disque ici contient une seule partition, qui fait l'ensemble du disque. 

Le code de démarrage ici nous intéresse peu. D'ailleurs, il manque le reste, qui s'occupe ici de charger un fichier du système de fichier FAT32. La partie intéressante se situe entre les labels `oem_name` et `fs_label` inclusivement. C'est la que se trouvent les données intéressantes. Voici une structure C organisée selon cette information:

```C
typedef struct BIOS_Parameter_Block_struct {
    uint8 BS_jmpBoot[3];
    uint8 BS_OEMName[8];
    uint8 BPB_BytsPerSec[2];  // 512, 1024, 2048 or 4096
    uint8 BPB_SecPerClus;     // 1, 2, 4, 8, 16, 32, 64 or 128
    uint8 BPB_RsvdSecCnt[2];  // 1 for FAT12 and FAT16, typically 32 for FAT32
    uint8 BPB_NumFATs;        // should be 2
    uint8 BPB_RootEntCnt[2];
    uint8 BPB_TotSec16[2];
    uint8 BPB_Media;
    uint8 BPB_FATSz16[2];
    uint8 BPB_SecPerTrk[2];
    uint8 BPB_NumHeads[2];
    uint8 BPB_HiddSec[4];
    uint8 BPB_TotSec32[4];
    uint8 BPB_FATSz32[4];
    uint8 BPB_ExtFlags[2];
    uint8 BPB_FSVer[2];
    uint8 BPB_RootClus[4];
    uint8 BPB_FSInfo[2];
    uint8 BPB_BkBootSec[2];
    uint8 BPB_Reserved[12];
    uint8 BS_DrvNum;
    uint8 BS_Reserved1;
    uint8 BS_BootSig;
    uint8 BS_VolID[4];
    uint8 BS_VolLab[11];
    uint8 BS_FilSysType[8];
} BPB;
```

Les nombres sont encodées en `little endian`. C'est pour cela que les nombres de plus de 8 bits
sont enregistrés dans un tableau: on utilise une macro pour les reconstruires correctement. Regardons les champs:

-    `uint8 BS_jmpBoot[3];`: L'instruction `jmp` au début du code. Permet au processeur de sauter par dessus les données
-    `uint8 BS_OEMName[8];`: Le nom du système
-    `uint8 BPB_BytsPerSec[2]; // 512, 1024, 2048 or 4096`: le nombre de bytes dans un secteur
-    `uint8 BPB_SecPerClus;     // 1, 2, 4, 8, 16, 32, 64 or 128`: le nombre de secteurs dans un cluster
-    `uint8 BPB_RsvdSecCnt[2];  // 1 for FAT12 and FAT16, typically 32 for FAT32`: le nombre de secteur réservés (pour ce que l'on veut)
-    `uint8 BPB_NumFATs;        // should be 2`: Le nombre de tables FAT (que l'on verra par après)
-    `uint8 BPB_RootEntCnt[2];` Le nombre d'entrées root (0 en FAT32)
-    `uint8 BPB_TotSec16[2];` Le nombre total de secteurs (pas en FAT32)
-    `uint8 BPB_Media;` Le type de média physique (`0xF8` pour un disque dur)
-    `uint8 BPB_FATSz16[2];` Le nombre de secteur pour une table FAT
-    `uint8 BPB_SecPerTrk[2];` Le nombre de secteur par track
-    `uint8 BPB_NumHeads[2];` Le nombre de têtes
-    `uint8 BPB_HiddSec[4];` Le nombre de secteurs cachés (on assumera 0)
-    `uint8 BPB_TotSec32[4];` Le nombre total de secteurs, cette fois pour FAT 23
-    `uint8 BPB_FATSz32[4];` La taille d'une table FAT (pour fat 32)
-    `uint8 BPB_ExtFlags[2];` On ignore ce champ
-    `uint8 BPB_FSVer[2];` La version du système de fichier
-    `uint8 BPB_RootClus[4];` Le cluster de la table du dossier root (ici on assumera que c'est toujours 2)
-    `uint8 BPB_FSInfo[2];` La version (on ignore)
-    `uint8 BPB_BkBootSec[2];`
-    `uint8 BPB_Reserved[12];`
-    `uint8 BS_DrvNum;` On ignore
-    `uint8 BS_Reserved1;` On ignore (réservé)
-    `uint8 BS_BootSig;` On ignore
-    `uint8 BS_VolID[4];` On ignore
-    `uint8 BS_VolLab[11];` On ignore
-    `uint8 BS_FilSysType[8];` Une chaîne de caractère qui devrait dire FAT32

Après avoir lu cette structure au début d'un disque, il est maintenant possible de correctement lire un système de fichier FAT32. En effet, on retrouve tous les paramètres tel que le nombre de bytes dans un secteur ainsi que le nombre de secteurs par cluster.


## Architecture d'un système de fichier FAT32

Il y a trois principales sections à un système de fichier FAT32. La première est composée des secteurs réservées et cachées. Normalement, outre le MBR, que le vient de lire, nous n'avons plus besoin de cette section. Par la suite, on retrouve les tables FATs, au nombre indiqué dans le MBR. La troisième section consiste aux données. Le partitionnement du disque peut changer ces détails, mais nous allons ici considérer le cas d'un disque qui est partitionné comme le précédent, c'est à dire une unique partition qui couvre l'ensemble du disque.

| Section    | Longueur                        |
|------------|---------------------------------|
| Entête     | 0 + $rsvd + $hidden             |
| Tables FAT | Selon le MBR                    |
| Données    | Selon le MBR / taille du disque |


### Les tables FAT et leur structure

On retrouve plusieurs tables FAT (*file allocation table)*. Nous n'en avons cependant besoin que d'une seule. En effet, le système FAT32 permet d'avoir plusieurs tables d'allocation afin de permettre d'avoir de la redondance en cas que certains secteurs ne peuvent pas être lus. La table FAT permet de lire les fichiers et dossiers qui requierent plus qu'un seul cluster d'espace. On pourrait croire que cette table n'est pas nécéssaire et qu'il suffit d'aller lire le cluster suivant, mais ce n'est pas le cas. En effet, les clusters d'un fichier ne sont pas nécéssairement consécutif (pour permettre les changements de tailles de fichier même une fois le système initialisé). 

Un table FAT est un tableau contigüe d'entiers de 32 bits de large (d'ou le nom FAT*32*). Chaque cellule du tableau FAT contient le cluster qui suit le cluster identifié par l'index de la cellule. Voici un exemple:

| Cluster n | Cluster n+1 | Cluster n+2 | Cluster n+3 |
|-----------|-------------|-------------|-------------|
| 0xFDDA   | 0xABCD       | 0xAE12BCD   |  0xA213A    |

Ici, on a un extrait de la chaîne. Les valeurs de la deuxième rangées sont celles qui sont réellement écrites: la première rangée sert à donner du contexte sur les valeurs que l'on voit. 

À l'entrée n, le nombre 0xFDDA indique que le cluster qui suit le cluster n est le cluster 0xFDDA. La même logique s'applique pour tous les autres clusters. Il y a cependant quelques valeurs spéciales:

- Une valeur de 0 indique que le cluster n'est pas utilisé. Il est donc libre d'être affecté comme premier cluster d'un nouveau fichier, ou d'être utilisé comme un cluster à une position arbitraire dans une autre chaîne.
- Une valeur plus grande ou égale à 0xFFFFFF8 indique qu'il s'agit du dernier cluster d'une chaîne. Le fichier est donc terminé à ce cluster.
- Un numéro de cluster, bien qu'écrit sur 32 bits, n'utilise que 28 bits. Les 4 bits du haut sont donc réservés et pourraient avoir une valeur arbitraire. Il faut donc les ignorer.
- Les numéros de clusters sont aussi écrit en `little endian`, comme les données du MBR.

Il est important de se rappeler que le premier cluster des données n'est pas nécéssairement le cluster 0. Les données contenue dans le MBR indiquent (la plupart du temps) que le premier cluster est le cluster 2. Les cluster 0 et 1, dans ce cas, contiennent des valeurs réservées.

### La zone de données

Après les tables FAT, on retrouve les données. Le premier fichier, appelé le root entry, correspond au dossier au niveau root du disque. Tous les autres fichiers sont des descendants de ce dossier. Son numéro de cluster est indiqué dans le bloc de paramètres qui est placé au début du disque. La zone de données est relative au début de ce fichier. On calcule donc le LBA d'un cluster en fonction de celui-ci. Si `$root-entry` correspond au numéro de cluster de ce dossier, et que `$begin = $rsvd + $hidden + $no-fats * $fat-size`, les paramètres correspondant dans la table, on calcule le LBA de cette façon:

```php
lba($cluster) = $begin + ($cluster - $root-entry) * $sectors-per-clusters
```

Ainsi, étant donné un numéro de cluster, le premier secteur identifié par ce cluster correspond au résultat de la formule. On obtient que la zone de données commence au secteur `lba($root-entry)`.

Le reste de la zone des données correspond à des contenus de fichiers, ou des contenus de dossier, dépendamment de ou on se trouve. Regardons maintenant comment ces données sont structurées.

### La structure d'un fichier

Un fichier FAT32 ne contient pas de structure particulières: les données sont écrites comme elles se trouvent dans le fichier. Il suffit donc de lire les bytes correctement, et de suivre les clusters correctement pour lire un fichier.

### La structure d'un dossier

Afin de pouvoir identifier les fichiers contenus dans un dossier, une structure spéciale est donnée au contenu des dossiers. Il s'agit de blocs contiguës de ce que l'on appel des "entrées de fichiers". Celles-ci ont cette structure:

| Nom                             | Largeur (bytes) |
|---------------------------------|-----------------|
| Nom du fichier                  | 11              |
| Attributs du fichier            | 1               |
| Reservé                         | 1               |
| 10iem de secondes               | 1               |
| Heure de création               | 2               |
| Date de création                | 2               |
| Date du dernier accès           | 2               |
| Partie haute du premier cluster | 2               |
| Dernière heure d'écriture       | 2               |
| Dernière date d'écriture        | 2               |
| Partie basse du premier cluster | 2               |
| Taille du fichier               | 4               |

Une structure C pourrait donc être construite ainsi:

```C
typedef struct FAT_directory_entry_struct {
    uint8 DIR_Name[FAT_NAME_LENGTH];
    uint8 DIR_Attr;
    uint8 DIR_NTRes;
    uint8 DIR_CrtTimeTenth;
    uint8 DIR_CrtTime[2];
    uint8 DIR_CrtDate[2];
    uint8 DIR_LstAccDate[2];
    uint8 DIR_FstClusHI[2];
    uint8 DIR_WrtTime[2];
    uint8 DIR_WrtDate[2];
    uint8 DIR_FstClusLO[2];
    uint8 DIR_FileSize[4];
} FAT_entry;
```

Si `$cluster_high` et `$cluster_low` sont les parties hautes et basse du premier cluster du fichier respectivement, le premier cluster du fichier est donc:

```php
$cluster = ($cluster_high << 16) + $cluster_low
```

La largeur totale d'une entrée est de 32 bytes, ce qui garantie que dans une même secteur, un nombre entier d'entrées s'y trouvent. Cela facilite la lecture des entrées dans un dossier.

#### Détail des champs

##### Name
Le champ `name` contient le nom du fichier (ou dossier). Le nom est toujours en majuscule. Le caractère espace doit être traduit par un symbol vide. Cela implique qu'un espace n'est pas un caractère valide dans un nom de fichier. Les trois derniers caractères sont l'extension, mais le point n'est pas là: il est implicite. Voici quelques examples:

- `unfichie.txt:_|UNFICHIETXT|`
- `petit.txt____:|PETIT___TXT|`
- `dossier______:|DOSSIER____|`

Les `|` sont simplement pour délimiter les noms et indiquer que le champ est toujours 11 bytes de long. Les `_` indiquent un espace (le caractère 32 de la table ascii).

Si le premier caractère du nom est 0xE5, l'entrée à été supprimée. Cependant, l'entrée suivante pourrait être utilisée. Par contre, si le premier caractère est 0x00, l'entrée n'est pas utilisée et les prochaines entrées ne sont pas utilisées non plus.

##### Attribut

Le champ attribut contient les attributs du fichier. On ne détaillera pas toutes les valeurs ici, mais si la valeur masquée par 0x10 est non-nulle, on peut conclure que l'entrée indique un dossier et non fichier.

### Navigation dans le système

Nous avons donc maintenant assez d'information sur la structure de FAT32 pour pouvoir naviguer dans une archive. Étant donné un chemin, il suffit de le décomposer en dossier à suivre, de lire le contenu de chaque dossier pour trouver le premier cluster du prochain dossier / fichier et de continuer à naviguer ainsi. Il faut faire attention lors de la lecture d'une entrée à bien passer d'un cluster à l'autre, ce qui permet de lire la suite du fichier / dossier. 


## Commande pour explorer une archive

Plusieurs outils peuvent être utilisés pour explorer une archive FAT32. Le premier, mais pas nécéssairement le plus utile, est la commande `mount` qui permet de monter une archive comme un disque (après tout, c'est exactement ce que c'est). La commande mount est utile, mais ne permet pas de voir la représentation binaire des objets. Lorsque vous implémenter un _pilote_ tel que le TP4 le demande, il est pratique de voir exactement la structure binaire des fichiers.

Pour cela, la commande `hexdump` est beaucoup plus pratique. Elle permet de lire un fichier binaire et d'imprimer les bytes lus en format hex (en fait, vous pouvez changer la représentation). Personnellement, je conseil d'utiliser la commande en mode canonique (option `-C`) qui va imprimer une représentation ASCII des charactères. En plus d'avoir l'air d'un hacker, vous pouvez lire les noms de fichiers, ou le contenu de ceux-ci, ce qui vous permet de vous repérer si vous n'êtes pas certain de votre positionnement dans le fichier.

Voici un exemple, qui contient le boot block montré plus haut:

```sh
> hexdump -C -n 512 floppy.img
00000000  eb 58 00 4d 49 4d 4f 53  41 00 00 00 02 01 20 00  |.X.MIMOSA..... .|
00000010  02 00 00 00 00 f8 00 00  12 00 02 00 00 00 00 00  |................|
00000020  00 00 01 00 f8 01 00 00  00 00 00 00 02 00 00 00  |................|
00000030  02 00 ff ff 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000040  80 01 29 ff ff ff ff 4d  49 4d 4f 53 41 20 4f 53  |..)....MIMOSA OS|
00000050  00 00 46 41 54 33 32 20  20 20 fa 31 c9 8e d9 8e  |..FAT32   .1....|
00000060  d1 bc 00 00 fb 88 16 40  7c 66 50 66 53 be 03 7c  |.......@|fPfS..||
00000070  e8 25 01 be 33 7d e8 1f  01 66 5b 66 58 b4 08 cd  |.%..3}...f[fX...|
00000080  13 72 71 fe c6 c1 ea 08  89 16 1a 7c 80 e1 3f 30  |.rq........|..?0|
00000090  ed 89 0e 18 7c 66 b8 01  00 00 00 66 bb 00 7e 00  |....|f.....f..~.|
000000a0  00 e8 19 00 be a9 7f e8  ee 00 b8 20 00 31 d2 f7  |........... .1..|
000000b0  26 11 7c f7 36 0b 7c a3  11 7d e9 43 01 60 66 89  |&.|.6.|..}.C.`f.|
000000c0  c2 66 c1 ea 10 f7 36 18  7c 42 88 d1 31 d2 f7 36  |.f....6.|B..1..6|
000000d0  1a 7c c0 e4 06 08 e1 88  c5 88 d6 8a 16 40 7c 66  |.|...........@|f|
000000e0  89 d8 66 c1 e8 04 8e c0  83 e3 0f b8 01 02 cd 13  |..f.............|
000000f0  72 02 61 c3 66 50 66 53  be 36 7d e8 9a 00 be 6a  |r.a.fPfS.6}....j|
00000100  7d e8 94 00 66 5b 66 58  30 e4 cd 16 ea f0 ff 00  |}...f[fX0.......|
00000110  f0 00 00 00 00 00 00 00  00 0a 0d 50 72 65 73 73  |...........Press|
00000120  20 61 6e 79 20 6b 65 79  20 74 6f 20 72 65 62 6f  | any key to rebo|
00000130  6f 74 00 0a 0d 00 49 4f  20 45 72 72 6f 72 2e 20  |ot....IO Error. |
00000140  54 68 65 20 73 79 73 74  65 6d 20 66 61 69 6c 65  |The system faile|
00000150  64 20 74 6f 20 6c 6f 61  64 2e 20 50 6c 65 61 73  |d to load. Pleas|
00000160  65 20 72 65 62 6f 6f 74  2e 00 50 72 65 73 73 20  |e reboot..Press |
00000170  61 6e 79 20 6b 65 79 20  74 6f 20 72 65 62 6f 6f  |any key to reboo|
00000180  74 2e 2e 2e 00 42 4f 4f  54 20 20 20 20 53 59 53  |t....BOOT    SYS|
00000190  b4 0e bb 07 00 cd 10 46  8a 04 84 c0 75 f2 c3 00  |.......F....u...|
000001a0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000001b0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 80 00  |................|
000001c0  00 00 0c 00 00 00 00 00  00 00 00 00 01 00 00 00  |................|
000001d0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|
00000200
```

L'étoile (\*) indique que les lignes sont répétées. L'option `-v` vous permet de désactiver le groupement des lignes similaires si vous le préférez.

L'option `-n` vous permet de spécifier le nombre de **bytes** que vous voulez lire. Cela vous permet de vous concentrer sur la zone que vous voulez lire.

L'option `-s` vous permet de sauter un nombre de bytes. Vous pouvez changer l'unité de mesure si vous voulez. 

Si on veut voir le dossier root de l'archive, par exemple:

```sh
> hexdump -C -s 532480 -n 512 floppy.img
00082000  5a 4f 4c 41 20 20 20 20  54 58 54 20 00 2b a3 78  |ZOLA    TXT .+.x|
00082010  83 50 83 50 00 00 a3 78  83 50 03 00 bb cb 00 00  |.P.P...x.P......|
00082020  53 50 41 4e 49 53 48 20  20 20 20 10 00 94 57 7b  |SPANISH    ...W{|
00082030  83 50 83 50 00 00 57 7b  83 50 69 00 00 00 00 00  |.P.P..W{.Pi.....|
00082040  41 46 4f 4c 44 45 52 20  20 20 20 10 00 2e fb 79  |AFOLDER    ....y|
00082050  83 50 83 50 00 00 fb 79  83 50 3d 01 00 00 00 00  |.P.P...y.P=.....|
00082060  48 45 4c 4c 4f 20 20 20  54 58 54 20 00 50 fb 7a  |HELLO   TXT .P.z|
00082070  83 50 83 50 00 00 fb 7a  83 50 f7 02 1a 00 00 00  |.P.P...z.P......|
00082080  e5 41 4d 42 49 54 20 20  20 20 20 10 00 64 a7 58  |.AMBIT     ..d.X|
00082090  4b 4f 4b 4f 00 00 a7 58  4b 4f 5f 03 00 00 00 00  |KOKO...XKO_.....|
000820a0  e5 67 00 73 00 69 00 00  00 ff ff 0f 00 c4 ff ff  |.g.s.i..........|
000820b0  ff ff ff ff ff ff ff ff  ff ff 00 00 ff ff ff ff  |................|
000820c0  e5 53 49 20 20 20 20 20  20 20 20 20 00 00 a8 58  |.SI         ...X|
000820d0  4b 4f 4b 4f 00 00 a8 58  4b 4f 00 00 00 00 00 00  |KOKO...XKO......|
000820e0  e5 67 00 73 00 69 00 2e  00 65 00 0f 00 ce 78 00  |.g.s.i...e....x.|
000820f0  65 00 00 00 ff ff ff ff  ff ff 00 00 ff ff ff ff  |e...............|
00082100  e5 53 49 20 20 20 20 20  45 58 45 20 00 00 a8 58  |.SI     EXE ...X|
00082110  4b 4f 4b 4f 00 00 a8 58  4b 4f 00 00 00 00 00 00  |KOKO...XKO......|
00082120  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
```

Ici on peut voir les fichiers `ZOLA.TXT`, `HELLO.TXT` et les dossiers `SPANISH` ainsi que `AFOLDER`. Évidemment, il faut connaître le bon emplacement de chaque chose. Ici, on peut voir que le début du premier secteur de données correspond aux bytes qui suivent 532480. Ce genre d'information peut être obtenu en analyant le block de paramètres au début de l'archive.
