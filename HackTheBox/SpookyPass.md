# HTB Binary Analysis Report — SpookyPass

**Source:** Downloaded from HTB room (local analysis)  
**Author:** PortMortem 
**Date:** 2025-09-13

---

## 1. Summary / Objective
This document records a short local analysis of the ELF binary `pass` file. The aim was to identify how the program validates input and to extract embedded secrets/flags. Using simple ELF inspection and runtime execution, the embedded password string was identified with `strings`, which allowed the flag to be obtained.

**Outcome:** HTB flag recovered: `[REDACTED]`

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
[REDACTED]
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
# enter: [REDACTED]
# Output: Welcome inside!
# Output: HTB{[REDACTED]}
```

**Flag recovered:** `HTB{[REDACTED]}`

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

orginial notes:
<details>
so we download the file and unzip it

we run file and get: 

pass: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=3008217772cc2426c643d69b80a96c715490dd91, for GNU/Linux 4.4.0, not stripped

learned a new tool called “filetype -f” that provides good info like:

pass: elf (application/x-executable)

another good one is checksec fo checking security: Check security properties of executables.


checksec --file=pass


RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH  Symbols         FORTIFY Fortified       FortifiableFILE
Partial RELRO   Canary found      NX enabled    PIE enabled     No RPATH   No RUNPATH   31 Symbols    No    0               2          pass

now we update the file to execute with chmod +x pass

it states: 

Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password: 


we do strings pass again and get:

/lib64/ld-linux-x86-64.so.2
fgets
stdin
puts
__stack_chk_fail
__libc_start_main
__cxa_finalize
strchr
printf
strcmp
libc.so.6
GLIBC_2.4
GLIBC_2.2.5
GLIBC_2.34
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
PTE1
u3UH
Welcome to the 
[1;3mSPOOKIEST
[0m party of the year.
Before we let you in, you'll need to give us the password: 
s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
Welcome inside!
You're not a real ghost; clear off!
;*3$"
GCC: (GNU) 14.2.1 20240805
GCC: (GNU) 14.2.1 20240910
main.c
_DYNAMIC
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_start_main@GLIBC_2.34
_ITM_deregisterTMCloneTable
puts@GLIBC_2.2.5
stdin@GLIBC_2.2.5
_edata
_fini
__stack_chk_fail@GLIBC_2.4
strchr@GLIBC_2.2.5
printf@GLIBC_2.2.5
parts
fgets@GLIBC_2.2.5
__data_start
strcmp@GLIBC_2.2.5
__gmon_start__
__dso_handle
_IO_stdin_used
_end
__bss_start
main
__TMC_END__
_ITM_registerTMCloneTable
__cxa_finalize@GLIBC_2.2.5
_init
.symtab
.strtab
.shstrtab
.interp
.note.gnu.property
.note.gnu.build-id
.note.ABI-tag
.gnu.hash
.dynsym
.dynstr
.gnu.version
.gnu.version_r
.rela.dyn
.rela.plt
.init
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.dynamic
.got
.got.plt
.data
.bss
.comment

the password behing : s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5

Welcome inside!
HTB{un0bfu5c4t3d_5tr1ng5}

</details>
