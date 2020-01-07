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
Se crea una instancia que actuará como servidor para bacula a la que posteriormente se asociará un volumen donde se guardarán las copias de seguridad que se realizan de los otras 3 máquina, salmorejo, croqueta y tortilla. Esta instancia, serranito, será una Debian Buster. 

### Aplicación Bacula
Para esta práctica se va a emplear la herramienta de copias de seguridad **Bacula** que funciona bajo una arquitectura cliente-servidor.

Para la instalación de Bacula es necesaria un gestor de base de datos, en nuestro caso se va a utilizar MariaDB. 

#### Webmin
Pero en la práctica no solo se instalará el gestor de bases de datos, sino toda una pila LAMP ya que se va a utilizar **Webmin**, una herramienta, que no es necesaria,pero permite la congiruación del sistema vía web para sistemas Linux. De esta forma, la configuración de Bacula será más amigable. 
~~~
debian@serranito:~$ sudo apt install apache2 mariadb-server mariadb-client php
~~~

Se continúa con la instalación de Webmin. En primer lugar se descarga desde su [página oficial](https://sourceforge.net/projects/webadmin/files/):
~~~
debian@serranito:~$ wget http://prdownloads.sourceforge.net/webadmin/webmin_1.870_all.deb
~~~

Y se instala:
~~~
debian@serranito:~$ sudo dpkg -i webmin_1.870_all.deb 
~~~

> Puede que en la instalación aparezcan errores de dependencias. En nuestro caso se han solventado con la instalación de las siguientes librerías:
~~~
debian@serranito:~$ sudo apt install libnet-ssleay-perl libauthen-pam-perl libio-pty-perl apt-show-versions 
~~~

Una vez instalado, ya está accesible Webmin desde la url que se indica al finalizar la instalación.


#### Instalación de Bacula
A continuación, se va a realizar la instalación de los paquetes de Bacula que son los siguientes:
~~~
debian@serranito:~$ sudo apt install bacula bacula-client bacula-common-mysql bacula-director-mysql bacula-server
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

El fichero para esta práctica se forma de la siguiente manera:
~~~
Director { 
 Name = serranito-dir
 DIRport = 9101
 QueryFile = "/etc/bacula/scripts/query.sql"
 WorkingDirectory = "/var/lib/bacula"
 PidDirectory = "/run/bacula"
 Maximum Concurrent Jobs = 20
 Password = "bacula"
 Messages = Daemon
 DirAddress = 10.0.0.10
}

JobDefs {
 Name = "copias"
 Type = Backup
 Level = Incremental
 Client = serranito-fd
 FileSet = "copia-completa"
 Schedule = "periodo"
 Storage = File
 Messages = Standard
 Pool = File
 SpoolAttributes = yes
 Priority = 10
 Write Bootstrap = "/var/lib/bacula/%c.bsr"
}

Job {
 Name = "cp-serranito"
 JobDefs = "copias"
 Client = "serranito-fd"
}

Job {
 Name = "cp-croqueta"
 JobDefs = "copias"
 Client = "croqueta-fd"
}

Job {
 Name = "cp-salmorejo"
 JobDefs = "copias"
 Client = "salmorejo-fd"
}

Job {
 Name = "cp-tortilla"
 JobDefs = "copias"
 Client = "tortilla-fd"
}

Job {
 Name = "rest-serranito"
 Type = Restore
 Client=serranito-fd
 FileSet="copia-completa"
 Storage = File
 Pool = File
 Messages = Standard
}

Job {
 Name = "rest-croqueta"
 Type = Restore
 Client= croqueta-fd
 FileSet="copia-completa"
 Storage = File
 Pool = File
 Messages = Standard
}

Job {
 Name = "rest-salmorejo"
 Type = Restore
 Client= salmorejo-fd
 FileSet="copia-completa"
 Storage = File
 Pool = File
 Messages = Standard
}

Job {
 Name = "rest-tortilla"
 Type = Restore
 Client=tortilla-fd
 FileSet="copia-completa"
 Storage = File
 Pool = File
 Messages = Standard
}

FileSet {
 Name = "copia-completa"
 Include {
 Options {
 signature = MD5
 compression = GZIP
 }
 File = /home
 File = /etc
 File = /var
 }
 Exclude {
 File = /var/lib/bacula
 File = /nonexistant/path/to/file/archive/dir
 File = /proc
 File = /var/cache
 File = /var/tmp
 File = /tmp
 File = /sys
 File = /.journal
 File = /.fsck
 }
}

Schedule {
 Name = "periodo"
 Run = Level=Full 1st friday 23:00
 Run = Level=Incremental saturday at 23:00
}

Client {
 Name = serranito-fd
 Address = 10.0.0.10
 FDPort = 9102
 Catalog = MyCatalog
 Password = "bacula"
 File Retention = 60 days
 Job Retention = 6 months
 AutoPrune = yes
}

Client {
 Name = croqueta-fd
 Address = 10.0.0.3
 FDPort = 9102
 Catalog = MyCatalog
 Password = "bacula"
 File Retention = 60 days
 Job Retention = 6 months
 AutoPrune = yes
}

Client {
 Name = salmorejo-fd
 Address = 10.0.0.18
 FDPort = 9102
 Catalog = MyCatalog
 Password = "bacula"
 File Retention = 60 days
 Job Retention = 6 months
 AutoPrune = yes
}

Client {
 Name = tortilla-fd
 Address = 10.0.0.11
 FDPort = 9102
 Catalog = MyCatalog
 Password = "bacula"
 File Retention = 60 days
 Job Retention = 6 months
 AutoPrune = yes
}

Storage {
 Name = File
 Address = 10.0.0.10
 SDPort = 9103
 Password = "bacula"
 Device = FileStorage
 Media Type = File
}

Catalog {
 Name = MyCatalog
 dbname = "bacula"; DB Address = "localhost"; dbuser = "bacula"; dbpassword = "bacula"
}

Pool {
 Name = File
 Pool Type = Backup
 Recycle = yes
 AutoPrune = yes
 Volume Retention = 365 days
 Maximum Volume Bytes = 50G
 Maximum Volumes = 100
 Label Format = "Remoto"
}

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

https://juanjoselo.wordpress.com/2017/12/27/instalacion-y-configuracion-de-sistema-de-copias-de-seguridad-con-bacula-en-debian-9/

Configuración de dispositivo de almacenamiento de las copias

https://www.juanluramirez.com/sistema-de-copia-de-seguridad-bacula/


### Selección de información

### Programación de las copias completas

### Programación de la copias incrementales/diferenciales

### Almacenamiento

### Sistema de copias a coconut

### Datos críticos

### Incorporación de los nuevos datos

### Equipo secundario
