# Instalación perzonalizada de Proxmox VE (Virtual Environment)

## Autor

- [Ixen Rodríguez Pérez - kurosaki1976](https://github.com/kurosaki1976)

## Preámbulo

Todo lo que se explicará a continuación, sólo es válido si se desea realizar una instalación personalizada del sistema operativo. Sin embargo, existe otra posibilidad de utilizar el resto del espacio en disco duro al concluir la instalación por defecto (sin personalizar valores) de Proxmox VE, tal y como era antes de la versión 4.2, donde se incluyó el almacenamiento `LVM-thin` que constituye una mejora permitiendo dividir o combinar particiones; incluso de diferentes discos.

Una vez concluido el proceso de instalación, acceder a la web admininistrativa de Proxmox VE, y en `Datacenter/Storage/` seleccionar `local-lvm` y clic en el botón `Remove`. Luego desde la consola del nodo ejecutar `lvremove /dev/pve/data` y aceptar que se desea eliminar el volumen `pve/data`.

En este punto se puede verificar el espacio libre disponible con el comando `vgdisplay | grep Free`, y posteriormente redefinir el tamaño del volumen `pve/root` (donde se encuentra la partición raíz `/`) ejecutando `lvresize -l 100%FREE /dev/pve/root`; agregando así, el espacio libre disponible a dicho volumen.

Finalmente, se debe reasignar el tamaño de la partición `Datacenter/Storage/local` ejecutando `resise2fs /dev/mapper/pve-root`.

## El pollo del arroz con pollo

En el proceso de instalación del Promxox VE se puede definir el espacio libre que tendrá el LVM, destinado para las salvas de los CTs/VMs. En el momento de escoger el HDD hacer click en el botón `Options`, se mostrará una ventana donde pueden ser configurados los parámetros siguientes:

- `hdsize`: define el tamaño total del disco duro que se utilizará; de esta forma, se puede reservar espacio libre en el disco duro para definir más particiones,
- `swapsize`: define el tamaño de la partición `swap` (el valor predeterminado es el tamaño de la memoria instalada, mínimo 4Gb y máximo 8Gb; el valor resultante no puede ser mayor que el cálculo `hdsize/8`),
- `maxroot`: define el tamaño máximo de la partición `root`, donde se almacena el sistema operativo (por defecto es el resultado del cálculo `hdsize/4`),
- `minfree`: define el espacio libre que queda en el grupo de volúmenes `LVM pve` (con más de 128Gb de almacenamiento disponible, el valor predeterminado es 16Gb; de lo contrario, se utilizará la resultante del cáculo `hdsize/8`),
- `maxvz`: define el tamaño máximo de la partición `data` (por defecto `/var/lib/vz` y se obtiene del cálculo `maxvz = hdsize-maxroot-swapsize-minfree`).

Lo recomendado es solamente definir el valor `minfree` y dejar los demás por defecto.

A los efectos de este tutorial, el valor de `maxroot` se obtendrá determinando el 10% de `hdsize`, es decir `maxroot = hdsize*10%`; mientras que `maxvz` será el resultado del cáculo `maxvz = hdsize-swapsize-maxroot` y, el valor `minfree`, el resultado de `minfree = maxvz/2`. Así, podrá disponerse de mayor capacidad de almacenamiento para las salvas.

Por ejemplo si se dispone de un disco de 1Tb para la instalación de Proxmox VE en un servidor con 8Gb de memoria RAM, el cálculo del valor del parámetro `minfree`, sería `minfree = (931-8-93)/2`.

## Asignar espacio disponible para volumen de salvas

Al concluir la instalación de Promxox VE, acceder a la consola y ejecutar los comandos siguientes:

### Mostrar espacio libre disponible

```bash
vgdisplay | grep Free
```

### Crear nuevo volumen lógico

```bash
lvcreate -l 100%FREE -n backups pve
```

### Dar formato al volumen

```bash
mkfs.ext4 /dev/pve/backups
```

### Crear subdirectorio de salvas

```bash
mkdir -p /var/lib/pve-backups
```

### Montar automáticamente el volúmen

#### Agregar nuevo punto de montaje

```bash
nano /etc/fstab

/dev/pve/backups /var/lib/pve-backups ext4 errors=remount-ro defaults,noatime,discard 0 2

ó

UUID=d233ae68-1ef3-454d-b34d-47ba38c81e27 /var/lib/pve-backups ext4 defaults,noatime,discard 0 2
```

> **NOTA**: El `UUID` puede obtenerse ejecutando `lsblk -f`.

#### Montar el volumen en el sistema de archivos

```bash
mount -a
```

### Crear almacenamiento para salvas

Desde la web admininistrativa de Proxmox VE, en `Datacenter/Storage/Add/Directory` añadir un directorio destinado a almacenar las salvas. También se puede editar el fichero `/etc/pve/storage.cfg` o ejecutar el utilitario `pvesm` desde la interfaz de línea de comandos.

#### Estracto ejemplo del fichero `/etc/pve/storage.cfg`:

```bash
dir: pve-backups
    path /var/lib/pve-backups
    content iso,vztmpl,backup
```

#### Ejemplo de ejecución del utilitario `pvesm`:

```bash
pvesm add dir pve-backups --path /var/lib/pve-backups --content iso,vztmpl,backup
pvesm set pve-backups --disable 0
```

#### Estracto ejemplo del fichero `/etc/cron.d/vzdump`, para automatización periódica de salvas:

```bash
0 0 * * * root vzdump --all --mailnotification always --mode snapshot --storage pve-backups --quiet 1 --mailto postmaster@example.tld --compress zstd
```

> **NOTA**: La automatización de las salvas periódicas debe definirse en la interfaz web administrativa, mediante la ruta `Datacenter/Backup/[Add|Remove|Edit]`. La edición del fichero `/etc/cron.d/vzdump`, no debe realizarse de forma manual si no se conocen con seguridad los parámetros de configuración.

## Referencias

* [Installation](https://pve.proxmox.com/wiki/Installation)
* [Advanced LVM Configuration Options](https://pve.proxmox.com/wiki/Installation#advanced_lvm_options)
* [Storage](https://pve.proxmox.com/wiki/Storage)
* [Backup and Restore](https://pve.proxmox.com/wiki/Backup_and_Restore)
* [The advanced installation option](https://subscription.packtpub.com/book/big_data_and_business_intelligence/9781788397605/1/ch01lvl1sec11/the-advanced-installation-option)
