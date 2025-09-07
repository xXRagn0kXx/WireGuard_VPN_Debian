# WireGuard Install on Debian 13

WireGuard es una de las VPN mas eficientes que existen para linux. 
Su sistema se basa en pares de claves(como en el SSH) lo que para nosotros seria como un "certificado".
En esta guia tambien se configurara su integracion con nftables, no vamos descuidar la seguridad.

![Portada de WireGuard Install](WireGuard_Install.png)

# 1 Instalar software
```bash
sudo apt install wireguard nftables
```
# 2 Activar redireccion de paquetes
2 Editar el sysctl.conf para permitir redireciones de trafico a nivel kernel.
```bash
        - nano /etc/sysctl.conf
```
Descomentar ependiendo si usamos IPv4 o IPv6
```conf
net.ipv4.ip_forward = 1
#net.ipv6.conf.all.forwarding=1
```
Guardamos el archivo

Aplicar con:
```bash
         - sysctl -p
```

# 3 Permitir trafico en el nftables
```bash 
sudo nano /etc/nftables.conf
```

```conf
#!/usr/sbin/nft -f
flush ruleset

table inet filter {

chain input {
                type filter hook input priority 0; policy drop;

        # Permitir conexiones ya establecidas o relacionadas
                ct state established,related accept

        # Permitir tráfico en la interfaz local (loopback)
                iifname "lo" accept

        # Permitir ICMP (ping) - solo echo-request y echo-reply
                ip protocol icmp icmp type { echo-request, echo-reply } accept

        # [Opcional] Permitir conexiones TCP (puerto 22) y limitar nuevas conexiones a 10 por minuto añadiendolas a un contador
        #       tcp dport 22 ct state new limit rate 10/minute counter accept
        # [Opcional] Permitir conexiones aolo SSH (puerto 22) y limitar nuevas conexiones a 10 por minuto añadiendolas a un contador
        #        tcp dport 22 ct state new tcp flags syn limit rate 4/minute counter accept
        
        # WireGuard (protegido igual que SSH pero para UDP para WireGuard en el puerto 51820)
                udp dport 51820 ct state new limit rate 10/minute counter accept
}

chain forward {
                type filter hook forward priority 0; policy drop;

         # Permitir tráfico entre WireGuard y la red local
                iifname "wg0" oifname "enP3p49s0" accept  # Cambia "eth0" por tu interfaz LAN
                iifname "enP3p49s0" oifname "wg0" ct state established,related accept

        # Permitir tráfico específico desde 10.10.0.1 hacia 192.168.1.0/24
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

Guardamos

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
Habilitamos arrnque automatico:
```bash
sudo systemctl enable nftables.service
```
# 4 Generar la clave del servidor

4.1 Generar la clave privada del servidor Wireguard
        Una vez instalado el paquete wireguard, la siguiente tarea es generar los certificados del servidor,
        lo que puede hacerse utilizando la herramienta de línea de comandos wg.

        Ejecuta el siguiente comando para generar la clave privada del servidor wireguard en /etc/wireguard/server.key
        A continuación, cambia el permiso de la clave privada del servidor a 0400, lo que significa que deshabilitarás el acceso de escritura al archivo.

                - sudo wg genkey | sudo tee /etc/wireguard/server.key
                - sudo chmod 0400 /etc/wireguard/server.key

        NOTA: Para modificar ese fichero de nuevo habra que volver a modificarle los permisos antes de editar.

4.2 A continuación, ejecuta el siguiente comando para generar la clave pública del servidor wireguard en /etc/wireguard/server.pub.

        - sudo cat /etc/wireguard/server.key | wg pubkey | sudo tee /etc/wireguard/server.pub
        
        Securizamos el fichero con:
        - sudo chmod 0400 /etc/wireguard/server.pub

    Comprobamos:
        - cat /etc/wireguard/server.key
        - cat /etc/wireguard/server.pub

5 Generar la clave del cliente

	- mkdir -p /etc/wireguard/clients/user
        - wg genkey | tee /etc/wireguard/clients/user/user.key
        - cat /etc/wireguard/clients/user/user.key | wg pubkey | tee /etc/wireguard/clients/user/user.pub

    Comprobamos:
        - cat /etc/wireguard/clients/user/user.key
        - cat /etc/wireguard/clients/user/user.pub


6 Crear el ficheo de configuracion del servidor 

Crea una nueva configuración de Wireguard /etc/wireguard/wg0.conf
        - sudo nano /etc/wireguard/wg0.conf

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


Guardamos
###########################################################################################################

7 Gestion del servidor 

        Para iniciar y habilitar el servidor wireguard, ejecuta el siguiente comando systemctl.
        Con el nombre de servicio wg-quick@wg0, iniciarás el Wireguard dentro de la interfaz wg0,
        que se basa en la configuración /etc/wireguard/wg0.conf.

        - sudo systemctl start wg-quick@wg0.service
        - sudo systemctl enable wg-quick@wg0.service
   Comprobamos:
        - sudo systemctl status wg-quick@wg0.service

A continuación, ejecuta el siguiente comando ip para mostrar los detalles de la interfaz wg0 de wireguard. Y deberías ver que la interfaz wg0 de wireguard tiene una dirección IP 10.10.0.1.
        - sudo ip a show wg0

También puedes iniciar o detener el wireguard manualmente mediante el comando wg-quick que aparece a continuación.

sudo wg-quick up /etc/wireguard/wg0.conf
sudo wg-quick down /etc/wireguard/wg0.conf


