# 从零做一个可读回的向量加法外设

这个原创实验把链路训练、枚举和 PCIe 报文留在 VM 工程里，聚焦你需要自己能写和能验证的部分：寄存器映射、握手、向量计算、错误响应和可重复测试。接口是教学用的简化总线，地址为字节偏移，数据宽度为 16 位。有两路 8 位无符号加法，各路分别截断到 8 位。

## 1. 先写规格

| 字节地址 | 名称 | 读写规则 |
| --- | --- | --- |
| `0x0` | A | 读写两个操作数，`req_strb[0]` 控制低字节，`req_strb[1]` 控制高字节 |
| `0x2` | B | 同 A |
| `0x4` | CMD/STATUS | 低字节有效时写 bit 0 启动。读回 bit 0 为 busy，其余位为 0 |
| `0x6` | RESULT | 只读，返回两个加法结果 |

每个时钟上升沿只有 `req_valid && req_ready` 才接受请求。接受后产生一个响应，`rsp_valid && rsp_ready` 才取走它。接收方暂停时，响应保持原值，模块停止接受下一条请求。这让一个寄存器就能保存未完成响应，代价是吞吐量低。非法地址及写 RESULT 返回错误，成功写操作的响应数据为 0。复位清零所有状态。命令被接受后 busy 为高一个周期，在下一次上升沿锁存结果。教学接口的请求和响应分离只是为了练习握手，和 AXI 的五个通道有区别。

例题：A 是 `16'hff09`，B 是 `16'h0207`。低字节是 `0x09 + 0x07 = 0x10`，高字节是 `0xff + 0x02 = 0x101`，截断得到 `0x01`。因此 RESULT 应为 `16'h0110`。若你把整个 16 位数一次相加，低字节进位有可能错误传到高字节。测试会再用一组低字节溢出的数据专门检查这点。

## 2. 写 RTL

把下面的代码保存为 `study_vec_regs.sv`。寄存器写入和响应更新都在时钟边沿发生。写 A 或 B 时逐字节更新。计算结果分两次赋值给各自的 8 位切片，位宽本身给出了截断行为。

```systemverilog
`timescale 1ns/1ps
module study_vec_regs (
    input  logic        clk, rst_n,
    input  logic        req_valid, req_write,
    output logic        req_ready,
    input  logic [3:0]  req_addr,
    input  logic [15:0] req_data,
    input  logic [1:0]  req_strb,
    output logic        rsp_valid, rsp_error,
    input  logic        rsp_ready,
    output logic [15:0] rsp_data
);
    logic [15:0] a, b, result;
    logic busy;

    assign req_ready = rst_n && !rsp_valid;

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            a <= '0;
            b <= '0;
            result <= '0;
            busy <= 1'b0;
            rsp_valid <= 1'b0;
            rsp_error <= 1'b0;
            rsp_data <= '0;
        end else begin
            if (busy) begin
                result[7:0]  <= a[7:0]  + b[7:0];
                result[15:8] <= a[15:8] + b[15:8];
                busy <= 1'b0;
            end

            if (rsp_valid && rsp_ready)
                rsp_valid <= 1'b0;

            if (req_valid && req_ready) begin
                rsp_valid <= 1'b1;
                rsp_error <= 1'b0;
                rsp_data <= '0;
                case (req_addr)
                    4'h0: begin
                        if (req_write) begin
                            if (req_strb[0]) a[7:0]  <= req_data[7:0];
                            if (req_strb[1]) a[15:8] <= req_data[15:8];
                        end else rsp_data <= a;
                    end
                    4'h2: begin
                        if (req_write) begin
                            if (req_strb[0]) b[7:0]  <= req_data[7:0];
                            if (req_strb[1]) b[15:8] <= req_data[15:8];
                        end else rsp_data <= b;
                    end
                    4'h4: begin
                        if (req_write) begin
                            if (req_strb[0] && req_data[0]) busy <= 1'b1;
                        end else rsp_data <= {15'b0, busy};
                    end
                    4'h6: begin
                        if (req_write) rsp_error <= 1'b1;
                        else rsp_data <= result;
                    end
                    default: rsp_error <= 1'b1;
                endcase
            end
        end
    end
endmodule
```

## 3. 写可自动判断通过的 testbench

把下面保存为 `tb_study_vec_regs.sv`。testbench 在下降沿驱动，给 DUT 下一上升沿采样留下稳定窗口。在下降沿检查响应，避开时序块刚采样时的调度竞争。`!==` 把未知位也当作失败。`stall_cycles` 用来检查响应停住时的数据、错误位和握手条件。最后用全局 watchdog 避免等待永远挂住。

```systemverilog
`timescale 1ns/1ps
module tb_study_vec_regs;
    logic clk = 0, rst_n = 0;
    logic req_valid = 0, req_write = 0, req_ready;
    logic [3:0] req_addr = '0;
    logic [15:0] req_data = '0;
    logic [1:0] req_strb = '0;
    logic rsp_valid, rsp_error, rsp_ready = 0;
    logic [15:0] rsp_data;

    always #5 clk = ~clk;
    study_vec_regs dut (.*);

    task automatic transfer(
        input bit wr,
        input logic [3:0] addr,
        input logic [15:0] data,
        input logic [1:0] strb,
        input logic [15:0] expected,
        input bit expected_error,
        input int stall_cycles
    );
        @(negedge clk);
        req_valid = 1;
        req_write = wr;
        req_addr = addr;
        req_data = data;
        req_strb = strb;
        do @(posedge clk); while (!req_ready);
        @(negedge clk);
        req_valid = 0;
        for (int cycle = 0; cycle <= stall_cycles; cycle++) begin
            if (cycle != 0) @(negedge clk);
            if (rsp_valid !== 1 || req_ready !== 0 ||
                rsp_data !== expected || rsp_error !== expected_error)
                $fatal(1, "addr=%h expected=%h/%b actual=%h/%b",
                       addr, expected, expected_error, rsp_data, rsp_error);
        end
        rsp_ready = 1;
        @(negedge clk);
        rsp_ready = 0;
        if (rsp_valid !== 0) $fatal(1, "response did not clear");
    endtask

    initial begin
        repeat (2) @(negedge clk);
        rst_n = 1;
        transfer(0, 4'h6, '0, 0, 16'h0000, 0, 0);       // reset
        transfer(1, 4'h0, 16'hff09, 3, '0, 0, 0);
        transfer(1, 4'h2, 16'h0207, 3, '0, 0, 0);
        transfer(1, 4'h4, 16'h0001, 1, '0, 0, 0);
        transfer(0, 4'h6, '0, 0, 16'h0110, 0, 3);       // overflow + stall
        transfer(1, 4'h0, 16'h00fe, 1, '0, 0, 0);
        transfer(0, 4'h0, '0, 0, 16'hfffe, 0, 0);       // byte enable
        transfer(1, 4'h4, 16'h0001, 1, '0, 0, 0);
        transfer(0, 4'h6, '0, 0, 16'h0105, 0, 0);       // no lane carry
        transfer(1, 4'h6, '1, 3, '0, 1, 0);            // read-only
        transfer(0, 4'h6, '0, 0, 16'h0105, 0, 0);
        transfer(0, 4'h1, '0, 0, '0, 1, 0);            // unaligned
        $display("PASS: study_vec_regs");
        $finish;
    end

    initial begin
        #2000;
        $fatal(1, "test timeout");
    end
endmodule
```

在两个文件所在的实验目录运行下面一组命令。VCS 选项分别指定 64 位、SystemVerilog、顶层模块和输出文件。Vivado 的对应步骤是 `xvlog` 分析源码，`xelab` 展开为快照，`xsim` 执行。

```bash
vcs -full64 -sverilog -timescale=1ns/1ps -top tb_study_vec_regs \
  study_vec_regs.sv tb_study_vec_regs.sv -o simv
./simv
```

```bash
# 在 Vivado 2023.1 可执行文件已加入 PATH 的环境
xvlog -sv study_vec_regs.sv tb_study_vec_regs.sv
xelab tb_study_vec_regs -s study_vec_sim
xsim study_vec_sim -runall
```

预期输出 `PASS: study_vec_regs`，该代码已在 VM 的 VCS V-2023.12-SP2 和 Vivado XSim 2023.1 实跑通过。在 Vivado IDE 也可以把 RTL 加到 Design Sources，把 testbench 加到 Simulation Sources，设为仿真顶层后运行 Behavioral Simulation。

## 4. 从能跑进阶到能验证

1. **验证 testbench 会抓错。** 在实验副本中把高字节相加改成相减。重新编译后应在第一次结果读回报错。恢复 RTL，再看到 PASS。保存首次失败行和复现命令。
2. **从固定数据扩展到组合。** 用固定种子生成两路操作数，以整数求和后分别取低 8 位，按字节组包作为预测值。加入清零、全 1、低字节溢出、高字节溢出、单字节写和两种响应背压长度。参考模型从请求序列更新状态。
3. **把检查放到接口边界。** Monitor 只在有效握手采集请求和响应。Scoreboard 按接受顺序比对地址、数据和错误位。SVA 可检查 `rsp_valid && !rsp_ready |=> rsp_valid && $stable({rsp_data, rsp_error})`，并在复位时禁用。功能覆盖采样成功握手的交易，选择读写方向、地址类别、字节使能、加法溢出和有无背压几个维度。
4. **迁移到 UVM。** 把 task 的入参做成 sequence item，把下降沿驱动和握手处理移到 driver，把读写和错误场景做成独立 sequence。Monitor 向 scoreboard 和 coverage collector 广播实测交易。最后把一个基础测试和一个带背压测试加入回归，用日志记录 seed 与检查数。
5. **和 VM 项目对照。** 教学代码的直接请求相当于 Endpoint 用户侧看到的访问。真实工程由 RC 侧 sequence 发 TLP，经 BAR 匹配、控制器和 AXI 才到寄存器。回读再经控制器变成 Completion。按第 3 章的顺序标出自己能在两份工程里定位到的对应环节。

拓展练习可以给 busy 长度参数化、定义复位中断未完成交易的行为，或为多笔未完成读维护 tag 到期望响应的映射。每次只添加一个行为，同时补一条规格、一项测试和一条可以从波形或日志确认的通过标准。

[下一章：把实验接到验证计划与数字 IC 流程](05-ic-flow.md)
