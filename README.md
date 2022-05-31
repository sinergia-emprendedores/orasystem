# Manual Instalación Zabbix Proxy

## Objetivo
El objetivo de este manual es servir de apoyo bibliográfico del proceso de instalación de un Zabbix Proxy Server.

### Arquitectura
La idea es que este servidor este en dependencias del cliente el cual pertenece a una red ajena a la del servidor de Zabbix y su Front-end para así asumir el rol de recolector local de métricas en caso de algun incidente de conectividad por ejemplo.

![Zabbix ARCH](https://www.linuxidc.com/upload/2016_07/160712200622117.png "Zabbix Proxy ARCH")

### Requisitos y Consideraciones
- Contar con un servidor de Zabbix
- Contar con un servidor para Zabbix Proxy
- Las redes deben ser alcanzables una de otra
- Zabbix Proxy por defecto viene en **modo activo** y es lo recomendado
- Zabbix Proxy y Zabbix Server no pueden usar la misma base de datos

### Actividades a realizar
- Instalación repositorio
- Creación de la base de datos
- Importación esquema
- Configurar la base de datos para el Zabbix proxy
- Iniciar el proceso de Zabbix proxy
- Configuración de la interfaz

#### Instalación repositorio
```bash
$ sudo rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm
$ sudo yum clean all
```
#### Instalación paquetes
```bash
$ sudo yum install zabbix-proxy-mysql zabbix-agent
```
#### Creación de la base de datos
```bash
# Ejecutar como root
# '<serverID>' es el nombre de host del servidor que se conectará. Para permitir el acceso desde cualquier ubicación configurar en '%'
mysql -uroot -p
<password>
mysql> create database zabbix character set utf8 collate utf8_bin;
mysql> create user 'zabbix'@'<serverID>' identified by '<NewZabbixPassword>';
mysql> grant all privileges on zabbix.* to 'zabbix'@'<serverID>' with grant option;
mysql> quit;
```
#### Importación esquema
```bash
$ zcat /usr/share/doc/zabbix-proxy-mysql*/schema.sql.gz | mysql -u<user> -p <password>
```
#### Configurar la base de datos para el Zabbix proxy
```bash
$ sudo vi /etc/zabbix/zabbix_proxy.conf
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=<contraseña>
```
#### Configuración Zabbix Proxy
Editar el archivo de configuración
`$ sudo vim /etc/zabbix/zabbix_proxy.conf`

Modifique los valores segun la siguiente configuración
```bash
Server=<Application Server IP Address>
ServerPort=10051
EnableRemoteCommands=1
LogRemoteCommands=1
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=<database password>
ProxyLocalBuffer=0
ProxyOfflineBuffer=72
HeartbeatFrequency=60
DataSenderFrequency=3
StartPollers=25
StartPollersUnreachable=5
StartTrappers=25
StartVMwareCollectors=3
```
#### Configurar PHP
Editar archivo php.ini
`$ sudo vim /etc/php.ini`

Cambie los valores a la siguiente configuración
```bash
max_execution_time = 600
max_input_time = 600
memory_limit = 256M
post_max_size = 32M
upload_max_filesize = 16M
date.timezone = UTC
expose_php = Off
```
#### Configurar Firewall
Abrir los puertos en el firewall
```bash
$ sudo firewall-cmd --permanent --add-port=10050/tcp
$ sudo firewall-cmd --permanent --add-port=10051/tcp
$ sudo firewall-cmd --reload
```

#### Iniciar el proceso de Zabbix proxy
```bash
$ sudo systemctl start zabbix-proxy
$ sudo systemctl enable zabbix-proxy
```
Comprobar el estado del servicio
`$ sudo systemctl status zabbix-proxy`

#### Configurar Zabbix Agent en Proxy
```bash
$ vi /etc/zabbix/zabbix_agentd.conf
PidFile=/var/run/zabbix/zabbix_agentd.pid
LogFile=/var/log/zabbix/zabbix_agentd.log
LogFileSize=0
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=zbxprxy01
#HostMetadata=97d7c2030d2ccf29f059f4c24a8c0796
Include=/etc/zabbix/zabbix_agentd.d/*.conf

$ sudo systemctl enable zabbix-agent.service
$ sudo systemctl start zabbix-agent.service
$ sudo systemctl status zabbix-agent.service
```

#### Configuración de la interfaz
Creamos el proxy en Zabbix Server.
**ADMINISTRATION -> PROXIES -> CREATE PROXY**

Muy importante que al crear el proxy **tenga el mismo nombre que el hostname** especificado en el archivo de configuración

![Create proxy](https://www.zabbix-es.com.es/images/2/2a/CZP102.png "Create proxy")

Si el proxy presenta algun problema al reflejar el estado, ejecutar lo siguiente

![Proxy status](https://www.zabbix-es.com.es/images/thumb/a/ad/CZP103.png/1536px-CZP103.png "Proxy status")

```bash
$ sudo zabbix_server -R config_cache_reload
$ sudo systemctl restart zabbix-proxy.service
```

![Proxy estado OK](https://www.zabbix-es.com.es/images/thumb/7/72/CZP104.png/1536px-CZP104.png "Proxy estado OK")

# GRACIAS!
