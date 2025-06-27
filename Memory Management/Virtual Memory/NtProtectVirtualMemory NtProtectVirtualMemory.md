# NtProtectVirtualMemory

> Native API zum Ändern der Zugriffsrechte eines Speicherbereichs.
> 🔥 Einer der wichtigsten NT-Funktionen für Malware-Entwicklung, Shellcode Execution, AV-Evasion und Runtime-Unpacking.

---

## 📜 Funktionsprototyp

```c
NTSTATUS NtProtectVirtualMemory(
  IN HANDLE       ProcessHandle,            // Pflicht
  IN OUT PVOID   *BaseAddress,              // Pflicht
  IN OUT PSIZE_T  NumberOfBytesToProtect,   // Pflicht
  IN ULONG        NewAccessProtection,      // Pflicht
  OUT PULONG      OldAccessProtection       // Optional (NULL = ignorieren)
);
```

---

## 📌 Parameter

| Parameter                | Typ       | Pflicht?    | Beschreibung                                                                             |
| ------------------------ | --------- | ----------- | ---------------------------------------------------------------------------------------- |
| `ProcessHandle`          | `HANDLE`  | ✅ Ja        | Handle auf Zielprozess mit `PROCESS_VM_OPERATION` Rechten.                               |
| `BaseAddress`            | `PVOID*`  | ✅ Ja        | Adresse des Speicherbereichs, der geschützt werden soll. Wird auf Seitenanfang gerundet. |
| `NumberOfBytesToProtect` | `PSIZE_T` | ✅ Ja        | Wie viele Bytes geschützt werden sollen. Auf Page-Größe (4 KB) gerundet.                 |
| `NewAccessProtection`    | `ULONG`   | ✅ Ja        | Neue Rechte: z. B. `PAGE_EXECUTE_READWRITE`, `PAGE_READONLY` usw.                        |
| `OldAccessProtection`    | `PULONG`  | ⚠️ Optional | Alter Schutz wird hier gespeichert (z. B. für Restore).                                  |

---

## 🧠 Warum ist das für Malware wichtig?

* **Shellcode Execution**: RW ➔ RX ➔ Jump. Du lädst Shellcode in `PAGE_READWRITE` und änderst dann auf `PAGE_EXECUTE_READ`.
* **AV/EDR-Evasion**: RWX (Read/Write/Execute) Speicher ist extrem auffällig. RW ➔ RX reduziert Detection.
* **Obfuscation & Unpacking**: Loader dekomprimiert Payload in RW-Speicher, macht sie dann RX und springt hinein.

---

## 🧪 Beispiel: Shellcode RW ➔ RX

```c
#include <windows.h>
#include <winternl.h>
#include <stdio.h>

// NtProtectVirtualMemory Prototyp definieren
typedef NTSTATUS (NTAPI *pNtProtectVirtualMemory)(
    HANDLE ProcessHandle,
    PVOID *BaseAddress,
    PSIZE_T RegionSize,
    ULONG NewProtect,
    PULONG OldProtect
);

int main() {
    HANDLE hProc = GetCurrentProcess();
    PVOID base = NULL;
    SIZE_T size = 0x1000;

    // Speicher allokieren
    NtAllocateVirtualMemory(hProc, &base, 0, &size, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

    // Shellcode oder Payload reinschreiben
    strcpy((char*)base, "\x90\x90\xC3"); // NOP NOP RET

    // Rechte ändern
    ULONG oldProtect = 0;
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    pNtProtectVirtualMemory NtProtect = (pNtProtectVirtualMemory)
        GetProcAddress(ntdll, "NtProtectVirtualMemory");

    NTSTATUS status = NtProtect(
        hProc,
        &base,
        &size,
        PAGE_EXECUTE_READ,
        &oldProtect
    );

    printf("NTSTATUS: 0x%X\n", status);
    printf("Alter Schutz: 0x%X\n", oldProtect);

    // Ausführen (gefährlich: nur Beispiel!)
    ((void(*)())base)();

    return 0;
}
```

> 🛡️ Hinweis: `OldAccessProtection` kann auch `NULL` sein, wenn du den alten Status nicht brauchst.

---

## 🔓 Wichtige PAGE\_\* Flags

| Flag                     | Beschreibung                                      |
| ------------------------ | ------------------------------------------------- |
| `PAGE_READWRITE`         | Lesen & Schreiben erlaubt (nicht ausführbar)      |
| `PAGE_EXECUTE_READ`      | Lesen & Ausführen erlaubt (typisch für Shellcode) |
| `PAGE_NOACCESS`          | Kein Zugriff erlaubt                              |
| `PAGE_EXECUTE_READWRITE` | Lesen, Schreiben & Ausführen (auffällig!)         |

---

## 📁 Requirements

* **DLL**: `ntdll.dll`
* **Header (inoffiziell)**: `winternl.h`
