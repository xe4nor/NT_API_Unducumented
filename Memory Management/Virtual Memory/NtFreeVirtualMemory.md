# NtFreeVirtualMemory

> Native API zum Freigeben von mit `NtAllocateVirtualMemory` allokiertem Speicher.
> 🔧 Unerlässlich zum Aufräumen von Speicher, AV-Evasion durch temporäre Speicherfreigabe und Anti-Analysis-Techniken.

---

## 📜 Funktionsprototyp

```c
NTSTATUS NtFreeVirtualMemory(
  IN HANDLE        ProcessHandle,          // Pflicht
  IN PVOID        *BaseAddress,            // Pflicht
  IN OUT PSIZE_T   RegionSize,             // Optional (NULL = gesamter Bereich)
  IN ULONG         FreeType                // Pflicht
);
```

---

## 📌 Parameter

| Parameter       | Typ       | Pflicht?    | Beschreibung                                                                            |
| --------------- | --------- | ----------- | --------------------------------------------------------------------------------------- |
| `ProcessHandle` | `HANDLE`  | ✅ Ja        | Handle auf Zielprozess mit `PROCESS_VM_OPERATION`.                                      |
| `BaseAddress`   | `PVOID*`  | ✅ Ja        | Zeiger auf Startadresse des Bereichs, der freigegeben werden soll.                      |
| `RegionSize`    | `PSIZE_T` | ⚠️ Optional | Bei `NULL` wird der komplette zusammenhängende Speicherbereich freigegeben.             |
| `FreeType`      | `ULONG`   | ✅ Ja        | `MEM_DECOMMIT` (nur Seiten-Inhalt löschen) oder `MEM_RELEASE` (ganze Region freigeben). |

---

## 🧠 Nutzung in Malware-Entwicklung

* Aufräumen nach Shellcode-Ausführung, um Forensik zu erschweren
* Temporäres Freigeben sensibler Speicherbereiche (Anti-Dump)
* RAM-basierte Loader, die sich selbst löschen (`self-delete` nach Codeausführung)

---

## 🧪 Beispiel: Speicher freigeben

```c
#include <windows.h>
#include <winternl.h>
#include <stdio.h>

// NtFreeVirtualMemory Prototyp

typedef NTSTATUS (NTAPI *pNtFreeVirtualMemory)(
    HANDLE    ProcessHandle,
    PVOID     *BaseAddress,
    PSIZE_T   RegionSize,
    ULONG     FreeType
);

int main() {
    HANDLE hProc = GetCurrentProcess();
    PVOID base = NULL;
    SIZE_T size = 0x1000;

    // Speicher allokieren wie in NtAllocateVirtualMemory
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    pNtAllocateVirtualMemory NtAlloc = (pNtAllocateVirtualMemory)
        GetProcAddress(ntdll, "NtAllocateVirtualMemory");

    NtAlloc(hProc, &base, 0, &size, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

    // Speicher verwenden (Demo-Zweck)
    strcpy((char*)base, "TEMPORARY");

    // Speicher freigeben mit NtFreeVirtualMemory
    pNtFreeVirtualMemory NtFree = (pNtFreeVirtualMemory)
        GetProcAddress(ntdll, "NtFreeVirtualMemory");

    NTSTATUS status = NtFree(hProc, &base, &size, MEM_RELEASE);

    printf("Freigabe Status: 0x%X\n", status);
    return 0;
}
```

> 💡 Wichtig: `MEM_RELEASE` erfordert, dass `RegionSize` gesetzt ist oder auf `NULL` zeigt – ganze Region wird dann freigegeben.

---

## 📁 Requirements

* **DLL**: `ntdll.dll`
* **Header (inoffiziell)**: `winternl.h`
