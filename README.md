# Despligue de App En Azure

**Realizado por:** [Cmauricio Parra](https://github.com/c-m-a)
**Twitter:** [@cma_io](https://twitter.com/@cma_io)

## Objetivos

En este documento encontrara los pasos para realizar un despliegue de una WebApp monolítica utilizando React, Node con una sección de análisis de datos en Python usando le infraestructura de Azure, además se realiza un análisis para optimizar los costos, teniendo en cuenta los siguientes requerimientos.

-  El proceso que realiza el componente de analítica tarda alrededor de 10 minutos.

## Proceso De Despliegue

El proceso de despliegue lo haremos desde Ubuntu en la cual podemos utilizar Azure CLI para creación de Resource Group, VMs, Web y Storage.

### Instalación de Azure CLI en Ubuntu 20.04

```bash
sudo apt install -y azure-cli
```

### Autenticándose contra Azure

```bash
az login
```

### Creando un "Resource Group"

Teniendo en cuenta que es una aplicación monolítica una de las estrategias para organizar el Resource Group es por ciclo de vida de software esto nos permite crear varios ambientes uno de desarrollo llamado **APP-DEV-RG** y otro de producción **APP-PRO-RG**, por defecto el ambiente de **APP-DEV-RG** puede ser implementado en la nube privada para una mejor optimización de costos.

![service-group-container.png](https://raw.githubusercontent.com/c-m-a/despliegue-app-node-python-en-azure/main/img/service-group-container.png)

El ambiente **APP-PRO-RG** contendrá los servicios necesarios como **SQL** o base de datos, **WEB** o servidor Web, **VM** o Maquina(s) virtual(es), **STORAGE** o almacenamiento, para ejecutar la aplicación y aplicar políticas de group para un mejor control.

![create-resource-option.png](https://raw.githubusercontent.com/c-m-a/despliegue-app-node-python-en-azure/main/img/create-resource-option.png)

### Creando Un Resource Group

En la creación del **Resource Group** supondremos que los usuarios de la aplicación se encuentran localizados en la región de estados unidos por simplicidad, por lo tanto escogeremos (US) East US por ser la mas cercana.

![create-resource-group-region.png](https://raw.githubusercontent.com/c-m-a/despliegue-app-node-python-en-azure/main/img/create-resource-group-region.png)

![create-resource-group-review.png](https://raw.githubusercontent.com/c-m-a/despliegue-app-node-python-en-azure/main/img/create-resource-group-review.png)

### Script Para Crear un Resource Group Usando AZURE CLI (AZCLI)

Se puede realizar la creacion de RG usando el siguiente comando.

```bash
az group create --name app-pro-rg --location "East US"
```

### Creación de "VM Machine Scale Set" Para Alojar El Código Fuente.

Ya que se esta desplegando una aplicación monolítica se decidió crear un servicio máquinas virtuales escalables o **"Virtual Machine Scale Set"**, ya que esto permite expandir y contraer la infraestructura cuando la carga de la CPU u otras políticas definidas para auto escala, para esta máquina se decidió instalar un Servidor **Ubuntu 18.04 LTS** en un servidor **E2as_v4** **con 2 VCPUs 16GB de RAM y 25GB de disco duro** ya que son máquinas para propósitos **in-memory analytics** se acomoda al proceso de análisis de datos que realiza Pyhon, con esto también se pretende reducir el tiempo de análisis de datos.

Se busca el servicio **"Virtual Machine Scale Set"**

![virtual-machine-set-specs.png](https://raw.githubusercontent.com/c-m-a/despliegue-app-node-python-en-azure/main/img/virtual-machine-set-specs.png)

En la pestaña Scaling se define las políticas de Scale Out que este caso esta basada en la carga del **CPU** cuando esta sea mayor a un **70%** y el Scale In ocurrirá cuando la carga sea menor al 30%.

![virtual-machine-network-specs.png](https://raw.githubusercontent.com/c-m-a/despliegue-app-node-python-en-azure/main/img/virtual-machine-network-specs.png)

### Instalación de Node.js y App

Supondremos que nuestra **APP** se encuentra comprimida en un tarball y que ya tenemos lista para el despliegue en producción.

- Copiando app.
```bash
scp app-pro-rg-release.1.5.tar.gz azureuser@20.240.25.260:~/.
```
- Entrando al servidor web
```bash
ssh azureuser@20.240.25.260
```

```bash
$ sudo apt update # Actualizando repositorios de paquetes
$ curl -sL https://deb.nodesource.com/setup_14.x -o nodesource_setup.sh # Descarga script de instalacion de Node
$ sudo bash nodesource_setup.sh
$ sudo apt install nodejs
sudo apt install build-essential
$ sudo mkdir opt app-pro && cd $_
$ sudo npm install pm2@latest -g  #install PM2 Node Process Manager
```

### Instalación de Nginx 

```bash
$ sudo apt install -y nginx
$ sudo ufw allow 'Nginx HTTP' # Permitiendo trafico HTTP en el Firewall
```

### Configuración Conexión De La APP Hacia Postgres 

En este proceso se debe realizar configuración de variables de entorno que contengan nombre del servicio de Postgres. Para este caso supondremos que todas las configuraciones ya se realizaron y se hicieron las verificaciones de conexión hacia la base de datos. 

### Creación De Servidor Postgres.

Para la base de datos puede escoger un servidor Postgres de propósito general con 500GB y dos 2Cores.

![postgres-single-server.png](https://raw.githubusercontent.com/c-m-a/despliegue-app-node-python-en-azure/main/img/postgres-single-server.png)
Especificaciones.
![postgres-machine-details.png](https://raw.githubusercontent.com/c-m-a/despliegue-app-node-python-en-azure/main/img/postgres-machine-details.png)
Costos del servicio de Postgres

![postgres-machine-costs.png](https://raw.githubusercontent.com/c-m-a/despliegue-app-node-python-en-azure/main/img/postgres-machine-costs.png)

### Creación de Servicio Postgres Usando Azule CLI

```bash
az postgres server create --resource-group app-pro-rg --name app-postgres --location eastus --admin-user pgadmin --admin-password "Estee$3l5abor" --sku-name GP_Gen5_2 
```

## Costos de la Infraestructura

Se va suponer el peor de los casos en el sistema **(Virtual Machine Set)** para el calculo de costos.
Precios en Dolares obtenidos por la pagina **Azure**.

| Servicio | Cantidad | Costo Uni | Costo Total x Mes |
|-|:-:|-:|-:|
|**(Virtual Machine Set) E2ds_v4** Máxima escalabilidad x 2 - Servidor de Aplicación Ubuntu|2|$105.12|$210.24|
|Servidor de Postgres Simple 500GB SSD, 2CPUs|1|$155.55|$155.55|
|**Total**|||$365.79|


## Optimización De Costos

Teniendo en cuenta que es una aplicación monolítica y no se puede desintegrar y que el costo de ejecutar la analítica es demasiado alto.

La forma de reducir es instalando un **Reverse Proxy Layer 7** utilizando **NGINX** Con **Caching** activado para el URL donde ejecuta la analítica dándole un tiempo de cache de 4 horas. También se debe crear en el servidor de aplicación Ubuntu un **Server Side Redering** para React, asi permita que los datos esten combinados en **HTML** y listos para el Reverse Proxy haga **Caching** a la pagina donde se ejecuta la analítica. Para la instalación del Proxy se utilizó una maquina de **B2s** de proposito general con un costo de $30.37/mes.

Teniendo en cuenta que el proceso de analítica gasta 10 mins en ejecutarse, la escalabilidad estaría activada 6 veces cada 24 horas que al mes 6 x 30 dias = 180 veces con un total de 1800 mins con un total de 30 horas/mes el costo de tener un **E2ds_v4** es de $0.146 dolares cada vez que expande o se realiza un **scale out** del servicio se factura una hora así se haya utilizado 10 mins por lo tanto el valor es $0.146 * 180 = $26.28.

La reducción de costos es del 13.3% mensual, lo cual es un valor aceptable, ya que se supone que el tiempo mínimo para actualizar la analítica es cada 2 horas, por ende se puede hacer varias modificaciones al tiempo para reducir el costo 5% si se mantiene el caching por 23 horas.

| Servicio | Cantidad | Costo Uni | Costo Total x Mes |
|-|:-:|-:|-:|
|**B2s** Ubuntu Nginx Reverse Proxy Layer 7|1|$30.37|$30.37|
|**(Virtual Machine Set) E2ds_v4** Maxima escalabilidad x 2 - Servidor de Aplicación Ubuntu|2|$105.12|$105.12|
|**(Virtual Machine Set) E2ds_v4** Activa|180 horas|$26.28|$26.28|
|Servidor de Postgres Simple 500GB SSD, 2CPUs|1|$155.55|$155.55|
|**Total**|||$317.32|
