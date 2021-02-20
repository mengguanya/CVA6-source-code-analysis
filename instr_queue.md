# instr_queue.sv

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

