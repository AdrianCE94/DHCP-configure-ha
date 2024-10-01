# DHCP-configure-ha
<div align="center">
  <img src="image.png" alt="Logo" width="300">
</div>

Configuración de servidor DHCP con failover y relay para alta disponibilidad. En este repositorio, encontrarás scripts y configuraciones para establecer un entorno de DHCP con redundancia y failover, que incluye la configuración de un relay DHCP para encaminar solicitudes DHCP entre subredes. Asegura la disponibilidad del servicio DHCP en tu red.

## 1.STACK 

- 4 MAQUINAS VIRTUALES (debian12)
  - 1 Servidor DHCP Primario
  - 1 Servidor DHCP Secundario (Failover)
  - 1 Router con DHCP Relay
  - 1 Cliente

- Virtualbox 7.0 (con extension pack)  

**_NOTA_**: modo promiscuo en las interfaces de red siempre activado
## 2.TOPOLOGÍA

DHCPSERVER -> RELAY -> CLIENTE

DHCPERVER RESPALDO

![top](image-1.png)

## 3.CONFIGURACIÓN DHCP SERVER PRIMARIO

Con el adaptador de red en modo puente para descargar el isc-dhcp-server

```bash
sudo apt update
sudo apt install isc-dhcp-server -y
```

![install](image-2.png)


**_NOTA_**: al comprobar el estado del servicio veras que esta failed , no te preocupes es normal pues no esta configurado. Si utilizamos `journalctl -xe` o `journalctl -u isc-dhcp-server` podremos ver el error.

![error](image-3.png)


Ahora cambiamos el adaptador de red a red interna y configuramos el archivo `/etc/network/interfaces`

`ip a` para ver nuestras interfaces de red
![ipa](image-4.png)

![stato](image-5.png)

En mi caso , voy a asignarle a mi servidor la ip 192.168.1.1
Aplicamos los cambios(ctrl O + ctrl X) y reiniciamos el servicio
```bash
systemctl restart networking.service
ip a #para comprobar que se ha asignado la ip
ip r #para comprobar la tabla de rutas
```
## 3.1 ARCHIVOS DE CONFIGURACION DEL SERVIDOR PRIMARIO

- `/etc/default/isc-dhcp-server` : aquí añadiremos nuestra interfaz “enp0s3” en INTERFACESv4.

![enpose](image-6.png)

- `/etc/dhcp/dhcpd.conf` : aqui añadiremos en el apartado para una red interna, la red, la mascara y el rango que va a proporcionar de ips.

![range](image-7.png)
---
En mi caso , subnet 192.168.1.0 netmask 255.255.255.0 { range 192.168.1.10 192.168.1.50;
option routers 192.168.1.1;
}

**_NOTA_**: es opcional pero se recomienda añadir el option routers, nos servirá posteriormente para el relay.

Por último, reiniciamos el servicio y comprobamos que no haya errores y ya quedaria levantado el servidor primario.

![running](image-8.png)

```bash
systemctl restart isc-dhcp-server
systemctl status isc-dhcp-server
```

**_NOTA_**: `/var/lib/dhcp/dhcpd.leases` : archivo donde se guardan las ips asignadas a los clientes.De momento estará vacío.

## 4.CONFIGURACIÓN DHCP CLIENT

En la maquina cliente, cambiamos el adaptador de red a red interna y vamos a usar dos comandos para obtener una ip. Recordar que puede ser que necesitemos ejecutar el comando como superusuario.
```bash
dhclient -r
dhclient -v
```

![offer](image-9.png)
![request](image-10.png)

aqui podemos ver como el cliente ha obtenido la ip de nuestro servidor dhcp.Le asigna la ip tras una petición de la ip. El servidor recibe una request , hace una offer y el cliente la acepta.

con `ip a` podemos ver la ip asignada al cliente.
en mi caso nos ha dado una del rango confiuurado en el servidor.
