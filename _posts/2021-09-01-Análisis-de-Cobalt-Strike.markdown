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

![]({{site.baseurl}}/images/redirectors.jpg)

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
		![]({{site.baseurl}}/images/beacon_memory.png)

 ---
## Evasión de la defensa con CB:
 	
El factor común de todos los ataques suele ser la inyección de código malicios en los procesos para tratar de no ser detectados y comenzar con la escalada de privilegios. En los dominios de Windows, suele ser típico inyectarlo en *lsass.exe* para extraer las credenciales de la memoria.

Cuando los atacantes se inyecta en un proceso remoto, están generando una nueva sesión en el contexto del usuario al que pertenece el proceso inyectado. Cobalt strike dispone de muchos módulos que facilitan la inyección de código malicioso en los procesos. Como por ejemplo:

* *inject/shinject*: Permite inyectar código en cualquier proceso remoto, algunos módulos de post-explotación incorporados también pueden inyectarse en un proceso remoto particular a través de la herramienta. Cobalt Strike hizo esto porque inyectar shellcode en una nueva sesión sería más seguro que migrar la sesión directamente a otro C2.
*  *shspawn*: Su función es iniciar un proceso e inyectarle shellcode. Los parámetros solo necesitan seleccionar la arquitectura del programa. Este proceso es más estable ya que no se corre el riesgo de que el proceso finalice. 
*  *process-inject*: Permite la configuración de un bloque de inyección de comandos en un perfil maleable del C2. 

![]({{site.baseurl}}/images/shinject.png)

Tal y como se puede observar en la imagen, selecciona el PID y la arquitectura y crea un subproceso del proceso al cual se ha inyectado, pasando totalmente desapercibido. Para la detección de la inyección de procesos podemos hacer uso de Sysmon, buscando los siguientes IDs en este orden consecutivo:

````
10 - Proceso creado (Se lleva a cabo la inyección en un proceso)
8 - CreateRemoteThread detectado (Se crea un hilo del proceso)
22/3- Consulta de red / DNS (Este hilo lleva a cabo peticiones DNS, tratando de conectar con el C2)
````

---
## Escaneo y descubrimiento con CB:

En muchos de los casos reales en los que se detecta a Cobalt Strike vemos a los actores ejecutando comandos de reconocimiento con la ayuda del comando "shell". Los comandos se basan en utilidades nativas de Windows como nltest.exe, whoami.exe y net.exe para ayudar con el escaneo y descubrimiento de vulnerabilidades de los equipos, así como redes internas.

> Los comandos se basan en utilidades nativas de Windows como nltest.exe, whoami.exe y net.exe para ayudar con el escaneo y descubrimiento...

En los entornos Windows, **las relaciones de confianza** o *Domain Trust Discovery* juegan un papel fundamental a la hora de determinar quién puede acceder a qué recursos. Para determinar qué cuentas de usuario tienen acceso a qué sistemas, un adversario debe comprender las cuentas de usuario que existen dentro de un dominio determinado y las relaciones de confianza entre ese dominio y otros.

Nltest es una herramienta de línea de comandos nativa de Microsoft que los administradores suelen utilizar para enumerar controladores de dominio (DC) y determinar el estado de confianza entre dominios, por nombrar algunas características importantes.

> Las herramientas más utilizadas para explotar las relaciones de confianza son AdFind y BloodHound.

---
## RECOMENDACIONES PARA EL BLUE TEAM:

* Configurar Sysmon (Vital importancia)
* Los defensores deben prestar mucha atención a los eventos de la línea de comandos que rundll32 está ejecutando sin ningún argumento.
* Configurar los eventos 17 y 18 de Sysmon para registrar las canalizaciones con nombre. 
* Investigar si los procesos de Sysmon 10, 8 y 22/3 van continuados.
* Detección de los intentos de relación de confianza, para ello monitorear los principales binarios que permiten llevar a cabo este tipo de acciones. Para ello, hacer uso de herramientas de Red Team que permitan detectarlas.

---
#### Referencias:  
* https://thedfirreport.com/2021/08/29/cobalt-strike-a-defenders-guide/
* https://blog.cobaltstrike.com/2014/01/14/cloud-based-redirectors-for-distributed-hacking/
* https://conpilar.es/los-desafios-de-la-toma-de-huellas-dactilares-del-servidor-cobalt-strike/
* https://www.sidertia.com/cobalt-strike-el-componente-perfecto-para-los-ciberdelincuentes-ii/
* https://hideandsec.sh/books/red-teaming-tactics/page/cobalt-strike-process-injection
* https://redcanary.com/threat-detection-report/techniques/domain-trust-discovery/
