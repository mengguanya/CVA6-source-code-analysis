# 1 frontend.sv

## 1.1  交互端口（顶层封装）

分析交互端口之前，我们首先需要明确的前端的主要任务，即包括：***确定取指地址***（包括分支预测，接收中断等信号）、***访问cache取指***、***发送指令到指令队列***以供处理器后端使用等三个部分。

### 1.1.1  确定取值地址

取值地址的来源有7个，除了默认PC+4和由前端内部产生的分支预测之外，还有来自其他模块的信号包括：分支预测失败、函数返回、例外/中断、刷新流水线、debug。

#### 1.1.1.1  分支预测失败

分支预测结果信号（resolved_branch_i）来源于控制器（controller），而它是执行阶段产生，传输到控制器的。当分支预测结果为失败也就是（resolved_branch_i.is_mispredict == true）时，会发生取值地址的改变，并更新BTB表（分支历史表）。

```
// mispredict
	input  bp_resolve_t        resolved_branch_i,  // from controller signaling a branch_predict -> update BTB
```

分支预测结果信号包括有效信号、PC值、目的地址、预测是否正确、是否发生了转移、分支类型等。

```
// branch-predict
// this is the struct we get back from ex stage and we will use it to update
// all the necessary data structures
// bp_resolve_t
typedef struct packed {
    logic                   valid;           // prediction with all its values is valid
    logic [riscv::VLEN-1:0] pc;              // PC of predict or mis-predict
    logic [riscv::VLEN-1:0] target_address;  // target address at which to jump, or not
    logic                   is_mispredict;   // set if this was a mis-predict
    logic                   is_taken;        // branch is taken
    cf_t                    cf_type;         // Type of control flow change
} bp_resolve_t;
```

#### 1.1.1.2  刷新流水线

刷新流水线的信号来自于提交阶段（commit）。包括是否刷新和刷新后的PC。刷新流水线信号的原因是是CSR寄存器的改变。

```
// from commit, when flushing the whole pipeline
    input  logic               set_pc_commit_i,    // Take the PC from commit stage
    input  logic [riscv::VLEN-1:0] pc_commit_i,        // PC of instruction in commit stage
```

#### 1.1.1.3  环境调用返回（return from environment call）

进程通过ECALL指令向执行环境（通常是操作系统）发起系统函数调用请求，并执行完成之后，返回。

```
// CSR input
    input  logic [riscv::VLEN-1:0] epc_i,              // exception PC which we need to return to
    input  logic               eret_i,             // return from exception
```

#### 1.1.1.4  例外/中断

发生中断之后，设置中断向量。

```
input  logic [riscv::VLEN-1:0] trap_vector_base_i, // base of trap vector
```

#### 1.1.1.5  debug

```
input  logic               set_debug_pc_i,     // jump to debug address
```

### 1.1.2   访问cache

前端的cache交互部分被封装成了两个独立的接口，非常简洁。我们可以从ariane_pkg.sv文件中找到关于这两个接口的定义。

```
// Instruction Fetch
output icache_dreq_i_t   icache_dreq_o,
input icache_dreq_o_t   icache_dreq_i,
```

首先是icache_dreq_i_t，该数据类型表示前端向cache发出的请求包。包括请求有效位、请求指令所在的地址、请求是否是推测执行的、取消当前请求或上一条请求。具体内容我们在cache部分详细讨论。

另外，在这里叙述一下system Verilog的数据类型。此处是定义了结构体类型，类似于C语言。但是我们发现 **packed**参数，这是表示 **合并结构以连续的bit集来存放数据**，合并struct和非合并struct的区别在于仿真的效率上。

```
// I$ data requests
typedef struct packed {
    logic                     req;     // we request a new word
    logic                     kill_s1; // kill the current request
    logic                     kill_s2; // kill the last request
    logic                     spec;    // request is speculative
    logic [riscv::VLEN-1:0]   vaddr;   // 1st cycle: 12 bit index is taken for lookup
} icache_dreq_i_t;
```

然后是icache_dreq_o_t，该数据类型表示指令cache向前端发出的响应包。包括cache就绪信号、cache例外信号、返回数据以及其有效信号和所在地址。

```
typedef struct packed {
    logic                     ready;          // icache is ready
    logic                     valid;          // signals a valid read
    logic [FETCH_WIDTH-1:0]   data;           // 2+ cycle out: tag
    logic [riscv::VLEN-1:0]   vaddr;          // virtual address out
    exception_t               ex;             // we've encountered an exception
} icache_dreq_o_t;
```

上面提到了cache例外信号，我们来看一下cva6中关于例外信号的封装（在ariane_pkg.sv中），包括例外有效信号、例外类型、造成例外的其他信息。

```
// Only use struct when signals have same direction
// exception
typedef struct packed {
    riscv::xlen_t       cause; // cause of exception
    riscv::xlen_t       tval;  // additional information of causing exception (e.g.: instruction causing it),
    // address of LD/ST fault
    logic        valid;
} exception_t;
```

### 1.1.3  发送指令到指令队列

前端发送到译码阶段的信息包括译码所需信息及其有效信号、译码阶段的响应确认信号。在ariane_pkg.sv中可以发现关于译码所需信号的封装。

```
// instruction output port -> to processor back-end
output fetch_entry_t       fetch_entry_o,       // fetch entry containing all relevant data for the ID stage
output logic               fetch_entry_valid_o, // instruction in IF is valid
input  logic               fetch_entry_ready_i  // ID acknowledged this instruction
```

前端提供给译码阶段的信号，包括指令及其地址、之前发送的指令可能发生的例外信息、之前发送的指令可能发生的分支转移信息。

```
// store the decompressed instruction
typedef struct packed {
    logic [riscv::VLEN-1:0] address;        // the address of the instructions from below
    logic [31:0]            instruction;    // instruction word
    branchpredict_sbe_t     branch_predict; // this field contains branch prediction information regarding the forward branch path
    exception_t             ex;             // this field contains exceptions which might have happened earlier, e.g.: fetch exceptions
} fetch_entry_t;
```

分支转移信息中包括控制流预测类型和预测地址。类型是通过枚举定义的，包括：不是控制流预测、分支指令、立即数跳转、寄存器跳转、函数返回。另外，分支转移信息也是计分板表项中的一项。计分板表项在scoreboard.sv部分，再做详细解读。

```
// branchpredict scoreboard entry
// this is the struct which we will inject into the pipeline to guide the various
// units towards the correct branch decision and resolve
typedef struct packed {
    cf_t                    cf;              // type of control flow prediction
    logic [riscv::VLEN-1:0] predict_address; // target address at which to jump, or not
} branchpredict_sbe_t;
```

以下是关于分支转移**cf_t**的定义。在此补充一下，关于SV的数据结构定义。和C语言类似，SV使用**enum表示枚举类型**，默认是int类型，但是也可以在enum之后加参数比如该例子中的logic [2:0]以指定数据类型。

```
typedef enum logic [2:0] {
    NoCF,   // No control flow prediction
    Branch, // Branch
    Jump,   // Jump to address from immediate
    JumpR,  // Jump to address from registers
    Return  // Return Address Prediction
} cf_t;		
```

## 1.2  内部信号

### 1.2.1 指令cache信号寄存器

system Verilog中的logic数据类型可以作为**reg**或者**wire**，在此是作为reg以寄存指令cache的信号。

```
// Instruction Cache Registers, from I$
    logic [FETCH_WIDTH-1:0] icache_data_q;
    logic                   icache_valid_q;
    ariane_pkg::frontend_exception_t icache_ex_valid_q;
    logic [riscv::VLEN-1:0] icache_vaddr_q;
```

### 1.2.2  指令queue信号寄存器

ready信号表示queue已经可以接收指令；consumed信号表示queue里的指令已经发出。

INSTR_PER_FETCH表示每次取指令的条数。

```
    logic                   instr_queue_ready;
    logic [ariane_pkg::INSTR_PER_FETCH-1:0] instr_queue_consumed;
```

### 1.2.3  上一周期的分支预测

```
    // upper-most branch-prediction from last cycle
    btb_prediction_t        btb_q;
    bht_prediction_t        bht_q;
```

### 1.2.4  fetch阶段就绪信号和PC

需要注意的是，前端是分为generate-PC和fetch两个阶段的。if_ready 和 npc指示fetch阶段。

```
// instruction fetch is ready
    logic                   if_ready;
    logic [riscv::VLEN-1:0] npc_d, npc_q; // next PC
```

### 1.2.5  指令重新执行

如果分支预测失败，那么需要从特定的地址开始取指令。

```
    logic                   replay;
    logic [riscv::VLEN-1:0] replay_addr;
```

### 1.2.6  指令扫描(instr scan)

instr scan是在取指阶段进行的指令初步解析，仅仅用于分析指令是否是分支跳转类型的指令，以用于分支预测。rvi和rvc分别表示对普通指令和对压缩指令的的分析结果。

```
// RVI ctrl flow prediction
    logic [INSTR_PER_FETCH-1:0]       rvi_return, rvi_call, rvi_branch,
    									rvi_jalr, rvi_jump;
    logic [INSTR_PER_FETCH-1:0][riscv::VLEN-1:0] rvi_imm;
// RVC branching
    logic [INSTR_PER_FETCH-1:0]       rvc_branch, rvc_jump, rvc_jr, rvc_return,
    									rvc_jalr, rvc_call;
    logic [INSTR_PER_FETCH-1:0][riscv::VLEN-1:0] rvc_imm;
```

### 1.2.7  重新对齐指令

从指令cache中获取的数据需要进一步进行重新对齐操作，因为存在压缩指令。

```
// re-aligned instruction and address (coming from cache - combinationally)
    logic [INSTR_PER_FETCH-1:0][31:0] instr;
    logic [INSTR_PER_FETCH-1:0][riscv::VLEN-1:0] addr;
    logic [INSTR_PER_FETCH-1:0]       instruction_valid;
```

### 1.2.8  预测结果

加上shifted表示分支预测的结果已经进行的对齐操作。

```
// BHT, BTB and RAS prediction
    bht_prediction_t [INSTR_PER_FETCH-1:0] bht_prediction;
    btb_prediction_t [INSTR_PER_FETCH-1:0] btb_prediction;
    bht_prediction_t [INSTR_PER_FETCH-1:0] bht_prediction_shifted;
    btb_prediction_t [INSTR_PER_FETCH-1:0] btb_prediction_shifted;
    ras_t            ras_predict;
```

### 1.2.9  更新预测器

在分支预测失败等情况下，需要对RAS或者PHT分支历史表进行更新。

```
// branch-predict update
    logic            is_mispredict;
    logic            ras_push, ras_pop;
    logic [riscv::VLEN-1:0]     ras_update;
```

### 1.2.10  传递到queue的预测结果

指令queue除了指令之外，还需要记录指令是否是预测执行的。

```
// Instruction FIFO
    logic [riscv::VLEN-1:0]                 predict_address;
    cf_t  [ariane_pkg::INSTR_PER_FETCH-1:0] cf_type;
    logic [ariane_pkg::INSTR_PER_FETCH-1:0] taken_rvi_cf;
    logic [ariane_pkg::INSTR_PER_FETCH-1:0] taken_rvc_cf;
```

## 1.3 信号传递

### 1.3.1 选择正确的分支预测结果

需要避免分支预测的地址是非对齐的，因此，如果预测非对齐，那么就采取之前预测的结果。

```
// select the right branch prediction result
// in case we are serving an unaligned instruction in instr[0] we need to take
// the prediction we saved from the previous fetch
    assign bht_prediction_shifted[0] = (serving_unaligned) ? bht_q : bht_prediction[0]; 
    assign btb_prediction_shifted[0] = (serving_unaligned) ? btb_q : btb_prediction[0];
// for all other predictions we can use the generated address to index
// into the branch prediction data structures
    for (genvar i = 1; i < INSTR_PER_FETCH; i++) begin : gen_prediction_address
    assign bht_prediction_shifted[i] = bht_prediction[addr[i][$clog2(INSTR_PER_FETCH):1]];
    assign btb_prediction_shifted[i] = btb_prediction[addr[i][$clog2(INSTR_PER_FETCH):1]];
    end
// for the return address stack it doens't matter as we have the
// address of the call/return already
```

### 1.3.2  判断分支指令

如果指令是有效的，并且普通指令或者压缩指令有其一是某分支指令即可。

```
    logic [INSTR_PER_FETCH-1:0] is_branch;
    logic [INSTR_PER_FETCH-1:0] is_call;
    logic [INSTR_PER_FETCH-1:0] is_jump;
    logic [INSTR_PER_FETCH-1:0] is_return;
    logic [INSTR_PER_FETCH-1:0] is_jalr;

    for (genvar i = 0; i < INSTR_PER_FETCH; i++) begin
      // branch history table -> BHT
      assign is_branch[i] =  instruction_valid[i] & (rvi_branch[i] | rvc_branch[i]);
      // function calls -> RAS
      assign is_call[i] = instruction_valid[i] & (rvi_call[i] | rvc_call[i]);
      // function return -> RAS
      assign is_return[i] = instruction_valid[i] & (rvi_return[i] | rvc_return[i]);
      // unconditional jumps with known target -> immediately resolved
      assign is_jump[i] = instruction_valid[i] & (rvi_jump[i] | rvc_jump[i]);
      // unconditional jumps with unknown target -> BTB
      assign is_jalr[i] = instruction_valid[i] & ~is_return[i] & ~is_call[i] & (rvi_jalr[i] | rvc_jalr[i] | rvc_jr[i]);
    end
```

### 1.3.3  判断是否跳转

这部分是主要的功能实现函数。实现了对taken_rvi_cf、taken_rvc_cf、predict_address、 cf_type、ras_push、ras_pop、ras_update的赋值。

**always_comb**实现了逻辑电路的循环执行，不需要触发条件。循环开始的时候初始化这些变量。然后，针对不同的分支条件branch、return、jump、jalrs更新变量。如果是寄存器跳转，则根据btb，如果btb预测是有效的，那么就跳转到相应的地址；如果是无条件跳转，则直接jump到立即数；如果是return函数返回，则如果ras部件中预测跳转有效，并且现在这条指令已经被执行了，则可以根据ras部件中预测的进行跳转。如果是branch，那么，若预测有效那么，采取动态预测，如果预测无效，则采取静态预测（取立即数的最高位表示正负，如果为负数，则进行采取静态预测）。如果是call，并且这么地址确实被执行了，那么就保存这个被调用的地址（用于下次预测）。

在此需要注意的是，unique关键字指明遍历的几种情况只可以符合其中之一。

```
  // taken/not taken
    always_comb begin
      taken_rvi_cf = '0;
      taken_rvc_cf = '0;
      predict_address = '0;

      for (int i = 0; i < INSTR_PER_FETCH; i++)  cf_type[i] = ariane_pkg::NoCF;

      ras_push = 1'b0;
      ras_pop = 1'b0;
      ras_update = '0;

      // lower most prediction gets precedence
      for (int i = INSTR_PER_FETCH - 1; i >= 0 ; i--) begin
        unique case ({is_branch[i], is_return[i], is_jump[i], is_jalr[i]})
          4'b0000:; // regular instruction e.g.: no branch
          // unconditional jump to register, we need the BTB to resolve this
          4'b0001: begin
            ras_pop = 1'b0;
            ras_push = 1'b0;
            if (btb_prediction_shifted[i].valid) begin
              predict_address = btb_prediction_shifted[i].target_address;
              cf_type[i] = ariane_pkg::JumpR;
            end
          end
          // its an unconditional jump to an immediate
          4'b0010: begin
            ras_pop = 1'b0;
            ras_push = 1'b0;
            taken_rvi_cf[i] = rvi_jump[i];
            taken_rvc_cf[i] = rvc_jump[i];
            cf_type[i] = ariane_pkg::Jump;
          end
          // return
          4'b0100: begin
            // make sure to only alter the RAS if we actually consumed the instruction
            ras_pop = ras_predict.valid & instr_queue_consumed[i];
            ras_push = 1'b0;
            predict_address = ras_predict.ra;
            cf_type[i] = ariane_pkg::Return;
          end
          // branch prediction
          4'b1000: begin
            ras_pop = 1'b0;
            ras_push = 1'b0;
            // if we have a valid dynamic prediction use it
            if (bht_prediction_shifted[i].valid) begin
              taken_rvi_cf[i] = rvi_branch[i] & bht_prediction_shifted[i].taken;
              taken_rvc_cf[i] = rvc_branch[i] & bht_prediction_shifted[i].taken;
            // otherwise default to static prediction
            end else begin
              // set if immediate is negative - static prediction
              taken_rvi_cf[i] = rvi_branch[i] & rvi_imm[i][riscv::VLEN-1];
              taken_rvc_cf[i] = rvc_branch[i] & rvc_imm[i][riscv::VLEN-1];
            end
            if (taken_rvi_cf[i] || taken_rvc_cf[i]) cf_type[i] = ariane_pkg::Branch;
          end
          default:;
            // default: $error("Decoded more than one control flow");
        endcase
          // if this instruction, in addition, is a call, save the resulting address
          // but only if we actually consumed the address
          if (is_call[i]) begin
            ras_push = instr_queue_consumed[i];
            ras_update = addr[i] + (rvc_call[i] ? 2 : 4);
          end
          // calculate the jump target address
          if (taken_rvc_cf[i] || taken_rvi_cf[i]) begin
            predict_address = addr[i] + (taken_rvc_cf[i] ? rvc_imm[i] : rvi_imm[i]);
          end
      end
    end
```

### 1.3.4  判断分支预测是否有效

如果各种分支预测有其中一个有效，则把分支预测有效信号置位有效。return是特殊情况，需要预测正确才可以。

```
    // or reduce struct
    always_comb begin
      bp_valid = 1'b0;
      // BP cannot be valid if we have a return instruction and the RAS is not giving a valid address
      // Check that we encountered a control flow and that for a return the RAS 
      // contains a valid prediction.
      for (int i = 0; i < INSTR_PER_FETCH; i++) bp_valid |= ((cf_type[i] != NoCF & cf_type[i] != Return) | ((cf_type[i] == Return) & ras_predict.valid));
    end
    assign is_mispredict = resolved_branch_i.valid & resolved_branch_i.is_mispredict;
```

### 1.3.5  cache接口

当指令queue准备好了之后，就可以开始访问（发出访问请求）。当预测错误、需要刷新前端、因为指令cache满了而停止取值三种情况下，会刷新cache（第一级）流水线。如果刷新了（第一级）或者分支预测正确的时候，则需要刷新第二阶段cache流水线。

```
    // Cache interface
    assign icache_dreq_o.req = instr_queue_ready;
    assign if_ready = icache_dreq_i.ready & instr_queue_ready;
    // We need to flush the cache pipeline if:
    // 1. We mispredicted
    // 2. Want to flush the whole processor front-end
    // 3. Need to replay an instruction because the fetch-fifo was full
    assign icache_dreq_o.kill_s1 = is_mispredict | flush_i | replay;
    // if we have a valid branch-prediction we need to only kill the last cache request
    // also if we killed the first stage we also need to kill the second stage (inclusive flush)
    assign icache_dreq_o.kill_s2 = icache_dreq_o.kill_s1 | bp_valid;
```

### 1.3.6  更新BTB和BHT

根据来自控制器的信号（resolved_branch_i）判断分支预测的正确与否，以更新BTB和BHT的信息。

```
    // Update Control Flow Predictions
    bht_update_t bht_update;
    btb_update_t btb_update;

    // assert on branch, deassert when resolved
    logic speculative_q,speculative_d;
    assign speculative_d = (speculative_q && !resolved_branch_i.valid || |is_branch || |is_return || |is_jalr) && !flush_i;
    assign icache_dreq_o.spec = speculative_d;

    assign bht_update.valid = resolved_branch_i.valid
                                & (resolved_branch_i.cf_type == ariane_pkg::Branch);
    assign bht_update.pc    = resolved_branch_i.pc;
    assign bht_update.taken = resolved_branch_i.is_taken;
    // only update mispredicted branches e.g. no returns from the RAS
    assign btb_update.valid = resolved_branch_i.valid
                                & resolved_branch_i.is_mispredict
                                & (resolved_branch_i.cf_type == ariane_pkg::JumpR);
    assign btb_update.pc    = resolved_branch_i.pc;
    assign btb_update.target_address = resolved_branch_i.target_address;
```

### 1.3.7  确定next PC

根据PC的6个来源，确定PC

```
    // -------------------
    // Next PC
    // -------------------
    // next PC (NPC) can come from (in order of precedence):
    // 0. Default assignment/replay instruction
    // 1. Branch Predict taken
    // 2. Control flow change request (misprediction)
    // 3. Return from environment call
    // 4. Exception/Interrupt
    // 5. Pipeline Flush because of CSR side effects
    // Mis-predict handling is a little bit different
    // select PC a.k.a PC Gen
    always_comb begin : npc_select
      automatic logic [riscv::VLEN-1:0] fetch_address;
      // check whether we come out of reset
      // this is a workaround. some tools have issues
      // having boot_addr_i in the asynchronous
      // reset assignment to npc_q, even though
      // boot_addr_i will be assigned a constant
      // on the top-level.
      if (npc_rst_load_q) begin
        npc_d         = boot_addr_i;
        fetch_address = boot_addr_i;
      end else begin
        fetch_address    = npc_q;
        // keep stable by default
        npc_d            = npc_q;
      end
      // 0. Branch Prediction
      if (bp_valid) begin
        fetch_address = predict_address;
        npc_d = predict_address;
      end
      // 1. Default assignment
      if (if_ready) npc_d = {fetch_address[riscv::VLEN-1:2], 2'b0}  + 'h4;
      // 2. Replay instruction fetch
      if (replay) npc_d = replay_addr;
      // 3. Control flow change request
      if (is_mispredict) npc_d = resolved_branch_i.target_address;
      // 4. Return from environment call
      if (eret_i) npc_d = epc_i;
      // 5. Exception/Interrupt
      if (ex_valid_i) npc_d = trap_vector_base_i;
      // 6. Pipeline Flush because of CSR side effects
      // On a pipeline flush start fetching from the next address
      // of the instruction in the commit stage
      // we came here from a flush request of a CSR instruction or AMO,
      // as CSR or AMO instructions do not exist in a compressed form
      // we can unconditionally do PC + 4 here
      // TODO(zarubaf) This adder can at least be merged with the one in the csr_regfile stage
      if (set_pc_commit_i) npc_d = pc_commit_i + {{riscv::VLEN-3{1'b0}}, 3'b100};
      // 7. Debug
      // enter debug on a hard-coded base-address
      if (set_debug_pc_i) npc_d = ArianeCfg.DmBaseAddress[riscv::VLEN-1:0] + dm::HaltAddress[riscv::VLEN-1:0];
      icache_dreq_o.vaddr = fetch_address;
    end
```

### 1.3.8  更新指令cache信号寄存器

根据来自cache的信号，对寄存器进行赋值。

```
always_ff @(posedge clk_i or negedge rst_ni) begin
      if (!rst_ni) begin
        npc_rst_load_q    <= 1'b1;
        npc_q             <= '0;
        speculative_q     <= '0;
        icache_data_q     <= '0;
        icache_valid_q    <= 1'b0;
        icache_vaddr_q    <= 'b0;
        icache_ex_valid_q <= ariane_pkg::FE_NONE;
        btb_q             <= '0;
        bht_q             <= '0;
      end else begin
        npc_rst_load_q    <= 1'b0;
        npc_q             <= npc_d;
        speculative_q    <= speculative_d;
        icache_valid_q    <= icache_dreq_i.valid;
        if (icache_dreq_i.valid) begin
          icache_data_q        <= icache_data;
          icache_vaddr_q       <= icache_dreq_i.vaddr;
          // Map the only three exceptions which can occur in the frontend to a two bit enum
          if (icache_dreq_i.ex.cause == riscv::INSTR_PAGE_FAULT) begin
            icache_ex_valid_q <= ariane_pkg::FE_INSTR_PAGE_FAULT;
          end else if (icache_dreq_i.ex.cause == riscv::INSTR_ACCESS_FAULT) begin
            icache_ex_valid_q <= ariane_pkg::FE_INSTR_ACCESS_FAULT;
          end else icache_ex_valid_q <= ariane_pkg::FE_NONE;
          // save the uppermost prediction
          btb_q                <= btb_prediction[INSTR_PER_FETCH-1];
          bht_q                <= bht_prediction[INSTR_PER_FETCH-1];
        end
      end
    end
```

## 1.4 例化