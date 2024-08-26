---
title: COM Hijacking
categories: [Deep Dives]
tags: [windows,persistence]
---


![FirtsPic]({{ '/assets/img/ComHijacking/ComHijacking.jpg' | relative_url }}){: .center-image } 

- [English](#com-hijacking)
- [Español](#secuestro-del-com)


### COM Hijacking

A COM object (Component Object Model) is a technology of Microsoft that allows different softwares written in different languages to work together. The references to these objects are in the Windows's registry.

The COM Hijacking is a technique that allows that, when an application uses a COM object, this executes malicius code. This mostly works to get persistence or privilege escalation. 

It is possible to hijack a COM object being used, but this may cause breaking applications. In this example I will be using a safer way which is to search for COM objects that doesn't exists but are trying to be loaded. To do that I will use [Process Monitor](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) of Microsoft, which is a tool that shows us files, processes and registries.

The easiest way would be to search for COM objects in our own machine and after finding a CLSID modify the registry in the target.
To search for the processes that we want, we can filter them in **procmon** for:

- RegOpenKey operations
- where the Result is NAME NOT FOUND
- the Path ends with InprocServer32
- Exclude if path starts with HKLM

![SecPic]({{ 'assets/img/ComHijacking/Pasted image 20240824170113.png' | relative_url }}){: .center-image }

Once these filters has been applied, the procmon will display more and more processes, to be faster we can open programs in our machine like the file explorer, this will display even more processes.

After some seconds we will see hundreds of processes that we can try to use, but a important point is the number of times that the CLSID is loaded, if we choose one that is loaded often it's possible that the machine may have big problems, that's why is interesting to search for one that is loaded frequently but not as often.

In my case I found a CLSID loaded by *C:\\Windows\System32\DllHost.exe* 

```
HKCU\Software\Classes\CLSID\{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}\InprocServer32
`````

As we can see **DllHost.exe** is trying to load that CLSID in HKCU (HKEY\_CURRENT\_USER), which we can modify.

From the Powershell console we can check with the command below if that CLSID exists in HKLM (HKEY_LOCAL_MACHINE) and HKCU

````powershell
Get-Item -Path "HKLM:\Software\Classes\CLSID\{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}\InprocServer32"
`````

![3Pic]({{ 'assets/img/ComHijacking/Pasted image 20240824171210.png' | relative_url }}){: .center-image }

````powershell
Get-Item -Path "HKCU:\Software\Classes\CLSID\{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}\InprocServer32"
`````

![4Pic]({{ 'assets/img/ComHijacking/Pasted image 20240824171449.png' | relative_url}}){: .center-image }

As we can see in the images, the CLSID exits in HKLM but doesn't exist in HKCU, where we have modification rights.
Therefore we will try to create an entry in the HKCU registry to scope to our payload also DllHost is trying to load this CLSID, so it will execute our payload.

In this example I am using a Command and Control (Cobalt Strike) server but It can be done with a Powershell console from the target machine, the only different thing is the way we can transfer the payload.

To create the entry in the registry we will use the next command followed by our CLSID

````powershell
New-Item -Path "HKCU:Software\Classes\CLSID" -Name "{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}"
`````

Now we have to indicate the path to our payload

````powershell
New-Item -Path "HKCU:Software\Classes\CLSID\{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}" -Name "InprocServer32" -Value "C:\Windows\Temp\http_x64.dll"
`````

````powershell
New-ItemProperty -Path "HKCU:Software\Classes\CLSID\{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}\InprocServer32" -Name "ThreadingModel" -Value "Both"
`````

Now when the DllHost.exe is executed and load this COM object, it will execute our payload

![5Pic]({{ 'assets/img/ComHijacking/Pasted image 20240825145326.png' | relative_url}}){: .center-image }

To clean-up the COM hijack, just remove the registry entry from the registry editor and remove the payload.

The lab used for this example was the one given in the [CRTO](https://training.zeropointsecurity.co.uk/courses/red-team-ops) certification by [ZeroPoint Secuirty](https://training.zeropointsecurity.co.uk/)

---

### Secuestro del COM

Un objeto COM (Component Object Model) es una tecnología de Microsoft que permite que distintos softwares escritos en distintos lenguajes puedan comunicarse entre sí. Las referencias a estos objetos se encuentran en el registro de Windows.

El COM hijacking o secuestro del COM consiste en lograr que, a la hora de emplear un determinado objeto COM, este ejecute código malicioso. Esto mayoritariamente se emplea para lograr persistencia o escalar privilegios.

Es posible secuestrar un objeto COM que esté en uso, pero podría causar que algunas aplicaciones rompan o no funcionen bien. En este ejemplo para hacerlo de forma segura se buscarán por objetos COM inexistentes pero que estén tratando de ser cargados por alguna aplicación. Para ello se usará [Process Monitor](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) de Microsoft, que es una herramienta que muestra los archivos cargados en el sistema, procesos y registros.

Lo más sencillo sería buscar por objetos COM en tu propia máquina y después modificar el registro en la máquina objetivo. Para buscar por los procesos que nos interesan, en **procmon** debemos filtrar por:

- RegOpenKey operations
- where the Result is NAME NOT FOUND
- the Path ends with InprocServer32
- Exclude if path starts with HKLM

![SecPic]({{ 'assets/img/ComHijacking/Pasted image 20240824170113.png' | relative_url }}){: .center-image }

Una vez aplicados estos filtros, empezarán a salir más y más procesos, para agilizar la tarea podemos abrir programas como el explorador de archivos para que vayan apareciendo más procesos. 

Al cabo de unos segundos tendremos cientos de procesos que podemos tratar de utilizar, pero un punto muy importante a tener en cuenta es el número de veces que se carga el CLSID, si usamos uno que se carga con mucha frecuencia, es posible que la máquina de serios problemas, por lo que es interesante buscar por alguno que se cargue con frecuencia pero no tanta.

En mi caso encontré un CLSID cargado por *C:\\Windows\\System32\\DllHost.exe*

````
HKCU\Software\Classes\CLSID\{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}\InprocServer32
`````

Como podemos ver **DllHost.exe** está tratando de cargar ese CLSID en HKCU (HKEY\_CURRENT\_USER), el cual podemos modificar.

Desde la consola de Powershell podemos comprobar con el siguiente comando si ese CLSID existe en HKLM (HKEY_LOCAL_MACHINE) y HKCU

````powershell
Get-Item -Path "HKLM:\Software\Classes\CLSID\{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}\InprocServer32"
`````

![3Pic]({{ 'assets/img/ComHijacking/Pasted image 20240824171210.png' | relative_url }}){: .center-image }


````powershell
Get-Item -Path "HKCU:\Software\Classes\CLSID\{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}\InprocServer32"
`````

![4Pic]({{ 'assets/img/ComHijacking/Pasted image 20240824171449.png' | relative_url}}){: .center-image }

Como se muestra en las imágenes, el CLSID existe en HKLM pero no en HKCU, que es donde tenemos permisos para modificarlo. Por lo que trataremos de crear una entrada en el registro HKCU para que apunte a nuestro payload y como DllHost está tratando de cargar este CLSID, ejecutará nuestro payload.

En este ejemplo estoy usando un servidor de comando y control (Cobalt Strike) pero se puede hacer de la misma manera desde una consola de Powershell de la máquina víctima, lo único que cambia será el cómo transferir el payload a la máquina víctima.

Para crear la entrada en el registro se usará este comando con el CLSID que hayamos seleccionado

````powershell
New-Item -Path "HKCU:Software\Classes\CLSID" -Name "{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}"
`````

Ahora indicaremos la ruta donde se encuentra el payload dentro de la máquina víctima

````powershell
New-Item -Path "HKCU:Software\Classes\CLSID\{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}" -Name "InprocServer32" -Value "C:\Windows\Temp\http_x64.dll"
`````

````powershell
New-ItemProperty -Path "HKCU:Software\Classes\CLSID\{AB8902B4-09CA-4bb6-B78D-A8F59079A8D5}\InprocServer32" -Name "ThreadingModel" -Value "Both"
`````

Ahora cuando DllHost.exe se ejecute y cargue este objeto COM, se ejecutará nuestro payload

![5Pic]({{ 'assets/img/ComHijacking/Pasted image 20240825145326.png' | relative_url}}){: .center-image }

Para eliminar lo causado basta con borrar la entrada de HKCU en el editor de registros y eliminar el payload de su ruta.

El laboratorio empleado para este ejemplo es proporcionado por [ZeroPoint Security](https://training.zeropointsecurity.co.uk/) para la certificación [CRTO](https://training.zeropointsecurity.co.uk/courses/red-team-ops).
