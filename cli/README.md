# Administrar máquinas virtuales con la CLI de Azure
https://aka.ms/AzureCLI3

## Intro

El Portal de Azure es ideal para tareas únicas, pero cuando se trata de múltiples tareas, navegar por los distintos paneles agrega tiempo y no es productivo.

¡Aquí es donde brilla la línea de comandos! Utiliza comandos y scripts para ejecutar tareas repetitivas.

Con Azure, tiene dos herramientas de líneas de comando diferentes con las que puede trabajar:

- Azure PowerShell (administradores de Windows tienden a preferir este)
- CLI de Azure (administradores de Linux tienden a preferir este)

Con cualquiera de ellos, puede escribir scripts para verificar el estado de los servidores en la nube, implementar nuevas configuraciones, abrir puertos en el firewall o conectarse a una máquina virtual para cambiar una configuración.

En esta sesión nos centraremos en la CLI de Azure, pero cualquier tarea se puede realizar en la versión de PowerShell.

## ¿Qué es la CLI de Azure?

La CLI de Azure es la herramienta de línea de comandos multiplataforma de Microsoft para administrar los recursos de Azure. Puede usarlo en Mac OS, Linux, Windows y desde un navegador a través de Azure Cloud Shell.

En esta sesión vamos a trabajar con máquinas virtuales, comencemos primero con la creación de una.

# Creación de una máquina virtual con la CLI de Azure

Con la CLI azure, cada comando comenzará con `az`

El comando `vm` se usa para trabajar con estas máquinas virtuales en la CLI de Azure. Algunos de los subcomandos más comunes son:

- `create`
- `deallocate`
- `delete`
- `list`
- `open-port`
- `restart`
- `show`
- `start`
- `stop`
- `update`

Para obtener una lista completa de comandos, consulte [Azure CLI reference documentation](https://docs.microsoft.com/cli/azure/reference-index?view=azure-cli-latest). 

Comencemos con el primero: `az vm create`

# az vm create

Este comando se utiliza para crear una máquina virtual en un grupo de recursos. Los cuatro parámetros que se deben suministrar son:

- `--resource-group` El grupo de recursos que será el propietario de la máquina virtual.
- `--name` El nombre de la máquina virtual. Debe ser único dentro del grupo de recursos.
- `--image` La imagen del sistema operativo que se utilizará para crear la VM.
- `--location` La región en la que colocar la VM. Por lo general, estaría cerca del consumidor de la VM.


Además, proporcionaremos estos parámetros:

- `--admin-username` Estamos especificando la cuenta de administrador. De forma predeterminada, el comando usará su nombre de usuario actual, pero dado que las reglas para los nombres de cuenta son diferentes para cada sistema operativo, es más seguro especificar un nombre.
- `generate-ssh-keys` Este parámetro se usa para distribuciones de Linux y crea un par de claves de seguridad para que podamos usar la herramienta ssh para acceder a la máquina virtual de forma remota. El par de archivos se colocan en la carpeta `.ssh` en su máquina local y en la VM.
- `--verbose` Útil para ver el progreso mientras se crea la VM.
    
    Si ya tiene una clave SSH llamada `id_rsa` en la carpeta de destino, se utilizará en lugar de tener una nueva clave generada.
    

Vamos a juntarlo todo.

```bash
az vm create --resource-group {resourcegroup} --location eastus2 --name {vmname} --image UbuntuLTS --admin-username azureuser --generate-ssh-keys --verbose
```

Una vez que finalice el comando, obtendrá una respuesta JSON que incluye el estado actual de la máquina virtual y sus direcciones IP públicas y privadas asignadas por Azure.

# Conéctese a nuestra VM con SSH

Podemos probar que la máquina virtual Linux está en funcionamiento utilizando la dirección IP pública en la herramienta ssh. Generamos un par de claves SSH cuando creamos la VM, por lo que no necesitamos una contraseña. Dado que establecemos el nombre de administrador en `azureadmin`, podemos usar el siguiente comando para conectar

```bash
ssh azureuser@publicipaddress
```

# VM images

Usamos la imagen `UbuntuLTS` pero Azure tiene varias imágenes estándar de VM que podemos usar. Para enumerarlos, use el comando `az vm image list --output table`. Esto generará las imágenes más populares que forman parte de una lista sin conexión integrada en la CLI de Azure.

Tenga en cuenta que también puede crear y cargar sus imágenes personalizadas para crear máquinas virtuales basadas en configuraciones únicas.

Si desea obtener una lista completa, agregue la marca `--all` al comando. Ayuda a filtrar la lista con las opciones `--publisher`` --sku` u `--offer`.

Si queremos ver todas las imágenes de Wordpress, podemos usar

```bash
az vm image list --sku Wordpress --output table --all
```

Algunas imágenes solo están disponibles en determinadas ubicaciones. Puede agregar la marca `--location` al comando para limitar los resultados a los disponibles en la región donde desea crear la máquina virtual.

```bash
az vm image list --location eastus2 --output table 
```

# Dimensionar las VM correctamente

Una máquina virtual sin la cantidad correcta de memoria o CPU fallará bajo carga o funcionará con demasiada lentitud para ser efectiva.

Azure define un conjunto de tamaños de máquina virtual predefinidos para Linux y Windows para elegir. Los tamaños disponibles cambian según la región en la que está creando la máquina virtual. Puede obtener una lista de los tamaños disponibles usando el comando `az vm list-size`.

No especificamos un tamaño en nuestro comando de creación, Azure seleccionó un tamaño de uso general predeterminado para nosotros. Podemos especificar el tamaño usando el parámetro `--size`. Podríamos crear una máquina virtual de 2 núcleos:

```bash
az vm create --resource-group {resourcegroup} --name {vmname} --image UbuntuLTS --admin-username azureuser --generate-ssh --verbose --size "Standard_DS2_v2"
```

También podemos cambiar el tamaño de una máquina virtual existente. Primero tenemos que verificar si el tamaño deseado está disponible en el clúster del que forma parte nuestra VM.

`vm list-vm-resize-options`

```bash
az vm list-vm-resize-options --resource-group {resourcegroup} --name {vm} --output table
```

Esto devolverá una lista de todas las configuraciones de tamaño posibles disponibles en el grupo de recursos. Si el tamaño que queremos no está disponible en nuestro clúster, pero está disponible en la región, podemos desasignar la VM. Esto detendría la máquina virtual en ejecución y la eliminaría del clúster actual sin perder ningún recurso. Luego podemos cambiar su tamaño.

Para cambiar el tamaño de una máquina virtual, usamos `vm resize`

```bash
az vm resize --resource-group {resourcegroup} --name {vm} --size Standard_b2s
```

# Consulta información del systema

Podemos usar otros comandos vm para obtener información sobre nuestra máquina recién creada.

- `az vm list` devolverá todas las máquinas definidas en esta suscripción. Filtrar la salida con `--resource-group`
- Hemos estado usando la marca `--output` con` table`, esto es para formatear la salida de una manera que sea más fácil de leer que el JSON predeterminado.
- `az vm list-ip-addresses` mostrará una lista de las direcciones IP públicas y privadas de una máquina virtual.
- `az vm show` nos dará información más detallada sobre una VM específica. Esto devolverá una cantidad bastante grande de datos JSON, incluidos dispositivos de almacenamiento adjuntos, interfaces de red y más. Esta es una gran oportunidad para usar JMESPath, un lenguaje de consulta integrado para JSON.

# Filtrar nuestras consultas de la CLI de Azure

```bash
az vm show --resource-group {resourcegroup} --name {vm} --query "osProfile.adminUsername"
```

También puede resultarle útil emparejarse con el parámetro `--output tsv`. También es útil para la creación de secuencias de comandos; por ejemplo, puede extraer un valor de su cuenta de Azure y almacenarlo en un entorno o variable de secuencia de comandos.

# Inicie y detenga su máquina virtual con la CLI de Azure

## Stop

Podemos detener una máquina virtual en ejecución con el comando vm stop. Debe pasar el nombre y el grupo de recursos, o el ID único de la VM:

```bash
az vm stop \
--name {vm} \
--resource-group [sandbox resource group name]
```

Podemos verificar que se haya detenido al intentar hacer ping a la dirección IP pública, usando ssh o mediante el comando vm get-instance-view. Este enfoque final devuelve los mismos datos básicos que vm show, pero incluye detalles sobre la instancia en sí. Intente ingresar el siguiente comando en Azure Cloud Shell para ver el estado de ejecución actual de su máquina virtual:

```bash
az vm get-instance-view \
    --name {vmname} \
    --resource-group {resourcegroup} \
    --query "instanceView.statuses[?starts_with(code, 'PowerState/')].displayStatus" -o tsv
```

Este comando debería devolver VM detenida como resultado.

## Start

Podemos hacer lo contrario a través del comando vm start.

```bash
az vm start \
    --name {vmname} \
    --resource-group [sandbox resource group name]
```

Este comando iniciará una máquina virtual detenida. Podemos verificarlo a través de la consulta vm get-instance-view, que ahora debería devolver la VM en ejecución.

## **Reiniciar una VM**

Podemos reiniciar una VM si hemos realizado cambios que requieran un reinicio ejecutando el comando `vm restart`. Puede agregar la marca `--no-wait` si desea que la CLI de Azure vuelva inmediatamente sin esperar a que se reinicie la máquina virtual.

## **Eliminar una VM**

```bash
az vm delete \
    --name {vmname} \
    --resource-group {resourcegroup}
```

# Instalación de software en su máquina virtual

1. Busque la dirección IP pública de su máquina virtual SampleVM Linux.

```bash
az vm list-ip-addresses --name {vmname} --output table
```

2. A continuación, abra una conexión ssh a SampleVM.

```bash
ssh azureuser@<PublicIPAddress>
```

3. Una vez que haya iniciado sesión en la máquina virtual, ejecute el siguiente comando para instalar el servidor web nginx.

```bash
sudo apt-get -y update && sudo apt-get -y install nginx
```

4. Salga de Secure Shell.

```bash
exit
```

5. En Azure Cloud Shell, use curl para leer la página predeterminada de su servidor web Linux con el siguiente comando, reemplazando <PublicIPAddress> con la IP pública que encontró anteriormente. Alternativamente, puede abrir una nueva pestaña del navegador e intentar buscar la dirección IP pública.

```bash
curl -m 10 <PublicIPAddress>
```

6. Este comando fallará porque la máquina virtual Linux no expone el puerto 80 (http) a través del grupo de seguridad de red que asegura la conectividad de red a la máquina virtual. Podemos cambiar esto con el comando vm open-port de la CLI de Azure.

7. Escriba lo siguiente en Cloud Shell para abrir el puerto 80:

```bash
az vm open-port \
    --port 80 \
    --resource-group [sandbox resource group name] \
    --name SampleVM
```

Tomará un momento agregar la regla de red y abrir el puerto a través del firewall.

8. Ejecute el comando curl nuevamente.

```bash
curl -m 10 <PublicIPAddress>
```

9. Esta vez debería devolver datos. También puede ver la página en un navegador.

```bash
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
body {
    width: 35em;
    margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif;
}
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```