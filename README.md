# Infraestructura de FINAL SRI

## Descripción del Proyecto

Implementación detallada de una infraestructura de red empresarial con servicios integrados de DHCP, DNS y FTP, diseñada para un entorno académico y de laboratorio.

## Características Principales

- **Topología de Red**
  - Red 1: 172.16.79.0/24 (Red principal)
  - Red 2: 172.16.49.0/24 (Red secundaria)

- **Servicios Implementados**
  - Servidor DHCP con failover
  - Servicios DNS (zonas primarias y secundarias)
  - Servidor FTP seguro
  - Firewall y NAT configurados

## Arquitectura de Red

### Componentes Principales

1. **Router**
   - Interfaces: 
     * Externa: 192.168.35.79
     * Interna: 172.16.79.1

2. **Servidor DHCP**
   - Dirección IP: 172.16.79.10
   - Funcionalidades avanzadas de asignación y configuración

3. **Servidores DNS**
   - Primario (172.16.79.2): alumno79.com
   - Secundario (172.16.79.3): alumno79.com, pana.com

4. **Servidor FTP**
   - Dirección IP: 172.16.79.4
   - Configuración segura con autenticación de usuarios virtuales

## Características de Seguridad

- Firewall con reglas de iptables
- Network Address Translation (NAT)
- Redireccionamiento de puertos
- Entornos chroot para servicios
- Usuarios virtuales para acceso FTP

## Requisitos del Sistema

- Sistema Operativo: Debian/Ubuntu
- Servicios:
  * ISC DHCP Server
  * BIND9 DNS Server
  * vsftpd
- Herramientas de red: iptables, netplan

## Contribuciones

¡Las contribuciones son bienvenidas! Por favor, sigue estos pasos:

1. Haz un fork del repositorio
2. Crea una nueva rama (`git checkout -b feature/mejora-importante`)
3. Realiza tus cambios y haz commit (`git commit -am 'Añadir nueva funcionalidad'`)
4. Sube tus cambios (`git push origin feature/mejora-importante`)
5. Abre una Pull Request


---

**Nota**: Esta infraestructura de red está diseñada para entornos de laboratorio y requiere adaptación para entornos de producción.