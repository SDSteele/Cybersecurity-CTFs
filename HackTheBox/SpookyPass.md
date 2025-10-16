# HTB Binary Analysis Report — `pass` (ELF x86_64)

**Target binary:** `pass`  
**Source:** Downloaded from HTB room (local analysis)  
**Author:** [your name / handle]  
**Date:** 2025-09-13

---

## 1. Summary / Objective
This document records a short local analysis of the ELF binary `pass`. The aim was to identify how the program validates input and to extract embedded secrets/flags. Using simple ELF inspection and runtime execution, the embedded password string was identified with `strings`, which allowed the flag to be obtained.

**Outcome:** HTB flag recovered: `HTB{un0bfu5c4t3d_5tr1ng5}`

---

## 2. Environment & initial steps
1. Downloaded and unzipped the challenge files into the analysis directory.  
2. Verified the binary type and metadata:
   ```bash
   file pass
   # ELF 64-bit LSB pie executable, x86-64, dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, not stripped
   ```
3. Used `filetype -f` for additional classification (optional):
   ```bash
   filetype -f pass
   # pass: elf (application/x-executable)
   ```

---

## 3. Binary hardening check
Ran `checksec` to determine protections:
```bash
checksec --file=pass
```
**Result summary:**
- Partial RELRO
- Stack Canary found
- NX enabled
- PIE enabled
- 31 Symbols present (binary is not stripped)

These protections are present, but the binary is not stripped and contains readable symbols and strings.

---

## 4. Runtime execution & interaction
Made the binary executable and executed it:

```bash
chmod +x pass
./pass
```

Program prompt:
```
Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password:
```

The program expects an input password. To locate the password, static string scanning was performed.

---

## 5. Static string inspection
Using `strings` on the binary revealed several human-readable strings. Notable excerpt:

```
Welcome to the 
SPOOKIEST
Before we let you in, you'll need to give us the password: 
s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
Welcome inside!
You're not a real ghost; clear off!
...
```

The embedded plaintext password discovered: `s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5`.

---

## 6. Verification & flag retrieval
Provided the discovered password to the running program. The binary responded with the success message and printed the flag.

**Steps to reproduce:**
```bash
./pass
# enter: s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
# Output: Welcome inside!
# Output: HTB{un0bfu5c4t3d_5tr1ng5}
```

**Flag recovered:** `HTB{un0bfu5c4t3d_5tr1ng5}`

---

## 7. Analysis notes & observations
- The binary is **not stripped**; presence of symbols and readable strings made extraction trivial.  
- Typical mitigations (canary, NX, PIE) do not prevent secret leakage via embedded strings.  
- Tools used: `file`, `filetype`, `checksec`, `strings`, `chmod`, and direct execution. For deeper analysis, consider `objdump`, `Ghidra`, or `gdb`.

---

## 8. Recommendations (for challenge authors / defenders)
- Strip binaries prior to distribution to avoid leaking symbols and readable strings (`strip`).  
- Remove or encrypt sensitive strings before compiling; use runtime-provided secrets.  
- Consider additional obfuscation or runtime string protection if embedding secrets is unavoidable.

---

## 9. Reproducible commands (reference)
```bash
unzip challenge.zip
cd challenge_dir
file pass
filetype -f pass
checksec --file=pass
chmod +x pass
strings pass | grep -i 's3cr3t\|password\|HTB'
./pass   # provide discovered password when prompted
```

---

## 10. Artifacts / evidence
- Binary: `pass` (original)  
- Extracted strings output: `strings_pass.txt`  
- Execution log: `pass_run.log` (recorded interactive run)

---

*End of report.*
