# NtLockVirtualMemory

> Native API zum Sperren eines Speicherbereichs im RAM.
> 🔐 Verhindert, dass der Speicher in die Auslagerungsdatei geschrieben wird – wichtig für Forensik-Resistenz, Anti-Swap-Techniken und sichere Payload-Verarbeitung.

---

## 📜 Funktionsprototyp

```c
NTSTATUS NtLockVirtualMemory(
  IN HANDLE       ProcessHandle,             // Pflicht
  IN PVOID        BaseAddress,               // Pflicht
  IN OUT PULONG   NumberOfBytesToLock,       // Pflicht
  IN ULONG        LockOption                 // Pflicht
);
```

---

## 📌 Parameter

| Parameter             | Typ      | Pflicht? | Beschreibung                                                                  |
| --------------------- | -------- | -------- | ----------------------------------------------------------------------------- |
| `ProcessHandle`       | `HANDLE` | ✅ Ja     | Handle auf Prozess, dessen Speicher gesperrt werden soll.                     |
| `BaseAddress`         | `PVOID`  | ✅ Ja     | Startadresse des zu sperrenden Speicherbereichs. Muss bereits allokiert sein. |
| `NumberOfBytesToLock` | `PULONG` | ✅ Ja     | Anzahl der zu sperrenden Bytes. Wird auf Page-Größe gerundet.                 |
| `LockOption`          | `ULONG`  | ✅ Ja     | Entweder `VM_LOCK_1` (0x0001) oder `VM_LOCK_2` (0x0002).                      |

---

## 🧠 Warum ist das relevant?

Durch Sperren von Speicher mit `NtLockVirtualMemory` kannst du verhindern, dass der Speicher **in die Auslagerungsdatei geschrieben wird**. Das ist vor allem wichtig bei:

* **Kryptografischen Schlüsseln** (nicht persistieren!)
* **Shellcode im RAM**, der nicht auf der Platte auftauchen darf
* **EDR-Evasion**, indem sensitive Payloads nicht ausgelagert werden
* **Forensik-Resistenz**, z. B. gegen RAM-Dumps

---

## 🔐 LockOption Werte

```c
#define VM_LOCK_1 0x0001 // Wird z. B. bei VirtualLock verwendet
#define VM_LOCK_2 0x0002 // Benötigt SE_LOCK_MEMORY_NAME Privileg
```

> 💡 Du kannst beide Flags kombinieren mit `|`, z. B. `VM_LOCK_1 | VM_LOCK_2`

---

## 🧪 Beispiel: RAM-Bereich sperren

```c
#include <windows.h>
#include <winternl.h>
#include <stdio.h>

// Prototyp definieren

typedef NTSTATUS (NTAPI *pNtLockVirtualMemory)(
    HANDLE ProcessHandle,
    PVOID BaseAddress,
    PULONG NumberOfBytesToLock,
    ULONG LockOption
);

int main() {
    SIZE_T size = 0x1000;
    PVOID base = NULL;

    // Speicher reservieren
    NtAllocateVirtualMemory(GetCurrentProcess(), &base, 0, &size, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

    // Inhalt schreiben (z. B. Payload, Schlüssel)
    strcpy((char*)base, "SensitiveData");

    // Speicher sperren
    HMODULE ntdll = GetModuleHandleA("ntdll.dll");
    pNtLockVirtualMemory NtLock = (pNtLockVirtualMemory)GetProcAddress(ntdll, "NtLockVirtualMemory");

    ULONG len = (ULONG)size;
    NTSTATUS status = NtLock(GetCurrentProcess(), base, &len, 0x0001);

    printf("Lock Status: 0x%X\n", status);

    return 0;
}
```

> 🔐 Nutze `SE_LOCK_MEMORY_NAME`, wenn du `VM_LOCK_2` nutzen willst (z. B. als Dienst mit Privilegien).

---

## ⚠️ Hinweis

* Der Speicher **muss vorher allokiert** worden sein.
* Locking funktioniert nur mit aktivierten Privilegien (für `VM_LOCK_2`).

---

## 📁 Requirements

* **DLL**: `ntdll.dll`
* **Privilege**: `SE_LOCK_MEMORY_NAME` (für `VM_LOCK_2`)
* **Header**: `winternl.h`
