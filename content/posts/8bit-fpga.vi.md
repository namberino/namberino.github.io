---
title: "Dự án máy tính 8-bit trên FPGA"
date: 2024-02-26T10:01:26+07:00
tags:
  - lập trình
  - phần cứng
  - fpga
author: "Nam Nguyễn"
description: "Làm 1 máy tính 8-bit bằng Verilog FPGA"
---

Máy tính là 1 công cụ khá là thần bí với đa số mọi người. Mình đảm bảo là nếu như mình hỏi random 1 người trên đường về cách máy tính hoạt động kiểu gì thì có khả năng rất cao là họ sẽ trả lời là "*Tôi không biết*".

Mình cũng từng là 1 người như thế, mình có ý tưởng mơ hồ về cách máy tính hoạt động: lập trình code, dịch code xuống mã nhị phân, máy tính đọc mã nhị phân rồi chạy. Nhưng mà mình không hiểu rõ máy tính làm gì để dịch code xuống mã nhị phân và làm gì với mã nhị phân để chạy chương trình. Thế nên mình quyết định là mình sẽ tìm hiểu về cách máy tính hoạt động. Và cách tìm hiểu về 1 cái gì đó tốt nhất là *tự tay* làm cái gì đó.

Lúc mình mới bắt đầu dự án này thì mình cũng muốn làm máy tính trên bảng cắm dây như [Ben Eater](https://www.youtube.com/playlist?list=PLowKtXNTBypGqImE405J2565dvjafglHU). Nhưng mình nhận ra là mình không có tiền để mua các bộ phận cần thiết để làm máy tính, thế nên mình quyết định là làm máy tính này bằng phần mềm. Và bởi vì mình cũng đang học *Verilog*, mình quyết định là sẽ dùng *Verilog* để làm máy tính này.

___

# Môi trường phát triển FPGA
Mình muốn sử dụng các công cụ phát triển FPGA mã nguồn mở như là: GTKWave, iverilog, yosys, vv. Mình tìm được công cự gọi là [*Apio*](https://github.com/FPGAwars/apio). Nó như là 1 hộp dụng cụ có chứa các công cụ phát triển FPGA mã nguồn mở. Thế nên mình quyết định là sẽ dùng *Apio*.

Cách tải Apio:
- Tải [Python](https://www.python.org/downloads/)
- Tải Apio với `pip` (nếu như không có `pip` thì chạy `easy_install pip`)
```
$ pip install -U apio
```

- Tải các package cần thiết:
```
$ apio install -a
```

Thế là tải xong Apio. Xem trang [quick start](https://apiodoc.readthedocs.io/en/stable/source/quick_start.html) của Apio để học cách dùng nó.

___

# Cấu trúc của máy tính

Mình dựa cấu trúc của máy tính này trên máy tính *SAP-1* trong [Digital Computer Electronics](https://www.amazon.com/Digital-Computer-Electronics-Jerald-Malvino-dp-0074622358/dp/0074622358/ref=dp_ob_image_bk).

![8bit architecture pic](/img/8bit-architecture.png)

Các mô-đun đều có 1 vài tín hiệu giống nhau: *clk*, *rst* và *out*.
- *clk*: Tín hiệu của clock
- *rst*: Tín hiệu cài đặt lại (Chuyển mọi thứ trở về 0)
- *out*: Đường truyền ra của mô-đun (kết nối với bus để các mô-đun có thể giao tiếp, truyền dữ liệu cho nhau)

Cấu trúc của mình có 1 vài điểm khác so với cấu trúc của *SAP-1*. Mình kết hợp mô-đun *MAR* với mô-đun *RAM* để tạo nên mô-đun memory (bộ nhớ). Một vài đường tín hiệu có có tên khác so với *SAP-1* bởi vì mình đang dựa trên cấu trúc *SAP-1* trong trí nhớ mình.

Cấu trúc này sẽ không giống *SAP-1* 100% nhưng nó vẫn sẽ là 1 máy tính 8-bit hoàn chỉnh.

Với cấu trúc hệ thống hoàn thiện rồi thì mình sẽ bắt đầu làm máy tính này.

___

# Giải thích các mô-đun

Đây là cách hoạt động của các mô-đun trong máy tính:
1. *Bus*: Đây là nơi mà mọi dữ liệu sẽ được truyền qua. Nó rộng 8-bit và nó là đường giao tiếp và truyền dữ liệu giữa các mô-đun khác nhau. Bus sẽ có các tín hiệu *enable* để có thể chọn mô-đun nào sẽ được truyền thông tin qua bus tại 1 thời điểm nhất định.
2. *Clock*: Mô-đun này sẽ đồng bộ hóa các mô-đun trong máy tính. Nó như là nhạc trưởng điều hành 1 ban nhạc. Mô-đun này sẽ cho tín hiệu *clk_in* đi qua nếu như tín hiệu *hlt* là 0, và sẽ cho tín hiệu 0 đi qua nếu như *hlt* là 1. Tín hiệu *hlt* là tín hiệu halt (ngừng). Nó dược dùng để làm cho máy tính ngừng việc thực thi câu lệnh.
3. *Program Counter (Bộ đếm chương trình)*: Mô-đun này sẽ lưu địa chỉ của câu lệnh cần được thực thi. Bởi vì bộ nhớ của máy tính này chỉ có 16 byte, mô-đun này sẽ đếm từ địa chỉ `0x0` đến địa chỉ `0xF` (đó là số thập lục phân). Tín hiệu *inc* là tín hiệu cho mô-đun này biết là nó cần đếm đến địa chỉ tiếp theo.
4. *Instruction Register (Thanh ghi câu lệnh)*: Mô-đun này sẽ load câu lệnh từ bộ nhớ và tách opcode và địa chỉ dữ liệu với nhau. *Opcode* là mã của câu lệnh. Máy tính sẽ đọc opcode để biết được câu lệnh nào cần được thực thi. Mỗi câu lệnh sẽ là *8-bit*, 4 bit đầu tiên sẽ là opcode, 4 bit cuối cùng sẽ là địa chỉ của dữ liệu mà câu lệnh đó sử dụng. Với các câu lệnh mà không cần dữ liệu (như là **HLT**) thì 4 bit cuối sẽ không được đọc.
5. *Thanh ghi A*: Đây là thanh ghi chính của máy tính. Nó là thanh ghi lưu trữ dự liệu chính trong lúc thực thi câu lệnh. Nó cần tín hiệu *load* để có thể load dữ liệu từ bộ nhớ vào.
6. *Thanh ghi B*: Đây là thanh ghi hỗ trợ của máy tính. Nó được dùng để lưu dữ liệu được sử dụng cho việc tính toán với dữ liệu trong thanh ghi A. Nó cũng sử dụng tín hiệu *load* để load dữ liệu từ bộ nhớ vào.
7. *Adder*: Đây là mô-đun phụ trách việc tính toán với dự liệu trong bộ nhớ. Nó có thể cộng (A + B) hoặc trừ (A - B). Nó không cần tín hiệu clock bởi vì nó luôn luôn tính toán và cho ra kết quả dựa trên giá trị trong thanh ghi A và B.
8. *Memory (Bộ nhớ)*: Đây là bộ nhớ 16 byte của máy tính. Mô-đun này có thanh ghi 4-bit gọi là *Memory Address Register* (*MAR*), dịch ra tiếng việt, thanh ghi này là *thanh ghi địa chỉ bộ nhớ*. Thanh ghi này có trách nghiệm là tạm thời lưu trữ địa chỉ của câu lệnh hay dữ liệu cần lấy trong bộ nhớ. Địa chỉ trong *MAR* sẽ được gửi vào bộ nhớ và từ đó mà câu lệnh hoặc dữ liệu sẽ được đọc. Máy tính này cần *2 chu kỳ clock* để đọc từ bộ nhớ: Chu kỳ 1 sẽ load địa chỉ cần đọc vào trong *MAR*; Chu kỳ 2 sẽ đọc dữ liệu trong bộ nhớ từ địa chỉ chứa trong *MAR*. Máy tính sẽ load dữ liệu vào trong bộ nhớ nhờ file *program.bin*.
9. *Controller (Bộ điều khiển)*: Đây là mô-đun phức tạp nhất trong máy tính này. Nó sẽ quyết định hành động tiếp theo của máy tính bằng cách gửi các *tín hiệu điều khiển* (có *12 tín hiệu* điều khiển khác nhau) cho các mô-đun khác nhau. Mình sẽ giải thích các tín hiệu điều khiển trong phần tiếp theo.

# Các giai đoạn thực thi câu lệnh:
Việc thực thi câu lệnh xảy ra trong nhiều *đoạn* (mỗi đoạn sẽ mất 1 chu kỳ clock). Máy tính này có *6 đoạn* thực thi (**0** đến **5**). Nó sẽ bắt đầu từ đoạn 0, đếm lên đoạn 5 và quay lại đoạn 0 (nó sẽ đếm bằng thanh ghi 3-bit).

*Opcode* sẽ được truyền vào *thanh ghi câu lệnh* và rồi được truyền vào *bộ điều khiển* để nó có thể gửi các tín hiệu điều khiển cho các mô-đun trong máy tính. Đầu ra của *bộ điều khiển* sẽ là 12 tín hiệu điều khiển, được sử dụng để điều khiển hành động của các mô-đun khác nhau. Mỗi đoạn thực thi của câu lệnh khác nhau sẽ cần tổ hợp tín hiệu điều khiển khác nhau để làm điều khác nhau.

Tín hiệu điều khiển:
- *hlt*: dừng thực thi lệnh
- *pc_inc*: tăng bộ đếm chương trình (*PC*)
- *pc_en*: cho dữ liệu trong *PC* lên bus
- *mar_load*: cho địa chỉ cần truy cập vào *MAR*
- *mem_en*: cho dữ liệu trong bộ nhớ lên bus
- *ir_load*: cho dữ liệu trong bus vào thanh ghi câu lệnh (*IR*)
- *ir_en*: cho dữ liệu trong ir lên bus
- *a_load*: cho dữ liệu trong bus vào thanh ghi A
- *a_en*: cho dữ liệu trong thanh ghi A lên bus
- *b_load*: cho dữ liệu trong bus vào thanh ghi B
- *adder_sub*: chuyển adder sang chế độ trừ (A - B)
- *adder_en*: cho dữ liệu trong adder lên bus

# Câu lệnh của máy tính
Máy tính này có **4** câu lệnh:
| Opcode | Câu lệnh | Miêu tả |
| :----: | ----------- | ----------- |
| *0000* | **LDA $x** | Cho dữ liệu tại địa chỉ *$x* trong bộ nhớ vào A |
| *0001* | **ADD $x** | Cộng dữ liệu tại địa chỉ *$x* trong bộ nhớ với dữ liệu trong A |
| *0010* | **SUB $x** | Trừ dữ liệu trong A với dữ liệu trong bộ nhớ tại địa chỉ $x |
| *1111* | **HLT** | Ngừng việc thực thi câu lệnh của máy tính |

Câu lệnh nào cũng có *3 đoạn đầu* giống nhau:
- **Đoạn 0**: Cho dữ liệu trong *PC* lên bus và load dữ liệu đó vào *MAR* (*pc_en* -> *mar_load*)
- **Đoạn 1**: Tăng dữ liệu trong *PC* (*pc_inc*)
- **Đoạn 2**: Cho dữ liệu trong bộ nhớ tại địa chỉ *MAR* lên bus và load dữ liệu đó vào *IR* (*mem_en* -> *ir_load*)

Mỗi câu lệnh khác nhau sẽ có *3 đoạn cuối* khác nhau:
| Đoạn | LDA | ADD | SUB | HLT |
| ----- | --- | --- | --- | --- |
| **Đoạn 3** | Cho dữ liệu trong *IR* lên bus và load dữ liệu đó vào *MAR* (*ir_en* -> *mar_load*)  | Cho dữ liệu trong *IR* lên bus và load dữ liệu đó vào *MAR* (*ir_en* -> *mar_load*) | Cho dữ liệu trong *IR* lên bus và load dữ liệu đó vào *MAR* (*ir_en* -> *mar_load*) | Ngừng clock (*hlt*) |
| **Đoạn 4** | Cho dữ liệu trong bộ nhớ tại địa chỉ *MAR* lên bus và load dữ liệu đó vào *A* (*mem_en* -> *a_load*) | Cho dữ liệu trong bộ nhớ tại địa chỉ *MAR* lên bus và load dữ liệu đó vào *B* (*mem_en* -> *b_load*) | Cho dữ liệu trong bộ nhớ tại địa chỉ *MAR* lên bus và load dữ liệu đó vào *B* (*mem_en* -> *b_load*) | Chạy không (Idle) |
| **Đoạn 5** | Chạy không (Idle) | Cho dữ liệu ở đầu ra của *adder* lên bus và load dữ liệu đó vào *A* (*adder_en* -> *a_load*) | Chuyển *adder* sang chế độ trừ và cho dữ liệu ở đầu ra của *adder* lên bus và load dữ liệu đó vào *A* (*adder_sub* -> *adder_en* -> *a_load*) | Chạy không (Idle) |

# Lập trình Verilog

Các mô-đun này sẽ được lập trình trong ngôn ngữ *Verilog*. Máy tính này sẽ có mô-đun tên là `top_design` để kết nối các mô-đun này với nhau. Mình sẽ lập trình *testbench* cho mô-đun `top_design` này để kiểm tra xem máy tính có hoạt động đúng hay không.

## Clock
```verilog
module clock(
	input hlt, // tín hiệu halt
	input clk_in,

	output clk_out
);

	assign clk_out = hlt ? 1'b0 : clk_in;

endmodule
```

## Program Counter (Bộ đếm chương trình)
```verilog
module pc(
	input clk,
	input rst,
	input inc,

	output[7:0] out
);
	
	reg[3:0] pc;
	
	always @ (posedge clk, posedge rst)
	begin
		if (rst)
		begin
			pc <= 4'b0;
		end else if (inc)
		begin
			pc <= pc + 1;
		end
	end

	assign out = pc;

endmodule
```

## Instruction Register (Thanh ghi câu lệnh)
```verilog
module ir(
	input clk, 
	input rst,
	input load,
	input[7:0] bus,

	output[7:0] out
);

	reg[7:0] ir;

	always @ (posedge clk, posedge rst)
	begin
		if (rst)
		begin
			ir <= 8'b0;
		end else if (load)
		begin
			ir <= bus;
		end
	end

	assign out = ir;

endmodule
```

## Thanh ghi A
```verilog
module reg_a(
	input clk,
	input rst,
	input load,
	input[7:0] bus,

	output[7:0] out
);

	reg[7:0] reg_a;

	always @ (posedge clk, posedge rst)
	begin
		if (rst)
		begin
			reg_a <= 8'b0;
		end else if (load)
		begin
			reg_a <= bus;
		end
	end

	assign out = reg_a;

endmodule
```

## Thanh ghi B
```verilog
module reg_b(
	input clk,
	input rst,
	input load,
	input[7:0] bus,

	output[7:0] out
);

	reg[7:0] reg_b;

	always @ (posedge clk, posedge rst)
	begin
		if (rst)
		begin
			reg_b <= 8'b0;
		end else if (load)
		begin
			reg_b <= bus;
		end
	end

	assign out = reg_b;

endmodule
```

## Adder
```verilog
module adder(
	input[7:0] a,
	input[7:0] b,
	input sub,

	output[7:0] out
);
	
	assign out = sub ? a - b : a + b;

endmodule
```

## Memory (Bộ nhớ)
```verilog
module memory(
	input clk,
	input rst,
	input load,
	input[7:0] bus,

	output[7:0] out
);

	// thiết lập bộ nhớ
	initial begin
		$readmemh("program.bin", ram);
	end

	reg[3:0] mar;
	reg[7:0] ram[0:15];		// 16 byte bộ nhớ (1 byte = 8 bit)

	always @ (posedge clk, posedge rst)
	begin
		if (rst)
		begin
			mar <= 4'b0;
		end else if (load)
		begin
			mar <= bus[3:0];
		end
	end

	assign out = ram[mar];

endmodule
```

## Controller (Bộ điều khiển)
```verilog
/*
Tín hiệu điều khiển:
    *hlt*: dừng thực thi lệnh
    *pc_inc*: tăng bộ đếm chương trình (*PC*)
    *pc_en*: cho dữ liệu trong *PC* lên bus
    *mar_load*: cho địa chỉ cần truy cập vào *MAR*
    *mem_en*: cho dữ liệu trong bộ nhớ lên bus
    *ir_load*: cho dữ liệu trong bus vào thanh ghi câu lệnh (*IR*)
    *ir_en*: cho dữ liệu trong ir lên bus
    *a_load*: cho dữ liệu trong bus vào thanh ghi A
    *a_en*: cho dữ liệu trong thanh ghi A lên bus
    *b_load*: cho dữ liệu trong bus vào thanh ghi B
    *adder_sub*: chuyển adder sang chế độ trừ (A - B)
    *adder_en*: cho dữ liệu trong adder lên bus
*/

module controller(
	input clk,
	input rst,
	input[3:0] opcode,

	output[11:0] out
);

	localparam HLT = 11;
	localparam PC_INC = 10;
	localparam PC_EN = 9;
	localparam MAR_LOAD = 8;
	localparam MEM_EN = 7;
	localparam IR_LOAD = 6;
	localparam IR_EN = 5;
	localparam A_LOAD = 4;
	localparam A_EN = 3;
	localparam B_LOAD = 2;
	localparam ADDER_SUB = 1;
	localparam ADDER_EN = 0;

	localparam OP_LDA = 4'b0000;
	localparam OP_ADD = 4'b0001;
	localparam OP_SUB = 4'b0010;
	localparam OP_HLT = 4'b1111;

	reg[2:0] stage;
	reg[11:0] ctrl_word;

    // tăng đoạn thực thi
	always @ (posedge clk, posedge rst)
	begin
		if (rst)
		begin
			stage <= 0;
		end else 
		begin
			if (stage == 5)
			begin
				stage <= 0;
			end else
			begin
				stage <= stage + 1;
			end
		end
	end

    // check đoạn thực thi và opcode
	always @ (*)
	begin
		ctrl_word = 12'b0;

		case (stage)
			0:
				begin
					ctrl_word[PC_EN] = 1;
					ctrl_word[MAR_LOAD] = 1;
				end
			1:
				begin
					ctrl_word[PC_INC] = 1;
				end

			2: 
				begin
					ctrl_word[MEM_EN] = 1;
					ctrl_word[IR_LOAD] = 1;
				end

			3:
				begin
					case (opcode)
						OP_LDA: 
                            begin
                                ctrl_word[IR_EN] = 1;
                                ctrl_word[MAR_LOAD] = 1;
                            end

                        OP_ADD: 
                            begin
                                ctrl_word[IR_EN] = 1;
                                ctrl_word[MAR_LOAD] = 1;
                            end

                        OP_SUB: 
                            begin
                                ctrl_word[IR_EN] = 1;
                                ctrl_word[MAR_LOAD] = 1;
                            end

                        OP_HLT: 
                            begin
                                ctrl_word[HLT] = 1;
                            end
                    endcase
				end
            4: 
                begin
                    case (opcode)
                        OP_LDA: 
                            begin
                                ctrl_word[MEM_EN] = 1;
                                ctrl_word[A_LOAD] = 1;
                            end

                        OP_ADD: 
                            begin
                                ctrl_word[MEM_EN] = 1;
                                ctrl_word[B_LOAD] = 1;
                            end

                        OP_SUB: 
                            begin
                                ctrl_word[MEM_EN] = 1;
                                ctrl_word[B_LOAD] = 1;
                            end
                    endcase
                end

            5: 
                begin
                    case (opcode)
                        OP_ADD: 
                            begin
                                ctrl_word[ADDER_EN] = 1;
                                ctrl_word[A_LOAD] = 1;
                            end

                        OP_SUB:
                            begin
                                ctrl_word[ADDER_SUB] = 1;
                                ctrl_word[ADDER_EN] = 1;
                                ctrl_word[A_LOAD] = 1;
                            end
                    endcase
                end
        endcase
	end

    assign out = ctrl_word;

endmodule
```

## Mô-đun top_design
```verilog
module top_design(
	input CLK
);

	reg[7:0] bus;

	// chọn giữa các đầu ra của các mô-đun
	always @ (*)
	begin
		if (ir_en) 
		begin
			bus = ir_out;
		end else if (adder_en) 
		begin
			bus = adder_out;
		end else if (a_en) 
		begin
			bus = a_out;
		end else if (mem_en) 
		begin
			bus = mem_out;
		end else if (pc_en) 
		begin
			bus = pc_out;
		end else 
		begin
			bus = 8'b0;
		end
	end

    // clock
    wire rst;
    wire hlt;
    wire clk;
    clock clock (
        .hlt(hlt),
        .clk_in(CLK),
        .clk_out(clk)
    );


    // program counter
    wire pc_inc;
    wire pc_en;
    wire[7:0] pc_out;
    pc pc(
        .clk(clk),
        .rst(rst),
        .inc(pc_inc),
        .out(pc_out)
    );


    // memory
    wire mar_load;
    wire mem_en;
    wire[7:0] mem_out;
    memory mem(
        .clk(clk),
        .rst(rst),
        .load(mar_load),
        .bus(bus),
        .out(mem_out)
    );


    // register A
    wire a_load;
    wire a_en;
    wire[7:0] a_out;
    reg_a reg_a(
        .clk(clk),
        .rst(rst),
        .load(a_load),
        .bus(bus),
        .out(a_out)
    );


    // register B
    wire b_load;
    wire[7:0] b_out;
    reg_b reg_b(
        .clk(clk),
        .rst(rst),
        .load(b_load),
        .bus(bus),
        .out(b_out)
    );


    // adder 
    wire adder_sub;
    wire adder_en;
    wire[7:0] adder_out;
    adder adder(
        .a(a_out),
        .b(b_out),
        .sub(adder_sub),
        .out(adder_out)
    );


    // instruction register
    wire ir_load;
    wire ir_en;
    wire[7:0] ir_out;
    ir ir(
        .clk(clk),
        .rst(rst),
        .load(ir_load),
        .bus(bus),
        .out(ir_out)
    );


    // controller
    controller controller(
        .clk(clk),
        .rst(rst),
        .opcode(ir_out[7:4]), // 4 bit đầu
        .out( // các tín hiệu out được kết nối với các tín hiệu bên trên
        {
            hlt,
            pc_inc,
            pc_en,
            mar_load,
            mem_en,
            ir_load,
            ir_en,
            a_load,
            a_en,
            b_load,
            adder_sub,
            adder_en
        })
    );

endmodule
```

## Testbench cho top_design
```verilog
module top_design_tb();

    initial begin
        $dumpfile("top_design_tb.vcd");
        $dumpvars(0, top_design_tb);

        // reset máy tính
        rst = 1;
        #1
        rst = 0;
    end

    // chọn giữa các đầu ra
    wire[4:0] bus_en = {pc_en, mem_en, ir_en, a_en, adder_en};
    reg[7:0] bus;
    always @ (*) 
    begin
        case (bus_en)
            5'b00001: bus = adder_out;
            5'b00010: bus = a_out;
            5'b00100: bus = ir_out;
            5'b01000: bus = mem_out;
            5'b10000: bus = pc_out;
            default: bus = 8'b0;
        endcase
    end

    // tín hiệu clock
    reg clk_in = 0;
    integer i;
    initial begin
        for (i = 0; i < 128; i++)
        begin
            #1
            clk_in = ~clk_in;
        end
    end

    wire clk;
    wire hlt;
    reg rst;
    clock clock(
        .hlt(hlt),
        .clk_in(clk_in),
        .clk_out(clk)
    );

    wire pc_inc;
    wire pc_en;
    wire[7:0] pc_out;
    pc pc(
        .clk(clk),
        .rst(rst),
        .inc(pc_inc),
        .out(pc_out)
    );

    wire mar_load;
    wire mem_en;
    wire[7:0] mem_out;
    memory mem(
        .clk(clk),
        .rst(rst),
        .load(mar_load),
        .bus(bus),
        .out(mem_out)
    );


    wire a_load;
    wire a_en;
    wire[7:0] a_out;
    reg_a reg_a(
        .clk(clk),
        .rst(rst),
        .load(a_load),
        .bus(bus),
        .out(a_out)
    );


    wire b_load;
    wire[7:0] b_out;
    reg_b reg_b(
        .clk(clk),
        .rst(rst),
        .load(b_load),
        .bus(bus),
        .out(b_out)
    );


    wire adder_sub;
    wire adder_en;
    wire[7:0] adder_out;
    adder adder(
        .a(a_out),
        .b(b_out),
        .sub(adder_sub),
        .out(adder_out)
    );


    wire ir_load;
    wire ir_en;
    wire[7:0] ir_out;
    ir ir(
        .clk(clk),
        .rst(rst),
        .load(ir_load),
        .bus(bus),
        .out(ir_out)
    );

    controller controller(
        .clk(clk),
        .rst(rst),
        .opcode(ir_out[7:4]),
        .out(
        {
            hlt,
            pc_inc,
            pc_en,
            mar_load,
            mem_en,
            ir_load,
            ir_en,
            a_load,
            a_en,
            b_load,
            adder_sub,
            adder_en
        })
    );

endmodule
```

# Lập trình cho máy tính
Để lập trình trên máy tính này, chúng ta có thể lập trình từng byte trong file `program.bin`. File này sẽ được load vào mô-đun bộ nhớ khi máy tính được khởi động. Đây là 1 chương trình mẫu:
```bin
0D 2E 1F F0 00 00 00 00 00 00 00 00 00 05 04 02
```

Giải thích từng byte trong file `program.bin` (số hex đầu tiên sẽ là opcode, số hex thứ 2 sẽ là địa chỉ của dữ liệu):
```bin
$0      0D      // LDA $D   Load dữ liệu tại địa chỉ $D vào A
$1      1E      // ADD $E   Cộng dữ liệu trong A với dữ liệu tại địa chỉ $E
$2      2F      // SUB $F   Trừ dữ liệu trong A với dữ liệu tại địa chỉ $F
$3      F0      // HLT      Ngừng thực thi câu lệnh
$4      00      // Byte trống
$5      00      // Byte trống
$6      00      // Byte trống
$7      00      // Byte trống
$8      00      // Byte trống
$9      00      // Byte trống
$A      00      // Byte trống
$B      00      // Byte trống
$C      00      // Byte trống
$D      05      // Dữ liệu
$E      04      // Dữ liệu
$F      02      // Dữ liệu
```

Cuối cùng thì chúng ta đã làm xong máy tính 8-bit có thể hoạt động được. Đây là mô phỏng (*simulation*) của máy tính:

![8bit computer waveforms](/img/8bit-computer-waveforms.png)

> Bạn có thể đọc mã nguồn của dự án này tại [đây](https://github.com/namberino/8bit-computer).

# Tài liệu tham khảo
- [Series máy tính 8-bit của Ben Eater](https://www.youtube.com/playlist?list=PLowKtXNTBypGqImE405J2565dvjafglHU)
- [Digital Computer Electronics](https://www.amazon.com/Digital-Computer-Electronics-Jerald-Malvino-dp-0074622358/dp/0074622358/ref=dp_ob_title_bk)
- [SAP-1 Implementation Report](https://drive.google.com/file/d/17fH-JBU5OX_4AG123AO47y879YxzmDwX/view)
