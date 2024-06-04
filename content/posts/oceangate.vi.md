---
title: "Áp lực nước của tàu OceanGate Titan"
date: 2024-02-19T12:27:06+07:00
toc: false
tags:
  - vật lý
  - toán
author: "Nam Nguyễn"
description: "Con tàu ngầm OceanGate Titan chịu áp lực mạnh đến mức nào?"
math: true
---

Mình vừa mới tìm hiểu về các thảm họa kĩ thuật kinh khủng nhất trong lịch sử. Thảm họa kĩ thuật mà mình thấy tò mò nhất là *thảm họa tàu ngầm OceanGate Titan*. Mình tự hỏi là con tàu đó chịu áp lực cao đến đâu để mà bị nổ tung như vậy. Thế nên mình đã lôi máy tính ra và bắt đầu tính toán.

Điều nguy hiểm nhất về biển có lẽ là áp suất thủy tĩnh. Áp suất đó tăng với độ sâu từ bề mặt nước đo trở xuống. Càng sâu, áp lực mà biển áp đặt được sẽ tăng dần lên.

Áp suất thủy tĩnh có thể được tính toán bằng công thức sau (giả sử áp lực đc đo trên 1 khối chất lỏng có diện tích bề mặt là *$1m^2$*):

$$
P = \rho g h
$$

*$P$*: Áp suất của chất lỏng ($N/m^2$)

*$\rho$*: Khối lượng riêng của chất lỏng ($kg/m^3$)

*$g$*: Gia tốc do trọng lực ($9.8 m/s^2$)

*$h$*: Độ sâu của khối chất lỏng

{{< image src="/img/oceangate/pressure-vi.png" alt="pressure diagram" position="center" style="padding: 0.1rem" >}}

Mình tính áp suất trên bề mặt trước rồi nhân nó với độ sâu của tàu OceanGate Titan.

Có 1 vài biến đã được biết sẵn:
- Áp suất (*$P$*) trên bề mặt là $1atm$ = $100,000 N/m^2$
- Trọng lượng riêng của nước là $1,000 kg/m^3$
- *$g$* là $9.8 m/s^2$ nhưng mà mình làm tròn nó lên $10 m/s^2$

Từ đó mình tính ra:

$$
\frac{100,000 N/m^2}{1,000 kg/m^3 \cdot 10 m/s^2} = 10m
$$

Vậy cứ đi sâu $10m$ thì áp suất nước sẽ tăng $1atm$. Và bởi vì tàu OceanGate Titan ở khoảng $4,000m$ khi mà nó nổ, mình tính ra áp suất là:

$$
\frac{4,000m}{10m} = 400atm
$$

Vậy tàu OceanGate Titan phải chịu khoảng áp suất $400atm$. Gấp 400 lần áp suất mà con người thường chịu đựng.

Con số $400atm$ chắc cũng không có ý nghĩa gì nhiều nếu như bạn không rành về vật lý. Thế nên để có thể tưởng tượng đc $400atm$ mạnh đến đâu, đây là 1 video về 1 lon soda nổ tung dưới áp suất *$1atm$*:

{{< youtube atsgIvOUFhA >}}
