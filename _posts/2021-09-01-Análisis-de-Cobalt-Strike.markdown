---
layout: post
title: "Análisis de Cobalt Strike para Blue Team"
date: 2021-09-01 09:00:00 +0200
image: cobalt.jpg
tags: [CobaltStrike, analista]
categories: BlueTeam
---
											

**Cobalt Strike** es una herramienta elegida por las APT's para llevar a cabo la segunda fase de sus ataques, ya que ofrece capacidades mejoradas de post-explotación, a lo que se añade su facilidad de uso y extensibilidad. Por lo tanto, los defensores deben saber cómo detectar Cobalt Strike en varias etapas de su ejecución. El propósito principal de esta publicación es exponer las técnicas más comunes que vemos en las intrusiones que rastreamos y proporcionamos detecciones. Dicho esto, no se discutirán todas las características de Cobalt Strike.

---

## Funciones utilizadas principalmente:

* Cargar y descargar cargas útiles y archivos: 
	* ``Download [archivo] | Upload [archivo]``

* Ejecución de comandos:
	* ``shell [comando] | run [comando] | powershell [comando]``

* Inyección de proceso: 
	* ``inject <pid> | dllinject <pid> (para inyección de dll reflectante) | dllload <pid> ( para cargar una DLL en el disco en la memoria) | spawnto <arch> <full-exe-path> ( para vaciar el proceso).``

* Proxy de SOCKS:
	*  ``socks <número de puerto>``

* Escalada de privilegios:
	*  ``getsystem  (suplantación de la cuenta del SISTEMA utilizando canalizaciones con nombre) | elevate svc-exe [listener]  (crea un servicio que ejecuta una carga útil como SISTEMA)``

* Hashes y credenciales:
	*  ``hashdump | logonpasswords (usando Mimikatz) | chromedump  (Recuperar contraseñas de Google Chrome del usuario actual)``

* Enumeración de red:
	*  ``portscan [destinos] [puertos] [método de descubrimiento] | net <comandos>  (comandos para encontrar objetivos en el dominio)``

* Movimiento lateral:
	*  ``jump psexec  (Ejecutar el servicio EXE en el host remoto) | jump psexec_psh  (Ejecute una línea única de PowerShell en un host remoto a través de un servicio) | jump winrm (Ejecute un script de PowerShell a través de WinRM en un host remoto) | remote-exec <cualquiera de los anteriores> (Ejecute un solo comando usando los métodos anteriores en el host remoto)``

---
## Infraestructura de Cobalt Strike:
	
Los ataques serios cuando se llevan a cabo registran muchos dominios y configuran varios sistemas para que actúen como redirectores (puntos de pivote) de regreso a sus C2. Cobalt Strike tiene soporte completo para redirectores. Un **redirector** es un sistema que envía todo el tráfico a su C2, estos no necesitan ningún tipo de software especial. Un poco de iptables o socat para redireccionar el tráfico y listo.

Cobalt Strike tiene la opción de **crear perfiles maleables** y permite a los actores de las amenazas personalizar casi todos los aspectos del marco C2. Esto dificulta en gran medida la vida y el trabajo de los defensores, ya que la huella puede cambiar con cada modificación del perfil. Los cibercriminales tienen la **capacidad de cambiar cualquier característica**, desde la comunicación de la red (como el agente de usuario, los encabezados, los URI predeterminados) hasta las funciones individuales posteriores a la explotación, como la inyección de procesos y las capacidades de ofuscación de la carga útil.

![Estructura redirectores]({{site.baseurl}}/images/redirectors.jpg)

El componente **Beacon** es altamente versátil y admite comunicaciones asíncronas o interactivas. El módulo asíncrono es muy útil para un atacante para crear distorsiones temporales, estableciendo conexiones basados en valores temporales aleatorios, que permite evitar que se puedan realizar asociaciones de ataques con patrones de comunicaciones concretas. En este sentido el componente de Beacon llamará al C2 evaluará si hay tareas, las descargará y las podrán en ejecución cuando sea necesario.

Los indicadores de red de Beacon son flexibles, empleado el lenguaje maleable C2 propio de Cobalt Strike. Esto le permite ocultar la actividad de Beacon para que se disfrace como otro malware o se haga pasar como tráfico legítimo.

---
## Cobalt Strike en acción:

La gran mayoría de las herramientas posteriores a la explotación de Cobalt Strike se implementan como archivos DLL de Windows. Esto significaría que cada vez que un actor de amenazas ejecuta estas herramientas integradas, se genera un proceso temporal que usa <rundll32.exe> para inyectar código malicioso en él y comunicar los resultados a la baliza usando canalizaciones con nombre. La herramienta permite cambiar el nombre de los operadores de las tuberias a cualquiera que elijan configurando el perfil C2 maleable. Aunque esto no es algo que se vea de normal por el atacante promedio. 

* Las principales tuberías predeterminadas son las siguientes:
	* \postex_(*)
	* \postex_ssh_(*)
	* \status_(*)
	* \msagent_(*)
	* \MSSE-(*)
	* \(*)-server


* Acciones comunes por los atacantes:

	1. Usar PowerShell para cargar e inyectar shellcode directamente en la memoria: 
		* ``%PC% /b /c start /b /min powershell -nop -w -hidden -encodedcommand [TEXTO EN BASE64]``


	2. Descargar en disco y ejecutar manualmente en el objetivo:
		* Tener en cuenta los **ID de Sysmon** (11 - Creación de archivos, 7 - Imagen cargada, 3 - Conexión de red y 1 - Creación de procesos)
		* Tener en cuenta los **registros de seguridad** (4663 - Creación de archvos, 5156 - Conexión de red y 4688 - Creación de procesos)


	3. Ejecutar la baliza en la memoria a través de la infección inicial de malware:
		![imagen con malware]({{site.baseurl}}/images/beacon_memory.png)

 ---
## Evasión de la defensa con CB:
 	
El factor común de todos los ataques suele ser la inyección de código malicios en los procesos para tratar de no ser detectados y comenzar con la escalada de privilegios. En los dominios de Windows, suele ser típico inyectarlo en *lsass.exe* para extraer las credenciales de la memoria.

Cuando los atacantes se inyecta en un proceso remoto, están generando una nueva sesión en el contexto del usuario al que pertenece el proceso inyectado. Cobalt strike dispone de muchos módulos que facilitan la inyección de código malicioso en los procesos. Como por ejemplo:

* *inject/shinject*: Permite inyectar código en cualquier proceso remoto, algunos módulos de post-explotación incorporados también pueden inyectarse en un proceso remoto particular a través de la herramienta. Cobalt Strike hizo esto porque inyectar shellcode en una nueva sesión sería más seguro que migrar la sesión directamente a otro C2.
*  *shspawn*: Su función es iniciar un proceso e inyectarle shellcode. Los parámetros solo necesitan seleccionar la arquitectura del programa. Este proceso es más estable ya que no se corre el riesgo de que el proceso finalice. 
*  *process-inject*: Permite la configuración de un bloque de inyección de comandos en un perfil maleable del C2. 

![ejecucion comandos]({{site.baseurl}}/images/shinject.png)

Tal y como se puede observar en la imagen, selecciona el PID y la arquitectura y crea un subproceso del proceso al cual se ha inyectado, pasando totalmente desapercibido. Para la detección de la inyección de procesos podemos hacer uso de Sysmon, buscando los siguientes IDs en este orden consecutivo:

* 10 - Proceso creado (Se lleva a cabo la inyección en un proceso)
* 8 - CreateRemoteThread detectado (Se crea un hilo del proceso)
* 22/3- Consulta de red / DNS (Este hilo lleva a cabo peticiones DNS, tratando de conectar con el C2)

---
## Escaneo y descubrimiento con CB:

En muchos de los casos reales en los que se detecta a Cobalt Strike vemos a los actores ejecutando comandos de reconocimiento con la ayuda del comando "shell". Los comandos se basan en utilidades nativas de Windows como nltest.exe, whoami.exe y net.exe para ayudar con el escaneo y descubrimiento de vulnerabilidades de los equipos, así como redes internas.

> Los comandos se basan en utilidades nativas de Windows como nltest.exe, whoami.exe y net.exe para ayudar con el escaneo y descubrimiento...

En los entornos Windows, **las relaciones de confianza** o *Domain Trust Discovery* juegan un papel fundamental a la hora de determinar quién puede acceder a qué recursos. Para determinar qué cuentas de usuario tienen acceso a qué sistemas, un adversario debe comprender las cuentas de usuario que existen dentro de un dominio determinado y las relaciones de confianza entre ese dominio y otros.

Nltest es una herramienta de línea de comandos nativa de Microsoft que los administradores suelen utilizar para enumerar controladores de dominio (DC) y determinar el estado de confianza entre dominios, por nombrar algunas características importantes.

> Las herramientas más utilizadas para explotar las relaciones de confianza son AdFind y BloodHound.

---
## Escalada de privilegios con comandos de CB:

La técnica más común que usan los actores de amenazas para obtener privilegios de nivel de SISTEMA es el  método GetSystem a través de la suplantación de canalización con nombre. Tal y como se puede observar en la imagen siguiente, de un ejemplo real en un sistema que fue comprometido con TrickBot:
![Getsystem]({{site.baseurl}}/images/getsystem.png)

Existen otros métodos que permiten la escalada de privilegios, como el comando *elevate*. Este comando utiliza dos opciones para escalar privilegios:
1. El primero es ***svc-exe***. Intenta colocar un ejecutable en "C:\Windows" y crear un servicio para ejecutar la carga útil como SYSTEM.
2. El segundo es el método ***uac-token-duplication***, que intenta generar un nuevo proceso con privilegios de SYSTEM en el contexto de un usuario sin privilegios con un token robado de un proceso con privilegios.

Para la detección de estos procesos, vuelve a ser necesario Sysmon. Hay que monitorizar los procesos:
* 11 - Creación de un archivo
* 1 - Crea proceso
* 25 - Manipulación de un proceso
* 12/13 - Se escribe el valor de un registro

En relación a los eventos de Windows:
* Instalación de un servicio: 4697 (seguridad) y 7045 (sistema)
* Creación de procesos: 4688

---
## Credenciales de acceso:

Después de obtener acceso al objetivo usando Cobalt Strike, una de las primeras tareas que realizan los operadores es recopilar credenciales y hashes de LSASS. Hay un par de formas de lograrlo con Cobalt Strike. El primero usa el comando ***hashdump*** para volcar hashes de contraseñas; el segundo usa el comando ***logonpasswords*** para volcar credenciales de texto plano y hashes NTLM con Mimikatz.

Cobalt Strike ha implementado la funcionalidad DCSync introducida por mimikatz. DCSync utiliza las API de Windows para la replicación de Active Directory a fin de recuperar el hash NTLM de un usuario específico o de todos los usuarios.

Para su detección:
ID de Sysmon 1,8,10,17: (el ID de evento 8 no siempre estará presente según la técnica utilizada).

---
## Command and Control:

Cobalt Strike está utilizando solicitudes GET y POST para comunicarse con el servidor C2. Los ciberdelincuentes pueden elegir entre la comunicación mediante HTTP, HTTPS y DNS. Cuando se trata de C2, normalmente vemos balizas HTTP y HTTPS. De forma predeterminada, Cobalt Strike utilizará solicitudes GET para recuperar información y solicitudes POST para enviar información al servidor. Aunque como explicamos anteriormente, todos estos perfiles son totalmente modificables, aunque no suele verse con mucha frecuencia.

Ejemplo petición GET:
![GET]({{site.baseurl}}/images/GET.png)

Ejemplo petición POST:
![POST]({{site.baseurl}}/images/POST.png)

---
## Movimiento lateral con CB:

Una vez los ciberdelicuentes han conseguido acceso a un equipo y han recopilado información de su PC y del entorno gracias a la fase de descubrimiento mencionada anteriormente, comienza la fase de los movimientos laterales.  La gran mayoría de informes que analizan los ataques perpetrados indican que las técnicas más usadas con CB por los atacantes son:


##### Ejecución y transferencia ejecutable SMB / WMI:
Este método es el más utilizado por los atacantes. De normal suele estar cargando su ejecutable desde el host deseado con el comando *upload* de Cobalt Strike y lo ejecutan usando el comando *remote-exec*. Tambien suelen usar *psexec*, *winrm* o *wmi* para ejecutar un comando. 

Los IDs que deberíamos buscar para detectarlo son:
* 4264 - Logon
* 4672 - Special Logon
* 4673 - Sensitive Privilege Use
* 4688 - Process Creation
* 4697 - Security System Extension
* 4674 - Sensitive Privilige Use
* 5140 - File Share

- - - -
##### Pass the hash
Cobalt Strike puede usar Mimikatz para generar y hacerse pasar por un token que luego se puede usar para realizar tareas en el contexto de ese recurso de usuario elegido. La baliza Cobalt Strike también puede usar este token para interactuar con los recursos de la red y ejecutar comandos remotos. 

Detección:

![]({{site.baseurl}}/images/PTH.jpg)


- - - -
##### Ejecución de servicios remotos 
En muchos de los ataques que hemos analizado, los atacantes ejecutan el comando *jump psexec* para crear un servicio remoto (Domain Controler) y ejecutar el .exe malicioso en el servidor. Cobalt strike es capaz de especificar el ejecutable para crear el servicio remoto.  Antes de que pueda hacer eso, tendrá que transferir el ejecutable del servicio al host de destino. El nombre del ejecutable del servicio se crea con siete caracteres alfanuméricos aleatorios.

Para identificarlo, podemos observar los eventos de Windows que se generan:

![ids]({{site.baseurl}}/images/ID.png)

En resumen, para su detección hay que monitorizar los procesos de Windows que van continuados con el siguiente ID:


* 4624: Inicio de sesión 
* 4672: Inicio de sesión especial 
* 4673: Uso de privilegios confidenciales 
* 4688: Creación de procesos 
* 5140: Uso compartido de archivos 
* 4674: Eventos de creación de servicios de uso de privilegios confidenciales 
* 4697: Se instaló un servicio en el sistema. (security.evtx) 
* 7045: Se instaló un servicio en el sistema. (system.evtx) 
* 7034: un servicio terminó inesperadamente

---
## Recomendaciones para el Blue Team:

> Sysmon:
> > * Configurar los eventos 17 y 18 de Sysmon para registrar las canalizaciones con nombre. 
> > * Investigar si los procesos de Sysmon 10, 8 y 22/3 van continuados.
> > * Investigar si los procesos de Sysmon 11, 1, 25 y 12/13 van continuados.
> > * Investigar si los procesos de Sysmon 1, 8, 10 y 17 van continuados.
> 
> Los analistas deben prestar especial atención a los eventos en los que rundll32 se está ejecutando sin ningún argumento.
>
> Detección de los intentos de relación de confianza, para ello monitorear los principales binarios que permiten llevar a cabo este tipo de acciones. Para ello, hacer uso de herramientas de Red Team que permitan detectarlas.
> 
> Monitorización de las peticiones GET y POST, desde donde se hace y hacia donde va. Comprobando reputación de la IP, dominio, etc.
> 
> Eventos de seguridad de Windows:
> > * Investigar los procesos de Seguridad de Windows 4264, 4672, 4673, 4688, 4697, 4674 y 5140 en conjunto.
> > * Investigar los procesos de Seguridad de Windows 4264, 4672, 4673, 4688, 5140, 4674, 4697, 7045 y 7035 en conjunto.

---
#### Referencias:  
**URL ORIGINAL DE DONDE SALE EL BLOQUE PRINCIPAL DE INFORMACIÓN:**
* https://thedfirreport.com/2021/08/29/cobalt-strike-a-defenders-guide/

**EXTRAS AÑADIDOS:**
* https://blog.cobaltstrike.com/2014/01/14/cloud-based-redirectors-for-distributed-hacking/
* https://conpilar.es/los-desafios-de-la-toma-de-huellas-dactilares-del-servidor-cobalt-strike/
* https://www.sidertia.com/cobalt-strike-el-componente-perfecto-para-los-ciberdelincuentes-ii/
* https://hideandsec.sh/books/red-teaming-tactics/page/cobalt-strike-process-injection
* https://redcanary.com/threat-detection-report/techniques/domain-trust-discovery/
