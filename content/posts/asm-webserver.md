---
title: "Let's make an HTTP server with Assembly"
date: 2025-01-17T22:37:58+07:00
toc: true
tags:
  - programming
  - low level system
  - assembly
author: "Nam Nguyen"
description: "Building an HTTP server with x86 Assembly and Linux syscalls"
---

One of the things that makes me happy is understanding how something works at its core. That's basically the main reason why I built this HTTP web server entirely in x86 Assembly. Assembly is the lowest level you can go in a computer before you start working with machine code and building an HTTP server requires an understanding of networking.

This project was really fun. It really put my Assembly, multi-processing, and syscall knowledge to the test and I hope you will learn something about Assembly, networking, and low level systems. Let's start with some basics to set the stage.

## How the Internet works

Have you ever wondered what happens when you enter a link into your browser? How did the browser get the website?

When you enter a link into your browser's search bar, the browser will make an HTTP request, particularly a GET request, to the web server associated with the domain of in the URL.

Here's a high-level break down of all the processes that goes into a website request:
- DNS resolution: The browser will give the DNS (Domain Name System) the request domain to get an IP address.
- Server connection: The browser starts connecting to the server with the specified IP address usually through TCP.
- HTTP request: After connecting to the server, the browser sends a GET request to request resources from the server.
- Server response: The server will process the request and send back an HTTP response, which may contain the requested resources or an error message if the server couldn't process the request.
- Response acceptance: The browser will take the HTTP response and render the website or modify the rendered website.

## Intro to Assembly

Assembly knowledge is a prerequisite for this project. I'll go over Assembly briefly as a refresher but if you don't know Assembly, I highly recommend learning it from [**here**](https://www.cs.virginia.edu/~evans/cs216/guides/x86.html).

Assembly is a low level programming language, it's the lowest you could go before you start writing machine code itself. It basically provides a way to write instructions that interacts directly with the underlying computer architecture. Assembly allows for deep control over hardware and system resources.

Each different computer architecture has a different flavor of Assembly, which contain will different instructions, though some instructions will persist across architectures.

When programming with Assembly, the main idea is moving data around between memory and registers and invoking interrupts to perform some calculation or processing. Look into any Assembly project and you'll see the myriad of `mov` instructions used to move data around and many interrupt instructions. Heck, there's even an entire [**project**](https://github.com/xoreaxeaxeax/movfuscator) that translate your C code into a bunch of `mov` instructions.

`mov` is not the only thing Assembly can do though. Each flavor of Assembly usually comes with some arithmetic operations on registers and memory like `add`, `sub`, along with control flow instructions that allows you to jump to different parts of the code like `jmp`, `je`, `jne`, etc.

```asm
.intel_syntax noprefix
.global _start

_start:
mov edx, [rdi]
mov eax, [rdi+4]
mov ebx, [rdi+8]
mov ecx, [rdi+12]

cmp edx, 0x7f454c46
je CON_1
cmp edx, 0x00005A4D
je CON_2
```

Interrupts will *interrupt* the currently executing process to process an event. We'll be working with interrupts a lot for this, mainly because syscall is essentially interrupts. When an interrupt is invoked, it transfers the program flow to the specified handler. The interrupt handler is specified in what's called the [**Interrupt Vector Table**](https://en.wikipedia.org/wiki/Interrupt_vector_table) (IVT).

We're particularly interested in the UNIX syscall interrupt, specified by the interrupt vector `0x80`. We'll be using this extensively, but first, let's take a look at what syscalls are.

## Syscalls basics

System calls (or syscalls for short) are interfaces between the user space and the kernel space. User space is where user applications and software is executed. Kernel space is where the kernel resides. The kernel is the most important part of an OS. It sits between the higher-level applications and the lower-level hardware and facilitates communication between the two.

{{< image src="/img/asm-webserver/kernel-space-diagram.png" alt="Kernel space diagram" position="center" style="padding: 20px" >}}

Applications in the user space interract with the kernel through syscalls. Syscalls allows the applications to request services from the OS, like networking, file operations, process control, etc.

On UNIX systems, syscalls are invoked through the interrupt vector `0x80`. The syscall service that we want to request is specified in the `eax` register. We'll be following the [**System V AMD64 ABI**](https://en.wikipedia.org/wiki/X86_calling_conventions#System_V_AMD64_ABI) when we're passing parameters to these syscalls. Below is an example on how to call the `exit` syscall, which will exit the executing program with a specified exit code.

```asm
# exit(0)
mov rdi, 0        # exit code
mov rax, 60       # sys_exit
syscall
```

So we passed the exit code into the `rdi` register, pass the syscall service number that we want to request into the `rax` register, then invoke the syscall interrupt. There are many different syscalls with different syscall service number, you can find all of them in the [**syscall table**](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/).

## All about sockets

If you've ever studied about networking, you'd be quite familiar with the concept of *sockets*. Sockets are software objects that allows you to bind and listen on an endpoint and send or receive data through that endpoint to other devices and other networks. It's used extensively in the Internet. The socket address will contain the triad of the Internet protocol: transport protocol, IP address, and port number.

Socket programming means connecting 2 sockets together to communicate with each other. One socket will listen on a particular port at a particular IP while the other socket will connect to that listening socket to form a connection. Below is a diagram of how this connection and communication process works.

{{< image src="/img/asm-webserver/socket-flow-diagram.png" alt="Socket flow diagram" position="center" style="padding: 40px" >}}

So how exactly do sockets work and how can we program one? There's a few steps to making a socket:
1. *Create a socket*: We can use the `socket` syscall to create a socket object. We can specify the type of IP address to use and which communication protocol to use.
2. *Bind the socket*: We can use the `bind` syscall to bind the created socket to a particular network interface on a particular port. The socket will then be able to communicate and listen on the port of that network interface.
3. *Listen for connection requests*: We can use the `listen` syscall to put the socket into listen mode, where it listens for connection requests to that particular port on that particular interface. It will process requests in queue, multiple pending connections will be put on a queue so that they could be processed sequentially. If the queue is full, new connection requests will be denied.
4. *Accept the connection request*: We can use the `accept` syscall to accept the first connection request in the connection request queue. Once a request has been accepted, we can communicate with the connected device via that connection.
5. *Communicate with the connection*: We can use the `write` and `read` syscall to exchange data with the connected device. These syscalls will operate on the accepted connection's file descriptor returned from the `accept` syscall.

## Assembly web server specification

Now comes the fun part, actually building the web server. First we'll need to specify what feature this web server will have.

The web server won't be super complicated. We'll build something that can accept multiple connections and handle multiple connections. It should also be able to handle 2 types of HTTP requests: the GET request and the POST request.

For Assembly flavor, we'll be using x86 Assembly with the Intel syntax. Our assembler is the GNU Assembler. Our targeted OS is Linux.

## Assembly socket programming

So using the above concepts, we can use syscalls to create our sockets that will listen for connections and accept HTTP requests.

### Creating the socket object

Firstly, we need to create a socket object with the `socket` syscall:

```asm
# create socket object
mov dil, 2        # AF_INET
mov sil, 1        # SOCK_STREAM
mov dl, 0         # IPPROTO_IP 
mov rax, 41       # sys_socket
syscall
mov r8, rax       # socket fd
```

This calls the socket syscall with 3 arguments and stores the file descriptor for the created socket object in the `r8` register. We're creating a socket object for the IPv4 protocol (`AF_INET`), with the TCP protocol (`SOCK_STREAM`), and for the Internet Protocol (`IPPROTO_IP`).

### Binding the socket object

Next, we bind the socket to a particular network interface using the `bind` syscall. The `bind` syscall takes in 3 arguments: socket file descriptor, socket address structure, address length. The socket file descriptor is taken from the output of the `socket` syscall. The socket address length is the size of the socket address structure, which will be 16 bytes. The socket address structure will look something like this:

```c
struct sockaddr_in {
	sa_family_t sin_family;
	in_port_t sin_port;
	struct in_addr sin_addr;
}
```

Here's what each properties of this structure means:
- `sin_family` will specify which version of IP we'll use (`AF_INET` for our case since we want to use IPv4).
- `sin_port` will specify which port the socket will listen on (`80` for our case, which is the standard HTTP port).
- `sin_addr` will specify which network interface we will listen on (`0.0.0.0` for our case, which means we'll listen on all network interface on the system).

Side note first, on Intel systems, we use the [little endian](https://en.wikipedia.org/wiki/Endianness) system, meaning the least significant byte will go in the lower address. so something like `0x1234` will be stored in memory as `0x34` then `0x12`. However, networking usually uses the big endian system instead, which is the opposite of little endian.

We can create our socket address structure by creating the structure in the `.data` section of our program. Since our structure has to be 16 bytes, we'll need to add some padding bytes to our structure.

```asm
.section .data
sockaddr:
    .word 2                 # AF_INET
    .word 0x5000            # port 80 (0x50 in hexadecimal)
    .double 0x00000000      # 0.0.0.0
    .byte 0,0,0,0,0,0,0,0   # padding bytes
```

With this structure, we can now bind the socket using the `bind` syscall:

```asm
# bind the socket
mov rdi, r8         # socket fd
lea rsi, sockaddr   # sockaddr struct
mov dl, 16          # sockaddr length
mov rax, 49         # sys_bind
syscall
```

### Listening for connection on the socket

Now that we have binded the socket object, we can start listening for connections. This is pretty simple, we just use the `listen` syscall.

The `listen` syscall takes in 2 parameters: the socket file descriptor and the backlog. The backlog determines the max length of the queue for pending connections. Let's just set the backlog to 0, because if `listen` receives 0 as the backlog argument, it will set the queue length to the implementation's minimum value (according to [**this**](https://pubs.opengroup.org/onlinepubs/009695399/functions/listen.html)), which means we won't have to bother with determining the minimum queue length.

```asm
# listen on the binded socket
mov rdi, r8     # get socket fd
mov rsi, 0      # queue length
mov rax, 50     # sys_listen
syscall
```

### Accepting a connection on the socket

Now we can accept connection requests. We can use the `accept` syscall.

The `accept` syscall is used with connection-based socket types like `SOCK_STREAM`. It extracts the first connection request on the queue of pending connections for the listening socket, creates an new connected socket, and returns a file descriptor referring to that new connected socket.

It takes in 3 parameters: the socket file descriptor, the `sockaddr` structure, and the address length. The `sockaddr` structure will contain the address of the peer for the accepted connection (the address of the client). The structure will be filled in by `accept` if we set it to `NULL`. Same goes for the address length, which contains the size of the peer address.

Since we want to accept any connection, we would want to set the structure argument to `NULL` since there's no way we can know what the client's address will be. We'll return the file descriptor to the accepted connection in the register `r9`.

```asm
# accept connections
mov rdi, r8     # get socket fd
mov rsi, 0x0    # NULL
mov rdx, 0x0    # NULL
mov rax, 43     # sys_accept
syscall
mov r9, rax     # accepted socket fd
```

### Communicating with the connection

Now that we've accepted a connection, we can communicate with it by writing to and reading the HTTP request from the accepted connection's file descriptor. We do this via the `write` syscall and the `read` syscall.

We can read the request from the connection using the `read` syscall. The `read` syscall takes in 3 parameters: the file descriptor to read from, the pointer to a buffer to read to, and the count, which is the size of the buffer. It will return the number of bytes read on success.

So we need to create a read buffer in the `.data` section to store the request. We do this by using the `.skip` directive.

```asm
read_buffer:
    .skip 1024
```

Now we can use the `read` syscall to read the request into the buffer through the accepted connection's file descriptor.

```asm
# read the request
mov rdi, r9            # get accepted socket fd
lea rsi, read_buffer   # read buffer
mov rdx, 1024          # buffer size
mov rax, 0             # sys_read
syscall
```

We can write a response to the connection using the `write` syscall. The `write` syscall takes in the same 3 parameters as the `read` syscall. It will write the data from the buffer, with the count parameter specifying how many bytes to write. For now, we'll write a `200 OK` response message. We can specify this response message in the `.rodata` section (Read-only data).

```asm
.section .rodata
response_msg:
    .string "HTTP/1.0 200 OK\r\n\r\n\0"
```

Now we can use the `write` syscall to write the response message to the accepted connection through its file descriptor. This reponse message will make its way to the client.

```asm
# write the response
mov rdi, r9             # get accepted socket fd
lea rsi, response_msg   # write buffer
mov rdx, 19             # buffer size
mov rax, 1              # sys_write
syscall
```

### Closing the connection

After we finished processing the current connection, we need to close the connection and listen to new connections. We do this using the `close` syscall, which takes in 1 argument, the file descriptor to close.

```asm
# close the accepted connection
mov rdi, r9       # get accepted socket fd
mov rax, 3        # sys_close
syscall
```

Then we need to jump back to the `accept` part of the code after closing the previous accepted connection. This would allow us to keep accepting new connections.

```asm
accept_conn:
    # accept connection to socket
    mov rdi, r8       # get socket fd
    mov rsi, 0x0      # NULL
    mov rdx, 0x0      # NULL
    mov rax, 43       # sys_accept
    syscall

    ...

    # close the accepted connection
    mov rdi, r9       # get accepted socket fd
    mov rax, 3        # sys_close
    syscall

    jmp accept_conn
```

## Handling GET requests

Now that we have the socket programmed, we need to create ways to handle the different types of HTTP requests that the client may make.

Let's focus on the GET request first. The GET request indicates resource request. It will specify an endpoint that it would like to get, and the server will process that and return that request. Of course this is really simplified, there's no security mechanism here because our server is very simple.

A simple GET request will look something like this:

```txt
GET /tmp/resource HTTP/1.1
```

This request will request the `/tmp/resource` file. The server will try to get the resource and return to the client. The `HTTP/1.1` specifies the version of HTTP that's being used. If we want to process the GET request, we'll need to extract the requested resource from the request. Since we've read the request into a buffer called `read_buffer`, we can extract information about the request from that buffer.

We need to create 2 new buffers using the `.skip` directive in the `.data` section, 1 for storing the request type, 1 for storing the requested resource.

```asm
filename:
    .skip 1024

file_content:
    .skip 1024
```

Now, we'll extract the request type. This will come in handy later on.

```asm
# get request type
lea rdi, request_type
lea rsi, read_buffer
mov rdx, 0              # start index
call get_substring
mov r10, rax            # end index
```

After that, we'll extract the requested resource.

```asm
# get filename
lea rdi, filename
lea rsi, read_buffer
mov rdx, r10            # start index
call get_substring
```

Both of these processes used a function called `get_substring`. This function will take in a string, an output buffer, along with an index, which specifies the starting location in the string for the function, and it will output the ending index. The function will read a substring starting at the specified starting index and ending when it hits a space, then it would write the substring into the specified output buffer and return the end index.

```asm
get_substring:
    mov r15, 0

substring_loop:
    # read each character from buffer
    cmp byte ptr [rsi+rdx], 32        # found a space
    je end_get_substring

    # copy current character to request type buffer
    mov r14b, byte ptr [rsi+rdx]
    mov [rdi+r15], r14b
    inc rdx
    inc r15
    jmp substring_loop

end_get_substring:
    mov byte ptr [rdi+r15], 0x00      # null terminating character
    inc rdx
    mov rax, rdx                      # return end index
    ret
```

Now we need to program a way to know what request type the current request is using. Since our server only handles 2 types of HTTP requests, we can just check the first character and see if it's a "G" or a "P" and jump to the section that handles that particular request.

```asm
# jump to correct request handling section
mov r15b, [request_type]
cmp r15b, 71     # 'G'
jne POST_request
```

Now that we have the filename, which is the requested resource, we can open up the file, read the content, and send the content back to the client.

```asm
GET_request:
    # open file to read
    lea rdi, filename
    mov rsi, 00000000 # O_RDONLY
    mov rax, 2        # sys_open
    syscall
    mov r10, rax      # opened file fd

    # read the opened file
    mov rdi, r10      # get opened file fd
    lea rsi, file_content
    mov rdx, 1024
    mov rax, 0        # sys_read
    syscall
    mov r15, rax      # get number of bytes read

    # close the opened file
    mov rdi, r10      # get opened file fd
    mov rax, 3        # sys_close
    syscall

    # write response message to socket connection
    mov rdi, r9       # get accepted socket fd
    lea rsi, response_msg
    mov rdx, 19
    mov rax, 1        # sys_write
    syscall

    # write the read file to socket connection
    mov rdi, r9       # get accepted socket fd
    lea rsi, file_content
    mov rdx, r15      # number of bytes in file content
    mov rax, 1        # sys_write
    syscall

    jmp serve_stop

...

serve_stop:
    # close socket connection
    mov rdi, r9       # get accepted socket fd
    mov rax, 3        # sys_close
    syscall

    # exit(0)
    mov rdi, 0
    mov rax, 60       # sys_exit
    syscall
```

## Handling POST requests

The GET request indicates resource creation/update. It will specify an endpoint that it would like to create/update, along with the content that it would like to put into that endpoint, and the server will process that.

A simple POST request will look something like this:

```txt
POST /tmp/resource HTTP/1.1

some_random_text
```

The content of the request will be separated by 2 newline characters. So to handle this request, we'll need to first extract the filename, then extract the content that will be written to the file. We can already get the filename from how we handle the GET request, now we need a way to extract the content of the request.

To do that we'll create a function called `find_string`. This function will take in the buffer to read from, the output buffer to write to, and the delimiter to find. We know that the content of the request will be separated by a double newline, so we can look for that in our request, once we've found that, we can read the text following that double newline, which will be the content.

We'll define the double newline in the `.rodata` section. When we call the `find_string` function, we can pass this double newline in as the 3rd parameter.

```asm
double_newline:
    .string "\r\n\r\n"
```

Now let's implement the `find_string` function. We'll basically go through each character of the request, compare the next 4 character with the double newline, and if they're a match, we'll start copying the substring starting at the index immediately after the double newline into the output buffer until we hit a null-terminating character (`\0`).

```asm
find_string:
    mov r15, 0
    mov r14, 0

next_char:
    cmp byte ptr [rdi+r15], 0x00               # check for null terminator
    je end_find

compare_loop:
    # only compare 4 bytes
    cmp r14, 4
    je found_match

    # compare the next 4 bytes
    mov rax, r14
    add rax, r15
    mov r11b, byte ptr [rdx+r14]               # byte from comparison source
    cmp byte ptr [rdi+rax], r11b
    jne no_match

    # byte matches
    inc r14
    jmp compare_loop

no_match:
    mov r14, 0
    inc r15
    jmp next_char

found_match:
    # r15 now has starting position of content string
    add r15, 4
    mov rax, 0

copy_content:
    # copy bytes to content buffer
    mov rbx, r15
    add rbx, rax
    
    # end of string
    cmp byte ptr [rdi+rbx], 0x00                # null terminator
    je end_find

    # copy bytes
    mov r14b, byte ptr [rdi+rbx]
    mov byte ptr [rsi+rax], r14b
    inc rax

    jmp copy_content

end_find:
    ret
```

Now we can call this `find_string` function during our POST request processing to get the content to write to the specified file.

```asm
POST_request:
    # open file to read
    lea rdi, filename
    mov rsi, 00000101 # O_WRONLY | O_CREAT
    mov rdx, 0777
    mov rax, 2        # sys_open
    syscall
    mov r10, rax      # opened file fd

    # get request content
    lea rdi, read_buffer
    lea rsi, content_buffer
    lea rdx, double_newline
    call find_string
    mov r11, rax      # get length of content

    # write to file with request content
    mov rdi, r10       # get opened file fd
    lea rsi, content_buffer
    mov rdx, r11      # number of bytes in content buffer
    mov rax, 1        # sys_write
    syscall

    # close the opened file
    mov rdi, r10      # get opened file fd
    mov rax, 3        # sys_close
    syscall

    # write response message to socket connection
    mov rdi, r9       # get accepted socket fd
    lea rsi, response_msg
    mov rdx, 19
    mov rax, 1        # sys_write
    syscall

serve_stop:
    ...
```

## Processing multiple requests

At this point, we have server that can process both the GET request and the POST request from a connection. However, we can only process 1 connection at a time. What if there's multiple clients who each wants to connect to our server at the same time? We'd have to implement multi-processing in order to handle multiple request at once.

In order to implement multi-processing, we'll need to use a syscall called `fork`. This syscall allows us to duplicate the current process, resulting in 2 of the same process. When `fork` is called, it creates 2 processes that are the same as each other and it will return a different value depending on the process. It will return the ID of the child process to the original parent process, and it will return 0 to the child process. This fact is really important as it allows us to determine if the current process is a child process or a parent process, which helps us determine which part of the code to execute. Usually the `exec` syscall is used with `fork` but for our purposes, which are pretty simple, we don't necessarily need to use `exec`.

{{< image src="/img/asm-webserver/fork-syscall-visual.png" alt="Socket flow diagram" position="center" style="padding: 20px" >}}

Another thing about duplicate processes is that they are independent of each other. If we were to say close a file descriptor on the parent process, that file descriptor would still be open on the child process. If the child process want to also close that file descriptor, it would have to do it by itself.

For our web server, the parent process will parent process will be responsible for accepting the socket connection, but not for serving and talking to the accepted connection. The child process will be responsible for serving and talking to the accepted connection, but not for accepting new socket connection.

When the parent process accepts a new connection, it will fork itself to create a child process to handle that connection, then close that accepted connection (since the child process still have access to that accepted connection to talk to), then listen for new connection. If there's another new connection, it does the same thing again, creating another different child process to handle that new connection.

The child process will close the listening socket file descriptor (since it doesn't have to listen for new connections) and process the current accepted connection by communicating through the accepted connection's file descriptor.

To determine whether a process is a child process, we can just compare the `fork` syscall's return value with 0 and jump to the serve connection section of the code. If it's not 0 then we just close the accepted connection and jump back to listening for a new connection.

```asm
accept_conn:
    ...
    mov rax, 57       # sys_fork
    syscall

    # serve the accepted socket connection if is child process
    cmp rax, 0        # child process always returns 0 on fork call
    je serve_conn

    # close the accepted socket connection if is parent process
    mov rdi, r9       # get accepted socket fd
    mov rax, 3        # sys_close
    syscall

    jmp accept_conn

serve_conn:
    # close the socket fd
    mov rdi, r8       # get binded socket fd
    mov rax, 3        # sys_close
    syscall
    ...
```

The `serve_conn` section will contain the GET request processing and the POST request processing code. After the child process is finished with processing the request, it will hit the `serve_stop` section, which will clean up the process, making sure the file descriptors are closed, and exit the child process as we have no more use for it.

```asm
serve_stop:
    # close socket connection
    mov rdi, r9       # get accepted socket fd
    mov rax, 3        # sys_close
    syscall

    # exit(0)
    mov rdi, 0
    mov rax, 60       # sys_exit
    syscall
```

## Conclusion

That's a basic web server written entirely in x86 Assembly in less than 300 lines of code. If you want to read the full code, you can check it out [**here**](https://gist.github.com/namberino/0cd2dfa288cc63f18a951d8620c1b17f). I definitely had a lot of fun working on this and got to learn a lot more about Assembly and systems programming.

This project was done as part of the "*Building a Web Server*" course by [pwn.college](https://pwn.college).

## References

- [Linux kernel source code](https://www.kernel.org/)
- [x86-64 calling conventions](https://en.wikipedia.org/wiki/X86_calling_conventions#x86-64_calling_conventions)
- [Linux syscall table](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)
- [man7.org](https://man7.org)
- [x86 registers list](https://www.tortall.net/projects/yasm/manual/html/arch-x86-registers.html)
- [GNU Assembler Manual](http://microelectronics.esa.int/erc32/doc/as.pdf)
- [MDN Web Docs](https://developer.mozilla.org/en-US/)
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/)
