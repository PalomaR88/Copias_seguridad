# Sistema de copias de seguridad
Implementar un sistema de copias de seguridad para las instancias del cloud, teniendo en cuenta las siguientes características:
- Selecciona una aplicación para realizar el proceso: bacula, amanda, shell script con tar, rsync, dar, afio, etc.
- Utiliza una de las instancias como servidor de copias de seguridad, añadiéndole un volumen y almacenando localmente las copias de seguridad que consideres adecuadas en él.
- El proceso debe realizarse de forma completamente automática
- Selecciona qué información es necesaria guardar (listado de paquetes, ficheros de configuración, documentos, datos, etc.)
- Realiza semanalmente una copia completa
- Realiza diariamente una copia incremental o diferencial (decidir cual es más adecuada)
- Implementa una planificación del almacenamiento de copias de seguridad para una ejecución prevista de varios años, detallando qué copias completas se almacenarán de forma permanente y cuales se irán borrando
- Añade tu sistema de copias a coconut cuando esté disponible
- Selecciona un directorio de datos "críticos" que deberá almacenarse cifrado en la copia de seguridad, bien encargándote de hacer la copia manualmente o incluyendo la contraseña de cifrado en el sistema
- Incluye en la copia los datos de las nuevas aplicaciones que se vayan instalando durante el resto del curso
- Utiliza saturno u otra opción que se te facilite como equipo secundario para almacenar las copias de seguridad. Solicita acceso o la instalación de las aplicaciones que sean precisas.

La corrección consistirá tanto en la restauración puntual de un fichero en cualquier fecha como la restauración completa de una de las instancias la última semana de curso.

### Instancia como servidor de copias de seguridad
Se utilizará una instancia con sistema operativo Ubuntu, tortilla, que actuará como servidor para bacula a la que posteriormente se asociará un volumen donde se guardarán las copias de seguridad que se realizan de las 3 máquina, salmorejo, croqueta y tortilla.

### Programación de las copias
Se ha decidido que se realizarán tres tipos de copias:
- Copias diarias e incrementales, que se borrarán a los 14 días.
- Copias semanales y completas, que se borrarán al mes.
- Copias mensuales y completas, que se borrarán al año.

Como se ha decidido borrar las parciales por día se va a utilizar la opción de copias incrementales ya que para la opción de copias diferenciales se necesita guardar todas las diferenciales que se hayan hecho desde el último full. 

### Aplicación Bacula
Para esta práctica se va a emplear la herramienta de copias de seguridad **Bacula** que funciona bajo una arquitectura cliente-servidor.

Para la instalación de Bacula es necesaria un gestor de base de datos, en nuestro caso se va a utilizar **MariaDB**. 

#### Instalación de Bacula
Para la instalación son necesarios los siguientes paquetes de Bacula:
~~~
ubuntu@tortilla:~$ sudo apt install bacula bacula-client bacula-common-mysql bacula-director-mysql bacula-server
~~~

En el proceso de instalación de **bacula-director-mysql** se pregunta si se quiere configurar la base de datos para bacula y su configuración.
~~~
 ┌────────────────┤ Configuring bacula-director-mysql ├────────────────┐
 │                                                                     │ 
 │ The bacula-director-mysql package must have a database installed    │ 
 │ and configured before it can be used. This can be optionally        │ 
 │ handled with dbconfig-common.                                       │ 
 │                                                                     │ 
 │ If you are an advanced database administrator and know that you     │ 
 │ want to perform this configuration manually, or if your database    │ 
 │ has already been installed and configured, you should refuse this   │ 
 │ option. Details on what needs to be done should most likely be      │ 
 │ provided in /usr/share/doc/bacula-director-mysql.                   │ 
 │                                                                     │ 
 │ Otherwise, you should probably choose this option.                  │ 
 │                                                                     │ 
 │ Configure database for bacula-director-mysql with dbconfig-common?  │ 
 │                                                                     │ 
 │                  <Yes>                     <No>                     │ 
 │                                                                     │ 
 └─────────────────────────────────────────────────────────────────────┘ 
  ┌────────────────┤ Configuring bacula-director-mysql ├────────────────┐
 │ Please provide a password for bacula-director-mysql to register     │ 
 │ with the database server. If left blank, a random password will be  │ 
 │ generated.                                                          │ 
 │                                                                     │ 
 │ MySQL application password for bacula-director-mysql:               │ 
 │                                                                     │ 
 │ ******_____________________________________________________________ │ 
 │                                                                     │ 
 │                  <Ok>                      <Cancel>                 │ 
 │                                                                     │ 
 └─────────────────────────────────────────────────────────────────────┘ 
 ~~~


#### Configuración del fichero /etc/bacula/bacula-dir.conf
En el fichero **/etc/bacula/bacula-dir.conf** se indica los parámetros más importantes para las copias a través de los siguientes apartados:
- Director: Indica los atributos de la máquina director
~~~
Director {
  Name = serranito-dir #Nombre que va a user el administrador del sistema.
  DIRport = 9101 #Puerto
  QueryFile = "/etc/bacula/scripts/query.sql" #Ruta donde se encuentran las sentendias SQL
  WorkingDirectory = "/var/lib/bacula" #Ruta donde el director coloca sus ficheros de estado.
  PidDirectory = "/run/bacula" #Ruta donde el director coloca sus fichero Id.
  Maximum Concurrent Jobs = 20 #Número máximo de trabajos a la vez
  Password = "bacula" #Contraseña
  Messages = Daemon #Indica el recurso mensaje, nombre de otro apartado
  DirAddress = 10.0.0.10 #Dirección del director
}
~~~

- Job: definición de los trabajos. Se configurarán un apartado para cada máquina y tipo de copia, y además un apartado para cada máquina para los recursos.
~~~
Job {
 Name = "Backup-dia-Serranito" #Nombre del trabajo
 Client = "serranito-fd" #Nombre del cliente sobre el que ejecutar el trabajo
 Type = Backup #Tipo: Backup, Restore, Verify, Admin, Migration, Copy
 Level = Incremental #Full, Incremental, Differential, VirtualFull...
 Pool = "Daily" #Nombre del apartado donde se define el grupo de volúmenes
 FileSet = "CopiaSerranito" #Nombre del apartado donde se especifica el conjunto de ficheros que se utilizarán
 Schedule = "Programa" #Nombre del apartado donde se define la programación para el trabajo
 Storage = Vol-Serranito # Nombre del apartado donde se define el almacenamiento
 Messages = Standard #Nombre del recurso de mensajes que va a user este trabajo
 SpoolAttributes = yes
 Priority = 10 #Número que indica la prioridad
 Write Bootstrap = "/var/lib/bacula/%c.bsr" #Ruta donde Bacula escribirá un archivo bootstrap
}
~~~

- FileSet: Indica los ficheros que se incluirán o excluirán en los trabajos donde este aparezca. Se van a configurar cuatro apartados, uno para cada máquina. Un ejemplo:
~~~
FileSet {
 Name = "CopiaCroqueta" #Nombre del apartado FileSet
 Include { #Ficheros a incluir
    Options { #Las opciones son obligatorios
        signature = MD5 #Se calculará una firma MD5 para todos los archivos guardados.
        compression = GZIP #Se usuará todos los ficheros usando GZIP 
    }
    File = /home
    File = /etc
    File = /var
 }
 Exclude { #Ficheros a excluir dentro de los ficheros que se han especificado anteriomente.
    File = /var/lib/bacula
    File = /nonexistant/path/to/file/archive/dir
    File = /proc
    File = /var/tmp
    File = /tmp
    File = /sys
    File = /.journal
    File = /.fsck
 }
}
~~~

- Schedule: Medio para programar los trabajos.
~~~
Schedule {
 Name = "Programa" #Nombre del apartado
 Run = Level=Full Pool=Monthly 1st sat at 23:59 #Indica cuando se va a iniciar un trabajo y el label
 Run = Level=Full Pool=Weekly 2nd-5th sat at 23:59
 Run = Level=Incremental Pool=Daily sun-fri at 23:59
}
~~~

- Client: En cada apartado se define un cliente, por lo tanto, hemos configurado 4 apartados Client. Ejemplo:
~~~
Client {
 Name = serranito-fd #Nombre para definir al cliente
 Address = 10.0.0.10 #Dirección del cliente
 FDPort = 9102 #Puerto
 Catalog = mysql-bacula #Indica el nombre del catálogo de la base de datos que va a usar el cliente
 Password = "bacula" 
 File Retention = 90 days #Periodo en el que el fichero va a estar grabado en el catalogo de la base de datos
 Job Retention = 6 months # Periodo en el que el trabajo va a estar grabado en el catalogo de la base de datos
 AutoPrune = yes # Para aplicar automáticamente el File Retention y el Job Retention al finalizar el trabajo.
}
~~~

- Storage: Define los demonios de almacenamiento.
~~~
Storage {
 Name = Vol-Serranito #Nombre del apartado.
 Address = 10.0.0.10 #Dirección de la máquina que contiene el volumen
 SDPort = 9103
 Password = "bacula"
 Device = FileAutochanger1 #Nombre del demonio de almacenamiento del recurso del dispositivo que utilizará.
 Media Type = File #Indica el tipo de medio
 Maximum Concurrent Jobs = 10 #Número máximo de trabajos en el almacenamiento a la vez.
}
~~~

- Catalog: Define qué catálogo utiliza para el trabajo actual.
~~~
Catalog {
 Name = mysql-bacula #Nombre del catálogo
 dbname = "bacula"; DB Address = "localhost"; dbuser = "bacula"; dbpassword = "bacula" #Especificaciones de la base de datos
}
~~~

- Pool: Conjunto de volúmenes de almacenamiento que Bacula utilizará para escribir los datos. Se han creado 3, según el tiempo que van a estar disponibles estas copias.
~~~
Pool {
 Name = Daily
 Pool Type = Backup #Tipo de trabajo que se va a ejecutar
 Recycle = yes #Indica si los volúmenes se pueden reciclar
 AutoPrune = yes
 Volume Retention = 14d
 Maximum Volumes = 10
 Label Format = "Remoto"
}
~~~

- Console: Indica las opciones de la consola:
~~~
Console {
 Name = serranito-mon
 Password = "bacula"
 CommandACL = status, .status
}
~~~

El fichero que se va a utilizar en esta práctica tiene la configuración de este [enlace](https://github.com/PalomaR88/Copias_seguridad/blob/master/bacula-dir.conf).

Para chequear el fichero se utiliza el siguiente comando:
~~~
debian@serranito:~$ sudo bacula-dir -tc /etc/bacula/bacula-dir.conf
~~~

Se inician todos los servicios:
~~~
debian@serranito:~$ sudo service bacula-dir start
debian@serranito:~$ sudo service bacula-sd start
debian@serranito:~$ sudo service bacula-fd start
~~~


#### Configuración del dispositivo de almacenamiento
Se crea un volumen llamado copias_seguridad con 10GiB de tamaño y se asocia a la máquina serranito.

En la máquina serranito se configura el volumen para que sirva de almacenamiento. Este nuevo volumen aparece como /dev/vdb:
~~~
debian@serranito:~$ lsblk -f
NAME FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda                                                                   
└─vda1
     ext4         6197e068-a892-45cb-9672-a05813e800ee    7.8G    16% /
vdb 

debian@serranito:~$ sudo fdisk /dev/vdb

Welcome to fdisk (util-linux 2.33.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xc79c0e0f.
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-20971519, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519): 

Created a new partition 1 of type 'Linux' and of size 10 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
~~~

Y se asigna un sistema de ficheros a la partición creada, en este caso será **ext4**:
~~~
debian@serranito:~$ sudo mkfs.ext4 /dev/vdb1
mke2fs 1.44.5 (15-Dec-2018)
Creating filesystem with 2621184 4k blocks and 655360 inodes
Filesystem UUID: d567a8bc-42da-4834-a333-72fc6c40ee4b
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 
~~~

Configuración de **/etc/fstab** para configurar el volumen montado:
~~~
UUID=d567a8bc-42da-4834-a333-72fc6c40ee4b	/bacula	ext4	defaults	0  0
~~~

Y se comprueba que se ha realizado correctamente:
~~~
debian@serranito:/bacula$ sudo mount -a
debian@serranito:/bacula$ lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    254:0    0  10G  0 disk 
└─vda1 254:1    0  10G  0 part /
vdb    254:16   0  10G  0 disk 
└─vdb1 254:17   0  10G  0 part /bacula
~~~

Después, se crea el directorio para bacula, que será la raíz de esta y cuyo propietario será bacula:
~~~
debian@serranito:~$ sudo mkdir /bacula
debian@serranito:~$ sudo mount /dev/vdb1 /bacula
debian@serranito:/bacula$ sudo chown bacula:bacula /bacula/backups/ -R
debian@serranito:/bacula$ sudo chmod 755 /bacula/backups/ -R
~~~

#### Configuración del fichero bacula-sd.conf
En el fichero **/etc/bacula/bacula-sd.conf** se configura la el almacenamiento de las copias. Los atributos de este fichero deben de corresponder con los atributos insdicados en el fichero que se configuró anteriormente, bacula-dir.conf. La sintaxis es la siguiente:

-Storage: Define el demonio del almacenamiento que usará el director:
~~~
Storage { 
 Name = serranito-sd 
 SDPort = 9103 o
 WorkingDirectory = "/var/lib/bacula" 
 Pid Directory = "/run/bacula" 
 Maximum Concurrent Jobs = 20
 SDAddress = 10.0.0.10
}
~~~

- Director: Define el director. En nuestro caso, se han definido dos directores, serranito-dir y serranito-mon para la consola.
~~~
Director {
 Name = serranito-dir
 Password = "bacula"
}


Director {
 Name = serranito-mon
 Password = "bacula"
 Monitor = yes
}
~~~

- Autochanger y Device: Debe coincidir la configuración con el apartado Storage de bacula-dir.conf:
~~~
Autochanger {
 Name = FileAutochanger1
 Device = DispositivoCopia
 Changer Command = ""
 Changer Device = /dev/null
}

Device {
 Name = DispositivoCopia
 Media Type = File
 Archive Device = /bacula/backups
 LabelMedia = yes;
 Random Access = Yes;
 AutomaticMount = yes;
 RemovableMedia = no;
 AlwaysOpen = no;
 Maximum Concurrent Jobs = 5
}
~~~

El último apartado es Messages, donde se especifica el director:
~~~
Messages {
  Name = Standard
  director = serranito-dir = all
}
~~~

[Aquí](https://github.com/PalomaR88/Copias_seguridad/blob/master/bacula-sd.conf) se encuentra el fichero configurado.

Para comprobar que el fichero se ha configurado correctamente se introduce el siguiente comando, como en la configuración del fichero /etc/bacula/bacula-dir.conf, y si no devuelve nada es que se ha configurado correctamente:
~~~
debian@serranito:/bacula$ sudo bacula-sd -tc /etc/bacula/bacula-sd.conf
~~~



### Inicio de los servicios
Se debe iniciar los servicios de -dir y -sd, se puede realizar a través de systemd:
~~~
debian@serranito:/bacula$ sudo systemctl restart bacula-sd.service
debian@serranito:/bacula$ sudo systemctl restart bacula-director.service
~~~

Y se comprueba que los servicios se han iniciado correctamente y están activos:
~~~
debian@serranito:/bacula$ sudo systemctl status bacula-sd.service
● bacula-sd.service - Bacula Storage Daemon service
   Loaded: loaded (/lib/systemd/system/bacula-sd.service; enabled; vendor 
   Active: active (running) since Thu 2020-01-09 17:32:30 UTC; 30s ago
     Docs: man:bacula-sd(8)
  Process: 10953 ExecStartPre=/usr/sbin/bacula-sd -t -c $CONFIG (code=exit
 Main PID: 10957 (bacula-sd)
    Tasks: 2 (limit: 1168)
   Memory: 1.1M
   CGroup: /system.slice/bacula-sd.service
           └─10957 /usr/sbin/bacula-sd -fP -c /etc/bacula/bacula-sd.conf
debian@serranito:/bacula$ sudo systemctl status bacula-director.service
● bacula-director.service - Bacula Director Daemon service
   Loaded: loaded (/lib/systemd/system/bacula-director.service; enabled; v
   Active: active (running) since Thu 2020-01-09 17:32:39 UTC; 44s ago
     Docs: man:bacula-dir(8)
  Process: 10964 ExecStartPre=/usr/sbin/bacula-dir -t -c $CONFIG (code=exi
 Main PID: 10967 (bacula-dir)
    Tasks: 3 (limit: 1168)
   Memory: 1.5M
   CGroup: /system.slice/bacula-director.service
           └─10967 /usr/sbin/bacula-dir -fP -c /etc/bacula/bacula-dir.conf
~~~

### Configuración del fichero bconsole
La configuración del fichero **/etc/bacula/bconsole.conf** permite acceder a la consola. Se configura de la sigueitne forma:
~~~
Director {
 Name = <nombre_director>
 DIRport = <puerto>
 address = <ip>
 Password = <contraseña>
}
~~~

En nuestro caso, el fichero termina configurado de la siguiente manera:
~~~
Director {
  Name = serranito-dir
  DIRport = 9101
  address = 10.0.0.10
  Password = "bacula"
}
~~~


### Instalación en los clientes
Para la configuración en los clientes hay que instalar el paquete **bacula-client**.
~~~
debian@croqueta:~$ sudo apt install bacula-client
~~~

> Aunque en la práctica se va a realizar la configuración de 4 máquinas (2 Debian, 1 Ubuntu, 1 Centos), en este apartado se va a explicar la configuración en una máquina Debian, serranito, y se explicará las diferencias que puede haber en los demás sistemas.

Configuración del fichero **/etc/bacula/bacula-fd.conf** para indicar los parámetros del director:
~~~
Director {
 Name = serranito-dir
 Password = "bacula"
}

Director {
 Name = serranito-mon
 Password = "bacula"
 Monitor = yes
}

FileDaemon { # this is me
 Name = serranito-fd
 FDport = 9102 # where we listen for the director
 WorkingDirectory = /var/lib/bacula # en Centos /var/spool/bacula
 Pid Directory = /run/bacula # en Centos /var/run
 Maximum Concurrent Jobs = 20
 Plugin Directory = /usr/lib/bacula # esta opción no debe incluirse en Centos
 FDAddress = 10.0.0.10 # esta opción no debe incluirse en centos
}

# Send all messages except skipped files back to Director
Messages {
 Name = Standard
 director = serranito-dir = all, !skipped, !restored
}
~~~

> En la máquina Salmorejo, con CentOS, hay que configurar el firewall:
~~~
[centos@salmorejo ~]$ sudo firewall-cmd --zone=public --permanent --add-port 9102/tcp
success
[centos@salmorejo ~]$ sudo firewall-cmd --reload
success
~~~

Tras configurar el fichero se inicia el servicio:
~~~
debian@serranito:/bacula$ sudo systemctl start bacula-fd
~~~

Cuando todos los clientes están configurados, los servicios inicidos y se ha comprobado que todos están activos se reinicia el servidor:
~~~
debian@serranito:/bacula$ sudo systemctl restart bacula-sd.service
debian@serranito:/bacula$ sudo systemctl restart bacula-director.service
debian@serranito:/bacula$ sudo systemctl restart bacula-fd.service
~~~

Se comprueba que los clientes se reconocen adecuadamente a través de la consola de bacula:
~~~
debian@serranito:~$ sudo bconsole
Connecting to Director 10.0.0.10:9101
1000 OK: 103 serranito-dir Version: 9.4.2 (04 February 2019)
Enter a period to cancel a command.
*status client
The defined Client resources are:
     1: serranito-fd
     2: croqueta-fd
     3: tortilla-fd
     4: salmorejo-fd
Select Client (File daemon) resource (1-4): 1
Connecting to Client serranito-fd at 10.0.0.10:9102

serranito-fd Version: 9.4.2 (04 February 2019)  x86_64-pc-linux-gnu debian buster/sid
Daemon started 11-Jan-20 18:36. Jobs: run=0 running=0.
 Heap: heap=114,688 smbytes=22,012 max_bytes=22,029 bufs=68 max_bufs=68
 Sizes: boffset_t=8 size_t=8 debug=0 trace=0 mode=0,0 bwlimit=0kB/s
 Plugin: bpipe-fd.so 

Running Jobs:
Director connected at: 11-Jan-20 18:37
No Jobs running.
====

Terminated Jobs:
====
You have messages.
*2
2: is an invalid command.
*status client
The defined Client resources are:
     1: serranito-fd
     2: croqueta-fd
     3: tortilla-fd
     4: salmorejo-fd
Select Client (File daemon) resource (1-4): 2
Connecting to Client croqueta-fd at 10.0.0.3:9102

croqueta-fd Version: 9.4.2 (04 February 2019)  x86_64-pc-linux-gnu debian buster/sid
Daemon started 11-Jan-20 18:36. Jobs: run=0 running=0.
 Heap: heap=114,688 smbytes=22,010 max_bytes=22,027 bufs=68 max_bufs=68
 Sizes: boffset_t=8 size_t=8 debug=0 trace=0 mode=0,0 bwlimit=0kB/s
 Plugin: bpipe-fd.so 

Running Jobs:
Director connected at: 11-Jan-20 18:37
No Jobs running.
====

Terminated Jobs:
====
*status client
The defined Client resources are:
     1: serranito-fd
     2: croqueta-fd
     3: tortilla-fd
     4: salmorejo-fd
Select Client (File daemon) resource (1-4): 3
Connecting to Client tortilla-fd at 10.0.0.11:9102

tortilla-fd Version: 9.0.6 (20 November 2017) x86_64-pc-linux-gnu ubuntu 18.04
Daemon started 11-Jan-20 18:36. Jobs: run=0 running=0.
 Heap: heap=110,592 smbytes=21,981 max_bytes=21,998 bufs=68 max_bufs=68
 Sizes: boffset_t=8 size_t=8 debug=0 trace=0 mode=0,0 bwlimit=0kB/s
 Plugin: bpipe-fd.so 

Running Jobs:
Director connected at: 11-Jan-20 19:05
No Jobs running.
====

Terminated Jobs:
====
*status client
The defined Client resources are:
     1: serranito-fd
     2: croqueta-fd
     3: tortilla-fd
     4: salmorejo-fd
Select Client (File daemon) resource (1-4): 4
Connecting to Client salmorejo-fd at 10.0.0.18:9102

salmorejo-fd Version: 9.0.6 (20 November 2017) x86_64-redhat-linux-gnu redhat (Core)
Daemon started 11-Jan-20 18:36. Jobs: run=0 running=0.
 Heap: heap=102,400 smbytes=21,984 max_bytes=22,001 bufs=68 max_bufs=68
 Sizes: boffset_t=8 size_t=8 debug=0 trace=0 mode=0,0 bwlimit=0kB/s
 Plugin: bpipe-fd.so 

Running Jobs:
Director connected at: 11-Jan-20 19:06
No Jobs running.
====

Terminated Jobs:
====
~~~

Para añadir el nombre al volumen e identificarlo:
~~~
debian@serranito:/bacula$ sudo bconsole
Connecting to Director 10.0.0.10:9101
1000 OK: 103 serranito-dir Version: 9.4.2 (04 February 2019)
Enter a period to cancel a command.
*label
Automatically selected Catalog: mysql-bacula
Using Catalog "mysql-bacula"
Automatically selected Storage: Vol-Serranito
Enter new Volume name: backups
Defined Pools:
     1: Default
     2: File
     3: Scratch
     4: Vol-Backup
Select the Pool (1-4): 4
Connecting to Storage daemon Vol-Serranito at 10.0.0.10:9103 ...
Sending label command for Volume "backups" Slot 0 ...
3000 OK label. VolBytes=226 VolABytes=0 VolType=1 Volume="backups" Device="DispositivoCopia" (/bacula/backups)
Catalog record for Volume "backups", Slot 0  successfully created.
Requesting to mount FileAutochanger1 ...
3906 File device ""DispositivoCopia" (/bacula/backups)" is always mounted.
You have messages.
~~~

