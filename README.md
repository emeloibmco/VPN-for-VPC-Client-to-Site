# VPN-for-VPC-Client-to-Site :computer:



<br />

## Tabla de contenido 📑

1. [Requisitos](#Requisitos-newspaper)
2. [Antes de empezar](#antes-de-empezar)
   * [Configuración de la autenticación client-to-site](#configuraci%C3%B3n-de-la-autenticaci%C3%B3n-client-to-site)
   * [Creación del grupo de acceso]()
   * [Creación de la instancia certificate manager]()
   * [Creación de la VPC y la subred]()
3. [creación del servidor VPN]()
   * [Crear servidor VPN]()
   * [Validar servidor VPN]()
   * [Crear ruta VPN]()
   * [Configurar cliente de VPN]()
4. [Conexión al servidor VPN]()
11. [Referencias](#Referencias-mag)
12. [Autores](#Autores-black_nib)
<br />

## Requisitos :newspaper:
- Contar con un sistema operativo Linux con el navegador Google Chrome instalado

- Tener una cuenta de [IBM Cloud](https://cloud.ibm.com/)

- :cloud: [IBM Cloud CLI](https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started&locale=en)

- :satellite: [OpenVPN](https://openvpn.net/)
- [Git](https://git-scm.com/downloads)

## Antes de empezar
Inicie sesión en su cuenta de [IBM Cloud](https://cloud.ibm.com/login).
## Configuración de la autenticación client-to-site
**Crear una autorización IAM sevice-to-service**
<br/>
Para crear una autorización IAM sevice-to-service para su servidor VPN y certificate manager siga los siguientes pasos:
1. Desde la consola de IBM Cloud, vaya a la página [Manage Autorizations](https://cloud.ibm.com/iam/authorizations) y dé clic en el botón ```Crear```
2. En el menú desplegable seleccione ```VPC Infrastructure Services``` y luego seleccione ```Resource based on selected attributes```
3. Seleccione ```Resource type``` > ```Client VPN for VPC```
4. En la opción Target Service seleccione ```Certificate Manager```
5. Seleccione la opción ```All resources``` y verifique la casilla ```Writer```
6. Dé clic en ```Authorize```

**Gestión de certificados de cliente y servidor VPN**
<br/>
A continuación se usará [OpenVPN easy-rsa](https://github.com/OpenVPN/easy-rsa) para generar los certificados y posteriormente importarlos al certificate manager.
1. Clone el repositorio Easy-RSA 3 en su carpeta local:
```
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3
```
<br/>
2. Cree un nuevo PKI y CA:

```
./easyrsa init-pki
./easyrsa build-ca nopass
```
<br/>

Verifique que el certificado CA esté generado en la ruta ```./pki/ca.crt```

<br/>
3. Genere un certificado de servidor VPN:

```
./easyrsa build-server-full vpn-server.vpn.ibm.com nopass
```

Verifique que la llave pública haya sido generada en la ruta ```./pki/issued/vpn-server.vpn.ibm.com.crt``` y la llave privada en la ruta ```./pki/private/vpn-server.vpn.ibm.com.key```
<br/>

4. Genere un certificado de cliente VPN:
```
./easyrsa build-client-full client1.vpn.ibm.com nopass
```
Verifique que la llave pública haya sido generada en la ruta ```./pki/issued/client1.vpn.ibm.com.crt``` y la llave privada en la ruta ```./pki/private/client1.vpn.ibm.com.key```
<br/>

Para importar los certificados al certificate manager siga estos pasos:
1. En el navegador Google Chrome navegue a la página de [Certificate Manager](https://cloud.ibm.com/catalog/services/certificate-manager), complete la información y dé clic en ```Create``` para crear una instancia.
2. Diríjase a la página ```Your Certificates``` e importe el certificado según los siguientes pasos:

   * Elija un nombre para su certificado, este no puede contener guiones, números ni mayúsculas (ej. vpcdemo)
   * Dé clic al botón ```Browse``` y seleccione el archivo de certificado ```./pki/issued/vpn-server.vpn.ibm.com.crt```
   * Dé clic al botón ```Browse``` y seleccione el archivo de llave privada ```./pki/private/vpn-server.vpn.ibm.com.key```
   * Dé clic al botón ```Browse``` y seleccione el archivo de certificado intermedio ```./pki/ca.crt```
   * Dé clic al botón ```Import```
   <br/>

Si el certificado es usado como certificado de servidor VPN, usted debe subir los archivos ```Certificate file```, ```Private key file``` e ```Intermediate certificate file```. si el certificado es usado como certificado de cliente VPN para autenticar el cliente, usted debe subir los archivos ```Certificate file``` e ```Intermediate certificate file```.
<br/>

**Ordenar un certificado usando Certificate Manager NOTA: creo que esto no lo hicimos**
<br/>

Usted puede usar IBM Cloud Certificate Manager para ordenar un certificado público SSL/TLS como certificado de servidor VPN. Certificate Manager solo almacena certificados intermedios, por lo cual usted necesitará los root certificates de Let's Encrypt, guardados como archivos ```.pem```. Los dos archivos requeridos puede encontrarlos en [https://letsencrypt.org/certs/lets-encrypt-r3.pem](https://letsencrypt.org/certs/lets-encrypt-r3.pem) y [https://letsencrypt.org/certs/isrgrootx1.pem](https://letsencrypt.org/certs/isrgrootx1.pem). Cuando descargue y actualice el certificado de cliente VPN, use este root certificate para reemplazar la sección ```<ca>``` en el perfil de cliente.
<br/>

Los certificados ordenados son certificados públicos SSL/TLS y deben ser usados como certificados de servidor VPN únicamente. No pueden ser usados para autenticar los clientes VPN.
<br/>

**Ubicar el certificado CRN NOTA: esto tampoco**
<br/>

Al configurar la autenticación de un servidor VPN client-to-site usando la UI, usted puede especificar el Certificate Manager y el certificado SSL, o el CRN del certificado. Esto se puede hacer si usted no tiene acceso a la instancia de Certificate Manager. Tenga en cuenta que usted debe ingresar el CRN si está usando la API para crear el servidor VPN client-to-site.
<br/>
Para encontrar el CRN del certificado, siga estos pasos:

1. En la [consola de IBM Cloud](https://cloud.ibm.com/vpc-ext) Vaya al ícono de menú y seleccione ```Resource List```
2. Dé clic para expandir ```Services and software``` y posteriormente seleccione el Certificate Manager del que desea obtener el CRN.
3. Seleccione cualquier parte en esa fila de la tabla para abrir el panel lateral de detalles. El CRN del certificado se encuentra listado allí.

**Configuración de IDs de usuario y contraseñas NOTA: no lo hicimos**
<br/>

Para configurar la autenticación en dos factores para usuarios de cliente VPN siga este proceso:

1. El administrador de la VPN invita al usuario del cliente VPN a la cuenta donde reside el servidor VPN.

2. El administrador de la VPN asigna el permiso IAM de usuario de cliente VPN, esto permite al usuario conectarse al servidor VPN. Para más información visite [Creating an IAM access group and granting the role to connect to the VPN server.](https://cloud.ibm.com/docs/vpc?topic=vpc-create-iam-access-group)

3. El usuario de cliente VPN abre la siguiente dirección web para generar una contraseña para su ID de usuario:

```
https://iam.cloud.ibm.com/identity/passcode
```
<br/>

4. El usuario de cliente VPN ingresa su contraseña en el cliente de OpenVPN e inicia la conexión al servidor VPN. Para más información vea [Setting up a client VPN environment and connecting to a VPN server.](https://cloud.ibm.com/docs/vpc?topic=vpc-vpn-client-environment-setup)



## Referencias :mag:

- [Documentación de IBM Cloud: About client-to-site VPN servers (Beta)](https://cloud.ibm.com/docs/vpc?topic=vpc-vpn-client-to-site-overview)


<br />

## Autores :black_nib:
Equipo *IBM Cloud Tech Sales Colombia*.
