Strings can only help you so far.

Download the task file and get started. The binary can also be found in the AttackBox inside the /root/Rooms/Compiled/ directory.

Note: The binary will not execute if using the AttackBox. However, you can still solve the challenge.

Answer the questions below
What is the password?

running it and changing commands it just says try again

└─$ strings Com*      
/lib64/ld-linux-x86-64.so.2
jKUhR
__cxa_finalize
__libc_start_main
strcmp
stdout
__isoc99_scanf
fwrite
printf
libc.so.6
GLIBC_2.7
GLIBC_2.2.5
GLIBC_2.34
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
PTE1
u+UH
StringsIH
sForNoobH
Password: 
DoYouEven%sCTF
__dso_handle
_init
Correct!
Try again!
;*3$"
GCC: (Debian 11.3.0-5) 11.3.0
Scrt1.o
__abi_tag
crtstuff.c
deregister_tm_clones
__do_global_dtors_aux
completed.0
__do_global_dtors_aux_fini_array_entry
frame_dummy
__frame_dummy_init_array_entry
zzz.c
__FRAME_END__
_DYNAMIC
__GNU_EH_FRAME_HDR
_GLOBAL_OFFSET_TABLE_
__libc_start_main@GLIBC_2.34
_ITM_deregisterTMCloneTable
stdout@GLIBC_2.2.5
_edata
_fini
printf@GLIBC_2.2.5
__data_start
strcmp@GLIBC_2.2.5
__gmon_start__
__dso_handle
_IO_stdin_used
_end
__bss_start
main
__isoc99_scanf@GLIBC_2.7
fwrite@GLIBC_2.2.5
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
.plt.got
.text
.fini
.rodata
.eh_frame_hdr
.eh_frame
.init_array
.fini_array
.dynamic
.got.plt
.data
.bss
.comment

downloaded and used Ghidra, looking at hte file we see that we can get "correct!" if we enter _init

now let's try DoYouEven_init

and that's the answer for the puzzle!
