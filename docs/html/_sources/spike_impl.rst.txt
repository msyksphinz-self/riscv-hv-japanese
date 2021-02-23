Spikeの実装を確認する。

Hypervisorをサポートするメンバ変数V
-----------------------------------

現在仮想モードで動いているかどうかを示す仮想モードVを示すのに、メンバ変数\ ``v``\ が定義されている。

-  ``riscv-isa-sim/riscv/processor.h``

.. code:: cpp

   // architectural state of a RISC-V hart
   struct state_t
   {
     void reset(reg_t max_isa);
   ...
     // control and status registers
     reg_t prv;    // TODO: Can this be an enum instead?
     bool v;       // これが仮想モードVを示す変数

Vが設定される条件
~~~~~~~~~~~~~~~~~

ビットVは以下の条件にて有効・無効化される。\ ``set_virt()``\ を呼び出すことによりVが設定されるのだが、\ ``set_virt()``\ を呼び出しているのは以下の部分だ。

-  ``riscv-isa-sim/riscv/processor.cc``

.. code:: cpp

   void processor_t::set_virt(bool virt)
   {
     reg_t tmp, mask;

     if (state.prv == PRV_M)
       return;
   /* ... 中略 ... */
       state.mstatus = (state.mstatus & ~mask) | (state.vsstatus & mask);
       state.vsstatus = tmp;
       state.v = virt; // 仮想モードの設定
     }
   }

命令として呼び出される場所
^^^^^^^^^^^^^^^^^^^^^^^^^^

-  ``MRET``\ 命令：\ ``mstatus.MPV``\ で取得されるビットをもとにVモードを設定している。

.. code:: cpp

   require_privilege(PRV_M);
   set_pc_and_serialize(p->get_state()->mepc);
   reg_t s = STATE.mstatus;
   reg_t prev_prv = get_field(s, MSTATUS_MPP);
   reg_t prev_virt = get_field(s, MSTATUS_MPV);    // MSTATUS.MPVから過去の仮想モードを取得している
   s = set_field(s, MSTATUS_MIE, get_field(s, MSTATUS_MPIE));
   s = set_field(s, MSTATUS_MPIE, 1);
   s = set_field(s, MSTATUS_MPP, PRV_U);
   p->set_csr(CSR_MSTATUS, s);
   p->set_privilege(prev_prv);
   p->set_virt(prev_virt);         // 仮想モードの設定

-  ``SRET``\ 命令：\ ``HSTATUS.SPV``\ で取得されるフィールドをもとにVモードを設定している。

.. code:: cpp

   p->set_csr(CSR_MSTATUS, s);
   p->set_privilege(prev_prv);
   if (!STATE.v) {
     reg_t prev_virt = get_field(STATE.hstatus, HSTATUS_SPV);  // HSTATUS.SPVから過去の仮想モードを取得している
     p->set_virt(prev_virt);   // 仮想モードの設定
   }

例外として呼び出される場所
^^^^^^^^^^^^^^^^^^^^^^^^^^

-  割り込み・例外発生時

-  ``riscv-isa-sim/riscv/processor.cc``

.. code:: cpp

   void processor_t::take_trap(trap_t& t, reg_t epc)
   {
   /* ... 中略 ... */
     } else if (state.prv <= PRV_S && bit < max_xlen && ((hsdeleg >> bit) & 1)) {
       // 移譲ビットによりHSモードに割り込み・例外が移譲される場合
       // Handle the trap in HS-mode
       set_virt(false);        // HSモードへの以上なので仮想化モードに遷移しない
       reg_t vector = (state.stvec & 1) && interrupt ? 4*bit : 0;
       state.pc = (state.stvec & ~(reg_t)1) + vector;
       state.scause = t.cause();
       state.sepc = epc;
   /* ... 中略 ... */
     } else {
       // Handle the trap in M-mode
       // Mモードで割り込み・例外を処理するので仮想化モードに遷移しない
       set_virt(false);
       reg_t vector = (state.mtvec & 1) && interrupt ? 4*bit : 0;
       state.pc = (state.mtvec & ~(reg_t)1) + vector;

``set_virt()``\ の中身
~~~~~~~~~~~~~~~~~~~~~~

``set_virt()``\ の動作についてソースコードを読んでみる。

-  現在の動作モードがMである場合、特に何も処理しない。

   -  .. code:: cpp

             if (state.prv == PRV_M)
               return;

   -  ``MRET``/``SRET``\ が実行された場合：最初に\ ``set_priviledge()``\ により動作モードが設定される。Mモードに遷移された場合は仮想モードに遷移しない

   -  ``take_trap()``\ の場合：例外を受け取った時の動作モードがMモードの場合、仮想モードに遷移しない。

-  現在の仮想化モードと設定される仮想化モードが異なる場合：

   -  現在仮想化モードで実行中で、仮想化モードから抜ける場合

      -  FS/VS/XSモードを設定して、仮想化モードが実行中であることを明記しておく必要あり（これは何のために必要なんだ？マシンモードにてレジスタ退避とかに使用するのか…？）

      -  .. code:: cpp

              // VSSTATUS.FSがInitialではない：つま仮想化モードにてFPUを実行済みの場合かつ
              // MSTATUS.FSがDirtyの場合：VSSTATUS.FSもDirtyにしてしまう？
                    if ((state.vsstatus & SSTATUS_FS) &&
                        ((state.mstatus & SSTATUS_FS) == SSTATUS_FS)) {
                      state.vsstatus |= SSTATUS_FS;
                    }
                    if (supports_extension('V') &&
                        (state.vsstatus & SSTATUS_VS) &&
                        ((state.mstatus & SSTATUS_VS) == SSTATUS_VS)) {
                      state.vsstatus |= SSTATUS_VS;
                    }
                    if ((state.vsstatus & SSTATUS_XS) &&
                        ((state.mstatus & SSTATUS_XS) == SSTATUS_XS)) {
                      state.vsstatus |= SSTATUS_XS;
                    }

      -  SDビットも設定する

      -  .. code:: cpp

                    /* Update SD bit of Host */
                    state.vsstatus &= (xlen == 64 ? ~SSTATUS64_SD : ~SSTATUS32_SD);
                    if (((state.mstatus & SSTATUS_FS) == SSTATUS_FS) ||
                        ((state.vsstatus & SSTATUS_VS) == SSTATUS_VS) ||
                        ((state.vsstatus & SSTATUS_XS) == SSTATUS_XS)) {
                       state.vsstatus |= (xlen == 64 ? SSTATUS64_SD : SSTATUS32_SD);
                    }

-  最後にMSTATUSとVSSTATUSを設定する。この辺のマスクはどういう設定なんだ？

.. code:: cpp

       mask = SSTATUS_VS_MASK;
       mask |= (supports_extension('V') ? SSTATUS_VS : 0);
       mask |= (xlen == 64 ? SSTATUS64_SD : SSTATUS32_SD);
       tmp = state.mstatus & mask;
       state.mstatus = (state.mstatus & ~mask) | (state.vsstatus & mask);
       state.vsstatus = tmp;
       state.v = virt;
