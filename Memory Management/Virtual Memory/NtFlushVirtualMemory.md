# NtFlushVirtualMemory

> Native API zum "Flushen" von gemapptem Speicher (z. B. von Sektionen) auf die zugeordnete Datei.
> 🚧 In Malware-Entwicklung selten genutzt, aber wichtig für Low-Level Dateisystemmanipulation und stealthy Persistence.

---

## 📜 Funktionsprototyp

```c
NTSTATUS NtFlushVirtualMemory(
  IN HANDLE             ProcessHandle,
  IN OUT PVOID         *BaseAddress,
  IN OUT PULONG         NumberOfBytesToFlush,  //Optional
  OUT PIO_STATUS_BLOCK  IoStatusBlock
);
```

---

## 📌 Parameter

| Parameter              | Typ                | Beschreibung                                                                                        |
| ---------------------- | ------------------ | --------------------------------------------------------------------------------------------------- |
| `ProcessHandle`        | `HANDLE`           | Handle auf den Zielprozess mit gemapptem View (i. d. R. durch `NtMapViewOfSection`).                |
| `BaseAddress`          | `PVOID*`           | Adresse des gemappten Bereichs, der geschrieben werden soll. Wird auf Page-Größe (0x1000) gerundet. |
| `NumberOfBytesToFlush` | `PULONG`           | Wie viele Bytes ab `BaseAddress` geflusht werden sollen. Ebenfalls auf Page Size gerundet.          |
| `IoStatusBlock`        | `PIO_STATUS_BLOCK` | Struktur mit Status über den I/O-Vorgang (enthält u. a. Ergebniscode & Transfergröße).              |

---

## 🧠 Wann wird das verwendet?

Diese Funktion ist nützlich, wenn du mit **gemappten Dateien arbeitest** (z. B. via `NtMapViewOfSection`) und sicherstellen willst, dass Änderungen am Memory auch in die Datei zurückgeschrieben werden.

In legitimen Szenarien:

* Bei selbstgebauten File-Mappers, Debuggern oder Loadern
* Persistence-Techniken (z. B. versteckter Code in PE-Dateien nachträglich reinschreiben)

In Malware:

* „Advanced“ Stealth: Shellcode wird in gemappter Datei gespeichert (in Memory), dann per Flush in das File geschrieben
* Manipulation von geladenen DLL-Dateien

---

## 🧪 Beispiel: Datei-Inhalt im Speicher verändern und flushen

```c
#include <windows.h>
#include <winternl.h>
#include <stdio.h>

typedef NTSTATUS (NTAPI *pNtFlushVirtualMemory)(
    HANDLE ProcessHandle,
    PVOID *BaseAddress,
    PULONG NumberOfBytesToFlush,
    PIO_STATUS_BLOCK IoStatusBlock
);

int main() {
    // Beispiel: Datei wird über CreateFileMapping + MapViewOfFile gemappt
    HANDLE hFile = CreateFileA("test.bin", GENERIC_READ | GENERIC_WRITE, 0, NULL, OPEN_EXISTING, 0, NULL);
    if (hFile == INVALID_HANDLE_VALUE) return 1;

    HANDLE hMap = CreateFileMappingA(hFile, NULL, PAGE_READWRITE, 0, 0, NULL);
    if (!hMap) return 1;

    LPVOID lpMap = MapViewOfFile(hMap, FILE_MAP_ALL_ACCESS, 0, 0, 0);
    if (!lpMap) return 1;

    // Dateiinhalt manipulieren
    strcpy((char*)lpMap, "HACKED\n");

    // NtFlushVirtualMemory aufrufen
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    pNtFlushVirtualMemory NtFlush = (pNtFlushVirtualMemory)
        GetProcAddress(ntdll, "NtFlushVirtualMemory");

    if (!NtFlush) return 1;

    ULONG bytes = 0x1000; // Flush eine Seite
    IO_STATUS_BLOCK io = { 0 };

    NTSTATUS status = NtFlush(
        GetCurrentProcess(),
        &lpMap,
        &bytes,
        &io
    );

    printf("Flush Status: 0x%X (IO-Status: 0x%X)\n", status, io.Status);

    UnmapViewOfFile(lpMap);
    CloseHandle(hMap);
    CloseHandle(hFile);

    return 0;
}
```

> 💡 Achtung: Diese Funktion funktioniert **nur auf Speicher, der per `NtMapViewOfSection` oder `MapViewOfFile`** gemappt wurde!

---

## ⚠️ Hinweis

* Wenn **mehrere Memory-Mappings** mit `NtMapViewOfSection` aktiv sind, können sie **nicht gleichzeitig** mit einem einzigen `NtFlushVirtualMemory`-Call geschrieben werden, selbst wenn sie auf dieselbe Datei zeigen.

---

## 📁 Requirements

* **DLL**: `ntdll.dll`
* **Header (inoffiziell)**: `winternl.h`
