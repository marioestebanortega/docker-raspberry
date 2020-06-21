# Plex sobre Docker en Raspberry

Con este repo podes crear tu propio server que descarga tus series y peliculas automáticamente, y cuando finaliza, las copia al directorio `media/` donde Plex las encuentra y las agrega a tu biblioteca.

También agregué un pequeño server samba por si querés compartir los archivos por red

Todo esto es parte de unos tutoriales que estoy subiendo a [Youtube](https://www.youtube.com/playlist?list=PLqRCtm0kbeHCEoCM8TR3VLQdoyR2W1_wv)

## Requerimientos iniciales

Agregar tu usuario (cambiar `kbs` con tu nombre de usuario)

```
sudo useradd kbs -G sudo
```

Agregar esto al sudoers para correr sudo sin password

```
%sudo   ALL=(ALL:ALL) NOPASSWD:ALL
```

Agregar esta linea a `sshd_config` para que sólo tu usuario pueda hacer ssh

```
echo "AllowUsers kbs" | sudo tee -a /etc/ssh/sshd_config
sudo systemctl enable ssh && sudo systemctl start ssh
```

Instalar paquetes básicos

```
sudo apt-get update && sudo apt-get install -y \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common \
     vim \
     fail2ban \
     ntfs-3g
```

Instalar Docker

```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
echo "deb [arch=armhf] https://download.docker.com/linux/debian \
     $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update && sudo apt-get install -y docker-ce docker-compose
```

Modificá tu docker config para que guarde los temps en el disco:

```
sudo vim /etc/default/docker
# Agregar esta linea al final con la ruta de tu disco externo montado
export DOCKER_TMPDIR="/mnt/storage/docker-tmp"
```

Agregar tu usuario al grupo docker 

```
# Add kbs to docker group
sudo usermod -a -G docker kbs
#(logout and login)
docker-compose up -d
```

## Cómo correrlo

Simplemente bajate este repo y modificá las rutas de tus archivos en el archivo (oculto) .env:


```
version: "2"

services:

  samba:
    image: dperson/samba:rpi
    restart: always
    command: '-u "pi;password" -s "media;/media;yes;no" -s "downloads;/downloads;yes;no"'  <--- estos son los directorios que vamos a compartir con samba y sus credenciales
    stdin_open: true
    tty: true
    ports:
      - 139:130
      - 445:445
    volumes:
      - /usr/share/zoneinfo/America/Argentina/Mendoza:/etc/localtime   <--- modifica esto con tu zona horaria si queres
      - ${MEDIA}:/media
      - ${DOWNLOADS}:/downloads

  rtorrent:
    image: linuxserver/rutorrent:arm32v7-v3.9-ls47
    environment:
      - PUID=1000
      - PGID=1000
    ports:
      - 80:80
      - 51413:51413
      - 6881:6881/udp
    volumes:
      - ${TORRENTS_CONFIG}:/config
      - ${MEDIA}:${MEDIA}
      - ${DOWNLOADS}:/downloads
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: always

  plex:
    image: jaymoulin/plex:1.16.3-armhf
    ports:
      - 32400:32400
      - 33400:33400
    volumes:
      - ${PLEX_CONFIG}:/root/Library/Application Support/Plex Media Server
      - ${MEDIA}:/media
    restart: always
    network_mode: "host"

```

## IMPORTANTE

Las raspberry son computadoras excelentes pero no muy potentes, y plex por defecto intenta transcodear los videos para ahorrar ancho de banda (en mi opinión, una HORRIBLE idea), y la chiquita raspberry no se aguanta este transcodeo "al vuelo", entonces hay que configurar los CLIENTES de plex (si, hay que hacerlo en cada cliente) para que intente reproducir el video en la máxima calidad posible, evitando transcodear y pasando el video derecho a tu tele o Chromecast sin procesar nada, de esta forma, yo he tenido 3 reproducciones concurrentes sin ningún problema. En android y iphone las opciones son muy similares, dejo un screenshot de Android acá:

<img src="https://i.imgur.com/F3kZ9Vh.png" alt="plex" width="400"/>

Más info acá: https://github.com/pablokbs/plex-rpi/issues/3
