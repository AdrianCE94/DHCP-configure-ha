# DHCP-configure-ha
<div align="center">
  <img src="imgs/logo" alt="Logo">
</div>

Configuración de servidor DHCP con failover y relay para alta disponibilidad. En este repositorio, encontrarás scripts y configuraciones para establecer un entorno de DHCP con redundancia y failover, que incluye la configuración de un relay DHCP para encaminar solicitudes DHCP entre subredes. Asegura la disponibilidad del servicio DHCP en tu red.

## 1. STACK 

- 4 MAQUINAS VIRTUALES (debian12)
  - 1 Servidor DHCP Primario
  - 1 Servidor DHCP Secundario (Failover)
  - 1 Router con DHCP Relay
  - 1 Cliente

- Virtualbox 7.0 (con extension pack)  

**_NOTA_**: modo promiscuo en las interfaces de red siempre activado
## 2. TOPOLOGÍA Y CONCEPTOS

![TOP](image.png)

DHCPSERVER PRIMARIO -> DHCPSERVER SECUNDARIO (RESPALDO FAILOVER)

DHCPSERVER PRIMARIO -> ROUTER (RELAY) (para encaminar solicitudes DHCP entre subredes)

DHCP CLIENT -> DHCPSERVER PRIMARIO (para obtener una dirección IP)


## 3. CONFIGURACIÓN DHCP SERVER PRIMARIO

Con el adaptador de red en modo puente para descargar el isc-dhcp-server

```bash
sudo apt update
sudo apt install isc-dhcp-server -y
```

![install](image-2.png)


**_NOTA_**: al comprobar el estado del servicio veras que esta failed , no te preocupes es normal pues no esta configurado. Si utilizamos `journalctl -xe` o `journalctl -u isc-dhcp-server` podremos ver el error.

![error](image-3.png)


Ahora cambiamos el adaptador de red a red interna(le llamaremos RED1) y configuramos el archivo `/etc/network/interfaces`

`ip a` para ver nuestras interfaces de red
![ipa](image-4.png)

En mi caso , voy a asignarle a mi servidor la ip 192.168.1.1 (Cuando utilicemos el relay pondremos la puerta de enlace como la ip del relay)
Aplicamos los cambios(ctrl O + ctrl X) y reiniciamos el servicio
![netstatic](image-1.png)



```bash
systemctl restart networking.service
ip a #para comprobar que se ha asignado la ip
ip r #para comprobar la tabla de rutas
```
## 3.1 ARCHIVOS DE CONFIGURACION DEL SERVIDOR PRIMARIO

- `/etc/default/isc-dhcp-server` : aquí añadiremos nuestra interfaz “enp0s3” en INTERFACESv4.

![enpose](image-6.png)

- `/etc/dhcp/dhcpd.conf` : aqui añadiremos en el apartado para una red interna, la red, la mascara y el rango que va a proporcionar de ips.
  ![conf](image-19.png)

En mi caso , subnet 192.168.1.0 netmask 255.255.255.0 
range 192.168.1.10 192.168.1.50;
---


**_NOTA_**: es opcional pero se recomienda añadir el option routers, nos servirá posteriormente para el relay.

Por último, reiniciamos el servicio y comprobamos que no haya errores y ya quedaria levantado el servidor primario.

![running](image-8.png)

```bash
systemctl restart isc-dhcp-server
systemctl status isc-dhcp-server
```

**_NOTA_**: `/var/lib/dhcp/dhcpd.leases` : archivo donde se guardan las ips asignadas a los clientes.De momento estará vacío.

## 4. CONFIGURACIÓN DHCP CLIENT

En la maquina cliente, cambiamos el adaptador de red a red interna(RED1) y vamos a usar dos comandos para obtener una ip. Recordar que puede ser que necesitemos ejecutar el comando como superusuario.
```bash
dhclient -r
dhclient -v
```
![alt text](image-9.png)
![client2](image-10.png)

aqui podemos ver como el cliente ha obtenido la ip de nuestro servidor dhcp.Le asigna la ip tras una petición de la ip. El servidor recibe una request , hace una offer y el cliente la acepta.

con `ip a` podemos ver la ip asignada al cliente.
en mi caso nos ha dado una del rango confiuurado en el servidor.

Si vamos al archivo `/var/lib/dhcp/dhcpd.leases` veremos que se ha añadido una entrada con la ip asignada al cliente.
Podemos usar `tail -f /var/lib/dhcp/dhcpd.leases` para ver en tiempo real las ips asignadas a los clientes.

![leases](image-11.png)



## 5. CONFIGURACIÓN DHCP RELAY

 ### 5.1 instalación y configuración del relay
Antes de nada, vamos a instalar el paquete isc-dhcp-relay en nuestra maquina saliendo a internet con adaptador puente.

```bash
sudo apt update
sudo apt install isc-dhcp-relay -y
```

![rel](image-12.png)
Aqui le asignamos la ip del servidor dhcp primario.(en nuestro caso 192.168.1.1) y aprovechamos para añadir la ip del failover que tendra la ip 192.168.1.2 según nuestro esquema.

![alt text](image-13.png)
Aqui le ponemos la interfaz por la que va a escuchar las peticiones dhcp.(en nuestro caso enp0s3 Y enp0s8)

En la tercera opcion no añadiremos nada

**_NOTA_**: el fichero donde configuramos el relay es `/etc/default/isc-dhcp-relay` por si después queremos modificar algo.
---
Tenemos una máquina virtual con dos adaptadores de red interna, red 1  para el servidor dhcp y failover y red 2 para el cliente.Todo esto tenemos que hacerlo en virtualbox en los ajustes de la configuración de la máquina.
![VIPER](image-14.png)

 `ip a` par ver las interfaces de red -> tenemos enp0s3 y enp0s8

![interfaces](image-15.png)

(tenemos ip porque el dhcp server esta funcionando y la interfaz enp0s3 esta en en la misma red que el servidor)

enp0s3 -> red interna con el servidor dhcp

enp0s8 -> red interna con el cliente

¡¡¡¡¡¡¡¡¡NO OLVIDAR!!!!!!!

 ### 5.2 configuración de las interfaces de red

Configuramos el archivo `/etc/network/interfaces` para asignarle una ip a la interfaz enp0s3 y a la interfaz enp0s8.
```bash
nano /etc/network/interfaces
```
![staitc relay](image-16.png)

guardamos los cambios y reiniciamos el servicio de red
```bash
systemctl restart networking.service
```
**_IMPORTANTE_**: EN NUESTRO SERVIDOR DHCP PONDREMOS COMO GATEWAY LA IP DE LA INTERFAZ ENP0S3 DEL RELAY.

![GATE](image-25.png)

### 5.3 habilitar el forwarding para el reenvio de paquetes entre interfaces de red
Modificación en /proc/sys/net/ipv4/ip_forward
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

![echo](image-18.png)
---
descomentar la linea `net.ipv4.ip_forward=1`
```bash
nano /etc/sysctl.conf
```
![descomentar](image-17.png)


---
por ultimo en este apartado vamos a añadir la interfaz
enp0s8 en el archivo `/etc/default/isc-dhcp-relay` para que escuche las peticiones dhcp si no lo hemos hecho ya.

```bash	
nano /etc/default/isc-dhcp-relay
```

Reiniciamos el servicio y comprobamos que no haya errores.

![alt text](image-27.png)
```bash


## 6. PROBAR CONECTIVIDAD

Ahora vamos a probar la conectividad entre el cliente y el servidor dhcp.Para ello pondremos una ip statica en el cliente y haremos ping al servidor dhcp utilizando como gateway la ip del relay de su red.(red2)

![client](image-28.png)
aqui podemos ver como el cliente tiene conectividad con el servidor dhcp.

ping a la ip del servidor dhcp



```bash