# NtAllocateVirtualMemory

> Native API zum Allokieren von virtuellem Speicher im Userland oder Remote-Prozess.
> 🔥 Beliebt in Malware-Entwicklung für **Shellcode-Injektion**, **RAM-only Execution** und **AV-Evasion**.

---

## 📜 Funktionsprototyp

```c
NTSTATUS NtAllocateVirtualMemory(
  IN HANDLE      ProcessHandle,
  IN OUT PVOID   *BaseAddress,
  IN ULONG_PTR   ZeroBits,
  IN OUT PSIZE_T RegionSize,
  IN ULONG       AllocationType,
  IN ULONG       Protect
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
| `Protect`        | `ULONG`     | Page Protection: z. B. `PAGE_READWRITE`, `PAGE_EXECUTE_READWRITE` usw.       |

---

## 🧠 Malware-Entwicklung: Nutzung & Tipps

### ✅ Typischer Ablauf für Shellcode Injection

1. **Speicher reservieren**:

   ```c
   NtAllocateVirtualMemory(..., MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
   ```
2. **Shellcode reinschreiben**:
   → z. B. via `NtWriteVirtualMemory`
3. **Speicher ausführbar machen**:

   ```c
   NtProtectVirtualMemory(..., PAGE_EXECUTE_READ);
   ```
4. **Code ausführen**:
   → per `NtCreateThreadEx`, `NtQueueApcThread`, oder `SetThreadContext`.

### 🕵️‍♂️ AV-Evasion-Taktiken

* **Keine `VirtualAlloc` benutzen**, sondern direkt `NtAllocateVirtualMemory` via:

  * `GetProcAddress(LoadLibraryA("ntdll.dll"), "NtAllocateVirtualMemory")`
  * **oder** direkt per Inline-Syscall (`syscall`)
* **Zuerst RW, dann RX!**

  * Erst `PAGE_READWRITE` nutzen → nach dem Schreiben zu `PAGE_EXECUTE_READ` wechseln
* **Syscall Stub verschleiern** (z. B. über XOR)
* **Inline Shellcode Loader** verwenden, der alle NT-Funktionen direkt via Syscall nutzt (z. B. über [SysWhispers2](https://github.com/jthuraisamy/SysWhispers2))

---

## 🧪 Beispielcode

```c
HANDLE hProc = GetCurrentProcess();
PVOID base = NULL;
SIZE_T size = 0x1000;

NTSTATUS status = NtAllocateVirtualMemory(
    hProc,
    &base,
    0,
    &size,
    MEM_COMMIT | MEM_RESERVE,
    PAGE_EXECUTE_READWRITE
);
```

> 💡 Achtung: Für stealth solltest du lieber `PAGE_READWRITE` nutzen und später mit `NtProtectVirtualMemory` auf `PAGE_EXECUTE_READ` ändern!

---

## 🔗 Siehe auch

* [`NtFreeVirtualMemory`](./NtFreeVirtualMemory.md)
* [`NtProtectVirtualMemory`](./NtProtectVirtualMemory.md)
* [`NtWriteVirtualMemory`](./NtWriteVirtualMemory.md)
* [`NtCreateThreadEx`](./NtCreateThreadEx.md)
* [`NtQueueApcThread`](./NtQueueApcThread.md)

---

## 📚 Quellen

* 📘 [ReactOS Source](https://github.com/reactos/reactos/tree/master/reactos/dll/ntdll)
* 🔬 [Geoff Chappell - Native API Research](https://www.geoffchappell.com/studies/windows/win32/ntdll/api/)
* 🧪 [Maldev Academy](https://maldevacademy.com/)
* 🛠 [SysWhispers2](https://github.com/jthuraisamy/SysWhispers2)

---

## 📁 Requirements

* **DLL**: `ntdll.dll`
* **Header (inoffiziell)**: `winternl.h`
  (oder eigene Definition)

---

*Dokumentiert von: BitBounty & ReactOS Community*
*Letztes Update: {{DATE}}*

