---
title: "Phá mã phần mềm điều khiển của jack cắm kết nối WiFi"
date: 2024-04-26T12:02:12+07:00
toc: true
tags:
  - bảo mật
  - hack
  - dịch ngược
  - lập trình
author: "Nguyễn Bình Nam"
description: "Phá mã phần mềm điều khiển của thiết bị NPort W2150A"
---

Mình đã học dịch ngược, hệ thống nhúng và phần mềm điều khiển hệ thống nhúng khá lâu rồi. Dạo gần đây mình lại nghĩ "Sao mình không kết hợp 2 kĩ năng này để làm 1 cái gì đấy hay?". Thế nên mình thử sức bản thân bằng cách phá mã của 1 phần mềm điều khiển đã được mã hóa của 1 jack cắm kết nối WiFi. Mình đã ghi lại quá trình phá mã và cách suy nghĩ của mình trong bài blog này.

## Về thiết bị

Gần đây mình cũng đọc được 1 lỗ hổng trong thiết bị jack cắm [*Moxa NPort W2150A Serial-To-Wifi*](https://www.moxa.com/en/products/industrial-edge-connectivity/serial-device-servers/wireless-device-servers/nport-w2150a-w2250a-series), lỗ hổng này sử dụng buffer overflow dạng stack. Xong mình nghĩ là mình cũng muốn thử làm 1 dự án liên quan đến bên bảo mật cho thiết bị này. Bởi vì thiết bị này có phần mềm đã được mã hóa, mình quyết định sẽ cố phá mã để có thể lấy được mã nguồn của phần mềm điều khiển.

Mình muốn bắt đầu với 1 phiên bản cũ hơn của phần mềm điều khiển của thiết bị này. Mình tìm được phiên bản [*v2.2*](https://www.moxa.com/Moxa/media/PDIM/S100000210/moxa-nport-w2150a-w2250a-series-firmware-v2.2.rom), được xuất bản vào khoảng 2019. MÌnh sẽ sử dụng phiên bản này.

## Phân tích phần mềm

Đầu tiên mình muốn thử trích xuất tất cả thông tin về phần mềm điều khiển này trước khi mình dịch ngược.

Mình thử sử dụng câu lệnh `file` để xem nó có thể phát hiện được phần mềm `.rom`  này là phần mềm loại gì không:

```bash
$ file moxa-nport-w2150a-w2250a-series-firmware-v2.2.rom
moxa-nport-w2150a-w2250a-series-firmware-v2.2.rom: data
```

`data` có nghĩa là câu lệnh `file` không thể phát hiện dấu hiệu của bất kì định dạng file nào phần mềm `.rom` này. Điều này có nghĩa là phần mềm này đã được mã hóa khá kĩ càng.

Mình thử chạy câu lệnh `hexdump` để xem cấu trúc của phần mềm này:
```bash
hexdump -C moxa-nport-w2150a-w2250a-series-firmware-v2.2.rom | head
```

{{< image src="/img/nport-firmware/nport-firmware-hexdump.png" alt="hexdump phần mềm điều khiển" position="center" style="padding: 10px" >}}

Ở đầu phần mềm này là `NPW2X50A8k`. Đây là tên của thiết bị. Sau đó là 1 vài byte null dùng để lấp chỗ trống. Sau đó là các byte nhìn rất random. Mình cũng không trích xuất được nhiều thông tin từ đây.

Since we can't really find anything using `file` and `hexdump`, let's use a more powerful tool. I'll use `binwalk` as that tool allows me to walk through the entire binary and find file signatures and compression methods. The tool also provides extensive binary analysis tools. Bởi vì mình không tìm được nhiều thông tin qua các câu lệnh `file` và `hexdump`, mình sẽ sử dụng công cụ mạnh hơn. Mình dùng `binwalk`, công cụ này cho phép mình "bước" qua cả file nhị phân và tìm các định dạng file và các định dạng nén trong phần mềm. Câu lệnh này cũng có nhiều công cụ phân tích nhị phân khác nhau.

```bash
binwalk moxa-nport-w2150a-w2250a-series-firmware-v2.2.rom
```

Khi mình chạy câu lệnh `binwalk` trên, mình được kết quả là 1 file `MySQL`. Đây là 1 báo động giả bởi vì mình không nghĩ là 1 jack cắm kết nối WiFi sẽ cần sử dụng database. Thế mình cũng không có thông tin gì hữu ích.

Sau đó thì mình bắt đầu tìm các phiên bản cũ hơn để xem mình có thể trích xuất thông tin gì từ đó. Trong khi mình đang đọc [note xuất bản](https://www.moxa.com/Moxa/media/PDIM/S100000210/W2250A%20Series_moxa-nport-w2150a-w2250a-series-firmware-1.11.rom_Software%20Release%20History.pdf) của phiên bản *1.11*, mình tìm thấy 1 phần khá thú vị:

{{< image src="/img/nport-firmware/nport-firmware-version11-release-note.png" alt="Note xuất bản phiên bản 1.11" position="center" style="padding: 10px" >}}

Phiên bản *1.11* là phiên bản tiên quyết cho phiên bản *2.2*. Tức là mình cần phiên bản *1.11* để có thể tải phiên bản *2.2*. Mình nghĩ là phần mã hóa cho phần mềm điều khiển được thêm vào trong phiên bản *2.2*. Thế nên mình đã tải phiên bản [v1.11](https://www.moxa.com/Moxa/media/PDIM/S100000210/moxa-nport-w2150a-w2250a-series-firmware-1.11.rom) và bắt đầu phân tích phiên bản này.

Đầu tiên mình thử `binwalk` phiên bản này:
```bash
binwalk moxa-nport-w2150a-w2250a-series-firmware-1.11.rom
```

{{< image src="/img/nport-firmware/nport-firmware-older-version-binwalk.png" alt="NPort firmware phiên bản 1.11 binwalk" position="center" style="padding: 10px" >}}

Output này xác nhận là phiên bản *1.11* không bị mã hóa. Trong output này có 2 điều khá thú vị: 2 hệ thống file `squashfs` đã được nén bằng `gzip`. `squashfs` là cả 1 hệ thống file của Linux.

Mình sẽ thử giải nén phần mềm này:
```bash
binwalk -e moxa-nport-w2150a-w2250a-series-firmware-1.11.rom
```

Câu lệnh này sẽ giải nén phiên bản *1.11* vào tệp `_moxa-nport-w2150a-w2250a-series-firmware-1.11.rom.extracted`:

{{< image src="/img/nport-firmware/nport-firmware-extracted-screenshot.png" alt="NPort firmware phiên bản 1.11 giải nén" position="center" style="padding: 10px" >}}

Trong folder đó có các folder con `squashfs-root`, các folder này có hệ thống file Linux của phần mềm này. Trước khi mình truy cập folder này thì mình cần phân quyền đúng cho folder đó:
```bash
chmod +x -R squashfs-root*
```

Giờ mình có thể truy cập các folder `squashfs-root`:

{{< image src="/img/nport-firmware/nport-firmware-old-version-filesystem.png" alt="NPort firmware phiên bản 1.11 hệ thống file đã được giải nén" position="center" style="padding: 10px" >}}

Sau khi nhìn qua các folder trong phần mềm này, mình tìm được 1 file có tên là `libupgradeFirmware.so` trong folder `lib` của folder `squashfs-root`. Bởi vì phiên bản *2.2* cần có phiên bản *1.11*, mình đoán là file `libupgradeFirmware.so` sẽ có thông tin về các phần mềm này đã được mã hóa. Mình sẽ phân tích và dịch ngược file nhị phân này:

## Dịch ngược file libupgradeFirmware.so

Mình sẽ dùng [`Ghidra`](https://ghidra-sre.org/) để dịch ngược.

Đầu tiên mình sẽ xem các hàm có trong file nhị phân này:

{{< image src="/img/nport-firmware/nport-firmware-ghidra-function-window.png" alt="NPort firmware Ghidra các hàm" position="center" style="padding: 10px" >}}

File này có sử dụng thuật toán mã hóa khối **AES** ở chế độ **ECB** (Electronic Code Block). Bởi vì chế độ **ECB** tạo text mã hóa (ciphertext) giống nhau với text thường (plaintext) giống nhau, hacker có thể suy ra khóa bí mật và phá mã dữ liệu đã được mã hóa bằng chế độ **ECB**. Đây là 1 lỗ hổng lớn mà mình có thể tấn công.

{{< image src="/img/nport-firmware/nport-firmware-ghidra-fw_decrypt.png" alt="NPort firmware Ghidra các hàm, hàm fw_decrypt" position="center" style="padding: 10px" >}}

Firmware này cũng có hàm `fw_decrypt`. Đây chắc là 1 hàm dùng để giải mã firmware, hàm này chắc là 1 hàm khá quan trọng. 

Sau khi đọc qua code của `fw_decrypt` 1 lượt thì mình thấy là `fw_decrypt` có gọi 1 hàm tên là `ecb128Decrypt`. Đây chắc là hàm giải mã AES 128 ở chế độ ECB. Hàm này trực tiếp gọi các hàm AES trong thư viện *OpenSSL*. Mình có thể dùng công cụ của *OpenSSL* để giải mã phần mềm firmware này. Nhưng mình sẽ cần khóa dùng cho việc mã hóa phần mềm này để có thể giải mã nó.

## Dịch ngược hàm ecb128Decrypt

Mình sẽ bắt đầu dịch ngược hàm `ecb128Decrypt` này:

{{< image src="/img/nport-firmware/nport-firmware-ecb128decrypt-function-reversed.png" alt="NPort firmware ecb128Decrypt hàm dịch ngược" position="center" style="padding: 10px" >}}

Trong lúc phân tích mình sẽ đặt tên lại và đặt kiểu dữ liệu lại cho các biến trong phần mềm firmware này.

Mình sẽ bắt đầu với hàm AES. Hàm `AES_set_decrypt_key` sẽ có input là khóa người dùng và mở rộng nó ra thành 1 khóa AES. Hàm `AES_set_decrypt_key` này sẽ sử dụng biến `auStack_30`. Trong [tài liệu chính thức của OpenSSL](https://www.openssl.org/docs/), input đầu tiên của hàm này là khóa người dùng. Thế nên mình có thể đặt tên`auStack_30` thành `user_key`. `AStack_124` là khóa AES thế nên mình sẽ đặt tên nó thành `aes_key`. Khóa AES này sẽ được sử dụng cho việc giải mã phần mềm.

Hàm này cũng cần lấy 1 tham số là kích cỡ khóa. Trong chương trình này thì kích cỡ khóa là *0x80* (là *128* trong số nguyên). Tức là firmware này sử dụng khóa AES với kích cỡ khóa là *128-bit*. Mình sẽ đổi kiểu dữ liệu của *0x80* thành số nguyên.

Tiếp theo, trên hàn `strncpy`, dòng này đang copy 16 byte từ `param_4` sang `auStack_30` (khóa người dùng `user_key`). Tức là `param_4` là 1 khóa giải mã bởi vì `user_key` sẽ được sử dụng trong hàm giải mã AES. Mình sẽ đặt tên cho biến `param_4` thành `decrypt_key`.

Tiếp theo, ở dòng có biến `in` and `out`, các biến này có chứa dữ liệu của `param_1` và `param_2` và các dữ liệu này được cộng với sai số là *0x10* (*16* trong số nguyên). 2 biến này cũng được cho vào hàm `AES_ecb_encrypt`. 

Hàm `AES_ecb_encrypt` sẽ cần 1 buffer đầu vào, 1 buffer đầu ra, 1 khóa AES và chế độ mã hóa. Bởi vì hàm `AES_ecb_encrypt` có chế độ mã hóa 0 ở trường hợp này, hàm `AES_ecb_encrypt` sẽ được đặt vào chế độ giải mã. Thế hàm này sẽ giải mã dữ liệu trong buffer đầu vào dùng khóa AES và cho dữ liệu đã được giải mã vào buffer đầu ra.

Mình suy ra là biến `param_1` và `param_2` là buffer đầu vào và buffer đầu ra. Mình sẽ đặt tên `param_1` thành `decrypt_in` và `param_2` thành `decrypt_out` và đổi kiễu dữ liệu thành `uchar*` bởi vì biến `in` và `out` đều là `uchar*`.

Tiếp theo, biến `iVar1` biến đếm được dùng cho vòng lặp, vòng lặp này chỉ dừng khi biến đếm bằng `param_3 + -0x28`. Thế nên `param_3` sẽ là kích cỡ của buffer đầu vào. Mình sẽ đặt tên `param_3` thành `decrypt_size`. `decrypt_size` cần được cộng với sai số là `-0x28`, điều này có nghĩa là file này sẽ có 1 vài byte đệm ở đầu file. Lúc mình dùng `hexdump`, mình tìm được 1 vài byte `00` ở đầu file, đó chắc là lý do tại sao `decrypt_size` cần được cộng với sai số *-0x28* (40 byte)

Đây là hàm đã được đặt tên lại:

{{< image src="/img/nport-firmware/nport-firmware-ecb128decrypt-reversed-renamed.png" alt="NPort firmware ecb128Decrypt hàm đã được đặt tên lại" position="center" style="padding: 10px" >}}

Thế là mình đã hiểu được cách hàm `ecb128Decrypt` hoạt động: Nó lấy 1 buffer đầu vào (`decrypt_in`), giải mã nó với khóa (`decrypt_key`), và cho kết quả vào đầu ra (`decrypt_out`).

## Dịch ngược hàm fw_decrypt

Tiếp theo, mình sẽ phân tích hàm `fw_decrypt` và dịch ngược nó lại.

> **Note**: Code của hàm`fw_decrypt` khá là dài thế nên mình sẽ copy nó vào 1 block code thay vì chụp ảnh

```c
undefined8 fw_decrypt(void *param_1,uint *param_2,undefined4 param_3)
{
  undefined4 uVar1;
  uint *puVar2;
  byte *pbVar3;
  uint decrypt_size;
  uint uVar4;
  void *__src;
  uint *local_24;
  undefined4 uStack_20;
  
  decrypt_size = *param_2;
  if (param_1 == (void *)0x0) {
    uVar1 = 0xffffffff;
  }
  else if (*(char *)((int)param_1 + 0xe) == '\x01') {
    if ((((decrypt_size < 0x29) || (decrypt_size < (*(byte *)((int)param_1 + 0xd) + 10) * 4)) ||
        (decrypt_size < *(uint *)((int)param_1 + 8))) || ((decrypt_size - 0x28 & 0xf) != 0)) {
      uVar1 = 0xfffffffe;
    }
    else {
      pbVar3 = &passwd.3309;
      while (pbVar3 + 4 != ubuf) {
        *pbVar3 = *pbVar3 ^ 0xa7;
        pbVar3[1] = pbVar3[1] ^ 0x8b;
        pbVar3[2] = pbVar3[2] ^ 0x2d;
        pbVar3[3] = pbVar3[3] ^ 5;
        pbVar3 = pbVar3 + 4;
      }
      local_24 = param_2;
      uStack_20 = param_3;
      ecb128Decrypt((uchar *)param_1,(uchar *)param_1,decrypt_size,&passwd.3309);
      uVar4 = *(uint *)((int)param_1 + 8);
      if (((0x28 < uVar4) && ((*(byte *)((int)param_1 + 0xd) + 10) * 4 < uVar4)) &&
         (*(char *)((int)param_1 + 0xe) == '\0')) {
        __src = (void *)((int)param_1 + (uint)*(byte *)((int)param_1 + 0xd) * 4 + 0x24);
        memcpy(&local_24,__src,4);
        puVar2 = (uint *)cal_crc32((int)__src + 4,uVar4 + (*(byte *)((int)param_1 + 0xd) + 10) * -4,
                                   0);
        if (puVar2 == local_24) {
          if ((int)decrypt_size < (int)uVar4) {
            uVar1 = 0xfffffffb;
          }
          else {
            *param_2 = uVar4;
            uVar1 = 0;
          }
          goto LAB_0001191c;
        }
      }
      uVar1 = 0xfffffffc;
    }
  }
  else {
    uVar1 = 0;
  }
LAB_0001191c:
  return CONCAT44(param_1,uVar1);
}
```

Ở dòng này:

```c
ecb128Decrypt((uchar *)param_1,(uchar *)param_1,decrypt_size,&passwd.3309);
```

Bởi vì mình biết cách hàm `ecb128Decrypt` hoạt động, mình có thể thấy là tham số `decrypt_in` và `decrypt_out` có cùng 1 biến: `param_1`. Điều này có nghĩa là biến `param_1` được giải mã vào chính nó. Mình sẽ đặt tên `param_1` thành `fw_buffer` và đặt lại kiểu dữ liệu thành `uchar*`.

Hàm `ecb128Decrypt` cũng lấy 1 tham số gọi là `decrypt_size`, tham số này có dữ liệu từ `param_2` (ở dòng `decrypt_size = *param_2;`). Mình sẽ đặt tên của `param_2` thành `fw_buffer_size`. 

```c
ecb128Decrypt(fw_buffer,fw_buffer,decrypt_size,&passwd.3309);
```

Ở dòng điều kiện "*if*", nó sẽ check xem `param_1` (`fw_buffer`) có giá trị *null* hay không. Nếu nó có giá trị là *null* thì biến `uVar2` sẽ được gán giá trị *0xffffffff*. Biến `uVar2` là biến đầu ra của hàm `fw_decrypt` thế nên mình sẽ đặt tên nó thành `return_value`. 

Đặt `return_value` thành *0xffffffff* sẽ overflow biến này `return_value` thành 1 giá trị âm. Mình có thể check xem số này là số nào bằng cách đổi kiểu dữ liệu của `return_value` từ `uint` sang `int`:

```c
if (fw_buffer == (uchar *)0x0) {
    return_value = -1;
}
```

Nó sẽ đặt `return_value` thành *-1*, có nghĩa là fail trong C. Mình có thể suy ra kiểu dữ liệu của hàm `fw_decrypt` từ biến `uVar2`.

```c
int fw_decrypt(uchar *fw_buffer,uint *fw_buffer_size,undefined4 param_3)
```

Hàm này đã trở nên dễ đọc hơn nhiều

Ở trong câu lệnh *if* ở trong câu lệnh *else if*, nó sẽ check lỗi và kích cỡ hàm không hợp lệ và trả về giá trị âm nếu không hợp lệ. Câu lệnh *else* sau đó nhìn nhìn khá thú vị, mình sẽ xem nó làm những gì:

```c
else {
  pbVar3 = &passwd.3309;
  while (pbVar3 + 4 != ubuf) {
    *pbVar3 = *pbVar3 ^ 0xa7;
    pbVar3[1] = pbVar3[1] ^ 0x8b;
    pbVar3[2] = pbVar3[2] ^ 0x2d;
    pbVar3[3] = pbVar3[3] ^ 5;
    pbVar3 = pbVar3 + 4;
  }
  local_24 = fw_buffer_size;
  uStack_20 = param_3;
  ecb128Decrypt(fw_buffer,fw_buffer,decrypt_size,&passwd.3309);
  uVar4 = *(uint *)(fw_buffer + 8);
  if (((0x28 < uVar4) && (bVar1 = fw_buffer[0xd], (bVar1 + 10) * 4 < uVar4)) &&
     (fw_buffer[0xe] == '\0')) {
    memcpy(&local_24,fw_buffer + (uint)bVar1 * 4 + 0x24,4);
    puVar2 = (uint *)cal_crc32((int)(fw_buffer + (uint)bVar1 * 4 + 0x24 + 4),
                               uVar4 + (fw_buffer[0xd] + 10) * -4,0);
    if (puVar2 == local_24) {
      if ((int)uVar4 <= (int)decrypt_size) {
        *fw_buffer_size = uVar4;
        return 0;
      }
      return -5;
    }
  }
  return_value = -4;
}
```

Mình sẽ đi qua từng đoạn và phân tích nó:

```c
pbVar3 = &passwd.3309;
```

`passwd.3309` nhìn giống như 1 biến chứa mật khẩu. Biến này cũng được dùng cho tham số `decrypt_key` của hàm `ecb128Decrypt`. Biến `pbVar3` sẽ giữ giá trị của `passwd.3309` thế nên mình sẽ đặt tên `pbVar3` thành `password`.

Tiếp theo là vòng lặp *while*, biến `password` sẽ được biến đổi qua 1 vài thao tác *XOR*:

```c
while (pbVar3 + 4 != ubuf) {
  *pbVar3 = *pbVar3 ^ 0xa7;
  pbVar3[1] = pbVar3[1] ^ 0x8b;
  pbVar3[2] = pbVar3[2] ^ 0x2d;
  pbVar3[3] = pbVar3[3] ^ 5;
  pbVar3 = pbVar3 + 4;
}
```

Đây có thể là 1 phương thức "làm mờ" hoặc mã hóa. Mình sẽ thử mô phỏng vòng lặp này bằng cách viết lại vòng lặp này trong Python.

Đầu tiên thì mình sẽ cần lấy dữ liệu mà `passwd.3309` đang trỏ đến, mình có thể làm thế bằng cách nhìn trong cửa số **Bytes** của Ghidra:

{{< image src="/img/nport-firmware/nport-firmware-fw_decrypt-passwd-bytes.png" alt="NPort firmware fw_decrypt passwd3309 byte" position="center" style="padding: 10px" >}}

Mình sẽ copy các byte được bôi đen vào 1 mảng trong Python:

```py
passwd = [0x95, 0xb3, 0x15, 0x32, 0xe4, 0xe4, 0x43, 0x6b, 0x90, 0xbe, 0x1b, 0x31, 0xa7, 0x8b, 0x2d, 0x05]
```

Mình sẽ làm lại vòng lặp *while* với các thao tác *XOR* như trong chương trình dịch ngược:

```py
i = 0
while (i < len(passwd)):
	passwd[i] ^= 0xa7
	passwd[i + 1] ^= 0x8b
	passwd[i + 2] ^= 0x2d
	passwd[i + 3] ^= 5
	
	i += 4
```

Sau đó mình có thể in ra mật khẩu:

```py
print("".join(chr(byte) for byte in passwd))
```

Đây là chương trình Python full:

```py
passwd = [0x95, 0xb3, 0x15, 0x32, 0xe4, 0xe4, 0x43, 0x6b, 0x90, 0xbe, 0x1b, 0x31, 0xa7, 0x8b, 0x2d, 0x05]

i = 0
while (i < len(passwd)):
	passwd[i] ^= 0xa7
	passwd[i + 1] ^= 0x8b
	passwd[i + 2] ^= 0x2d
	passwd[i + 3] ^= 5
	
	i += 4

print("".join(chr(byte) for byte in passwd))
```

Khi mình chạy chương trình Python này, nó sẽ in ra cái này:

{{< image src="/img/nport-firmware/nport-firmware-fw_decrypt-python-output.png" alt="NPort firmware fw_decrypt vòng lặp while trong python" position="center" style="padding: 10px" >}}

Thế là mình được khóa giải mã AES cho chương trình này là "*2887Conn7564*". Mình có thể sử dụng khóa này để giải mã firmware. Trước hết thì mình cần chuyển khóa này thành mã hex:

```py
print("".join(hex(byte)[2:] for byte in passwd))
```

Dòng này sẽ cho ra kết quả là *32383837436f6e6e373536340000*

## Giải mã phần mềm

Làm thế nào để mình có thể giải mã firmware này với khóa mình vừa lấy được?

Mình có thể dùng *OpenSSL*. Câu lệnh `openssl` có hỗ trợ mã hóa AES 128-bit ở chế độ ECB thế nên mình sẽ sử dụng nó.

Trước khi mình giải mã, khi mà mình dịch ngược hàm `ecb128Decrypt` và dùng câu lệnh `hexdump`, mình biết được là firmware đã bị mã hóa này có khoảng *0x28* byte đệm (40 byte trong số nguyên) 

Thế nên mình phải loại bỏ các byte đệm đó, nếu không thì nó sẽ cố giải mã các byte đệm đó, tạo ra dữ liệu xấu. Mình sẽ dùng câu lệnh `dd` cho việc này:

```bash
$ dd if=moxa-nport-w2150a-w2250a-series-firmware-v2.2.rom of=firmware-offseted.encrypted bs=1 skip=40
8874768+0 records in
8874768+0 records out
8874768 bytes transferred in 54.281718 secs (163495 bytes/sec)
```

> **Note**: `bs` có nghĩa là kích cỡ khối, `skip` có nghĩa là số byte để skip

Bây giờ mình có thể dùng `openssl` để giải mã file `firmware-offseted.encrypted`:

```bash
openssl aes-128-ecb -d -K "32383837436f6e6e373536340000" -in firmware-offseted.encrypted -out firmware.decrypted
```

Câu lệnh này sẽ cho ra file `firmware.decrypted`. Nếu như mình chạy câu lệnh `binwalk` lên file đã được giải mã này thì nó sẽ ra:

{{< image src="/img/nport-firmware/nport-firmware-firmware-decrypted-openssl.png" alt="NPort firmware đã được giải mã binwalk" position="center" style="padding: 10px" >}}

Mình sẽ giải nén file này ra `_firmware.decrypted.extracted`:

```bash
binwalk -e firmware.decrypted
cd _firmware.decrypted.extracted
```

Cho các folder `squashfs-root` quyền thực thi:

```bash
chmod +x -R squashfs-root*
```

Và thế là mình có quyền truy cập vào firmware phiên bản *2.2* của thiết bị này:

{{< image src="/img/nport-firmware/nport-firmware-firmware-decrypted-filesystem.png" alt="NPort firmware hệ thống file đã được giải mã" position="center" style="padding: 10px" >}}

## Kết luận

Đó là cách mình dịch ngược và giải mã 1 firmware đã được mã hóa. Mình học được nhiều về cách phân tích firmware để tìm ra lỗ hổng và cách tấn công các lỗ hổng đó.

> Các file Ghidra và Python được host ở [đây](https://github.com/namberino/NPort-firmware-decrypt)
