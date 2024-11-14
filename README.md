# Cliente-servidor-DNS
##Pasos para Configuración Cliente-Servidor DNS
###Paso 1: Crear el archivo docker-compose.yml
---

Este archivo configura tanto el servidor DNS como el cliente, ambos en contenedores de Docker.

    Crea un directorio para el proyecto: En tu máquina local, crea un directorio donde guardarás todos los archivos de configuración:
```
mkdir dns_project
cd dns_project
```
Crea el archivo docker-compose.yml: En el directorio del proyecto, crea un archivo llamado docker-compose.yml con el siguiente contenido:
```
version: '3'
```
services:
  asir_bind9:
    container_name: Practica6_bind9
    image: ubuntu/bind9
    platform: linux/amd64
    ports:
      - "54:53"  # Exponer el puerto 54 para que otros puedan hacer consultas DNS
    networks:
      bind9_subnet:
        ipv4_address: 172.28.5.1  # IP estática del servidor DNS
    volumes:
      - ./conf:/etc/bind/  # Montar la configuración del servidor BIND
      - ./zonas:/var/lib/bind/  # Montar el directorio de zonas
    restart: unless-stopped  # Reiniciar el contenedor si falla o si Docker se reinicia

  cliente:
    container_name: Prac6_alpine
    image: alpine
    platform: linux/amd64
    tty: true
    stdin_open: true
    dns:
      - 172.28.5.1  # El cliente usará el DNS del servidor asir_bind9
    networks:
      bind9_subnet:
        ipv4_address: 172.28.5.2  # IP estática del cliente en la misma subred
    restart: unless-stopped  # Asegurarse de que el cliente también se reinicie si falla

networks:
  bind9_subnet:
    driver: bridge  # Tipo de red 'bridge'
    ipam:
      config:
        - subnet: "172.28.0.0/16"  # Rango de subred
          ip_range: "172.28.5.0/24"  # Rango de IPs para los contenedores
          gateway: "172.28.5.254"  # Puerta de enlace de la red

- Explicación:

    asir_bind9: Configura el servidor DNS usando la imagen de Ubuntu con Bind9.
        Expone el puerto 53 (DNS) en el puerto 54 del host.
        Se le asigna una IP estática (172.28.5.1).
        Usa directorios locales para configurar Bind9 y las zonas DNS.
    cliente: Configura el cliente usando la imagen de Alpine Linux.
        Se le asigna una IP estática (172.28.5.2).
        Configura el DNS del cliente para que apunte al servidor DNS 172.28.5.1.

Crea los directorios de configuración: Dentro de dns_project, crea los directorios conf y zonas donde se almacenarán los archivos de configuración de Bind9:

    mkdir conf zonas

Paso 2: Crear Archivos de Configuración para Bind9

    Archivo conf/named.conf: Este archivo contiene la configuración básica de Bind9. Crea el archivo conf/named.conf con el siguiente contenido:

options {
  directory "/var/cache/bind";
  forwarders {
    8.8.8.8;
  };
  allow-query { any; };
};

zone "asircastelao.int" {
  type master;
  file "/var/lib/bind/db.asircastelao.int";
};

Explicación:

    Define las opciones básicas para Bind9.
    Establece un "forwarder" (servidor DNS al que redirigir las consultas no resueltas) a Google DNS (8.8.8.8).
    Crea una zona llamada asircastelao.int que usará el archivo db.asircastelao.int para los registros.

Archivo zonas/db.asircastelao.int: Este archivo define los registros DNS de la zona asircastelao.int. Crea el archivo zonas/db.asircastelao.int con el siguiente contenido:

    $TTL    86400
    @       IN      SOA     ns.asircastelao.int. root.asircastelao.int. (
                          2023111401 ; Serial
                          3600       ; Refresh
                          1800       ; Retry
                          1209600    ; Expire
                          86400 )    ; Minimum TTL

    @       IN      NS      ns.asircastelao.int.
    ns      IN      A       172.28.5.1
    test    IN      A       172.28.5.4

    Explicación:
        Define un registro SOA (Start of Authority) para la zona.
        Registros NS y A que indican que ns.asircastelao.int apunta a 172.28.5.1 y test.asircastelao.int apunta a 172.28.5.4.

Paso 3: Iniciar los Contenedores con docker-compose

    Levantar los contenedores: Una vez que hayas configurado el archivo docker-compose.yml y los archivos de configuración, puedes iniciar los contenedores con el siguiente comando:

    docker-compose up -d

    Esto descargará las imágenes necesarias (si no están disponibles) y levantará ambos contenedores: el servidor DNS y el cliente.

Paso 4: Configurar el Cliente para Usar el Servidor DNS

    Acceder al contenedor del cliente: Una vez que los contenedores estén en funcionamiento, accede al contenedor del cliente (Alpine) usando:

docker exec -it Prac6_alpine /bin/sh

Instalar dig: Dentro del contenedor cliente, instala las herramientas necesarias para realizar consultas DNS con el siguiente comando:

apk update && apk add bind-tools

Realizar consultas DNS con dig: Ahora puedes usar el comando dig para realizar consultas al servidor DNS. Ejemplo de consulta:

dig @172.28.5.1 test.asircastelao.int

Esto debería devolver la IP 172.28.5.4, que es la que configuraste en el archivo de zona.

También puedes probar con otros registros, como:

    dig @172.28.5.1 ns.asircastelao.int

    Resultado esperado: La consulta debería devolver la IP del servidor DNS 172.28.5.1.

Paso 5: Verificar y Depurar

    Si todo está correctamente configurado, el cliente debería poder resolver los dominios definidos en el servidor DNS. Si dig muestra un error como SERVFAIL, verifica los archivos de configuración de Bind9 y asegúrate de que los volúmenes estén montados correctamente.

    También puedes revisar los logs del servidor DNS con:

    docker logs Practica6_bind9

    Esto te mostrará cualquier error o advertencia relacionado con el servidor DNS.

Conclusión

Siguiendo estos pasos, has configurado con éxito un sistema cliente-servidor DNS usando Docker y docker-compose. El servidor DNS está ejecutándose en un contenedor con Bind9 y el cliente está configurado para hacer consultas a este servidor, validando la resolución de nombres a través de dig.
