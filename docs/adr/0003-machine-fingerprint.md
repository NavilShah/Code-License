# ADR-0003: Machine fingerprint and matching rule

| Field | Value |
|---|---|
| Status | Proposed. Validate in T9 on a physical PC, a VM and a VM clone |
| Date | 2026-10-23 |
| Related | [Requirements](../01-requirements.md) FR-SDK-2, FR-ACT-6, FR-ACT-7, NFR-3 |

## Context

Licenses are per machine (decision brief §2). The desktop apps run on Windows: .NET Framework
4.7+ is Windows-only, and the .NET Core and Python apps are assumed to be Windows too. Linux is
out of scope until a Linux desktop target exists.

The fingerprint must:

- survive common maintenance without reactivation (an OS reinstall, a disk swap);
- reject a license file copied to another machine;
- reject a cloned VM that the hypervisor gave a new UUID;
- need no administrator rights and no third-party packages (FR-SDK-6).

## Decision

### Components

| Key | Component | Source | Changes when |
|---|---|---|---|
| `mg` | Windows MachineGuid | Registry `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid`, read through the 64-bit registry view (`RegistryView.Registry64`) so that 32-bit processes are not redirected to `WOW6432Node` | The OS is reinstalled. Cloned images that were not sysprepped share it |
| `su` | SMBIOS system UUID | `GetSystemFirmwareTable('RSMB')`, SMBIOS structure type 1 | The motherboard is replaced; a new VM is created; most VM clones |
| `ds` | Serial number of the physical disk holding the Windows system volume | `DeviceIoControl` with `IOCTL_VOLUME_GET_VOLUME_DISK_EXTENTS` to find the disk, then `IOCTL_STORAGE_QUERY_PROPERTY` to read its serial | The disk is replaced. Virtual disks often have no serial |

**To verify in T9:** each call must work without administrator rights on Windows 10/11 and on
the Windows versions found on clients' industrial PCs.

### Normalization and hashing

1. **Normalize** each value: trim whitespace and NUL characters, then convert to upper case.
2. **Treat a value as missing** if it is:
   - empty;
   - an all-zero or all-F UUID;
   - a known firmware placeholder, such as "To be filled by O.E.M.", "Default string", "None"
     or "0".
3. **Hash:** compute the SHA-256 of the UTF-8 string `"{product_code}|{key}|{value}"` and store
   it as lowercase hex. Including the product code means the same machine yields different
   hashes in different products.
4. **Only the hashes leave the machine** (NFR-3). The hostname is sent separately as
   `machine_label`. It is used for display and misuse detection (FR-ACT-7), never for matching.

The Python SDK (Phase 2) must produce identical hashes. Cross-language fingerprint vectors,
mapping raw values to expected hashes, live in `test-vectors/`.

### Matching rule

The fingerprint matches when **both** of these hold:

1. at least 2 of the 3 components match, and
2. if the activation recorded a valid `su`, the current `su` also matches.

If the activation recorded fewer than 2 valid components, then every recorded component must
match, and the activation is flagged for review when it is created.

| Event | `mg` | `su` | `ds` | Result |
|---|---|---|---|---|
| OS reinstalled | changes | same | same | Valid |
| Disk replaced (OS image restored) | same | same | changes | Valid |
| Disk replaced and OS reinstalled | changes | same | changes | Invalid: reactivate |
| Motherboard replaced | same | changes | same | Invalid: Navil deactivates it from the admin without using a client reset (FR-ACT-6) |
| License file copied to another PC | differs | differs | differs | Invalid |
| VM cloned with a new UUID | same | changes | same or none | Invalid |
| VM moved with its UUID kept | same | same | same | Valid. A moved VM cannot be told apart from a copy offline; when online, FR-ACT-7 flags two machines refreshing one activation |

Making `su` mandatory is what stops VM clones. The price is that a motherboard replacement needs
a support deactivation, which is rare and handled from the admin.

## Alternatives considered

| Option | Why not chosen |
|---|---|
| MAC address | Changed by virtual adapters, USB network adapters, Wi-Fi toggling and address randomization |
| CPU `ProcessorId` | Identifies the CPU model, not the machine |
| Baseboard serial | Often a placeholder on industrial and white-box PCs. Could be added later as an optional fourth component |
| WMI (`System.Management`) | Needs a NuGet package on netstandard2.0, and is slower. Direct Win32 calls return the same data |
| A device-bound key (DPAPI or TPM) | Resists copying better, but adds complexity and makes recovery harder. Revisit if copying becomes a measured problem |
| Strict 3-of-3 matching | Every reinstall or disk swap would need reactivation and use up the client's resets |

## Consequences

**Benefits**

- No administrator rights and no dependencies.
- Survives the most common single-part maintenance.
- Only hashes are collected.

**Costs**

- A motherboard replacement, or a disk replacement combined with an OS reinstall, needs
  reactivation.
- Offline, a moved VM looks the same as a copied one.
- The Win32 interop code needs testing on real client hardware (T9).
