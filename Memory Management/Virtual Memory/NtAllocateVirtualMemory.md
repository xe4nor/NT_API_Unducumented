# NtAllocateVirtualMemory

> Native API zum Allokieren von virtuellem Speicher im Userland oder Remote-Prozess.
> 🔥 Beliebt in Malware-Entwicklung für **Shellcode-Injektion**, **RAM-only Execution** und **AV-Evasion**.

---

## 📜 Funktionsprototyp

```c
NTSTATUS NtAllocateVirtualMemory(
  IN HANDLE    ProcessHandle,
  IN OUT PVOID *BaseAddress,
  IN ULONG_PTR ZeroBits,
  IN OUT PSIZE_T RegionSize,
  IN ULONG     AllocationType,
  IN ULONG     Protect
);
```

---

## 📌 Parameter

| Parameter        | Typ         | Beschreibung                                                                 |
| ---------------- | ----------- | ---------------------------------------------------------------------------- |
| `ProcessHandle`  | `HANDLE`    | Handle auf den Zielprozess (`PROCESS_VM_OPERATION` nötig).                   |
| `BaseAddress`    | `PVOID*`    | Zieladresse. Wenn `NULL`, sucht das System automatisch einen freien Bereich. |
| `ZeroBits`       | `ULONG_PTR` | Anzahl der erforderlichen führenden Null-Bits. Meist `0`.                    |
| `RegionSize`     | `PSIZE_T`   | Ein-/Ausgabeparameter: gewünschte Größe in Bytes.                            |
| `AllocationType` | `ULONG`     | `MEM_COMMIT`, `MEM_RESERVE` oder beides.                                     |
| `Protect`        | `ULONG`     | Page Protection: z. B. `PAGE_READWRITE`, `PAGE_EXECUTE_READWRITE` usw.       |

---

## 🧠 Warum ist das für Malware-Entwickler wichtig?

Diese Funktion ist der **Low-Level-Zugang zu VirtualAlloc**, ohne dass du die WinAPI verwenden musst. Damit kannst du:

* Speicher **im eigenen oder fremden Prozess** allokieren
* Den Speicher als **ausführbar markieren** (für Shellcode)
* Shellcode **komplett im RAM** verarbeiten, ohne Datei
* AV/EDR umgehen, indem du keine verdächtigen Imports wie `kernel32.dll!VirtualAlloc` nutzt

---

## 🧪 Beispiel: Einfach & Verständlich (ohne ASM, ohne Imports)

```c
#include <windows.h>
#include <winternl.h>
#include <stdio.h>

// Definiere den Funktionsprototyp selbst
typedef NTSTATUS (NTAPI *pNtAllocateVirtualMemory)(
    HANDLE ProcessHandle,
    PVOID *BaseAddress,
    ULONG_PTR ZeroBits,
    PSIZE_T RegionSize,
    ULONG AllocationType,
    ULONG Protect
);

int main() {
    HANDLE hProc = GetCurrentProcess();
    PVOID base = NULL;
    SIZE_T size = 0x1000;

    // Lade ntdll.dll und resolve die Funktion zur Laufzeit
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    if (!ntdll) {
        printf("[!] Konnte ntdll.dll nicht laden\n");
        return 1;
    }

    pNtAllocateVirtualMemory NtAlloc = (pNtAllocateVirtualMemory)
        GetProcAddress(ntdll, "NtAllocateVirtualMemory");

    if (!NtAlloc) {
        printf("[!] NtAllocateVirtualMemory nicht gefunden\n");
        return 1;
    }

    // Speicher allokieren (READWRITE zuerst, später optional RX über NtProtectVirtualMemory)
    NTSTATUS status = NtAlloc(
        hProc,
        &base,
        0,
        &size,
        MEM_COMMIT | MEM_RESERVE,
        PAGE_READWRITE
    );

    if (status != 0) {
        printf("[!] NtAllocateVirtualMemory fehlgeschlagen: 0x%X\n", status);
        return 1;
    }

    printf("[+] Speicher bei %p reserviert (Größe: %llu Bytes)\n", base, size);

    // Beispiel: kleinen Shellcode simulieren (NOP + RET)
    unsigned char shellcode[] = { 0x90, 0x90, 0xC3 }; // NOP NOP RET
    memcpy(base, shellcode, sizeof(shellcode));

    // Ausführbar machen mit VirtualProtect (für Demo-Zwecke)
    DWORD oldProtect = 0;
    if (!VirtualProtect(base, size, PAGE_EXECUTE_READ, &oldProtect)) {
        printf("[!] VirtualProtect fehlgeschlagen\n");
        return 1;
    }

    printf("[+] Speicher jetzt ausführbar. Starte Shellcode...\n");

    // Shellcode ausführen
    ((void(*)())base)();

    return 0;
}
```

> 📝 Hinweis: In echter Malware ersetzt man `VirtualProtect` mit `NtProtectVirtualMemory` für maximale Stealth.

## 📁 Requirements

* **DLL**: `ntdll.dll`
* **Header (inoffiziell)**: `winternl.h` oder eigene Prototyp-Definition
