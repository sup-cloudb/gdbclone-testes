### GDBCLONE 
---------------------------------
PM : Ruggero Citton - RAC Pack, Cloud Innovation and Solution Engineering Team

GDBCLONE é uma ferramenta/orquestrador que serve para automatizar processos de clone/snap de bases de dados ORACLE. Em background os binarios de ACFS ( ACFSUTIL) e rman e sqlplus são utilizados.

O DOC ID **gDBClone Powerful Database Clone/Snapshot Management Tool (Doc ID 2099214.1)**  contem todos os cenarios de CLONE/SNAP/CONVERSAO/MIGRACAO/UPGRADE a partir da version 3.0 lancada em 2016 contemplando bases a partir da 11g e com GRID INFRASTRUCTURE minimo 12c+.



- **Instalacao do GDBCLONE** 


> rpm -ivh gDBClone-4.0.0-3.el8.x86_64.rpm

Como nao tem o comando tree no DBCS ou no EXACS seria necessario instalar : 

> [root@dbtryh ~]# yum install tree

![Image](https://github.com/user-attachments/assets/0fc641e6-5602-47ff-a52c-7f4bef2f2613)

Dessa forma podemos validar as dependencias do diretorio HOME do GDBCLONE 👍 

> [root@dbtryh ~]#  tree /opt/gDBClone

```
/opt/gDBClone
└── gDBClone.bin

0 directories, 1 file

```

Para fazer o sudo para o usuario ORACLE seria necessario primeiro criar a pasta /opt/gDBClone/out para nao dar o erro abaixo : 

![Image](https://github.com/user-attachments/assets/c08b57be-22ba-43c2-b214-a12bc74d203f)

> [root@dbtryh ~]# mkdir /opt/gDBClone/out
> [root@dbtryh ~]# chown -R oracle:oinstall /opt/gDBClone/out
> 

Adicionar o usuario no sudoers para os comandos do gdbclone : 

> [root@dbtryh ~]# sudo visudo

```
#includedir /etc/sudoers.d
opc ALL=(ALL) NOPASSWD: ALL
Cmnd_Alias GDBCLONE_CMDS = /opt/gDBClone/gDBClone.bin, /opt/gDBClone/gDBClone.bin *
oracle ALL=(ALL) NOPASSWD: GDBCLONE_CMDS

```

Ja com o usuario ORACLE, altere o .bash_profile : 


> alias gdbclone='sudo /opt/gDBClone/gDBClone.bin'


** EXEMPLOS DE GDBCLONE**

1. Criar o ADVM e o ACFS : 

_ @Sugestao : Pode-se criar 2 ACFS para servir de DATAACFS e outro para RECOACFS, utilizando assim melhor o outro diskgroup e tenho os archives em um outro VOLUME._

> ASMCMD> volinfo --all
> Diskgroup Name: DATA
> 
>          Volume Name: COMMONSTORE
>          Volume Device: /dev/asm/commonstore-363
>          State: ENABLED
>          Size (MB): 5120
>          Resize Unit (MB): 64
>          Redundancy: UNPROT
>          Stripe Columns: 8
>          Stripe Width (K): 1024
>          Usage: ACFS
>          Mountpath: /opt/oracle/dcs/commonstore
> 
> 
> 

> ASMCMD>   volcreate -G DATA -s 100G VL_SNAP_DTA

Para criar o outro volume no RECO, nao pode passar de 11 caracteres...

ASMCMD>  volcreate -G RECO  -s 50G vol_reco_snaps
ORA-15032: not all alterations performed
**ORA-15474: volume name is greater than 11 characters (DBD ERROR: OCIStmtExecute)**
```
ASMCMD>  volcreate  -G RECO  -s 50G VL_SNAP_RCO
ASMCMD>  volinfo --all
Diskgroup Name: DATA

         Volume Name: COMMONSTORE
         Volume Device: /dev/asm/commonstore-175
         State: ENABLED
         Size (MB): 5120
         Resize Unit (MB): 64
         Redundancy: UNPROT
         Stripe Columns: 8
         Stripe Width (K): 1024
         Usage: ACFS
         Mountpath: /opt/oracle/dcs/commonstore

         Volume Name: VL_SNAP_DTA
         Volume Device: /dev/asm/vl_snap_dta-175
         State: ENABLED
         Size (MB): 102400
         Resize Unit (MB): 64
         Redundancy: UNPROT
         Stripe Columns: 8
         Stripe Width (K): 1024
         Usage:
         Mountpath:

Diskgroup Name: RECO

         Volume Name: VL_SNAP_RCO
         Volume Device: /dev/asm/vl_snap_rco-470
         State: ENABLED
         Size (MB): 51200
         Resize Unit (MB): 64
         Redundancy: UNPROT
         Stripe Columns: 8
         Stripe Width (K): 1024
         Usage:
         Mountpath:


```



Formatar o device (ADVM) 

> [root@dbtryh ~]# mkfs.acfs /dev/asm/vl_snap_dta-175
> mkfs.acfs: version                   = 19.0.0.0.0
> mkfs.acfs: on-disk version           = 46.0
> mkfs.acfs: volume                    = /dev/asm/vl_snap_dta-175
> mkfs.acfs: volume size               = 107374182400  ( 100.00 GB )
> mkfs.acfs: Format complete.
> 

> [root@dbtryh ~]# mkfs.acfs /dev/asm/vl_snap_rco-470

```
[root@dbtryh ~]# mkfs.acfs /dev/asm/vl_snap_rco-470
mkfs.acfs: version                   = 19.0.0.0.0
mkfs.acfs: on-disk version           = 46.0
mkfs.acfs: volume                    = /dev/asm/vl_snap_rco-470
mkfs.acfs: volume size               = 53687091200  (  50.00 GB )


```
[root@dbtryh ~]# mkdir /snapreco
[root@dbtryh ~]# acfsutil registry -a /dev/asm/vl_snap_rco-470 /snapreco
acfsutil registry: mount point /snapreco successfully added to Oracle Registry


Criar o PATH e setar o OWNER para o usuario ORACLE : 

```
[root@dbtryh ~]# mkdir /snapdata
[root@dbtryh ~]# chown -R oracle:oinstall /snapdata
[root@dbtryh ~]# mount /dev/asm/vol_snaps-363 /snapdata
[root@dbtryh ~]# mount.acfs /dev/asm/vol_snaps-363 /snapdata
[root@dbtryh ~]# acfsutil registry -a /dev/asm/vol_snaps-363 /snapdata
acfsutil registry: mount point /snapdbs successfully added to Oracle Registry
There are files under the mount path '/snapdata'. Stop the filesystem in order to backup the files.

```
Dessa forma o file system vai subir automaticamente no reboot do host

```
[grid@dbtryh ~]$ crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.DATA.COMMONSTORE.advm
               ONLINE  ONLINE       dbtryh                   STABLE
ora.DATA.VOL_SNAPS.advm
               ONLINE  ONLINE       dbtryh                   STABLE
ora.LISTENER.lsnr
               ONLINE  ONLINE       dbtryh                   STABLE
ora.chad
               ONLINE  ONLINE       dbtryh                   STABLE
ora.data.commonstore.acfs
               ONLINE  ONLINE       dbtryh                   mounted on /opt/orac
                                                             le/dcs/commonstore,S
                                                             TABLE
ora.data.vol_snaps.acfs
               ONLINE  ONLINE       dbtryh                   mounted on /snapdbs,
                                                             STABLE
ora.net1.network
               ONLINE  ONLINE       dbtryh                   STABLE
ora.ons
               ONLINE  ONLINE       dbtryh                   STABLE
ora.proxy_advm
               ONLINE  ONLINE       dbtryh                   STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.ASMNET1LSNR_ASM.lsnr(ora.asmgroup)
      1        ONLINE  ONLINE       dbtryh                   STABLE
ora.DATA.dg(ora.asmgroup)
      1        ONLINE  ONLINE       dbtryh                   STABLE
ora.LISTENER_SCAN1.lsnr
      1        OFFLINE OFFLINE                               STABLE
ora.RECO.dg(ora.asmgroup)
      1        ONLINE  ONLINE       dbtryh                   STABLE
ora.asm(ora.asmgroup)
      1        ONLINE  ONLINE       dbtryh                   Started,STABLE
ora.asmnet1.asmnetwork(ora.asmgroup)
      1        ONLINE  ONLINE       dbtryh                   STABLE
ora.cvu
      1        ONLINE  ONLINE       dbtryh                   STABLE
ora.db0626_jcj_gru.db
      1        ONLINE  OFFLINE      dbtryh                   Instance Shutdown,ST
                                                             ARTING
ora.db0626_jcj_gru.db0626_db0626_pdb1.paas.oracle.com.svc
      1        ONLINE  OFFLINE                               STABLE
ora.dbtryh.vip
      1        ONLINE  ONLINE       dbtryh                   STABLE
ora.qosmserver
      1        ONLINE  ONLINE       dbtryh                   STABLE
ora.scan1.vip
      1        OFFLINE OFFLINE                               STABLE
--------------------------------------------------------------------------------

```

2. Gerar o arquivo com a senha do SYS : 
Criacao do syspwd para depois fazer o uso no SNAP sem ter que informar a senha do SYS  👍 

```

[oracle@dbtryh snapdbs]$  gdbclone syspwf --syspwf /snapdbs/acfs_passwd/password_file

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-03
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│


Please enter the password to encrypt:
Please re-enter the password to encrypt:

SYS password file created as '/snapdbs/acfs_passwd/password_file'

```
Pegando os DBHOMES na maquina : 
```
[oracle@dbtryh snapdbs]$ gdbclone listhomes
INFO: 2025-06-26 23:08:30: Please check the logfile '/opt/gDBClone/out/log/gDBClone_29471.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-03
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│

Oracle Home Name      Home Location
----------------      ------------
OraDB19000_home1      /u01/app/oracle/product/19.0.0.0/dbhome_1

```

Criando os diretorios para o destino dos arquivos (datafiles, redo e archives) no ACFS : 

```
[root@dbtryh snapdbs]# mkdir datastore
[root@dbtryh snapdbs]# mkdir redostore
[root@dbtryh snapdbs]# mkdir recostore
[root@dbtryh snapdbs]# chown -R oracle:oinstall datastore  redostore  recostore
[root@dbtryh snapdbs]# pwd
/snapdbs/gold
[root@dbtryh snapdbs]# ls -lrt
total 272
drwx------ 2 root   root     65536 Jun 26 22:49 lost+found
drwxr-xr-x 2 oracle oinstall 20480 Jun 26 23:06 acfs_passwd
drwxr-xr-x 2 oracle oinstall 20480 Jun 26 23:10 datastore
drwxr-xr-x 2 oracle oinstall 20480 Jun 26 23:10 redostore
drwxr-xr-x 2 oracle oinstall 20480 Jun 26 23:10 recostore
[root@dbtryh snapdbs]# pwd
/snapdbs/gold

```

Quando fui fazer o comando para os diretorios , **_DEU ERRO_**

![Image](https://github.com/user-attachments/assets/de2a3084-a5e6-48c4-88ff-ac7be4956e6a)

Dessa forma, nao temos que especificar o diretorio e sim so o nome do volume ACFS : 


> gdbclone clone -sdbname DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com -sdbscan dbtryh-scan \
> -tdbname GOLD \
> -tdbhome OraDB19000_home1 \
> -dataacfs /snapdbs \
> -redoacfs /snapdbs \
> -recoacfs /snapdbs \
> -syspwf /snapdbs/acfs_passwd/password_file 
> 

Sempre verificar o SERVICE_NAME, se esta com sufixo no output do comando lsnrctl status.
```
[oracle@dbtryh ~]$ gdbclone clone -sdbname DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com -sdbscan dbtryh-scan \
> -tdbname GOLD \
> -tdbhome OraDB19000_home1 \
> -dataacfs /snapdbs \
> -redoacfs /snapdbs \
> -recoacfs /snapdbs \
> --racmod 0 \
> -syspwf /snapdbs/acfs_passwd/password_file
INFO: 2025-06-26 23:20:51: Please check the logfile '/opt/gDBClone/out/log/gDBClone_45678.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-03
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│


MacroStep1 - Getting information and validating setup...
INFO: 2025-06-26 23:20:52: Validating environment
INFO: 2025-06-26 23:20:52: Checking superuser usage
INFO: 2025-06-26 23:20:52: Checking 'GOLD' target database name
INFO: 2025-06-26 23:20:52: Checking target database home 'OraDB19000_home1' existence
INFO: 2025-06-26 23:20:52: Checking if Oracle Restart
INFO: 2025-06-26 23:20:52: Checking ping to host 'dbtryh-scan'
INFO: 2025-06-26 23:20:52: Getting ORACLE_BASE path from orabase
INFO: 2025-06-26 23:20:52: Checking target database 'GOLD' existence
INFO: 2025-06-26 23:20:53: Checking ACFS snapshot 'GOLD' existence under '/snapdbs'
INFO: 2025-06-26 23:20:53: Checking listener on 'dbtryh-scan:1521'
INFO: 2025-06-26 23:20:53: Checking target ACFS storage options...
INFO: 2025-06-26 23:20:53: ...checking if dataacfs '/snapdbs' is an ACFS file system
INFO: 2025-06-26 23:20:53: ...checking if redoacfs '/snapdbs' is an ACFS file system
INFO: 2025-06-26 23:20:53: ...checking if recoacfs '/snapdbs' is an ACFS file system
INFO: 2025-06-26 23:20:53: Getting target database home 'OraDB19000_home1' version...
INFO: 2025-06-26 23:20:56: Checking source and target database version
INFO: 2025-06-26 23:20:57: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' size
INFO: 2025-06-26 23:20:57: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' role
INFO: 2025-06-26 23:20:57: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' log mode
INFO: 2025-06-26 23:20:57: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' flashcache
SUCCESS: 2025-06-26 23:20:58: Environment validation complete

MacroStep2 - Setting up clone environment...
INFO: 2025-06-26 23:20:58: Creating local pfile
INFO: 2025-06-26 23:20:58: Creating local password file
INFO: 2025-06-26 23:20:58: Creating local Audit folder
INFO: 2025-06-26 23:20:58: Checking local auxiliary listener existence
INFO: 2025-06-26 23:20:58: Creating local auxiliary listener configuration file

```


**Com o GDBCLONE ele ja vai restaurando no .ACFS** 

> 
> [root@dbtryh GOLD]# pwd
> /snapdbs/.ACFS/snaps/GOLD
> [root@dbtryh GOLD]#
```
[root@dbtryh GOLD]# pwd
/snapdbs/.ACFS/snaps/GOLD
[root@dbtryh GOLD]# ls -lrt
total 156
drwxrwx--- 2 oracle asmadmin 20480 Jun 26 23:06 acfs_passwd
drwxrwx--- 5 oracle asmadmin 20480 Jun 26 23:14 gold
drwxrwx--- 7 oracle asmadmin 20480 Jun 26 23:23 GOLD
[root@dbtryh GOLD]# cd GOLD/
[root@dbtryh GOLD]# ls
387A0BF19C47F54AE0630D00000A3A9F  387A296B506E2E09E0630D00000A8D4D  datafile  parameterfile  password
(failed reverse-i-search)`du ': c^CGOLD/
[root@dbtryh GOLD]# du -sk *
993396  387A0BF19C47F54AE0630D00000A3A9F
1019012 387A296B506E2E09E0630D00000A8D4D
2392140 datafile
52      parameterfile
52      password
[root@dbtryh GOLD]# cd 387A0BF19C47F54AE0630D00000A3A9F
[root@dbtryh 387A0BF19C47F54AE0630D00000A3A9F]# ls
datafile
[root@dbtryh 387A0BF19C47F54AE0630D00000A3A9F]# cd datafile/
[root@dbtryh datafile]# ls -lrt
total 993292
-rw-r----- 1 oracle asmadmin  52436992 Jun 26 23:24 o1_mf_undotbs1_n5w06gwb_.dbf
-rw-r----- 1 oracle asmadmin 503324672 Jun 26 23:24 o1_mf_system_n5w05z5l_.dbf
-rw-r----- 1 oracle asmadmin 450895872 Jun 26 23:24 o1_mf_sysaux_n5w066w4_.dbf

```

Porem quando o banco tem TDE, TEMOS QUE COPIAR MANUALMENTE  A TDE para o destino : 

ERRO : 

```
Oracle instance shut down
released channel: TGT1
released channel: TGT2
released channel: TGT3
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of Duplicate Db command at 06/26/2025 23:25:22
RMAN-05501: aborting duplication of target database
RMAN-03015: error occurred in stored script Memory Script
ORA-00283: recovery session canceled due to errors
RMAN-11003: failure during parse/execution of SQL statement: alter database recover logfile '/snapdbs/GOLD/GOLD/archivelog/2025_06_26/o1_mf_1_5_n5w07htg_.arc'
ORA-00283: recovery session canceled due to errors
ORA-28365: wallet is not open
RMAN Client Diagnostic Trace file : /u01/app/oracle/diag/clients/user_oracle/RMAN_1893052680_110/trace/ora_rman_47661_0.trc
RMAN Server Diagnostic Trace file : /u01/app/oracle/diag/rdbms/gold/GOLD/trace/GOLD_ora_49775.trc
RMAN Server Diagnostic Trace file : /u01/app/oracle/diag/rdbms/gold/GOLD/trace/GOLD_ora_49775.trc

```



```

[oracle@dbtryh dbs]$ export ORACLE_SID=GOLD
[oracle@dbtryh dbs]$ sqlplus  / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Fri Jun 27 07:37:33 2025
Version 19.27.0.0.0

Copyright (c) 1982, 2024, Oracle.  All rights reserved.

Connected to an idle instance.

SQL> startup nomount
ORACLE instance started.

Total System Global Area 6442447880 bytes
Fixed Size                  9192456 bytes
Variable Size            1107296256 bytes
Database Buffers         5184159744 bytes
Redo Buffers              141799424 bytes
SQL> select  * from v$encryption_wallet;

WRL_TYPE
--------------------
WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS                         WALLET_TYPE          WALLET_OR KEYSTORE FULLY_BAC
------------------------------ -------------------- --------- -------- ---------
    CON_ID
----------
FILE
/opt/oracle/dcs/commonstore/wallets/GOLD/tde/
NOT_AVAILABLE                  UNKNOWN              SINGLE    NONE     UNDEFINED
         1
```

Por isso tem que copiar a TDE ( .p12 e .sso) para esse novo diretorio.


```
[oracle@dbtryh tde]$ cp ewallet.p12 cwallet.sso /opt/oracle/dcs/commonstore/wallets/GOLD/tde/
[oracle@dbtryh tde]$ sqlplus  / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Fri Jun 27 07:40:28 2025
Version 19.27.0.0.0

Copyright (c) 1982, 2024, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c EE High Perf Release 19.0.0.0.0 - Production
Version 19.27.0.0.0

SQL> shut abort
ORACLE instance shut down.
SQL> startup nomount
ORACLE instance started.

Total System Global Area 6442447880 bytes
Fixed Size                  9192456 bytes
Variable Size            1107296256 bytes
Database Buffers         5184159744 bytes
Redo Buffers              141799424 bytes
SQL> select  * from v$encryption_wallet;

WRL_TYPE
--------------------
WRL_PARAMETER
--------------------------------------------------------------------------------
STATUS                         WALLET_TYPE          WALLET_OR KEYSTORE FULLY_BAC
------------------------------ -------------------- --------- -------- ---------
    CON_ID
----------
FILE
/opt/oracle/dcs/commonstore/wallets/GOLD/tde/
OPEN                           AUTOLOGIN            SINGLE    NONE     NO
         1



```

> [oracle@dbtryh ~]$ gdbclone acfslistsnaps --dataacfs /snapdbs


```
[oracle@dbtryh ~]$ gdbclone acfslistsnaps --dataacfs /snapdbs
INFO: 2025-06-27 07:44:53: Please check the logfile '/opt/gDBClone/out/log/gDBClone_36687.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-03
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│


ACFS snapshot list for '/snapdbs':
snapshot name:               GOLD
snapshot location:           /snapdbs/.ACFS/snaps/GOLD
RO snapshot or RW snapshot:  RW
parent name:                 /snapdbs
snapshot creation time:      Thu Jun 26 23:21:58 2025
file entry table allocation: 8650752   (   8.25 MB )
storage added to snapshot:   4519526400   (   4.21 GB )


    number of snapshots:  1
    kilosnap state:       ENABLED
    snapshot space usage: 4519772160  (   4.21 GB )


```


Deletar a "sujeira" : 

> [oracle@dbtryh ~]$ gdbclone acfsdelsnap --snapname GOLD  --dataacfs /snapdbs


```

[oracle@dbtryh ~]$ gdbclone acfsdelsnap --snapname GOLD  --dataacfs /snapdbs
INFO: 2025-06-27 07:46:34: Please check the logfile '/opt/gDBClone/out/log/gDBClone_38433.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-03
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│


You are going to drop ACFS snapshot 'GOLD', are you sure (Y/N)? y
SUCCESS: 2025-06-27 07:46:36: ACFS snapshot 'GOLD' deleted

```

Podemos escolher um DBNAME diferente da ORIGEM. O proprio GDBCLONE altera o dbname e o unique_name.

```
[oracle@dbtryh ~]$ gdbclone clone -sdbname DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com -sdbscan dbtryh-scan -tdbname GOLD -tdbhome OraDB19000_home1 -dataacfs /snapdbs -redoacfs /snapreco -recoacfs /snapdata -syspwf /snapdbs/acfs_passwd/password_file  -force
INFO: 2025-06-27 07:52:21: Please check the logfile '/opt/gDBClone/out/log/gDBClone_44212.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-03
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│


MacroStep1 - Getting information and validating setup...
INFO: 2025-06-27 07:52:22: Validating environment
INFO: 2025-06-27 07:52:22: Checking superuser usage
INFO: 2025-06-27 07:52:22: Checking 'GOLD' target database name
INFO: 2025-06-27 07:52:22: Checking target database home 'OraDB19000_home1' existence
INFO: 2025-06-27 07:52:22: Checking if Oracle Restart
INFO: 2025-06-27 07:52:22: Checking ping to host 'dbtryh-scan'
INFO: 2025-06-27 07:52:22: Getting ORACLE_BASE path from orabase
INFO: 2025-06-27 07:52:22: Checking target database 'GOLD' existence
INFO: 2025-06-27 07:52:23: Checking ACFS snapshot 'GOLD' existence under '/snapdbs'
INFO: 2025-06-27 07:52:23: Checking listener on 'dbtryh-scan:1521'
INFO: 2025-06-27 07:52:23: Checking target ACFS storage options...
INFO: 2025-06-27 07:52:23: ...checking if dataacfs '/snapdbs' is an ACFS file system
INFO: 2025-06-27 07:52:23: ...checking if redoacfs '/snapdbs' is an ACFS file system
INFO: 2025-06-27 07:52:23: ...checking if recoacfs '/snapdbs' is an ACFS file system
INFO: 2025-06-27 07:52:23: Getting target database home 'OraDB19000_home1' version...
INFO: 2025-06-27 07:52:26: Checking source and target database version
INFO: 2025-06-27 07:52:27: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' size
INFO: 2025-06-27 07:52:27: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' role
INFO: 2025-06-27 07:52:27: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' log mode
INFO: 2025-06-27 07:52:28: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' flashcache
SUCCESS: 2025-06-27 07:52:28: Environment validation complete

MacroStep2 - Setting up clone environment...
INFO: 2025-06-27 07:52:28: Creating local pfile
INFO: 2025-06-27 07:52:28: Creating local password file
INFO: 2025-06-27 07:52:28: Creating local Audit folder
INFO: 2025-06-27 07:52:28: Checking local auxiliary listener existence
INFO: 2025-06-27 07:52:28: Stopping local auxiliary listener
INFO: 2025-06-27 07:52:28: Creating local auxiliary listener configuration file
INFO: 2025-06-27 07:52:28: Starting local auxiliary listener
INFO: 2025-06-27 07:52:28: Waiting 60 secs for listener registration, please wait...
 INFO: 2025-06-27 07:53:28: Setting up ACFS storage
INFO: 2025-06-27 07:53:28: Make recovery manager dynamic scripts
INFO: 2025-06-27 07:53:29: Setup clone database to target ACFS from host 'dbtryh-scan'
INFO: 2025-06-27 07:53:29: ...creating recovery manager duplicate script
INFO: 2025-06-27 07:53:29: ...creating recovery manager spfile script
INFO: 2025-06-27 07:53:29: ...creating recovery manager script for spfile target to ACFS
INFO: 2025-06-27 07:53:29: Instantiating clone database
SUCCESS: 2025-06-27 07:53:29: Environment setup complete

MacroStep3 - Cloning database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' as 'GOLD'...
INFO: 2025-06-27 07:53:29: Getting cluster_interconnects values for 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com'
INFO: 2025-06-27 07:53:29: Database clone from active database in progress, please wait...
INFO: 2025-06-27 07:53:29:    (this can take a while depending on database size and/or network speed)
INFO: 2025-06-27 07:53:29:    (rman spool file '/opt/gDBClone/out/log/rman_clone_44212.log')
INFO: 2025-06-27 07:57:11: Create pfile from spfile
INFO: 2025-06-27 07:58:16: Register 'GOLD' database as cluster resource
INFO: 2025-06-27 07:58:17: ...creating password file for 'GOLD'
INFO: 2025-06-27 07:58:17: Register ACFS passwordfile '/snapdbs/.ACFS/snaps/GOLD/GOLD/password/orapwGOLD'
INFO: 2025-06-27 07:58:18: Updating spfile
INFO: 2025-06-27 07:58:20: Updating local pfile
INFO: 2025-06-27 07:58:20: Stopping database 'GOLD'
INFO: 2025-06-27 07:58:20: Modifying DB instance
INFO: 2025-06-27 07:58:21: Setup ACFS dependency
INFO: 2025-06-27 07:58:22: Starting database 'GOLD'
SUCCESS: 2025-06-27 07:58:33: Clone database 'GOLD' created successfully

INFO: 2025-06-27 07:58:33: Starting database 'GOLD'
SUCCESS: 2025-06-27 07:58:34: Successfully created clone database 'GOLD'


[oracle@raia2host ~]$ gdbclone clone --sdbname DB0605_crh_gru.subnetapp.maquina.oraclevcn.com   -sdbscan raiahost-scan.subnetapp.maquina.oraclevcn.com  -tdbname  GOLD4  -tdbhome OraDB19000_home1 -dataacfs /acfs_gdb --sga_max_size 4096  --sga_target 4096
INFO: 2025-06-06 15:24:32: Please check the logfile '/opt/gDBClone/out/log/gDBClone_12323.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-02
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│


MacroStep1 - Getting information and validating setup...

Please enter the 'SYS' for the database DB0605_crh_gru.subnetapp.maquina.oraclevcn.com:
Please re-enter the 'SYS' for the database DB0605_crh_gru.subnetapp.maquina.oraclevcn.com:
INFO: 2025-06-06 15:24:40: Validating environment
INFO: 2025-06-06 15:24:40: Checking superuser usage
INFO: 2025-06-06 15:24:40: Checking 'GOLD4' target database name
INFO: 2025-06-06 15:24:40: Checking target database home 'OraDB19000_home1' existence
INFO: 2025-06-06 15:24:40: Checking if Oracle Restart
INFO: 2025-06-06 15:24:40: Checking ping to host 'raiahost-scan.subnetapp.maquina.oraclevcn.com'
INFO: 2025-06-06 15:24:40: Getting ORACLE_BASE path from orabase
INFO: 2025-06-06 15:24:40: Checking target database 'GOLD4' existence
INFO: 2025-06-06 15:24:41: Checking ACFS snapshot 'GOLD4' existence under '/acfs_gdb'
INFO: 2025-06-06 15:24:41: Checking listener on 'raia2host-scan:1521'
INFO: 2025-06-06 15:24:41: Checking target ACFS storage options...
INFO: 2025-06-06 15:24:41: ...checking if dataacfs '/acfs_gdb' is an ACFS file system
INFO: 2025-06-06 15:24:41: ...redoacfs set to '/acfs_gdb'
INFO: 2025-06-06 15:24:41: ...recoacfs set to '/acfs_gdb'
INFO: 2025-06-06 15:24:41: Getting target database home 'OraDB19000_home1' version...
INFO: 2025-06-06 15:24:45: Checking source and target database version
INFO: 2025-06-06 15:24:45: Checking source database 'DB0605_crh_gru.subnetapp.maquina.oraclevcn.com' size
INFO: 2025-06-06 15:24:46: Checking source database 'DB0605_crh_gru.subnetapp.maquina.oraclevcn.com' role
INFO: 2025-06-06 15:24:46: Checking source database 'DB0605_crh_gru.subnetapp.maquina.oraclevcn.com' log mode
INFO: 2025-06-06 15:24:46: Checking source database 'DB0605_crh_gru.subnetapp.maquina.oraclevcn.com' flashcache
SUCCESS: 2025-06-06 15:24:47: Environment validation complete

MacroStep2 - Setting up clone environment...
INFO: 2025-06-06 15:24:47: Creating local pfile
INFO: 2025-06-06 15:24:47: Creating local password file
INFO: 2025-06-06 15:24:47: Creating local Audit folder
INFO: 2025-06-06 15:24:47: Checking local auxiliary listener existence
INFO: 2025-06-06 15:24:47: Stopping local auxiliary listener
INFO: 2025-06-06 15:24:47: Creating local auxiliary listener configuration file
INFO: 2025-06-06 15:24:47: Starting local auxiliary listener
INFO: 2025-06-06 15:24:47: Waiting 60 secs for listener registration, please wait...
INFO: 2025-06-06 15:25:47: Setting up ACFS storage
INFO: 2025-06-06 15:25:47: Make recovery manager dynamic scripts
INFO: 2025-06-06 15:25:48: Setup clone database to target ACFS from host 'raiahost-scan.subnetapp.maquina.oraclevcn.com'
INFO: 2025-06-06 15:25:48: ...creating recovery manager duplicate script
INFO: 2025-06-06 15:25:48: ...creating recovery manager spfile script
INFO: 2025-06-06 15:25:48: ...creating recovery manager script for spfile target to ACFS
INFO: 2025-06-06 15:25:48: Instantiating clone database
SUCCESS: 2025-06-06 15:25:48: Environment setup complete

MacroStep3 - Cloning database 'DB0605_crh_gru.subnetapp.maquina.oraclevcn.com' as 'GOLD4'...
INFO: 2025-06-06 15:25:48: Getting cluster_interconnects values for 'DB0605_crh_gru.subnetapp.maquina.oraclevcn.com'
INFO: 2025-06-06 15:25:48: Database clone from active database in progress, please wait...
INFO: 2025-06-06 15:25:48:    (this can take a while depending on database size and/or network speed)
INFO: 2025-06-06 15:25:48:    (rman spool file '/opt/gDBClone/out/log/rman_clone_12323.log')
INFO: 2025-06-06 15:29:05: Create pfile from spfile
INFO: 2025-06-06 15:30:06: Register 'GOLD4' database as cluster resource
INFO: 2025-06-06 15:30:08: ...creating password file for 'GOLD4'
INFO: 2025-06-06 15:30:08: Register ACFS passwordfile '/acfs_gdb/.ACFS/snaps/GOLD4/GOLD4/password/orapwGOLD4'
INFO: 2025-06-06 15:30:08: Updating spfile
INFO: 2025-06-06 15:30:11: Updating local pfile
INFO: 2025-06-06 15:30:11: Stopping database 'GOLD4'
INFO: 2025-06-06 15:30:11: Modifying DB instance
INFO: 2025-06-06 15:30:12: Setup ACFS dependency
INFO: 2025-06-06 15:30:13: Starting database 'GOLD4'
SUCCESS: 2025-06-06 15:30:26: Clone database 'GOLD4' created successfully

INFO: 2025-06-06 15:30:26: Starting database 'GOLD4'
SUCCESS: 2025-06-06 15:30:28: Successfully created clone database 'GOLD4'
[oracle@raia2host ~]$
```

FAZENDO AGORA PARA UM BANCO STANDBY : 

> [oracle@dbtryh ~]$ gdbclone clone -sdbname DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com -sdbscan dbtryh-scan -tdbname GOLD -tdbhome OraDB19000_home1 -dataacfs /snapdbs -redoacfs /snapdbs -recoacfs /snapdbs -syspwf /snapdbs/acfs_passwd/password_file   --standby  --pmode maxperf

```
[oracle@dbtryh ~]$ gdbclone clone -sdbname DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com -sdbscan dbtryh-scan -tdbname GOLD -tdbhome OraDB19000_home1 -dataacfs /snapdbs -redoacfs /snapdbs -recoacfs /snapdbs -syspwf /snapdbs/acfs_passwd/password_file   --standby  --pmode maxperf
INFO: 2025-06-27 08:11:56: Please check the logfile '/opt/gDBClone/out/log/gDBClone_74520.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-03
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│


MacroStep1 - Getting information and validating setup...
INFO: 2025-06-27 08:11:56: Validating environment
INFO: 2025-06-27 08:11:56: Checking superuser usage
INFO: 2025-06-27 08:11:56: Checking 'GOLD' target database name
INFO: 2025-06-27 08:11:56: Checking target database home 'OraDB19000_home1' existence
INFO: 2025-06-27 08:11:56: Checking if Oracle Restart
INFO: 2025-06-27 08:11:56: Checking ping to host 'dbtryh-scan'
INFO: 2025-06-27 08:11:56: Getting ORACLE_BASE path from orabase
INFO: 2025-06-27 08:11:56: Checking target database 'GOLD' existence
INFO: 2025-06-27 08:11:57: Checking ACFS snapshot 'GOLD' existence under '/snapdbs'
INFO: 2025-06-27 08:11:58: Checking listener on 'dbtryh-scan:1521'
INFO: 2025-06-27 08:11:58: Checking target ACFS storage options...
INFO: 2025-06-27 08:11:58: ...checking if dataacfs '/snapdbs' is an ACFS file system
INFO: 2025-06-27 08:11:58: ...checking if redoacfs '/snapdbs' is an ACFS file system
INFO: 2025-06-27 08:11:58: ...checking if recoacfs '/snapdbs' is an ACFS file system
INFO: 2025-06-27 08:11:58: Getting target database home 'OraDB19000_home1' version...
INFO: 2025-06-27 08:12:01: Checking source and target database version
INFO: 2025-06-27 08:12:01: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' size
INFO: 2025-06-27 08:12:01: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' role
INFO: 2025-06-27 08:12:02: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' log mode
INFO: 2025-06-27 08:12:02: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' Force Logging mode
INFO: 2025-06-27 08:12:02: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' 'log_archive_dest' settings
INFO: 2025-06-27 08:12:03: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' Flashback mode
WARN: 2025-06-27 08:12:03: Source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' is not in Flashback mode
SUCCESS: 2025-06-27 08:12:03: Environment validation complete

MacroStep2 - Setting up clone environment...
INFO: 2025-06-27 08:12:03: Creating local pfile
INFO: 2025-06-27 08:12:03: Creating local password file
INFO: 2025-06-27 08:12:03: Creating local Audit folder
INFO: 2025-06-27 08:12:03: Checking local auxiliary listener existence
INFO: 2025-06-27 08:12:03: Creating local auxiliary listener configuration file
INFO: 2025-06-27 08:12:03: Starting local auxiliary listener
INFO: 2025-06-27 08:12:03: Waiting 60 secs for listener registration, please wait...
INFO: 2025-06-27 08:13:03: Setting up ACFS storage
INFO: 2025-06-27 08:13:03: Make recovery manager dynamic scripts
INFO: 2025-06-27 08:13:04: Setup clone database to target ACFS from host 'dbtryh-scan' as standby database
INFO: 2025-06-27 08:13:04: ...creating recovery manager duplicate script
INFO: 2025-06-27 08:13:05: ...creating recovery manager spfile script
INFO: 2025-06-27 08:13:05: ...creating recovery manager script for spfile target to ACFS
INFO: 2025-06-27 08:13:05: Instantiating standby database
INFO: 2025-06-27 08:13:05: ...getting standby logs on source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com'
INFO: 2025-06-27 08:13:05: ...getting MAX redologs group on source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com'
INFO: 2025-06-27 08:13:05: ...getting THREAD# and redolog size on source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com'
INFO: 2025-06-27 08:13:06: ...creating standby logs on source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com'
SUCCESS: 2025-06-27 08:13:40: Environment setup complete

MacroStep3 - Cloning database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' as 'GOLD'...
INFO: 2025-06-27 08:13:40: Getting cluster_interconnects values for 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com'
INFO: 2025-06-27 08:13:40: Database clone from active database in progress, please wait...
INFO: 2025-06-27 08:13:40:    (this can take a while depending on database size and/or network speed)
INFO: 2025-06-27 08:13:40:    (rman spool file '/opt/gDBClone/out/log/rman_clone_74520.log')
INFO: 2025-06-27 08:17:10: Create pfile from spfile
INFO: 2025-06-27 08:17:40: Register 'GOLD' database as cluster resource
INFO: 2025-06-27 08:17:41: Register ACFS passwordfile '/snapdbs/.ACFS/snaps/GOLD/GOLD/password/orapwGOLD'
INFO: 2025-06-27 08:17:42: Updating spfile
INFO: 2025-06-27 08:17:44: Updating local pfile
INFO: 2025-06-27 08:17:44: Stopping database 'GOLD'
INFO: 2025-06-27 08:17:44: Modifying DB instance
INFO: 2025-06-27 08:17:45: Setup ACFS dependency
SUCCESS: 2025-06-27 08:17:46: Clone database 'GOLD' created successfully

MacroStep4 - Standby setup...
INFO: 2025-06-27 08:17:46: Restarting database 'GOLD'
INFO: 2025-06-27 08:17:56: Enabling standby file management for database 'GOLD'
INFO: 2025-06-27 08:17:56: Checking source database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' DG Broker configuration
INFO: 2025-06-27 08:17:57: Starting redo apply
INFO: 2025-06-27 08:18:03: Configuring primary database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com'
SUCCESS: 2025-06-27 08:18:06: Successfully created clone "GOLD" database as standby

```



**10.. Ao final do gdbclone clone, ele coloca o database no CRS como um resource.**

Se falhar ele nao registrara e para fazer o CLEAN UP sera necessario remover na mao no filesystem do ACFS.


```

Esse clone ele faz, de forma default, um DUPLICATE FROM ACTIVE

```
[root@dbtryh ~]# cat /opt/gDBClone/out/conf/rman_dup_74520.scr
spool log to /opt/gDBClone/out/log/rman_clone_74520.log
STARTUP CLONE NOMOUNT;
CONFIGURE COMPRESSION ALGORITHM CLEAR;
run {
ALLOCATE CHANNEL TGT1 DEVICE TYPE DISK;
ALLOCATE AUXILIARY CHANNEL AUX1 DEVICE TYPE DISK;
ALLOCATE CHANNEL TGT2 DEVICE TYPE DISK;
ALLOCATE AUXILIARY CHANNEL AUX2 DEVICE TYPE DISK;
ALLOCATE CHANNEL TGT3 DEVICE TYPE DISK;
ALLOCATE AUXILIARY CHANNEL AUX3 DEVICE TYPE DISK;
SET NEWNAME FOR DATABASE TO NEW;
DUPLICATE TARGET DATABASE
  FOR STANDBY
  FROM ACTIVE DATABASE
  DORECOVER
  SPFILE
    PARAMETER_VALUE_CONVERT='DB0626_jcj_gru','GOLD'
    SET CONTROL_FILES='/snapdbs/GOLD/GOLD/controlfile/control01.ctl'
    SET DB_CREATE_FILE_DEST="/snapdbs/.ACFS/snaps/GOLD"
    SET DB_RECOVERY_FILE_DEST="/snapdbs/GOLD"
    SET DB_CREATE_ONLINE_LOG_DEST_1="/snapdbs/GOLD"
    SET FILESYSTEMIO_OPTIONS='setall'
    SET DB_RECOVERY_FILE_DEST_SIZE='10G'
    SET SERVICE_NAMES=""
    SET DB_CREATE_ONLINE_LOG_DEST_2=""
    SET DB_CREATE_ONLINE_LOG_DEST_3=""
    SET DB_CREATE_ONLINE_LOG_DEST_4=""
    SET DB_CREATE_ONLINE_LOG_DEST_5=""
    SET LOG_ARCHIVE_DEST_2=""
    SET LOG_ARCHIVE_DEST_3=""
    SET LOG_ARCHIVE_DEST_4=""
    SET LOG_ARCHIVE_DEST_5=""
    SET LOG_ARCHIVE_FORMAT='%r_%s_%t.arc'
    SET LOG_ARCHIVE_CONFIG="DG_CONFIG=(DB0626_jcj_gru, GOLD)"
    SET DB_UNIQUE_NAME="GOLD"
    SET DB_NAME="DB0626"
    SET CLUSTER_DATABASE='false'
    SET LOG_ARCHIVE_DEST_2='service=dbtryh-scan:1521/DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com NOAFFIRM ASYNC REGISTER VALID_FOR=(online_logfile,primary_role) DB_UNIQUE_NAME=DB0626_jcj_gru'
    SET FAL_SERVER="dbtryh-scan:1521/DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com"
    SET REMOTE_LISTENER="dbtryh-scan:1521"
    SET LOCAL_LISTENER=''
    SET DIAGNOSTIC_DEST="/u01/app/oracle"
    SET AUDIT_FILE_DEST="/u01/app/oracle/admin/GOLD/adump"
    SET DB_DOMAIN=''
    SET LISTENER_NETWORKS=''
  NOFILENAMECHECK;
}

### **FAZENDO AGORA PARA UM BANCO STANDBY A PARTIR DO BACKUP no FILE SYSTEM :** 

Lembre-se que o script do rman deve conter :  

```
Note1: for a clone as standby from backup-based duplication you must have the "controlfile for standby" included in the backup:
example:

RMAN> RUN
{
ALLOCATE CHANNEL disk1 DEVICE TYPE DISK FORMAT '/mnt/backup/ORCL/%U';
BACKUP DATABASE;
BACKUP AS COPY CURRENT CONTROLFILE FORMAT '/mnt/backup/ORCL/control_%U';
BACKUP CURRENT CONTROLFILE FOR STANDBY FORMAT '/mnt/backup/ORCL/standby_control_%U';
BACKUP SPFILE FORMAT '/mnt/backup/ORCL/spfile_%U';
SQL 'ALTER SYSTEM ARCHIVE LOG CURRENT';
BACKUP ARCHIVELOG ALL;
}
Note2: With backup-based duplication you must copy the password file used on the primary to the standby, for Oracle Data Guard to ship logs.

```

Apos o backup estar no ACFS ou num NFS, pode ser criar um banco MASTER PRIMARY OU STANDBY : 

**PRIMARY**

> [oracle@dbtryh backup]$ gdbclone clone -sdbname DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com -sbckloc /snapdbs/backup/ -tdbname GOLD -tdbhome OraDB19000_home1 -dataacfs /snapdbs -redoacfs /snapdbs -recoacfs /snapdbs -syspwf /snapdbs/acfs_passwd/password_file


```
[oracle@dbtryh backup]$ gdbclone clone -sdbname DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com -sbckloc /snapdbs/backup/ -tdbname GOLD -tdbhome OraDB19000_home1 -dataacfs /snapdbs -redoacfs /snapdbs -recoacfs /snapdbs -syspwf /snapdbs/acfs_passwd/password_file
INFO: 2025-07-01 08:51:09: Please check the logfile '/opt/gDBClone/out/log/gDBClone_31227.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-03
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│


MacroStep1 - Getting information and validating setup...
INFO: 2025-07-01 08:51:10: Validating environment
INFO: 2025-07-01 08:51:10: Checking superuser usage
INFO: 2025-07-01 08:51:10: Checking 'GOLD' target database name
INFO: 2025-07-01 08:51:10: Checking target database home 'OraDB19000_home1' existence
INFO: 2025-07-01 08:51:10: Checking if Oracle Restart
INFO: 2025-07-01 08:51:10: Checking source backup location /snapdbs/backup/...
INFO: 2025-07-01 08:51:10: Getting ORACLE_BASE path from orabase
INFO: 2025-07-01 08:51:10: Checking target database 'GOLD' existence
INFO: 2025-07-01 08:51:11: Checking ACFS snapshot 'GOLD' existence under '/snapdbs'
INFO: 2025-07-01 08:51:11: Checking listener on 'dbtryh-scan:1521'
INFO: 2025-07-01 08:51:11: Checking target ACFS storage options...
INFO: 2025-07-01 08:51:11: ...checking if dataacfs '/snapdbs' is an ACFS file system
INFO: 2025-07-01 08:51:11: ...checking if redoacfs '/snapdbs' is an ACFS file system
INFO: 2025-07-01 08:51:12: ...checking if recoacfs '/snapdbs' is an ACFS file system
INFO: 2025-07-01 08:51:12: Getting target database home 'OraDB19000_home1' version...
SUCCESS: 2025-07-01 08:51:15: Environment validation complete

MacroStep2 - Setting up clone environment...
INFO: 2025-07-01 08:51:15: Creating local pfile
INFO: 2025-07-01 08:51:15: Creating local password file
INFO: 2025-07-01 08:51:15: Creating local Audit folder
INFO: 2025-07-01 08:51:15: Checking local auxiliary listener existence
INFO: 2025-07-01 08:51:15: Creating local auxiliary listener configuration file
INFO: 2025-07-01 08:51:15: Starting local auxiliary listener
INFO: 2025-07-01 08:51:15: Waiting 60 secs for listener registration, please wait...
INFO: 2025-07-01 08:52:15: Setting up ACFS storage
INFO: 2025-07-01 08:52:15: Make recovery manager dynamic scripts
INFO: 2025-07-01 08:52:16: Setup clone database to target ACFS from backup location '/snapdbs/backup/'
INFO: 2025-07-01 08:52:16: ...creating recovery manager duplicate script
INFO: 2025-07-01 08:52:16: ...creating recovery manager spfile script
INFO: 2025-07-01 08:52:16: ...creating recovery manager script for spfile target to ACFS
INFO: 2025-07-01 08:52:16: Instantiating clone database
SUCCESS: 2025-07-01 08:52:16: Environment setup complete

MacroStep3 - Cloning database 'DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com' as 'GOLD'...
INFO: 2025-07-01 08:52:16: Database clone from backup in progress, please wait...
INFO: 2025-07-01 08:52:16:    (this can take a while depending on database size and/or network speed)
INFO: 2025-07-01 08:52:16:    (rman spool file '/opt/gDBClone/out/log/rman_clone_31227.log')
INFO: 2025-07-01 08:55:57: Create pfile from spfile
INFO: 2025-07-01 08:57:04: Register 'GOLD' database as cluster resource
INFO: 2025-07-01 08:57:06: ...creating password file for 'GOLD'
INFO: 2025-07-01 08:57:06: Register ACFS passwordfile '/snapdbs/.ACFS/snaps/GOLD/GOLD/password/orapwGOLD'
INFO: 2025-07-01 08:57:07: Updating spfile
INFO: 2025-07-01 08:57:09: Updating local pfile
INFO: 2025-07-01 08:57:09: Stopping database 'GOLD'
INFO: 2025-07-01 08:57:09: Modifying DB instance
INFO: 2025-07-01 08:57:10: Setup ACFS dependency
INFO: 2025-07-01 08:57:11: Starting database 'GOLD'
SUCCESS: 2025-07-01 08:57:24: Clone database 'GOLD' created successfully

INFO: 2025-07-01 08:57:24: Starting database 'GOLD'
SUCCESS: 2025-07-01 08:57:25: Successfully created clone database 'GOLD'
```

Se quiser criar esse banco como standby, devera colocar a opcao do -sdbscan/-sconstring, **SENAO DARA ERRO** :

```
[oracle@dbtryh backup]$ gdbclone clone -sdbname DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com -sbckloc /snapdbs/backup/ -tdbname GOLD -tdbhome OraDB19000_home1 -dataacfs /snapdbs -redoacfs /snapdbs -recoacfs /snapdbs -syspwf /snapdbs/acfs_passwd/password_file   --standby  --pmode maxperf
INFO: 2025-07-01 08:49:39: Please check the logfile '/opt/gDBClone/out/log/gDBClone_29718.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-03
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│

Clonig as standby database from backup location needs source '-sdbscan/-sconstring!

```
Agora com a chamada correta apontantando para o scan name do ambiente de origem : 

`[oracle@dbtryh ~]$ gdbclone clone -sdbname DB0626_jcj_gru.subnetapp.maquina.oraclevcn.com -sbckloc /snapdbs/backup/ -tdbname GOLD -tdbhome OraDB19000_home1 -dataacfs /snapdbs -redoacfs /snapdbs -recoacfs /snapdbs -syspwf /snapdbs/acfs_passwd/password_file     --standby  --pmode maxperf -sdbscan dbtryh-scan.subnetapp.maquina.oraclevcn.com
`

 e o check ao final ....

> [oracle@dbtryh ~]$ gdbclone listdbs

```

[oracle@dbtryh ~]$ gdbclone listdbs
INFO: 2025-07-02 09:45:57: Please check the logfile '/opt/gDBClone/out/log/gDBClone_47810.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-03
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│

Database Name    Database Type   Database Role      Master/Snapshot  Location/Parent
-------------    -------------   -------------      ---------------  ---------------
GOLD             SINGLE          PHYSICAL_STANDBY   Master           /snapdbs/.ACFS/snaps/
DB0626_jcj_gru   SINGLE          PRIMARY            n/a              ASM

```

[oracle@raia2host ~]$ gdbclone deldb --tdbname GOLD3
INFO: 2025-06-06 17:02:58: Please check the logfile '/opt/gDBClone/out/log/gDBClone_81429.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-02
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│


You are going to drop the database 'GOLD3', are you sure (Y/N)? y
The database 'GOLD3' is not defined as cluster resource.

```


É necessario nao so remover as pastas do diretorio padrao do ACFS e sim nas subpastas do .SNAP


![Image](https://github.com/user-attachments/assets/2b0c414c-cc6f-4080-85db-909cf5f6410d)
**_*Mesmo removendo o espaco ocupado continua o mesmo._**




O espaco mesmo esta no .ACFS

![Image](https://github.com/user-attachments/assets/cec33581-7d1a-4888-8534-5e130a804b10)


Se tentar remover via SO ( rm -rf ) vai tomar erro : 

![Image](https://github.com/user-attachments/assets/509b2921-d2ee-4583-86dd-e48d19e93f8a)






```
[oracle@raia2host ~]$ /sbin/acfsutil snap info   -h
Usage: acfsutil [-h] snap info [-t] [<snap_name>] <mountpoint>
                              - get information about snapshots
                [-t]          - display family tree starting at next name given
                [<snap_name>] - snapshot name
                <mountpoint>  - mount point
[oracle@raia2host ~]$ /sbin/acfsutil snap info   /acfs_gdb
snapshot name:               GOLD1
snapshot location:           /acfs_gdb/.ACFS/snaps/GOLD1
RO snapshot or RW snapshot:  RW
parent name:                 /acfs_gdb
snapshot creation time:      Fri Jun  6 14:00:26 2025
file entry table allocation: 262144   ( 256.00 KB )
storage added to snapshot:   294912   ( 288.00 KB )


snapshot name:               GOLD2
snapshot location:           /acfs_gdb/.ACFS/snaps/GOLD2
RO snapshot or RW snapshot:  RW
parent name:                 /acfs_gdb
snapshot creation time:      Fri Jun  6 14:12:26 2025
file entry table allocation: 8650752   (   8.25 MB )
storage added to snapshot:   8683520   (   8.28 MB )


snapshot name:               GOLD3
snapshot location:           /acfs_gdb/.ACFS/snaps/GOLD3
RO snapshot or RW snapshot:  RW
parent name:                 /acfs_gdb
snapshot creation time:      Fri Jun  6 14:39:42 2025
file entry table allocation: 8650752   (   8.25 MB )
storage added to snapshot:   4490436608   (   4.18 GB )


    number of snapshots:  3
    kilosnap state:       ENABLED
    snapshot space usage: 4518805504  (   4.21 GB )

```


Com o usuario ORACLE, ocore o erro : 

```
[oracle@raia2host ~]$ /sbin/acfsutil snap delete
acfsutil snap delete: ACFS-00535: insufficient arguments
Usage: acfsutil [-h] snap delete <snap_name> <mountpoint> - delete a file system snapshot
[oracle@raia2host ~]$ /sbin/acfsutil snap delete GOLD1  /acfs_gdb
acfsutil snap delete: CLSU-00107: operating system function: ioctl; failed with error data: 1; at location: OI_0
acfsutil snap delete: CLSU-00101: operating system error message: Operation not permitted
acfsutil snap delete: ACFS-03046: unable to perform snapshot operation on /acfs_gdb
[oracle@raia2host ~]$ exit
logout


- ### ALGUNS INSIGHTS & CASES : 


- CASO 1 ) Nesses case o cliente tinha um EXADB-D (PRIMARY-PROD) em @AZURE e o ambiente EXADB-D( STANDBY - NAO PROD) em @GOOGLE E com uma base de 200 TB . 

- [ ]               O step do CLONE fez 20TB/Hora ( rman duplicate) com 48 canais 
- [ ]               Havia 1 tablespace em READ-ONLY, o que gerou um erro ao fazer o snapshot standby devido a criar um novo CONTROLFILE no processo e gerar uma nova incarnation, pois como o header de um datafile READ ONLY, nao permite alteração, não é possivel fazer . O workaround foi fazer de fato colocar a tablespace em RW para que o processo fosse concluido ou incluir uma trigger de AFTER STARTUP.

- CASO 2 ) Com o GDBCLONE pode-se fazer um snap de um snap, portanto o cliente fez o masking de objetos no SNAP1 e  a partir dele foram realizados outros snaps com o masking nas tabelas.



```


Entao tem que ser com o grid : 

[opc@raia2host ~]$ sudo su - grid
Last login: Fri Jun  6 17:10:09 UTC 2025
[grid@raia2host ~]$  /sbin/acfsutil snap delete GOLD1  /acfs_gdb
acfsutil snap delete: Snapshot operation is complete.
[grid@raia2host ~]$  /sbin/acfsutil snap delete GOLD2  /acfs_gdb
acfsutil snap delete: Snapshot operation is complete.
```

Pode ser feito via gdbclone 👍 

![Image](https://github.com/user-attachments/assets/6d83db1f-1bae-458d-b193-4447825a83a4)

```
[oracle@raia2host ~]$ gdbclone acfslistsnaps --dataacfs /acfs_gdb
INFO: 2025-06-06 17:43:04: Please check the logfile '/opt/gDBClone/out/log/gDBClone_9162.log' for more details

│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│
 gDBClone - Version: 4.0.0-02
 Copyright (c) 2012-2025 Oracle and/or its affiliates.
 --------------------------------------------------------
 Author: Ruggero Citton <ruggero.citton@oracle.com>
 RAC Pack, Cloud Innovation and Solution Engineering Team
│▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒│


ACFS snapshot list for '/acfs_gdb':
snapshot name:               GOLD4
snapshot location:           /acfs_gdb/.ACFS/snaps/GOLD4
RO snapshot or RW snapshot:  RW
parent name:                 /acfs_gdb
snapshot creation time:      Fri Jun  6 17:23:57 2025
file entry table allocation: 8650752   (   8.25 MB )
storage added to snapshot:   9379840   (   8.95 MB )


snapshot name:               GOLD5
snapshot location:           /acfs_gdb/.ACFS/snaps/GOLD5
RO snapshot or RW snapshot:  RW
parent name:                 /acfs_gdb
snapshot creation time:      Fri Jun  6 17:28:31 2025
file entry table allocation: 8650752   (   8.25 MB )
storage added to snapshot:   9433088   (   8.99 MB )


snapshot name:               GOLD3
snapshot location:           /acfs_gdb/.ACFS/snaps/GOLD3
RO snapshot or RW snapshot:  RW
parent name:                 /acfs_gdb
snapshot creation time:      Fri Jun  6 14:39:42 2025
file entry table allocation: 8650752   (   8.25 MB )
storage added to snapshot:   4490436608   (   4.18 GB )


    number of snapshots:  3
    kilosnap state:       ENABLED
    snapshot space usage: 4528607232  (   4.22 GB )

```

