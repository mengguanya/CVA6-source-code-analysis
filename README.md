# CVA6
CVA6是一款使用System Verilog编写的基于RISC-V指令集架构的六级流水按序单发射处理器，计划通过分析其代码，以达到学习SV语言的目的。

需要注意的是，本文档完全按照我的递归学习过程书写，因此可能会有些混乱，全部完成之后，我会重新整理文档。

- [ ] frontend
  - [x] frontend.sv
  - [ ] bht.sv
  - [ ] btb.sv
  - [ ] ras.sv
  - [ ] instr_queue.sv
  - [ ] instr_scan.sv
- [ ] scoreboard.sv
- [ ] decoder.sv
- [ ] load_store_unit

把文件在VIVADO 2020中打开可以观察到文件（模块）之间的调用关系。

![image-20210220211916924](D:\DesktopImage\MPRC\E-core\CVA6-source-code-analysis\image\ariane.png)