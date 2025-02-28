# Documentación Completa: Implementación de Red con Servicios DHCP, DNS y FTP

## Índice
1. [Arquitectura de Red](#arquitectura-de-red)
2. [Router Firewall Red 1 (172.16.79.0/24)](#router-firewall-red-1)
3. [Router Firewall Red 2 (172.16.49.0/24)](#router-firewall-red-2)
4. [Servidor DHCP](#servidor-dhcp)
5. [Servidor DNS1 - Primario para alumno79.com](#servidor-dns1)
6. [Servidor DNS2 - Secundario para alumno79.com/Primario para pana.com](#servidor-dns2)
7. [Servidor FTP](#servidor-ftp)
8. [Pruebas de Funcionamiento](#pruebas-de-funcionamiento)
9. [Resolución de Problemas](#resolución-de-problemas)

## Arquitectura de Red

La implementación consta de dos redes interconectadas:
- **Red 1**: 172.16.79.0/24
- **Red 2**: 172.16.49.0/24

### Diagrama de Red

```
                           INTERNET
                               |
                               |
        +---------------------+----------------------+
        |                     |                      |
+-------+--------+   +--------+-------+    +---------+--------+
| Router Red 1   |   | Router Red 2   |    |  Acceso Externo  |
| 192.168.35.79  |   | 192.168.35.49  |    |                  |
+-------+--------+   +--------+-------+    +------------------+
        |                     |
        |                     |
+-------+--------+   +--------+-------+
| 172.16.79.1    |   | 172.16.49.1    |
| Red Interna    |   | Red Interna    |
+--------+-------+   +--------+-------+
         |                    |
    +----+----+          +---+----+
    |         |          |        |
+---+---+ +---+---+ +----+----+
|Servidor| |Servidor| |Relay    |
|DHCP    | |DNS1    | |DHCP     |
|.79.10  | |.79.2   | |.49.2    |
+-------+ +---+---+ +---------+
              |
          +---+---+  +-------+
          |Servidor|  |Servidor|
          |DNS2    |  |FTP    |
          |.79.3   |  |.79.4  |
          +-------+  +-------+
```

## Router Firewall Red 1

### Configuración de Interfaces

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.35.79/24]
      gateway4: 192.168.35.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
    ens37:
      dhcp4: no
      addresses: [172.16.79.1/24]
```
![[SR/DOCUMENTACIÓN-FINAL/capturas/1.png]]
![[SR/DOCUMENTACIÓN-FINAL/capturas/2.png]]

### Script de Firewall y Redirección de Puertos

```bash
#!/bin/sh

#BORRADO DE REGLAS ANTERIORES
iptables -F
iptables -X
iptables -Z
iptables -t nat -F

#POLITICA POR DEFECTO
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -t nat -P PREROUTING ACCEPT
iptables -t nat -P POSTROUTING ACCEPT

#ACCESO A LOCALHOST
iptables -A INPUT -i lo -j ACCEPT

#ACCESO DESDE RED LOCAL
iptables -A INPUT -s 172.16.79.0/24 -j ACCEPT

#ENMASCARAMIENTO RED LOCAL
iptables -t nat -A POSTROUTING -s 172.16.79.0/24 -o ens33 -j MASQUERADE

#ACTIVAR BIT FORWARD
echo 1 > /proc/sys/net/ipv4/ip_forward

#PERMITIMOS ACCESO PUERTO 22 (SSH)
iptables -A INPUT -s 0.0.0.0/0 -i ens33 -p tcp --dport 22 -j ACCEPT

#REDIRECCIONAMIENTO DE PUERTOS
iptables -t nat -A PREROUTING -i ens33 -p udp --dport 67 -j DNAT --to 172.16.79.2:67
iptables -t nat -A PREROUTING -i ens33 -p tcp --dport 2203 -j DNAT --to 172.16.79.3:22

#PERMITIR Y REDIRIGIR FTP
# Cargar módulos para seguimiento de conexiones FTP
modprobe nf_conntrack_ftp
echo 1 > /proc/sys/net/netfilter/nf_conntrack_helper

# Puerto de control FTP (21)
iptables -A INPUT -s 0.0.0.0/0 -i ens33 -p tcp --dport 21 -j ACCEPT
iptables -t nat -A PREROUTING -i ens33 -p tcp --dport 21 -j DNAT --to 172.16.79.4:21
iptables -A FORWARD -d 172.16.79.4 -p tcp --dport 21 -j ACCEPT

# Puertos para modo pasivo FTP (30000-31000)
iptables -A INPUT -s 0.0.0.0/0 -i ens33 -p tcp --dport 30000:31000 -j ACCEPT
iptables -t nat -A PREROUTING -i ens33 -p tcp --dport 30000:31000 -j DNAT --to 172.16.79.4
iptables -A FORWARD -d 172.16.79.4 -p tcp --dport 30000:31000 -j ACCEPT

#CERRAR PUERTOS
iptables -A INPUT -s 0.0.0.0/0 -i ens33 -p tcp --dport 1:1024 -j DROP
iptables -A INPUT -s 0.0.0.0/0 -i ens33 -p udp --dport 1:1024 -j DROP
```
![[SR/DOCUMENTACIÓN-FINAL/capturas/3.png]]
![[SR/DOCUMENTACIÓN-FINAL/capturas/4.png]]
### Verificación de Firewall y Enrutamiento

```bash
# Verificar que el bit de forward está habilitado
cat /proc/sys/net/ipv4/ip_forward

# Verificar módulos de conexión cargados
lsmod | grep conntrack
lsmod | grep nf_nat

# Verificar reglas de firewall
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v
sudo iptables -t filter -L FORWARD -n -v

# Verificar tabla de enrutamiento
ip route show
ip route get 172.16.79.4
ip route get 172.16.49.1
ip route get 8.8.8.8

# Verificar conexiones activas
sudo netstat -tulapn
sudo ss -tulapn

# Verificar configuración de sistema
sudo sysctl -a | grep forward
sudo sysctl -a | grep conntrack
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/5.png]]
![[SR/DOCUMENTACIÓN-FINAL/capturas/6.png]]

## Router Firewall Red 2

### Configuración de Interfaces

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.35.49/24]
      gateway4: 192.168.35.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
    ens37:
      dhcp4: no
      addresses: [172.16.49.1/24]
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/7.png]]

### Configuración del Relay DHCP

```
# /etc/default/isc-dhcp-relay
# Opciones para dhcrelay

# Servidor DHCP al que se reenviarán las solicitudes
SERVERS="172.16.79.10"

# Interfaz en la que escuchar las solicitudes de DHCP
INTERFACES="ens37"

# Opciones adicionales
OPTIONS=""
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/8.png]]

Verificación del funcionamiento del relay:

```bash
# Ver estado del servicio 
sudo systemctl status isc-dhcp-relay

# Ver logs del servicio
sudo journalctl -u isc-dhcp-relay

# Verificar procesos en ejecución
ps aux | grep dhcrelay

# Reiniciar el servicio si es necesario
sudo systemctl restart isc-dhcp-relay
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/9.png]]

## Servidor DHCP

### Configuración Principal

```
# /etc/dhcp/dhcpd.conf

# Configuración de la Red 1
subnet 172.16.79.0 netmask 255.255.255.0 {
  range 172.16.79.50 172.16.79.200;
  option routers 172.16.79.1;
  option domain-name-servers 172.16.79.2, 172.16.79.3;
  option domain-name "alumno79.com";
  default-lease-time 600;
  max-lease-time 7200;
}

# Configuración de la Red 2 (vía relay)
subnet 172.16.49.0 netmask 255.255.255.0 {
  range 172.16.49.50 172.16.49.200;
  option routers 172.16.49.1;
  option domain-name-servers 172.16.79.2, 172.16.79.3;
  option domain-name "alumno79.com";
  default-lease-time 600;
  max-lease-time 7200;
}

# Configuración de Failover DHCP
failover peer "dhcp-failover" {
  primary;
  address 172.16.79.10;
  peer address 172.16.79.3;
  max-response-delay 30;
  max-unacked-updates 10;
  load balance max seconds 3;
  mclt 1800;
  split 128;
}

# Reserva para servidor FTP
host servidor-ftp {
  hardware ethernet XX:XX:XX:XX:XX:XX;  # Dirección MAC del servidor FTP
  fixed-address 172.16.79.4;
}
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/10.png]]
![[SR/DOCUMENTACIÓN-FINAL/capturas/11.png]]

### Configuración de Red del Servidor DHCP

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses: [172.16.79.10/24]
      gateway4: 172.16.79.1
      nameservers:
        addresses: [172.16.79.2, 172.16.79.3]
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/12.png]]

### Verificación del Funcionamiento

```bash
# Verificar estado del servicio
sudo systemctl status isc-dhcp-server

# Ver logs del servicio
sudo journalctl -u isc-dhcp-server

# Verificar archivo de leases
sudo cat /var/lib/dhcp/dhcpd.leases
sudo cat /var/lib/dhcp/dhcpd.leases | grep -A 6 'lease'

# Verificar estadísticas del servidor DHCP
dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases

# Verificar puertos abiertos para DHCP
sudo netstat -tulapn | grep dhcp
sudo ss -tulapn | grep :67

# Reiniciar el servicio
sudo systemctl restart isc-dhcp-server

# Validar la configuración
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf

# Ver estado de failover
sudo omshell
server 127.0.0.1
connect
status failover
```

**![[SR/DOCUMENTACIÓN-FINAL/capturas/13.png]]

![[SR/DOCUMENTACIÓN-FINAL/capturas/14.png]]

![[SR/DOCUMENTACIÓN-FINAL/capturas/15.png]]
### Prueba de Asignación DHCP

```bash
# Desde un cliente, renovar la dirección IP:
sudo dhclient -r
sudo dhclient

# Ver la dirección IP asignada
ip addr show

# Ver la configuración de DNS recibida
cat /etc/resolv.conf

# Ver detalles de la asignación DHCP
sudo dhclient -v
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/16.png]]

## Servidor DNS1

### Configuración Principal

```
# /etc/bind/named.conf.local

# Zona primaria para alumno79.com
zone "alumno79.com" {
  type master;
  file "/etc/bind/zones/db.alumno79.com";
  allow-transfer { 172.16.79.3; };
  notify yes;
};

# Zona inversa para 172.16.79.0/24
zone "79.16.172.in-addr.arpa" {
  type master;
  file "/etc/bind/zones/db.172.16.79";
  allow-transfer { 172.16.79.3; };
  notify yes;
};
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/17.png]]

### Archivo de Zona Directa

```
; /etc/bind/zonas/db.alumno79.com
$TTL    604800
@       IN      SOA     ns1.alumno79.com. admin.alumno79.com. (
                  2024022801         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.alumno79.com.
@       IN      NS      ns2.alumno79.com.
@       IN      MX      10      mail.alumno79.com.
@       IN      A       172.16.79.1

ns1     IN      A       172.16.79.2
ns2     IN      A       172.16.79.3
mail    IN      A       172.16.79.10
www     IN      A       172.16.79.10
ftp     IN      A       172.16.79.4
dhcp    IN      A       172.16.79.10
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/18.png]]

### Archivo de Zona Inversa

```
; /etc/bind/zonas/db.172.16.79
$TTL    604800
@       IN      SOA     ns1.alumno79.com. admin.alumno79.com. (
                  2024022801         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.alumno79.com.
@       IN      NS      ns2.pana.com.

2       IN      PTR     ns1.alumno79.com.
3       IN      PTR     ns2.alumno79.com.
1       IN      PTR     firewall.alumno79.com.
4       IN      PTR     interna1.alumno79.com.
5       IN      PTR     interna2.alumno79.com.
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/19.png]]

### Verificación del Funcionamiento

```bash
# Verificar estado del servicio
sudo systemctl status bind9

# Validar archivos de configuración
sudo named-checkconf
sudo named-checkzone alumno79.com /etc/bind/zones/db.alumno79.com
sudo named-checkzone 79.16.172.in-addr.arpa /etc/bind/zones/db.172.16.79

# Ver logs del servicio
sudo journalctl -u bind9

# Consultar registros del servidor DNS
dig @172.16.79.2 alumno79.com
dig @172.16.79.2 ns1.alumno79.com
dig @172.16.79.2 ftp.alumno79.com
dig @172.16.79.2 -x 172.16.79.4

# Verificar transferencia de zona
dig @172.16.79.2 alumno79.com AXFR

# Verificar puertos abiertos
sudo netstat -tulapn | grep named
sudo ss -tulapn | grep :53

# Ver estadísticas del servidor DNS
sudo rndc stats
cat /var/cache/bind/named.stats

# Verificar funcionalidad con host
host ftp.alumno79.com 172.16.79.2
host 172.16.79.4 172.16.79.2
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/20.png]]

![[SR/DOCUMENTACIÓN-FINAL/capturas/21.png]]

![[SR/DOCUMENTACIÓN-FINAL/capturas/22.png]]

## Servidor DNS2

### Configuración Principal

```
# /etc/bind/named.conf.local

# Zona secundaria para alumno79.com
zone "alumno79.com" {
  type slave;
  file "/var/cache/bind/db.alumno79.com";
  masters { 172.16.79.2; };
};

# Secundaria para zona inversa
zone "79.16.172.in-addr.arpa" {
  type slave;
  file "/var/cache/bind/db.172.16.79";
  masters { 172.16.79.2; };
};

# Zona primaria para pana.com
zone "pana.com" {
  type master;
  file "/etc/bind/zones/db.pana.com";
};
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/23.png]]

### Archivo de Zona pana.com

```
; /etc/bind/zones/db.pana.com
$TTL    604800
@       IN      SOA     ns2.pana.com. root.pana.com. (
                  2024022801         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns2.pana.com.
@       IN      NS      ns1.alumno79.com.
@       IN      A       172.16.79.3

ns2     IN      A       172.16.79.3
debian2 IN      A       172.16.79.3
interna2 IN     A       172.16.79.5
debian4 IN      A       172.16.79.5
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/25.png]]

### Verificación del Funcionamiento

```bash
# Verificar estado del servicio
sudo systemctl status bind9

# Validar archivos de configuración
sudo named-checkconf
sudo named-checkzone pana.com /etc/bind/zones/db.pana.com

# Ver logs del servicio
sudo journalctl -u bind9

# Verificar que las zonas secundarias se hayan transferido
ls -la /var/cache/bind/

# Consultar registros del servidor DNS secundario
dig @172.16.79.3 alumno79.com
dig @172.16.79.3 ftp.alumno79.com
dig @172.16.79.3 -x 172.16.79.4

# Consultar registros del dominio primario
dig @172.16.79.3 pana.com
dig @172.16.79.3 www.pana.com

# Verificar transferencia de zona
dig @172.16.79.3 pana.com AXFR

# Forzar la actualización desde el servidor primario
sudo rndc retransfer alumno79.com
sudo rndc reload

# Verificar que el servidor está recibiendo consultas
sudo rndc status
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/26.png]]

![[SR/DOCUMENTACIÓN-FINAL/capturas/27.png]]

![[SR/DOCUMENTACIÓN-FINAL/capturas/28.png]]
## Servidor FTP

### Configuración de Red

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses: [172.16.79.4/24]
      gateway4: 172.16.79.1
      nameservers:
        addresses: [172.16.79.2, 172.16.79.3]
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/29.png]]

### Configuración de vsftpd

```
# /etc/vsftpd.conf
# Configuración básica
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES

# Configuración de chroot
chroot_local_user=YES
allow_writeable_chroot=YES

# Configuración de usuarios virtuales
guest_enable=YES
guest_username=ftp
user_config_dir=/etc/vsftpd/users
virtual_use_local_privs=YES
pam_service_name=vsftpd.virtual

# Configuración para modo pasivo
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=31000
pasv_address=192.168.35.79  # IP pública del router

# Habilitar registro
xferlog_std_format=YES
xferlog_file=/var/log/vsftpd.log
log_ftp_protocol=YES
listen=YES
listen_ipv6=NO
secure_chroot_dir=/var/run/vsftpd/empty
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/30.png]]

### Configuración de PAM para Usuarios Virtuales

```
# /etc/pam.d/vsftpd.virtual
auth required pam_pwdfile.so pwdfile=/etc/vsftpd/ftpd.passwd
account required pam_permit.so
```


![[SR/DOCUMENTACIÓN-FINAL/capturas/31.png]]
### Configuración de Usuarios

```bash
# Verificar el archivo de contraseñas
cat /etc/vsftpd/ftpd.passwd

# Listar configuraciones de usuarios virtuales
ls -la /etc/vsftpd/users/

# Ver la configuración del primer usuario
cat /etc/vsftpd/users/usuario1
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/32.png]]

### Directorios y Permisos

```bash
# Verificar directorios y permisos
ls -la /home/ftp/
ls -la /home/ftp/usuario1/
ls -la /home/ftp/usuario2/

# Verificar propietario del usuario del sistema
id ftp

# Verificar existencia del directorio chroot seguro
ls -la /var/run/vsftpd/empty
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/33.png]]

### Verificación del Funcionamiento

```bash
# Verificar estado del servicio
sudo systemctl status vsftpd

# Ver logs del servicio
sudo tail -50 /var/log/vsftpd.log

# Verificar puertos abiertos
sudo netstat -tulapn | grep vsftpd
sudo ss -tulapn | grep :21

# Verificar módulos del kernel para FTP
lsmod | grep conntrack_ftp

# Verificar conexiones actuales
sudo netstat -natp | grep vsftpd

# Verificar configuración
sudo vsftpd -v
sudo vsftpd -olisten=NO

# Reiniciar el servicio
sudo systemctl restart vsftpd
```

![[SR/DOCUMENTACIÓN-FINAL/capturas/34.png]]

## Pruebas de Funcionamiento

### Prueba de DNS

```bash
# Resolución de nombres desde distintos clientes
nslookup ftp.alumno79.com 172.16.79.2
nslookup ftp.alumno79.com 172.16.79.3
nslookup pana.com 172.16.79.3

# Resolución inversa
nslookup 172.16.79.4 172.16.79.2

# Consultas usando dig
dig @172.16.79.2 ftp.alumno79.com
dig @172.16.79.3 ftp.alumno79.com

# Verificar el uso del servidor DNS configurado
cat /etc/resolv.conf
host ftp.alumno79.com
```

![[35.png]]

### Prueba de Conectividad

```bash
# Ping a todos los servidores
ping -c 4 172.16.79.1
ping -c 4 172.16.79.2
ping -c 4 172.16.79.3
ping -c 4 172.16.79.4
ping -c 4 172.16.79.10
ping -c 4 172.16.49.1

# Traceroute a servidores
traceroute 172.16.79.4
traceroute 172.16.49.1

# Verificar rutas
ip route get 172.16.79.4
ip route get 172.16.49.1
ip route get 8.8.8.8
```

![[36.png]]

### Prueba de FTP desde Red Interna

```bash
# Conectar al servidor FTP por IP
ftp 172.16.79.4

# Conectar al servidor FTP por nombre DNS
ftp ftp.alumno79.com

# Comandos para probar una vez conectado
pwd
mkdir prueba_interna
cd prueba_interna
put archivo_prueba.txt
ls -la
```

![[37.png]]

### Prueba de FTP desde Red Externa

```bash
# Conectar al servidor FTP a través del router
ftp 192.168.35.79

# Comandos para probar una vez conectado
pwd
mkdir prueba_externa
cd prueba_externa
put archivo_prueba_externo.txt
ls -la

# Habilitar modo pasivo explícitamente
passive
put archivo_modo_pasivo.txt
```

![[38.png]]

### Verificación de Acceso a Internet

```bash
# Prueba de ping a servidores DNS públicos
ping -c 4 8.8.8.8
ping -c 4 1.1.1.1

# Resolución de nombres de Internet
nslookup www.google.com
nslookup www.debian.org

# Traceroute a Internet
traceroute www.google.com

# Verificar la tabla de NAT en el router
sudo iptables -t nat -L -n -v
```


---
