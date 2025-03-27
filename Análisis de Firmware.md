# Introducción
Este proyecto trata sobre el análisis de firmware de la cámara DCS-935L. Se llevó a cabo la extracción del binario y la exploración de sus carpetas, archivos y scripts en busca de credenciales y posibles vulnerabilidades.

El principal objetivo del proyecto es comprender en qué consiste el análisis de firmware, qué tipos de herramientas se pueden utilizar, qué tipo de información se puede encontrar y cómo un ciberdelincuentes podría aprovechar estas vulnerabilidades para llevar a cabo un ataque.

# Metodología de análisis de firmware
En este caso voy a tratar de guiarme por la metodología de [OWASP FTSM](https://github.com/scriptingxss/owasp-fstm)
La metodología consisten en 9 pasos pero en mi caso voy a reducir los pasos por falta de conocimiento.

1. **Recolección de información:** Obtener detalles técnicos del dispositivo.

2. **Obtención del firmware:** Adquirir el firmware.

3. **Análisis del firmware:** Estudiar sus características.

4. **Extracción del sistema de archivos:** Extraer los contenidos del firmware.

5. **Análisis del sistema de archivos:** Realizar análisis estáticos.

6. **Emulación del firmware:** Emular los componentes.

7. **Análisis dinámico:** Pruebas de seguridad dinámicas.

8. **Análisis en tiempo de ejecución:** Analizar binarios en ejecución.

9. **Explotación binaria:** Explorar vulnerabilidades encontradas.

# Recolección de información
La recolección de información es un proceso amplio que puede incluir detalles sobre el hardware, el software y las vulnerabilidades conocidas del dispositivo. Sin embargo, en este caso, omitiré esa parte y me centraré en explorar el contenido del binario del firmware.
Sitio oficial: [DCS‑935L](https://www.dlink.com/es/es/products/dcs-935l-monitor-hd)
# Obtener Firmware
Existen distintas manera de conseguir el firmware en mi caso lo descargo porque es publico,pero en muchos casos no lo vas a poder encontrar y vas a tener que buscar la forma de extraerlo manualmente manipulando el hardware del dispositivo.

Dejo un video de ejemplo para que explore: [Extraction Firmware](https://www.youtube.com/watch?v=eMVr_iAuAA4&pp=ygUUIEZpcm13YXJlIEV4dHJhY3Rpb24%3D)

![texto alternativo](/Imagenes/Descarga.png)

# Extracción del firmware y análisis
La guía recomienda usar algunos comandos para obtener información del binario, pero en mi caso, no apareció nada demasiado relevante. Así que en lugar de detenerme ahí, voy a seguir adelante y extraer su contenido.

Según la guía, a veces un análisis de binario no arroja datos interesantes por varias razones:

- Puede ser BareMetal, es decir, diseñado para ejecutarse directamente en el hardware sin un sistema operativo.

- Puede ser para un sistema operativo en tiempo real (RTOS) con un sistema de archivos personalizado.

- Puede estar encriptado, lo que dificulta la extracción de información.

Para verificar si un binario está cifrado, se puede analizar su entropía con binwalk:

```bash
binwalk -E firmware.bin
```
La entropía se mantiene mayormente alta, con algunas caídas en ciertas áreas. Esto podría indicar que:
- El firmware no está completamente cifrado.
- Algunas partes pueden estar comprimidas o contener código legible.

![[Entropia.png]]

Extraer contenido del binario.

```bash
binwalk -ev firmware.bin
```

Su contenido es este.
![[Contenido.png]]

Por el momento solo explorare la primer carpeta.
![[Contenido1.png]]

El contenido del archivo 2818.
```bash
strings 2818 | grep -iE "password|passwd|admin|user|key|credential|secret|username|pass"
```

Su salida no muestra algo tan relevante para mi pero si puede profundizar mas, se encontró una gran cantidad de cadenas relacionadas con credenciales y claves.
![[2818.png]]

Como objetivo principal explore la carpeta home y este contiene 3 archivos.
![[home.png]]

El text contiene hash y el comprimido contiene script y ejecutables.
![[ContenidoHome.png]]

En la carpeta server se encuentra varios archivos pero por ahora no buscare mucho.
![[Cserver.png]]

```bash
 grep -iE "password|username|admin|key|secret|pass|http|local|user|pass|cam|firmware|dlink|monitor|server|services|ip" *.ini        

xver.ini:Cabname =UltraRTCamX.cab
xver.ini:Cabname64 =UltraRTCamX64.cab
```

Navegando en la carpeta `/etc/ssl/certs` encontré certificados.
![[Cerst.png]]

En la carpeta `/etc/stunnel` encontré mas certificados pero en este caso son privados.
```bash
cat stunnel.pem 
```

En la carpeta `/etc/Wireless` encontré configuraciones.
![[Wir.png]]

Realizando un grep con algunos parámetros de interés encontré algunas cosas interesante:

```bash
grep -iE "password|username|admin|key|secret|credential|user|firmware|version|cam|port|ip|tcp|udp|open|close" *

```

Se puede ver puertos,servicios,canales y algunos usuario.Luego de usar grep decidí explorar con cat y encontré algunas cosas mas,como la configuración de servicios ,direcciones url,configuración de tunnel y configuración onvif.

![[User.png]]
![[ovnif.png]]
![[Tunnel.png]]

Este es todo el contenido de la carpeta etc:
![[ETC.png]]

Por ultimo explorare la carpeta web.
![[Web.png]]
# Emulación del firmware
Encontré la carpeta web y decidí montar todo el sistema de archivos para poder visualizar el sitio web de la cámara.

Permite identificar las estructuras dentro del firmware.
`binwalk firmware.bin`

Para extraer el contenido del binario, utilicé el siguiente comando:
`dd if=DIR850L_REVB.bin bs=1 skip=1704084 of=dir.squashfs`

Luego, descomprimí el sistema de archivos SquashFS con el siguiente comando:
`unsquashfs dir.squashfs`

 SquashFS es un sistema de archivos altamente comprimido. Esto permite almacenar una gran cantidad de datos en un espacio reducido. 

Finalmente, accedí a la carpeta squashfs-root, donde se encuentran los archivos extraídos. Dentro de esta carpeta, ejecuté el siguiente comando para verificar que el sistema contenía busybox:
`cat bin/busybox`

Una vez verificado monte tres carpetas principales `proc` , `dev` y `sys`:
- **/proc**: Proporciona datos sobre el estado del sistema y los procesos en tiempo real.

- **/dev**: Contiene archivos que representan los dispositivos hardware, permitiendo su acceso y manipulación.

- **/sys**: Ofrece detalles sobre el kernel y los dispositivos, permitiendo inspeccionar y modificar parámetros relacionados con el hardware.

```bash
sudo mount --bind /proc /home/thor/Desktop/Analisis_Firmware/Firmware/squashfs-root/proc

sudo mount --bind /dev /home/thor/Desktop/Analisis_Firmware/Firmware/squashfs-root/dev

sudo mount --bind /sys /home/thor/Desktop/Analisis_Firmware/Firmware/squashfs-root/sys
```

Una vez montado todo se procede a iniciar la shell:

```bash
sudo chroot . /bin/sh
```

![[Bin.png]]
Para comprobar el user use el comando `whoami` y busque el archivo `passwd`.
![[Whoami.png]]
![[Passdw.png]]

Ahora se ejecuta el script `rcS` que se inicia al arranque para configurar el sistema, montar sistemas de archivos, preparar el entorno, y ejecutar otros scripts que son necesarios para que el sistema esté listo para su uso.

![[Scripts.png]]

Hice una comprobación de puertos con `netstat`.
![[Port.png]]
Tiene tres puertos abierto y realice la comprobación con nmap para ver las versiones de los puertos.

Escaneo del puerto con nmap para ver la version.
```bash
nmap -sV  0.0.0.0 -p 80,8080,443 -vvv
```

![[nmap.png]]
Claramente nos muestra que contiene el sitio web de la cámara, para entrar al sitio me dirigí a la dirección `0.0.0.0:80`.

Se puede ver el la pagina principal del sitio y tipo de producto y su version de firmware.
![[Port80.png]]

Explorando un poco mas el sitio puedo ver mas información del dispositivo.
![[InfoDevices.png]]

En otro apartado del sitio se puede ver en Wireless que se puede configurar con dos modos de seguridad.
![[Wire.png]]
# Vulnerabilidades
Estuve realizando búsqueda de vulnerabilidades en internet y no logre encontrar ninguna pero posiblemente algunas vulnerabilidades de versiones mas alta a la del firmware 1.07 puedan ser probada en esta version.

De igual forma dejare algunas de otras versiones:
[CVE-2019-17146](https://nvd.nist.gov/vuln/detail/CVE-2019-17146)
[CVE-2019-17146](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2019-17146)
[CVE-2019-17146](https://app.opencve.io/cve/CVE-2019-17146)

# Conclusion
Se analizó el firmware de la cámara D-Link DCS-935L (versión 1.07). Aunque no se encontraron vulnerabilidades, el proceso ayudó a entender cómo analizar firmware en dispositivos IoT.

En resumen, el proyecto proporcionó conocimientos sobre cómo auditar dispositivos IoT y mejorar su seguridad.