# 1 frondend.sv

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



# 2  bht.sv





