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


### Aplicación Bacula
Para esta práctica se va a emplear la herramienta de copias de seguridad **Bacula** que funciona bajo una arquitectura cliente-servidor.

Para la instalación de Bacula es necesaria un gestor de base de datos, en nuestro caso se va a utilizar MariaDB. 

#### Webmin
Pero en la práctica no solo se instalará el gestor de bases de datos, sino toda una pila LAMP ya que se va a utilizar **Webmin**, una herramienta, que no es necesaria,pero permite la congiruación del sistema vía web para sistemas Linux. De esta forma, la configuración de Bacula será más amigable. Además de Apache2, debería estar instalada una base de datos, 
~~~
ubuntu@tortilla:~$ sudo apt install apache2  mariadb-client php
~~~

Se continúa con la instalación de Webmin. En primer lugar se descarga desde su [página oficial](https://sourceforge.net/projects/webadmin/files/):
~~~
ubuntu@tortilla:~$ wget http://prdownloads.sourceforge.net/webadmin/webmin_1.870_all.deb
~~~

Y se instala:
~~~
ubuntu@tortilla:~$ sudo dpkg -i webmin_1.870_all.deb
~~~

> Puede que en la instalación aparezcan errores de dependencias. En nuestro caso se han solventado con la instalación de las siguientes librerías:
~~~
ubuntu@tortilla:~$ sudo apt install libnet-ssleay-perl libauthen-pam-perl libio-pty-perl apt-show-versions
~~~

Una vez instalado, ya está accesible Webmin desde la url que se indica al finalizar la instalación.


#### Instalación de Bacula
A continuación, se va a realizar la instalación de los paquetes de Bacula que son los siguientes:
~~~
ubuntu@tortilla:~$ sudo apt install bacula bacula-client bacula-common-mysql bacula-director-mysql bacula-server
~~~

En el proceso de instalación de **bacula-director-mysql** se pregutna si se quiere configurar la base de datos para bacula y su configuración.
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

#### Configuración de Bacula
Por último hay que configurar el fichero **/etc/bacula/bacula-dir.conf** donde se indica los parámetros más importantes para las copias de la siguiente forma:
~~~
Director { # se define a sí mismo
 Name = <nombre_director>
 DIRport = <puerto> # where we listen for UA connections
 QueryFile = "/etc/bacula/scripts/query.sql"
 WorkingDirectory = "/var/lib/bacula"
 PidDirectory = "/run/bacula"
 Maximum Concurrent Jobs = 20
 Password = "<contraseña_director>"
 Messages = Daemon
 DirAddress = <ip_maquina_director>
}

JobDefs { # definición de tarea
 Name = "<nombre_tarea>"
 Type = <tipo_tarea> # backup para copias
 Level = <nivel_tarea> # por defecto Incremental
 Client = <cliente_donde_ejecutar>
 FileSet = "<nombre_info_copiara>" # se define en el apartado FileSet
 Schedule = "<nombre_programación>" # se define a continuación
 Storage = <cargador_virtual_automático> # se define a continuación
 Messages = Standard
 Pool = <nombre_apartado> # se define a continuación el volumen 
 SpoolAttributes = yes
 Priority = <numero_prioridad>
 Write Bootstrap = "<fichero_bacula>" # por defecto "/var/lib/bacula/%c.bsr"
}

Job { # definición de clientes en los que trabajar
 Name = "<nombre_job>"
 JobDefs = "<nombre_tarea>"
 Client = "<nombre_cliente>"
}

Job { # definición en los clientes donde se restaura
 Name = "<nombre_tarea>"
 Type = <tipo_tarea> # Restore para restaurar
 Client= <nombre_cliente>
 FileSet= "<nombre_info_copiara>"
 Storage = <cargador_virtual_automático>
 Pool = <nombre_apartado>
 Messages = Standard
}

FileSet { # tipo de copia, contenido y forma de compresión
 Name = "<nombre_info_copiara>"
 Include {
          Options {
                   signature = <tipo_encriptado>
                   compression = <tipo_almacenamiento>
          }
          File = <rutas_que_copiar>
 }
 Exclude {
          File = <turas_que_excluye>
         }
}

Schedule { # tipo de programación - cuándo
 Name = "<nombre_programación>"
 Run = Level=<tipo> <periodo>
}

Client { # definición de los clientes
 Name = <nombre_cliente>
 Address = <ip_máquina_cliente>
 FDPort = <puerto>
 Catalog = <nombre_catálogo>
 Password = <contraseña_cliente>
 File Retention = <tiempo_retención_fichero> #registro fichero en la BD del cat
 Job Retention = <tiempo_retención_tearea> #registro tareas en la BD del cat
 AutoPrune = yes # auto expirado en Jobs/Files
}

Storage { # tipo de almacenamiento
 Name = <cargador_virtual_automático>
 Address = <ip_almacenamiento> # no utilizar 'localhost'
 SDPort = <puerto>
 Password = <contraseña_dir>
 Device = <nombre_dipositivo_lógico> # se especifica a continuación
 Media Type = <tipo_de_medio>
}

Catalog {
 Name = <nombre_catálogo>
 dbname = "<nombre_BD>"; DB Address = <dirección_BD>; dbuser = "<usuario_BD>"; dbpassword = "<contraseña>"
}

Pool {
 Name = File
 Pool Type = Backup
 Recycle = yes # reciclar volúmenes
 AutoPrune = yes # Auto expirado de volumnes
 Volume Retention = <periodo_retención>
 Maximum Volume Bytes = <límite_tamaño
 Maximum Volumes = <límite_número_volumen>
 Label Format = "Remoto"
}

# Los demás apartados se dejan por defecto
Messages {
 Name = Standard
 mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: %t %e of %c %l\" %r"
 operatorcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula: Intervention needed for %j\" %r"
 mail = root = all, !skipped
 operator = root = mount
 console = all, !skipped, !saved
 append = "/var/log/bacula/bacula.log" = all, !skipped
 catalog = all
}

Messages {
 Name = Daemon
 mailcommand = "/usr/sbin/bsmtp -h localhost -f \"\(Bacula\) \<%r\>\" -s \"Bacula daemon message\" %r"
 mail = root = all, !skipped
 console = all, !skipped, !saved
 append = "/var/log/bacula/bacula.log" = all, !skipped
}

Pool {
 Name = Default
 Pool Type = Backup
 Recycle = yes # Bacula can automatically recycle Volumes
 AutoPrune = yes # Prune expired volumes
 Volume Retention = 365 days # one year
 Maximum Volume Bytes = 50G # Limit Volume size to something reasonable
 Maximum Volumes = 100 # Limit number of Volumes in Pool
}

Pool {
 Name = Scratch
 Pool Type = Backup
}

Console {
 Name = grafana-mon
 Password = "bacula"
 CommandACL = status, .status
}
~~~

El fichero que se va a utilizar en esta práctica tiene la configuración de este [enlace](enlace).

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


Después, se crea el directorio para bacula, que será la raíz de esta y cuyo propietario será bacula:
~~~
debian@serranito:~$ sudo mkdir /bacula
debian@serranito:~$ sudo mount /dev/vdb1 /bacula
debian@serranito:/bacula$ sudo chown bacula:bacula /bacula/backups/ -R
debian@serranito:/bacula$ sudo chmod 755 /bacula/backups/ -R
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

### Configuración del fichero bacula-sd.conf
En el fichero **/etc/bacula/bacula-sd.conf** se configura la el almacenamiento de las copias. La sintaxis del fichero es la siguiente:
~~~
Storage {
  Name = <nombre_del_equipo>-sd
  SDPort = <puerto>
  WorkingDirectory = "<directorio de trabajo>"
  Pid Directory = "/run/bacula"
  Maximum Concurrent Jobs = 20
  SDAddress = <ip>
}
Director { # llamamiendo al director que llama al daemon anterior
 Name = <nombre_director>-dir
 Password = "<contraseña>"
}

Director { # otorga permisos al director para ver el proceso
 Name = <nombre_director>-mon
 Password = "<contraseña>"
 Monitor = yes
}

Autochanger { # cargador automático virtual del punto de almacenamiento
 Name = <nombre_dispositivo_logico> # tiene que coinciddir con el atributo Device en bacula-dir.conf
 Device = <dispositivo_montaje>
 Changer Command = ""
 Changer Device = /dev/null
}

Device { # declaración de los Devices del apartado anterior
 Name = <nombre_disp_montage>
 Media Type = <tipo>
 Archive Device = <ruta_almacenamiento>
 LabelMedia = yes; # lets Bacula label unlabeled media
 Random Access = Yes;
 AutomaticMount = yes; # when device opened, read it
 RemovableMedia = no;
 AlwaysOpen = no;
 Maximum Concurrent Jobs = 5
}

Messages { # este último apartado sobre los mensajes se deja por defecto
  Name = Standard
  director = serranito-dir = all
}
~~~

[Aquí](link) se encuentra el fichero configurado.

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

### Selección de información
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




### Programación de las copias completas

### Programación de la copias incrementales/diferenciales

### Almacenamiento

### Sistema de copias a coconut

### Datos críticos

### Incorporación de los nuevos datos

### Equipo secundario
