# bedrock-hook

External memory patching for Minecraft: Bedrock Edition (Windows).  
Signature scanning, code cave allocation, detour hooking.

> Originally made for Minecraft, but it works with regular Windows processes too.

---

## Features

- **Signature scanning** — Find addresses using byte patterns with wildcards
- **Code cave hooking** — Allocate executable memory near the target and write a detour
- **Typed value storage** — Read/write `int` and `float` values through named hooks
- **Auto-reconnect** — Automatically re-hook when the process crashes and restarts
- **Eject** — Restore the original bytes and free the allocated memory

---

## Requirements

- Windows x64
- C++17
- `Psapi.lib`, `kernel32.lib`

---

## Quick Start

### 1. Initialize

```cpp
if (!Game::Initialize()) {
    std::cerr << "Failed to attach to process." << std::endl;
    return 1;
}
```

Looks for `Minecraft.Windows.exe` and opens a handle to it. Obviously fails if the game isn't running.

### 2. Define the signature and hook bytes

```cpp
// -1 is a wildcard
std::vector<int> sig = {
    0x89, 0x83, -1, -1, -1, -1,   // mov [rbx+??], eax
    0x8B, 0x05                      // mov eax, [...]
};

// E9 FF FF FF FF is a placeholder for the jump target (automatically filled in during inject)
std::vector<int> hook = {
    0xE9, 0xFF, 0xFF, 0xFF, 0xFF,  // jmp <detour>
    0x90                            // nop
};

std::vector<int> detour = {
    0x89, 0x05, 0xFF, 0xFF, 0xFF, 0xFF,  // mov [valueAddr], eax
    0xE9, 0xFF, 0xFF, 0xFF, 0xFF          // jmp back
};
```

### 3. Inject

```cpp
bool ok = inject("health", sig, hook, detour, (int)20);
if (!ok) {
    std::cerr << "inject() failed" << std::endl;
}
```

### 4. Read / Write

```cpp
std::variant<int, float> val;
if (read("health", val)) {
    std::cout << "Health: " << std::get<int>(val) << std::endl;
}

write("health", (int)100);
```

### 5. Eject

```cpp
eject("health");
```

---

## API Reference

### `Game::Initialize() → bool`

Finds the process and opens a handle to it. Also gets the module's base address.  
Returns `false` if the process can't be found or the module couldn't be retrieved.

### `inject(name, sig, hook, detour, defaultValue, varSize) → bool`

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string` | Key used to identify the hook |
| `sig` | `vector<int>` | Byte pattern used for scanning. `-1` is a wildcard |
| `hook` | `vector<int>` | Bytes written to the hook site. Must contain `E9 FF FF FF FF` |
| `detour` | `vector<int>` | Shellcode written to the code cave. Must end with `E9 FF FF FF FF` |
| `defaultValue` | `variant<int, float>` | Initial value for the data slot |
| `varSize` | `int` | Size of the value (default: `4`) |

Returns `false` and cleans everything up if the signature can't be found, a code cave can't be allocated, or the placeholder is missing.

### `eject(name) → bool`

Restores the original bytes and frees the allocated code cave.

### `write(name, value) → bool`

Writes an `int` or `float` value to the hook's data slot.

### `read(name, outValue) → bool`

Reads the value from the hook's data slot.

### `resolvePointer(name) → uintptr_t`

Reads the value in the data slot as an 8-byte pointer. Useful when the detour stores an address there.

### `getAllocatedAddress(name) → uintptr_t`

Returns the address of the data slot itself.

---

## About Placeholders

The `E9 FF FF FF FF` placeholder in both `hook` and `detour` gets replaced with the correct relative offset by `SetJump()` during injection.

The code expects exactly one placeholder in each array, so having multiple ones will break things.

---

## Auto-Reconnect

`MonitorThread()` monitors the process in the background every 5 seconds.  
If the process crashes and restarts, it automatically runs `Game::Initialize()` and re-applies the hooks.

```cpp
std::thread monitor(MonitorThread);
monitor.detach();
```

The hook metadata (`sig`, `hook`, `detour`, `lastValue`, `varSize`) is kept internally, so you don't need to write any extra code to handle restarts.

---

## Building

```text
cl /std:c++17 /O2 /EHsc main.cpp /link Psapi.lib
```

---

## License

MIT