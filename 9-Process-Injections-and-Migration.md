Problem with a basic shellcode runner:
┌─────────────────────────────────────────┐
│  Word.exe / PowerShell runs shellcode   │
│  → User closes the application          │
│  → Shell dies immediately               │
│                                         │
│  AV/EDR detects that Word.exe           │
│  is making outbound network connections │
│  → Blocks or kills the process          │
└─────────────────────────────────────────┘

The solution: move the payload into a different, more trusted process so it survives and blends in.

Key Concepts Before Starting
Process vs Thread
PROCESS
└── Virtual memory space (isolated per process)
    └── THREAD 1  ← executes actual code, has its own stack
    └── THREAD 2
    └── THREAD N
