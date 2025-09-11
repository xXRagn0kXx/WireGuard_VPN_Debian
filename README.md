# WireGuard VPN Install on Debian

WireGuard es una de las VPN mas eficientes que existen para linux. 
Su sistema se basa en pares de claves(como en el SSH) lo que para nosotros seria como un "certificado".

En esta guia tambien se configurara su integracion con nftables..

![Portada de WireGuard Install](WireGuard_Install.png)

# 1 Instalar software
Instalamos la paqueteria necesaria:
```bash
sudo apt install nftables install wireguard wireguard-tools 
```
# 2 Activar redireccion de paquetes
2 Editar el sysctl.conf para permitir redireciones de trafico a nivel kernel.
```bash
        - nano /etc/sysctl.conf
```
Descomentar dependiendo si usamos IPv4 o IPv6:
```conf
net.ipv4.ip_forward = 1
#net.ipv6.conf.all.forwarding=1
```

Aplicar cambios con:
```bash
sysctl -p
```

# 3 Configurar nftables
Por defecto nftables viene vacio completamente.

Este fichero es el nucleo de nuestro firewall, comprenderlo nos dara un profundo conocimiento
sobre la seguridad de nuestra maquina ya que todo pasara por aqui.

Habra que especificar todos aquellos servicios que queramos permitir, se han añadido comentados algunos como el ping o el SSH.
He creado esta estructura basica para que wireguard pueda usarlo

Creamos el fichero /etc/nftables.conf:
```bash 
sudo nano /etc/nftables.conf
```
Modificar plantilla al gusto:
```conf
#!/usr/sbin/nft -f
flush ruleset

table inet filter {

chain input {
                type filter hook input priority 0; policy drop;

        # Permitir conexiones ya establecidas o relacionadas
                ct state established,related accept

        # Permitir trafico en la interfaz local (loopback)
                iifname "lo" accept
        # WireGuard (permitir UDP para WireGuard en el puerto 51820)
        udp dport 51820 ct state new limit rate 10/minute counter accept

        # Permitir ICMP (ping) - solo echo-request y echo-reply
        #        ip protocol icmp icmp type { echo-request, echo-reply } accept

        # [Opcional] Permitir conexiones TCP (puerto 22) y limitar nuevas conexiones a 10 por minuto añadiendolas a un contador
        #       tcp dport 22 ct state new limit rate 10/minute counter accept
        # [Opcional] Permitir conexiones aolo SSH (puerto 22) y limitar nuevas conexiones a 10 por minuto añadiendolas a un contador
        #        tcp dport 22 ct state new tcp flags syn limit rate 4/minute counter accept
        

}

chain forward {
                type filter hook forward priority 0; policy drop;

         # Permitir trafico entre WireGuard y la red local
                iifname "wg0" oifname "enP3p49s0" accept  # Cambia "eth0" por tu interfaz LAN
                iifname "enP3p49s0" oifname "wg0" ct state established,related accept

        # Permitir trafico especifico desde 10.10.0.1 hacia 192.168.1.0/24
                ip saddr 10.10.0.1 ip daddr 192.168.1.0/24 accept
}

chain output {
                type filter hook output priority 0; policy accept;
        }
chain nat {
        type nat hook postrouting priority 100; policy accept;
        ip saddr 10.10.10.0/24 oifname "eth0" masquerade
        }
}
```

:warning: (IMPORTANTE) Una vez guardado el archivo, revisa la configuracion ejecutando:
   ```bash
   sudo nft -f /etc/nftables.conf
   ```
:white_check_mark: Si el comando no duelve nada el fichero esta correcto. Si todo esta ok activamos el firewall

Arrancar el servicio para aplicar:
```bash
sudo systemctl start nftables.service
```

Comprobamos que no tiene errores:
```bash
sudo systemctl status nftables.service
```

Habilitamos arranque automatico si todo esta correcto:
```bash
sudo systemctl enable nftables.service
```
:warning: Recuerda hacer "nft -f /etc/nftables.conf" cada vez que toques el fichero siempre. 

# 4 Generar las claves del servidor
En este apartado generaremos las claves necesarias para que la conexion sea posible.

* WireGuard no usa usuario y contraseña
* Funciona con pares de claves publica-privada
* Cada clave publica se crea a partir de su privada
* Las claves estan generadas en BASE-64
* Habra que crear un par para el servidor y otro para cada hosts que queramos conectar
* Asignaremos permisos restrictivos para no poder modificar las claves accidentalmente

## 4.1 Metodo automatizado
Tenemos 2 metodos para crear todos los certificados, mediante script o manualmente:

 ### Certificados mediante Script
Para no alargar esta guia lo he alojado en otro repositorio.
* Es un Script creado por mi para facilitar la gestion diaria.
* Permite Crear certificados del servidor y cliente
* Comprueba correspondencia entre las claves publicas y privadas
* Borrar certificados del servidor y cliente
* Elimina errores manuales de gestion

Para ver el script visita el repositorio de [WireGuard CertMaker Debian](https://github.com/xXRagn0kXx/WireGuard_CertMaker_Debian/blob/main/README.md).

## 4.2 Generar manualmente claves del servidor
Podemos crear los 2 pares a la vez o por separado, a la vez lo haremos asi:
```bash
cd /etc/wireguard/
wg genkey | tee server.key | wg pubkey > server.pub
sudo chmod 0400 /etc/wireguard/server.*
```

:warning: A continuacion desglosaremos por separado, si ya la creaste saltar al "4.3 Comprobar las claves".

## 4.3 Generar manualmente clave privada del servidor:

Una vez instalado wireguard, ahora tendremos la herrmaienta wg para crear pares de claves.

El siguiente paso es generar los certificados del servidor.

Esto puede hacerse utilizando la herramienta de linea de comandos wg.

Ejecuta el siguiente comando para generar la clave privada del servidor wireguard en /etc/wireguard/server.key

```bash
sudo wg genkey > server.key
sudo chmod 0400 /etc/wireguard/server.key
```
:warning: NOTA: Para modificar ese fichero de nuevo habra que volver a modificarle los permisos antes de editar.

## 4.2 Generar manualmente clave publica del servidor.
A continuacion, ejecuta el siguiente comando para generar la clave pública del servidor wireguard en /etc/wireguard/server.pub.

```bash
sudo wg pubkey < server.key > server.pub
sudo chmod 0400 /etc/wireguard/server.pub
```

## 4.3 Comprobar las claves
Comprobamos que contienen las claves:
```bash
cat /etc/wireguard/server.key
cat /etc/wireguard/server.pub
```

# 5 Generar manualmente la clave del cliente
Como wireguard por defecto viene vacio, primero crearemos una esctructura de directorios organizada donde iremos almacenando las claves de los clientes que generemos, asi podremos administrar la VPN de forma ordenada.
Todo los clientes penderan de la carpeta clients
Crearemos la carpeta "clients" y luego
```bash
sudo mkdir /etc/wireguard/clients
```
Dentro de la carpeta clients crearemos una por cada usuario donde se almacenaran sus claves.
```bash
sudo mkdir /etc/wireguard/clients/user1
```
Para crear las claves el procedimiento es similar a la publica pero cambiando la ruta y el nombre de los ficheros.

Los metodos son los mismos: 

```bash
cd /etc/wireguard/clients/user1
 wg genkey | tee user1.key | wg pubkey > user1.pub
```

Comprobamos:
```bash
cat /etc/wireguard/clients/user/user.key
cat /etc/wireguard/clients/user/user.pub
```

# 6 Crear el ficheo de configuracion del servidor 

Crea una nueva configuracion de Wireguard /etc/wireguard/wg0.conf
```bash
sudo nano /etc/wireguard/wg0.conf
```
```conf
[Interface]
# Wireguard Server private key - server.key
PrivateKey = cNBb6MGaKhmgllFxSq/h9BdYfZOdyKvo8mjzb2STbW8=
# Wireguard interface will be run at 10.10.0.1
Address = 10.10.0.1/24

# Clients will connect to UDP port 51820
ListenPort = 51820

# Ensure any changes will be saved to the Wireguard config file
#SaveConfig = true

[Peer]
# Wireguard first client public key - user.pub
PublicKey = 3ZoaoVgHOioZnKzCrF/XALAv70V4vyJXpl/UO7AKYzA=
# clients' VPN IP addresses you allow to connect
AllowedIPs = 10.10.0.2/32

[Peer]
# Wireguard second client public key - user2.pub
PublicKey = 54gyrteoagHOiozCrF/XtrLAv7V4vyJXpl/UO7AK36g=
# clients' VPN IP addresses you allow to connect
AllowedIPs = 10.10.0.3/32
```

# 7 Gestion del servidor 

Para iniciar y habilitar el servidor wireguard, ejecuta el siguiente comando systemctl.
Con el nombre de servicio wg-quick@wg0, iniciaras el Wireguard dentro de la interfaz wg0,
que se basa en la configuracion /etc/wireguard/wg0.conf.
```bash
sudo systemctl start wg-quick@wg0.service
sudo systemctl enable wg-quick@wg0.service
```
   Comprobamos:
```bash
sudo systemctl status wg-quick@wg0.service
```
A continuacion, ejecuta el siguiente comando ip para mostrar los detalles de la interfaz wg0 de wireguard. Y deberias ver que la interfaz wg0 de wireguard tiene una direccion IP 10.10.0.1.
```bash
sudo ip a show wg0
```
Tambien puedes iniciar o detener el wireguard manualmente mediante el comando wg-quick que aparece a continuacion.

```bash
sudo wg-quick up /etc/wireguard/wg0.conf
sudo wg-quick down /etc/wireguard/wg0.conf
```
