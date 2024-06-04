---
title: "Lập trình hệ điều hành: Khởi đầu"
date: 2024-06-04T22:12:15+07:00
toc: true
tags:
  - lập trình
  - hệ điều hành
  - hệ thống cấp thấp
author: "Nguyễn Bình Nam"
description: "Học về hệ điều hành bằng cách lập trình 1 cái từ đầu đến cuối"
---

Mình vừa học xong 1 môn về hệ điều hành vài kì học trước nhưng môn đó có rất nhiều lý thuyết và gần như không thực hành về hệ điều hành. Sau khi học xong môn đó thì mình không cảm thấy mình thực sử hiểu về cách hoạt động của hệ điều hành. Thế nên mình quyết định là sẽ tự tạo ra 1 hệ điều hành của riêng mình để có thể lấy kinh nghiệm thực tế về hệ điều hành và tăng vốn hiểu biết của mình về hệ điều hành.

Mình sẽ chia sẻ chi tiết về dự án này trên blog này. Mình sẽ cố gắng hết khả năng của mình để giải thích mọi thứ 1 cách chi tiết để những ai mà muốn đi theo con đường này có thể học hỏi từ đây và xây dựng nền tảng nhanh hơn.

Bài đăng này sẽ đi vào cách thiết lập 1 môi trường lập trình với các công cụ cần thiết đề làm cái hệ điều hành này.

## Chọn cấu trúc vi xử lý

Một hệ điều hành cần được lập trình để chạy trên 1 cấu trúc vi xử lý nhất định. Bạn có thể thấy là các hệ điều hành hiện đại đều được lập trình để có thể chạy trên được nhiều cấu trúc vi xử lý khác nhau. Ví dụ như Linux, bạn có thể mình trong [mã nguồn](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch?h=v6.10-rc2) của nó và thấy được tất cả các cấu trúc khác nhau mà nó hỗ trợ.

Chúng ta sẽ không thể lập trình 1 hệ điều hành mà có thể chạy trên được nhiều cấu trúc vi xử lý khác nhau bởi vì việc đó khá phức tạp. Thế nên mình sẽ chọn **một** cấu trúc vi xử lý và xây dựng hệ điều hành cho cấu trúc đó.

Mình đã chọn cấu trúc *x86* bởi vì cấu trúc này rất nổi tiếng và có rất nhiều tài liệu và hướng dẫn cho lập trình trên cấu trúc *x86* này. Điều này sẽ làm cho việc lập trình cho cấu trúc này dễ hơn. Mình sẽ lập trình cho cụ thể là gia đình vi xử lý *i386*. Gia đình vi xử lý này cũng khá cũ, chúng có nhiều tài liệu và chúng cũng không quá phức tạp.

## Công cụ

Giờ thì chúng ta sẽ bắt đầu cài công cụ và set up môi trường làm việc. Đầu tiên thì chúng ta se dùng một hệ điều hành dựa trên *Unix* bởi vì thường thì chúng sẽ có các công cụ chúng ta sẽ dùng hoặc chúng sẽ có cách để cài các công cụ đó 1 cách dễ dàng. Mình sẽ dùng Mac nhưng mà mình nghĩ Linux sẽ tốt hơn cho việc này bởi vì thường thì nó có các công cụ này sẵn rồi. 

Chúng ta sẽ cài 2 công cụ: [*QEMU*](https://www.qemu.org/) và [*NASM*](https://nasm.us/)

- *QEMU* là 1 chương trình giả lập. Nó cho phép chúng ta giả lập vi xử lý mà chúng ta đã chọn và chạy chương trình được lập trình cho cấu trúc đó. Bởi vì *i386* không được sản xuất nữa, chúng ta sẽ dùng giả lập này để chạy hệ điều hành.
- *NASM* là 1 chương trình dịch assembly. Nó cho phép chúng ta dịch chương trình assembly xuống file nhị phân có thể chạy được trên cấu trúc *i386*. File nhị phân này sẽ được upload lên *QEMU* để chạy.

## Set up môi trường

Với Mac, chúng ta sẽ cần [cài Homebrew](https://brew.sh/) và chạy câu lệnh này:

```sh
brew install qemu nasm
```

Với Linux (Họ Debian), chạy câu lệnh này nếu như bạn chưa có QEMU hay NASM:

```sh
sudo apt install qemu nasm
```

Nếu như bạn không dùng được công cụ thì bạn sẽ cần update cái biến PATH trong file `.bashrc` (hoặc `.zshrc` nếu như bạn dùng zsh) với đường dẫn tới các công cụ này. Đây là 1 ví dụ cho cách update biến PATH:

```sh
# trong .zshrc
export PATH="/usr/local/bin/nasm"
```

Với Windows thì mình cũng không rõ cách set up thế nào bởi vì mình không có máy nào chạy Windows. Nhưng mình nghĩ cài QEMU và NASM trên Windows vẫn được, bạn chỉ cần đảm bảo là bạn có thể dùng được các công cụ đó trên terminal. 

## Kết luận

Đó là cách set up môi trường làm việc cho dự án hệ điều hành này. Nếu như mình dùng thêm các công cụ khác thì mình sẽ nói về cách set up các công cụ đó sau. Hiện tại thì đây là các công cụ cần thiết đề làm dự án này.

Trong [bài đăng tiếp theo](./os-dev-2.vi.md), chúng ta sẽ học về boot sector và cách làm ra 1 cái boot sector.
