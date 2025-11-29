# Dynamic Linking in Linux: How Binaries Find Shared Libraries


## 1. **What is a binary/executable?**

A compiled program (machine code) that your computer runs directly.

- Example: `firefox`, `chrome`, `ls`

-----

## 2. **Two types of binaries:**

### **Statically linked:**

- Contains ALL code it needs inside itself (huge file)
- Doesn’t need external libraries
- Rare nowadays

### **Dynamically linked:**

- Small file, relies on external **shared libraries**
- Most binaries are this type
- More efficient (multiple programs share same library)

-----

## 3. **What are shared libraries (.so files)?**

Reusable code that multiple programs use.

- **Windows equivalent:** `.dll` files
- **Linux:** `.so` files (shared object)
- Example: `libc.so` (standard C library used by almost everything)

**Why they exist:** Instead of every program including the same code, they all point to one shared library file.

-----

## 4. **How does a binary find its libraries?**

When you run `./my-program`, it needs to find its `.so` files.

**The dynamic linker** (`ld.so`) does this job:

1. Binary says: “I need `libfoo.so.1`”
1. Dynamic linker searches standard locations:

- `/lib`
- `/usr/lib`
- `/lib64`

1. Finds it → loads it → program runs ✅

-----

## 5. **The NixOS problem:**

**Traditional Linux:**

```
/usr/lib/libfoo.so  ← Library is here
```

**NixOS:**

```
/nix/store/abc123-libfoo-1.0/lib/libfoo.so  ← Library is here
```

Binary looks in `/usr/lib` → doesn’t find it → crashes ❌

-----

## 6. **Solution 1: LD_LIBRARY_PATH (manual)**

`LD_LIBRARY_PATH` is an **environment variable** that tells the dynamic linker: “Also search in these directories.”

```bash
# Run binary with custom library path
LD_LIBRARY_PATH=/nix/store/abc123-libfoo-1.0/lib ./my-program
```

Now the dynamic linker searches:

1. Standard locations (`/lib`, `/usr/lib`)
1. **Plus** your custom path (`/nix/store/...`)
1. Finds library → works ✅

**Downside:** You have to set this every time you run the binary.

-----

## 7. **Solution 2: RPATH (permanent fix)**

**RPATH** is a list of directories **embedded inside the binary itself**.

Think of it as: “This binary has a note inside that says ‘my libraries are in these directories.’”

**Original binary:**

```
RPATH: /usr/lib  (expects FHS)
```

**Patched binary (Nix does this):**

```
RPATH: /nix/store/abc123-libfoo-1.0/lib
```

Now the dynamic linker automatically looks in the right place without needing `LD_LIBRARY_PATH`.

**Tool to modify RPATH:** `patchelf`

-----

## 8. **How NixOS automates this:**

When you install from nixpkgs:

**Step 1:** Nix builds/downloads the binary  
**Step 2:** Nix patches the RPATH to point to `/nix/store/...` locations  
**Step 3:** You run the binary → it already knows where its libraries are ✅

-----

## 9. **For random binaries NOT from nixpkgs:**

You have to manually help them find libraries:

### **Option A:** Set `LD_LIBRARY_PATH` each time

```bash
LD_LIBRARY_PATH=/nix/store/xyz.../lib ./random-binary
```

### **Option B:** Patch the binary yourself

```bash
patchelf --set-rpath /nix/store/xyz.../lib ./random-binary
```

### **Option C:** Use NixOS wrappers

- `steam-run ./random-binary` - runs in fake FHS environment
- `nix-ld` - provides compatible dynamic linker

-----

## 10. **Summary in one flow:**

**Traditional Linux:**

```
Binary → needs libfoo.so → looks in /usr/lib → finds it → runs
```

**NixOS (broken):**

```
Binary → needs libfoo.so → looks in /usr/lib → NOT THERE → crashes
```

**NixOS (your manual fix):**

```
LD_LIBRARY_PATH=/nix/store/.../lib
Binary → needs libfoo.so → looks in custom path → finds it → runs
```

**NixOS (automatic fix via nixpkgs):**

```
Binary has RPATH=/nix/store/.../lib embedded
Binary → needs libfoo.so → looks in own RPATH → finds it → runs
```

-----

Does this clear everything up? Want me to explain any specific part deeper?​​​​​​​​​​​​​​​​