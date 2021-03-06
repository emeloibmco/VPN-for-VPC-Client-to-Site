# VPN-for-VPC-Client-to-Site :computer:



<br />

## Tabla de contenido 馃搼

1. [Requisitos](#Requisitos-newspaper)
2. [Antes de empezar](#antes-de-empezar)
   * [Configuraci贸n de la autenticaci贸n client-to-site e importaci贸n de certificados al Certificate Manager](#configuraci贸n-de-la-autenticaci贸n-client-to-site-gear)
   * [Creaci贸n del grupo de acceso IAM y rol para conectarse al servidor VPN](#creaci贸n-del-grupo-de-acceso-iam-y-rol-para-conectarse-al-servidor-vpn-oldkey)
   * [Creaci贸n de la VPC y la subred](#creaci贸n-de-la-vpc-y-la-subred)
3. [creaci贸n del servidor VPN](#desplegar-servidor-vpn)
   * [Crear servidor VPN](#h3crear-servidor-vpnh3)
   * [Validar servidor VPN](#validar-servidor-vpn)
   * [Crear ruta VPN](#crear-ruta-vpn)
   *[Configurar Protocolo y reglas de entrada](#configurar-protocolo-y-reglas-de-entrada)
   * [Configurar cliente de VPN](#configurar-el-cliente-de-vpn)
4. [Autenticaci贸n al servidor VPN](#autenticaci贸n-al-servidor-vpn)
5. [Referencias](#Referencias-mag)
6. [Autores](#Autores-black_nib)
<br />

## Requisitos :newspaper:
- Contar con un sistema operativo Linux con el navegador Google Chrome instalado

- Tener una cuenta de [IBM Cloud](https://cloud.ibm.com/)

- :cloud: [IBM Cloud CLI](https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started&locale=en)

- :satellite: [OpenVPN](https://openvpn.net/)
- [Git](https://git-scm.com/downloads)

## Antes de empezar
Inicie sesi贸n en su cuenta de [IBM Cloud](https://cloud.ibm.com/login).
## Configuraci贸n de la autenticaci贸n client-to-site :gear:
**Crear una autorizaci贸n IAM sevice-to-service**
<br/>
Para crear una autorizaci贸n IAM sevice-to-service para su servidor VPN y certificate manager siga los siguientes pasos:
1. Desde la consola de IBM Cloud, vaya a la p谩gina [Manage Autorizations](https://cloud.ibm.com/iam/authorizations) y d茅 clic en el bot贸n ```Create```
2. En el men煤 desplegable seleccione ```VPC Infrastructure Services``` y luego seleccione ```Resource based on selected attributes```
3. Seleccione ```Resource type``` > ```Client VPN for VPC```
4. En la opci贸n Target Service seleccione ```Certificate Manager```
5. Seleccione la opci贸n ```All resources``` y verifique la casilla ```Writer```
6. D茅 clic en ```Authorize```

**Gesti贸n de certificados de cliente y servidor VPN**
<br/>
Para la gesti贸n de certificados hay dos opciones, usar OpenVPN para generar los certificados u ordenar un certificado usando Certificate Manager.

**Opci贸n 1. Generaci贸n de certificados usando OpenVPN**
<br/>
A continuaci贸n se usar谩 [OpenVPN easy-rsa](https://github.com/OpenVPN/easy-rsa) para generar los certificados y posteriormente importarlos al certificate manager.
1. Clone el repositorio Easy-RSA 3 en su carpeta local:

```
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3
```

2. Cree un nuevo PKI y CA:

```
./easyrsa init-pki
./easyrsa build-ca nopass
```

Verifique que el certificado CA est茅 generado en la ruta ```./pki/ca.crt```

3. Genere un certificado de servidor VPN:

```
./easyrsa build-server-full vpn-server.vpn.ibm.com nopass
```

Verifique que la llave p煤blica haya sido generada en la ruta ```./pki/issued/vpn-server.vpn.ibm.com.crt``` y la llave privada en la ruta ```./pki/private/vpn-server.vpn.ibm.com.key```

4. Genere un certificado de cliente VPN:
```
./easyrsa build-client-full client1.vpn.ibm.com nopass
```
Verifique que la llave p煤blica haya sido generada en la ruta ```./pki/issued/client1.vpn.ibm.com.crt``` y la llave privada en la ruta ```./pki/private/client1.vpn.ibm.com.key```
<br/>

Para importar el certificado del servidor al certificate manager siga estos pasos:
1. En el navegador Google Chrome dir铆jase a la p谩gina de [Certificate Manager](https://cloud.ibm.com/catalog/services/certificate-manager), complete la informaci贸n y d茅 clic en ```Create``` para crear una instancia.
2. Dir铆jase a la p谩gina ```Your Certificates``` e importe el certificado del servidor seg煤n los siguientes pasos:

   * Elija un nombre para su certificado, este no puede contener guiones, n煤meros ni may煤sculas (ej. vpcdemo)
   * D茅 clic al bot贸n ```Browse``` y seleccione el archivo de certificado ```./pki/issued/vpn-server.vpn.ibm.com.crt```
   * D茅 clic al bot贸n ```Browse``` y seleccione el archivo de llave privada ```./pki/private/vpn-server.vpn.ibm.com.key```
   * D茅 clic al bot贸n ```Browse``` y seleccione el archivo de certificado intermediario ```./pki/ca.crt```
   * D茅 clic al bot贸n ```Import```
   <br/>

<br/>

**Opci贸n 2. Ordenar un certificado usando Certificate Manager**
<br/>

Usted puede usar IBM Cloud Certificate Manager para ordenar un certificado p煤blico SSL/TLS como certificado de servidor VPN. Certificate Manager solo almacena certificados intermedios, por lo cual usted necesitar谩 los root certificates de Let's Encrypt, guardados como archivos ```.pem```. Los dos archivos requeridos puede encontrarlos en [https://letsencrypt.org/certs/lets-encrypt-r3.pem](https://letsencrypt.org/certs/lets-encrypt-r3.pem) y [https://letsencrypt.org/certs/isrgrootx1.pem](https://letsencrypt.org/certs/isrgrootx1.pem). Cuando descargue y actualice el certificado de cliente VPN, use este root certificate para reemplazar la secci贸n ```<ca>``` en el perfil de cliente.
<br/>

Los certificados ordenados son certificados p煤blicos SSL/TLS y deben ser usados como certificados de servidor VPN 煤nicamente. No deben ser usados para autenticar los clientes VPN.
<br/>

*Ubicar el certificado CRN*
<br/>

Al configurar la autenticaci贸n de un servidor VPN client-to-site usando la UI, usted puede especificar el Certificate Manager y el certificado SSL, o el CRN del certificado. Esto se puede hacer si usted no tiene acceso a la instancia de Certificate Manager. Tenga en cuenta que usted debe ingresar el CRN si est谩 usando la API para crear el servidor VPN client-to-site.
<br/>
Para encontrar el CRN del certificado, siga estos pasos:

1. En la [consola de IBM Cloud](https://cloud.ibm.com/vpc-ext) Vaya al 铆cono de men煤 y seleccione ```Resource List```
2. D茅 clic para expandir ```Services and software``` y posteriormente seleccione el Certificate Manager del que desea obtener el CRN.
3. Seleccione cualquier parte en esa fila de la tabla para abrir el panel lateral de detalles. El CRN del certificado se encuentra listado all铆.

## Creaci贸n del grupo de acceso IAM y rol para conectarse al servidor VPN :old_key:

Para crear un grupo de acceso IAM y permitir al rol de usuario conectarse al servidor VPN, siga estos pasos:

1. Desde la consolda de IBM Cloud, navegue a la p谩gina de [Access groups](https://cloud.ibm.com/iam/groups) (Manage > Access (IAM) > Access groups) y d茅 clic en ```Create```.
2. Digite un nombre para su grupo de acceso y d茅 clic en ```Create```.
3. D茅 clic en la pesta帽a ```Access Policies``` y luego en ```Assign access```.
4. En el men煤 desplegable seleccione ```VPC Infrastructure Services```.
5. Para acceso al servicio, seleccione ```Users of the VPN server need this role to connect to the VPN server``` y luego d茅 clic en ```Add```
6. Verifique el panel de resumen y d茅 clic en ```Assign```.
7. D茅 clic en la pesta帽a ```Users``` y posteriormente en ```Add users``` para agregar usuarios al nuevo grupo de acceso.

## Creaci贸n de la VPC y la subnet

**Creaci贸n de la VPC**
<br/>
Para crear una VPC en su cuenta de IBM Cloud siga los pasos que se indican a continuaci贸n:

1. D茅 click en el Men煤 de Navegaci贸n y seleccione la pesta帽a ```VPC Infrastructure```.

2. En la secci贸n de ```Network``` seleccione la opci贸n ```VPCs``` y posteriormente de click en el bot贸n ```Create```. Una vez le aparezca la ventana para la configuraci贸n y creaci贸n de la *VPC*, complete lo siguiente:

* ```Name```: asigne un nombre exclusivo para la *VPC*.
* ```Resource Group```: seleccione el grupo de recursos en el cual va a trabajar.
* ```Location```: seleccione la ubicaci贸n en la cual desea implementar la *VPC*.


| NAME | DISPLAY NAME |
| ------------- | :---: |
| au-syd        | Sydney          |     
| in-che        | Chennai         |     
| jp-osa        | Osaka           |     
| jp-tok        | Tokyo           |     
| kr-seo        | Seoul           |     
| eu-de         | Frankfurt       | 
| eu-gb         | London          | 
| ca-tor        | Toronto         |     
| us-south      | Dallas          | 
| us-south-test | Dallas Test     |
| us-east       | Washington DC   |
| br-sao        | Sao Paulo       |

* ```Default security group```: deje seleccionadas las opciones *Permitir SSH* y *Permitir ping*.
* ```Classic access```: deje el campo SIN seleccionar.
* ```Default address prefixes```: deje el campo SIN seleccionar, ya que posteriormente se crear谩 la subred en la que se va a trabajar.

Cuando ya tenga todos los campos configurados d茅 click en el bot贸n ```Create virtual private cloud```.

3. Espere unos minutos mientras la *VPC* aparece en estado disponible y aseg煤rese de tener seleccionada la regi贸n en la cual la implement贸.
4. Una vez haya sido aprovisonada la VPC, d茅 click en el nombre e ingrese a la pesta帽a ```Address prefixes```. En dicha pesta帽a, de click en ```Create``` e ingrese la direcci贸n IP que desee junto con la m谩scara.

> NOTA: Puede utilizar la IP y m谩scara sugeridas en las subnets creadas por defecto cuando se estaba aprovisionando la VPC.

<br />

**Creaci贸n de la subnet**
<br />
El siguiente paso consiste en crear una Subnet en la *VPC*. Para ello, en la secci贸n ```Network``` seleccione la opci贸n ```Subnets``` y d茅 click en el bot贸n ```Create```. Una vez le aparezca la ventana para la configuraci贸n y creaci贸n de la subnet, complete lo siguiente:

* ```Name```: asigne un nombre exclusivo para la subnet.
* ```Resource group```: seleccione el grupo de recursos en el cual va a trabajar (el mismo seleccionado en la creaci贸n de la *VPC*).
* ```Location```: seleccione la ubicaci贸n en la cual desea implementar la subnet (la misma seleccionada en la creaci贸n de la *VPC*).
* ```Virtual private cloud```: seleccione la *VPC* que cre贸 anteriormente.
* Los dem谩s par谩metros no los modifique, deje los valores establecidos por defecto.

Cuando ya tenga todos los campos configurados d茅 click en el bot贸n ```Create subnet```.

Luego de crear la subnet habilitar la opcion de pasarela publica.

![image](screens/pasarela-publica-subnet.png)

6. Espere unos minutos mientras la subnet aparece en estado disponible y aseg煤rese de tener seleccionada la regi贸n en la cual la implement贸.

<br />

## Configurar claves SSH :closed_lock_with_key:
<br />
Para poder desplegar una *VSI* en *VPC* es necesario realizar la respectiva configuraci贸n para las claves *SSH*. Con base en esto, realice lo siguiente:

1. Para generar una clave *SSH* acceda al *IBM Cloud Shell* y coloque el comando:
```
ssh-keygen -t rsa -C "user_id"
```

2. Al colocar el comando anterior, en la consola se pide que especifique la ubicaci贸n, en este caso oprima la tecla Enter para que se guarde en la ubicaci贸n sugerida. Posteriormente, cuando se pida la ```Passphrase``` coloque una constrase帽a que pueda recordar o gu谩rdela, ya que se utilizar谩 m谩s adelante.

3. Mu茅vase con el comando ```cd .ssh``` a la carpeta donde est谩n los archivos ```id_rsa.pub``` y ```id_rsa```. Estos archivos contienen las claves p煤blicas y privadas respectivamente. 

4. Visualice la clave p煤blica, ya que la necesitar谩 para la creaci贸n de la *VSI*. Utilice el comando:
```
cat id_rsa.pub
```
> NOTA: Por defecto la clave empieza con ssh_rsa y termina con el user_ID. Copie la clave para emplearla m谩s adelante.

<br />

**Desplegar VSI en VPC**
<br/>
Una vez ha configurado las claves *SSH* proceda con la creaci贸n de la *VSI* Linux en *VPC*. Complete los siguientes pasos:

1. Entre al men煤 desplegable y seleccione ```VPC Infrastructure```. En la secci贸n de ```Compute``` seleccione la opci贸n ```Virtual Server instances``` y posteriormente d茅 click en el bot贸n ```Create```. Una vez le aparezca la ventana para la configuraci贸n y creaci贸n de la *VSI*, complete lo siguiente:

* ```Name```: asigne un nombre exclusivo para la *VSI*.
* ```Resource group```: seleccione el grupo de recursos en el cual va a trabajar (el mismo seleccionado en la creaci贸n de la *VPC*).
* ```Location```: seleccione la ubicaci贸n en la cual desea implementar la subnet (la misma seleccionada en la creaci贸n de la *VPC*).
* ```Hosting type```: seleccione la opci贸n **Public**.
* ```Operating system```: seleccione la opci贸n **Ubuntu Linux**.
* ```Profile```: deje seleccionado el perfil que viene por defecto (**Balanced | bx2-2x8**).
* ```SSH keys```: d茅 click en el bot贸n ```Create key +```, asigne un nombre exclusivo para su clave *SSH*, seleccione el grupo de recursos y la ubicaci贸n y finalmente en **Public key** coloque la clave copiada en el 铆tem 3 del paso [Configurar claves SSH](#Configurar-claves-SSH-closed_lock_with_key). Posteriormente, d茅 click en el bot贸n ```Create```.
* ```Virtual private cloud```: seleccione la *VPC* creada anteriormente.
* Los dem谩s par谩metros no los modifique, deje los valores establecidos por defecto.

Cuando ya tenga todos los campos configurados d茅 click en el bot贸n ```Create virtual server instance```.

2. Espere unos minutos mientras la *VSI* aparece en estado disponible y aseg煤rese de tener seleccionada la regi贸n en la cual la implement贸.

<br />

## Desplegar servidor VPN
Dir铆jase al Panel en la parte izquierda de IBM Cloud y seleccione *Infraestructura VPC*

<img src="screens/VPC.png" alt="VPC" width="200"/>

## Crear servidor VPN

   Ahora seleccione el apartado de VPN y d茅 click en el bot贸n de crear.

   ![image](screens/vpn-crear.png)

   Luego seleccionamos el tipo de VPN que deseamos, en este caso *Client-to-site-server*.

   ![image](screens/vpn-type.png)

   En la seccion de detalles, se debe especificar la siguiente informacion:

   - **VPN server name:** Escoja un nombre para su servidor VPN, ejemplo: my-vpn-server.<br/>
   - **Resource group:** El grupo de recursos que seleccione debe ser el mismo que en donde se encuentra la VPC.
   - **Region:** La misma region donde se encuentra la VPC se usar谩 para el servidor de VPN.
   - **Virtual private cloud:** Escoger la VPC para el servidor de VPN.
   - **Client IPv4 address pool:** Ingrese un rango CIDR. Al cliente se le asigna una IP de este rango para su sesi贸n.

   ![image](screens/vpn-details.png)

   Ahora en la secci贸n de subnets, se debe seleccionar la modalidad del servidor VPN:

   - **Seleccionar la modalidad del servidor VPN:**
      - **High-availability mode:** Este modo despliega el servidor en dos subnets que se encuentran ubicadas en diferentes zonas.Ideal para los despliegues y soluciones donde el acceso VPN de el cliente es crucial.
      - **stand-alone mode:** Este modo despliega el servidor de VPN en una subnet en una sola zona. Ideal para despliegues en una sola zona.

      ![image](screens/vpn-subnets.png)
   - **Secci贸n de autenticaci贸n:** 
      - **Server authentication:** Seleccione el gestor de certificados y luego el certificado SSL del servidor.

      ![image](screens/vpn-server-authentication.png)

      - **Client authentication:** Se debe seleccionar qu茅 configuracion usar谩 el cliente para autenticarse en el servidor, ya sea a trav茅s de certificados o usando un ID y passcode, o ambas si se desea. Si se usa certificados, seleccione el mismo certificado que eligio en el paso anterior.

      ![image](screens/vpn-client-authentication.png)

   - **Secci贸n Security groups:** Debe seleccionar al menos un grupo de seguridad.Tambi茅n puede configurar m谩s grupos de seguridad si lo desea.

   ![image](screens/vpn-security-groups.png)

   - **Configuraci贸n adicional:** Mantenga la configuraci贸n recomendada.

   ![image](screens/vpn-additional-config.png)
      
## Validar servidor VPN
   Para asegurarse de que el servidor VPN se despleg贸 correctamente espere unos minutos y dir铆jase al apartado de VPN, en la seccion *client-to-site-servers* y aseg煤rese de que el estado del servidor sea *Estable* y el *Estado* sea *Buen estado*.

   ![image](screens/vpn-status.png)
## Crear ruta VPN
   Ingrese a su servidor de VPN y en la pesta帽a de *Rutas de servidor VPN* d茅 click en el boton *Crear*. 

   ![image](screens/vpn-routes.png)

   Luego as铆gnele un nombre a la ruta y escoja rango CIDR para la red destino:
      - **Para acceso a internet:** Elija 0.0.0.0/0
      - **Para la subred de la VPC:** Ingrese el CIDR de la subred de la VPC.

   Por 煤ltimo escoja la acci贸n de la ruta, en este caso ser谩 *Entregar* y finalice dando click en el boton crear.

   ![image](screens/vpn-config-route.png)

## Configurar Protocolo y reglas de entrada
   En la seccion de grupos de seguridad seleccionar el grupo de seguridad de la VPC y en el apartado de entrada seleccionar el protocolo UDP
   para el puerto 443 con la opci贸n de cualquier origen y la direccion IP 0.0.0.0/0.

   ![image](screens/config-vpn.png)

## Configurar el cliente de VPN
   - **Ingrese al servidor de VPN:** En la pesta帽a cliente d茅 click *Descargar perfil de cliente*. Descargar谩 un archivo de configuraci贸n llamado *<vpn_server>.ovpn*
   <img src="screens/vpn-client.png" alt="VPN client" width="600"/>

   - **Distribuya el archivo de perfil de cliente:** Env铆e el perfil de configuraci贸n a los usuarios de la VPN a trav茅s de un canal seguro.

   - **Configurar el archivo de perfil de cliente:** Los clientes de la VPN deben editar el perfil de cliente, para esto deben tener los certificados de la VPN, los cuales deben ser enviados por un canal seguro, estos certificados deben ser a帽adidos al perfil del cliente de la siguiente manera:

   ![image](screens/config-cliente-profile.png)

   Para editar el perfil del cliente puede usar un editor de c贸digo como *VS CODE* o el de su preferencia y agregar el certificado de cliente que genero al inicio y la private key del certificado al final del archivo perfil del cliente tal como se muestra en la imagen de arriba.

   <br/>
   Una opci贸n alternativa para conectarse al servidor VPN es por medio de un ID de usuario y una contrase帽a. Para configurar la autenticaci贸n en dos factores para usuarios de cliente VPN siga este proceso:

   * El administrador de la VPN invita al usuario del cliente VPN a la cuenta donde reside el servidor VPN.

   * El administrador de la VPN asigna el permiso IAM de usuario de cliente VPN, esto permite al usuario conectarse al servidor VPN. Para m谩s informaci贸n visite [Creating an IAM access group and granting the role to connect to the VPN server.](https://cloud.ibm.com/docs/vpc?topic=vpc-create-iam-access-group)

   * El usuario de cliente VPN abre la siguiente direcci贸n web para generar una contrase帽a para su ID de usuario:

   ```
   https://iam.cloud.ibm.com/identity/passcode
   ```

   * El usuario de cliente VPN ingresa su contrase帽a en el cliente de OpenVPN e inicia la conexi贸n al servidor VPN. Para m谩s informaci贸n vea [Setting up a client VPN environment and connecting to a VPN server.](https://cloud.ibm.com/docs/vpc?topic=vpc-vpn-client-environment-setup)
   
   <br/>

## Autenticaci贸n al servidor VPN

**Por certificado:**
Para conectarse debera importar el perfil de cliente que descargo y configuro en el paso anterior en **Open VPN** y conectarse.


![image](screens/import-Cprofile.png)

Asi debe lucir su conexi贸n al servidor VPN desde el cliente de **Open VPN**

![image](screens/crt-connect-opvpn.png)


<br/>

**Usuario y contrase帽a**

Para conectarse usando usuario y contrase帽a primero debe habilitar la opcion de usuario y codigo de acceso en el apartado de **Autenticaci贸n** del servidor VPN. 

![image](screens/habilitar-user-pass.png)

Luego debera importar el perfil de cliente que descargo al cliente de **Open VPN**.

Por ultimo recuerde que su usuario es el mismo que en IBM cloud y tambien debera generar un codigo de acceso para su usuario a traves del siguiente link:

```
   https://iam.cloud.ibm.com/identity/passcode
```
Asi debe lucir su conexi贸n al servidor VPN desde el cliente de **Open VPN**

![image](screens/conectar-opvn.png)

Para verificar que se hizo la conexi贸n adecuadamente, abra la p谩gina de detalles del servidor VPN. Luego verifique en la secci贸n de clientes todos los clientes de VPN que se han conectado en a 煤ltima hora.

![image](screens/conexiones.png)

## Referencias :mag:

- [Documentaci贸n de IBM Cloud: About client-to-site VPN servers (Beta)](https://cloud.ibm.com/docs/vpc?topic=vpc-vpn-client-to-site-overview)
- [Gu铆a VPC-Despliegue-VSI-Acceso-SSH IBM Colombia](https://github.com/emeloibmco/VPC-Despliegue-VSI-Acceso-SSH)

<br />

## Autores :black_nib:
Equipo *IBM Cloud Tech Sales Colombia*.
