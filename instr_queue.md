# instr_queue.sv

instr_queue.sv 是衔接前端和译码（decode）阶段的部件，其支持前端分支预测，并且这个queue只保留有效的指令。

## 1.1  交互端口（顶层封装）

指令队列是衔接前端和后端的部件，其与前端有交互。首先，ready_o信号是传递给前端的准备信号，consumed_o信号是传递给前端的用于标志分支预测的指令是否已经传递到后端（译码阶段）。

```
  output logic                                               ready_o,
  output logic [ariane_pkg::INSTR_PER_FETCH-1:0]             consumed_o,
```

例外信号，只可能是页表错误信号。因为前面已经对其他例外做了处理。

```
  // we've encountered an exception, at this point the only possible exceptions are page-table faults
  input  ariane_pkg::frontend_exception_t                    exception_i,
  input  logic [riscv::VLEN-1:0]                             exception_addr_i,
```

前端传递过来的分支预测信号。

```
  // branch predict
  input  logic [riscv::VLEN-1:0]                             predict_address_i,
  input  ariane_pkg::cf_t  [ariane_pkg::INSTR_PER_FETCH-1:0] cf_type_i,
```

队列满导致指令无法写入，需要前端重新取值。

```
  // replay instruction because one of the FIFO was already full
  output logic                                               replay_o,
  output logic [riscv::VLEN-1:0]                             replay_addr_o, // address at which to replay this instruction
```

把前端的所有信号传递到后端，包括指令、地址、有效信号、分支预测，这些都定义在fetch_entry_o中。

```
  // to processor backend
  output ariane_pkg::fetch_entry_t                           fetch_entry_o,
  output logic                                               fetch_entry_valid_o,
  input  logic                                               fetch_entry_ready_i
```

## 1.2  内部信号



## 1.3  信号传递

在此，例化了lzc模块。lzc模块是common cell库中定义的，主要作用是实现前导零和后补零的计数。

```
module lzc #(
  /// The width of the input vector.
  parameter int unsigned WIDTH = 2,
  parameter bit          MODE  = 1'b0, // 0 -> trailing zero, 1 -> leading zero
  // Dependent parameters. Do not change!
  parameter int unsigned CNT_WIDTH = WIDTH == 1 ? 1 : $clog2(WIDTH)
) (
  input  logic [WIDTH-1:0]     in_i,
  output logic [CNT_WIDTH-1:0] cnt_o,
  output logic                 empty_o // asserted if all bits in in_i are zero
);
```

例化fifo_v3生成queue。在common cell中定义了它（V3表示**版本3**的fifo，前面的两个版本已经弃用）。

```
module fifo_v3 #(
    parameter bit          FALL_THROUGH = 1'b0, // fifo is in fall-through mode
    parameter int unsigned DATA_WIDTH   = 32,   // default data width if the fifo is of type logic
    parameter int unsigned DEPTH        = 8,    // depth can be arbitrary from 0 to 2**32
    parameter type dtype                = logic [DATA_WIDTH-1:0],
    // DO NOT OVERWRITE THIS PARAMETER
    parameter int unsigned ADDR_DEPTH   = (DEPTH > 1) ? $clog2(DEPTH) : 1
)(
    input  logic  clk_i,            // Clock
    input  logic  rst_ni,           // Asynchronous reset active low
    input  logic  flush_i,          // flush the queue
    input  logic  testmode_i,       // test_mode to bypass clock gating
    // status flags
    output logic  full_o,           // queue is full
    output logic  empty_o,          // queue is empty
    output logic  [ADDR_DEPTH-1:0] usage_o,  // fill pointer
    // as long as the queue is not full we can push new data
    input  dtype  data_i,           // data to push into the queue
    input  logic  push_i,           // data is valid and can be pushed to the queue
    // as long as the queue is not empty we can pop new elements
    output dtype  data_o,           // output data
    input  logic  pop_i             // pop head from queue
);
```

