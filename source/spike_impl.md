Spikeの実装を確認する。

## Hypervisorをサポートするメンバ変数V

現在仮想モードで動いているかどうかを示す仮想モードVを示すのに、メンバ変数`v`が定義されている。

- `riscv-isa-sim/riscv/processor.h`

```cpp
// architectural state of a RISC-V hart
struct state_t
{
  void reset(reg_t max_isa);
...
  // control and status registers
  reg_t prv;    // TODO: Can this be an enum instead?
  bool v;		// これが仮想モードVを示す変数
```

### Vが設定される条件

ビットVは以下の条件にて有効・無効化される。`set_virt()`を呼び出すことによりVが設定されるのだが、`set_virt()`を呼び出しているのは以下の部分だ。

- `riscv-isa-sim/riscv/processor.cc`

```cpp
void processor_t::set_virt(bool virt)
{
  reg_t tmp, mask;

  if (state.prv == PRV_M)
    return;
/* ... 中略 ... */
    state.mstatus = (state.mstatus & ~mask) | (state.vsstatus & mask);
    state.vsstatus = tmp;
    state.v = virt;	// 仮想モードの設定
  }
}
```

#### 命令として呼び出される場所

- `MRET`命令：`mstatus.MPV`で取得されるビットをもとにVモードを設定している。

```cpp
require_privilege(PRV_M);
set_pc_and_serialize(p->get_state()->mepc);
reg_t s = STATE.mstatus;
reg_t prev_prv = get_field(s, MSTATUS_MPP);
reg_t prev_virt = get_field(s, MSTATUS_MPV);	// MSTATUS.MPVから過去の仮想モードを取得している
s = set_field(s, MSTATUS_MIE, get_field(s, MSTATUS_MPIE));
s = set_field(s, MSTATUS_MPIE, 1);
s = set_field(s, MSTATUS_MPP, PRV_U);
p->set_csr(CSR_MSTATUS, s);
p->set_privilege(prev_prv);
p->set_virt(prev_virt);			// 仮想モードの設定
```

- `SRET`命令：`HSTATUS.SPV`で取得されるフィールドをもとにVモードを設定している。

```cpp
p->set_csr(CSR_MSTATUS, s);
p->set_privilege(prev_prv);
if (!STATE.v) {
  reg_t prev_virt = get_field(STATE.hstatus, HSTATUS_SPV);	// HSTATUS.SPVから過去の仮想モードを取得している
  p->set_virt(prev_virt);	// 仮想モードの設定
}
```

#### 例外として呼び出される場所

- 割り込み・例外発生時

- `riscv-isa-sim/riscv/processor.cc`

```cpp
void processor_t::take_trap(trap_t& t, reg_t epc)
{
/* ... 中略 ... */
  } else if (state.prv <= PRV_S && bit < max_xlen && ((hsdeleg >> bit) & 1)) {
    // 移譲ビットによりHSモードに割り込み・例外が移譲される場合
    // Handle the trap in HS-mode
    set_virt(false);		// HSモードへの以上なので仮想化モードに遷移しない
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

```

### `set_virt()`の中身

`set_virt()`の動作についてソースコードを読んでみる。

- 現在の動作モードがMである場合、特に何も処理しない。

  - ```cpp
      if (state.prv == PRV_M)
        return;
    ```

  - `MRET`/`SRET`が実行された場合：最初に`set_priviledge()`により動作モードが設定される。Mモードに遷移された場合は仮想モードに遷移しない

  - `take_trap()`の場合：例外を受け取った時の動作モードがMモードの場合、仮想モードに遷移しない。

- 現在の仮想化モードと設定される仮想化モードが異なる場合：

  - 現在仮想化モードで実行中で、仮想化モードから抜ける場合

    - FS/VS/XSモードを設定して、仮想化モードが実行中であることを明記しておく必要あり（これは何のために必要なんだ？マシンモードにてレジスタ退避とかに使用するのか...？）

    - ```cpp
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
      ```

    - SDビットも設定する

    - ```cpp
            /* Update SD bit of Host */
            state.vsstatus &= (xlen == 64 ? ~SSTATUS64_SD : ~SSTATUS32_SD);
            if (((state.mstatus & SSTATUS_FS) == SSTATUS_FS) ||
                ((state.vsstatus & SSTATUS_VS) == SSTATUS_VS) ||
                ((state.vsstatus & SSTATUS_XS) == SSTATUS_XS)) {
               state.vsstatus |= (xlen == 64 ? SSTATUS64_SD : SSTATUS32_SD);
            }
      ```

- 最後にMSTATUSとVSSTATUSを設定する。この辺のマスクはどういう設定なんだ？

```cpp
    mask = SSTATUS_VS_MASK;
    mask |= (supports_extension('V') ? SSTATUS_VS : 0);
    mask |= (xlen == 64 ? SSTATUS64_SD : SSTATUS32_SD);
    tmp = state.mstatus & mask;
    state.mstatus = (state.mstatus & ~mask) | (state.vsstatus & mask);
    state.vsstatus = tmp;
    state.v = virt;
```

# 2ステージアドレス変換

仮想化モードが設定されている場合、RISC-Vのアドレス変換は2ステージを踏むことになっている。

1. VSレベルアドレス変換：**仮想アドレス** → **ゲスト物理アドレス**
2. Gレベルアドレス変換：**ゲスト物理アドレス** → **スーパーバイザー物理アドレス**

まずは**VSレベルアドレス変換**は

- ベースアドレスレジスタは`vsatp`で制御される

次は**Gレベルアドレス変換**は、

- ベースアドレス変換は`hgatp`で制御される

という前提条件を踏まえて、Spikeがどのようにアドレス変換を行っているのか探ってみる。

まずは基本的なロード命令から巡っていくことにする。`ld`命令の定義は以下でなされている。

- `riscv-isa-sim/riscv/insns/ld.h`

```cpp
require_rv64;
WRITE_RD(MMU.load_int64(RS1 + insn.i_imm()));
```

`MMU.load_int64()`という関数は実は`define`マクロで定義されていて、`mmu.h`の中を探しても見当たらないので最初は焦る。正しい場所はこちら。

- `riscv-isa-sim/riscv/mmu.h`

```cpp
  // template for functions that load an aligned value from memory
  #define load_func(type, prefix, xlate_flags) \
    inline type##_t prefix##_##type(reg_t addr, bool require_alignment = false) { \
      if ((xlate_flags) != 0) \
        flush_tlb(); \
      if (unlikely(addr & (sizeof(type##_t)-1))) { \
        if (require_alignment) load_reserved_address_misaligned(addr); \
/* ... 以下略 ... */

  // load value from memory at aligned address; zero extend to register width
  load_func(uint8, load, 0)
  load_func(uint16, load, 0)
  load_func(uint32, load, 0)
  load_func(uint64, load, 0)
      
  // load value from guest memory at aligned address; zero extend to register width
  load_func(uint8, guest_load, RISCV_XLATE_VIRT)
  load_func(uint16, guest_load, RISCV_XLATE_VIRT)
  load_func(uint32, guest_load, RISCV_XLATE_VIRT)
  load_func(uint64, guest_load, RISCV_XLATE_VIRT)
  load_func(uint16, guest_load_x, RISCV_XLATE_VIRT|RISCV_XLATE_VIRT_MXR)
  load_func(uint32, guest_load_x, RISCV_XLATE_VIRT|RISCV_XLATE_VIRT_MXR)

  // load value from memory at aligned address; sign extend to register width
  load_func(int8, load, 0)
  load_func(int16, load, 0)
  load_func(int32, load, 0)
  load_func(int64, load, 0)

  // load value from guest memory at aligned address; sign extend to register width
  load_func(int8, guest_load, RISCV_XLATE_VIRT)
  load_func(int16, guest_load, RISCV_XLATE_VIRT)
  load_func(int32, guest_load, RISCV_XLATE_VIRT)
  load_func(int64, guest_load, RISCV_XLATE_VIRT)
```

ここにきていくつかのロード命令のバリエーションがあることに気が付いた。`RISCV_XLATE_VIRT`とか、`guest_load`とかの意味は何だろう？`guest_load_int16()`とかは`HLV.H`命令で使用されているのは理解できる。

- `riscv-isa-sim/riscv/insns/hlv_h.h`

```cpp
require_extension('H');
require_novirt();
require_privilege(get_field(STATE.hstatus, HSTATUS_HU) ? PRV_U : PRV_S);
WRITE_RD(MMU.guest_load_int16(RS1));
```

とりあえず先に進もう。アドレス変換についてはMMUのPage Table Walkの部分に記述があるのでそれを眺めていく。

- `riscv-isa-sim/riscv/mmu.h`

```cpp
  // perform a page table walk for a given VA; set referenced/dirty bits
  reg_t walk(reg_t addr, access_type type, reg_t prv, bool virt, bool mxr);
```

- `riscv-isa-sim/riscv/mmu.cc`

```cpp
reg_t mmu_t::walk(reg_t addr, access_type type, reg_t mode, bool virt, bool mxr)
{
  reg_t page_mask = (reg_t(1) << PGSHIFT) - 1;
  reg_t satp = (virt) ? proc->get_state()->vsatp : proc->get_state()->satp;
  vm_info vm = decode_vm_info(proc->max_xlen, false, mode, satp);
  if (vm.levels == 0)
    return s2xlate(addr, addr & ((reg_t(2) << (proc->xlen-1))-1), type, type, virt, mxr) & ~page_mask; // zero-extend from xlen

```

まず、VSアドレスレベル変換についてだ。これは先ほど説明したようにVSアドレスレベル変換の場合はVSATPレジスタをベースアドレスとするので、その条件で`satp`レジスタの指定が切り替わっている。

`vm.info`についてはSATPレジスタのMODEレジスタの内容に応じてどの変換方式を採用するかを決定している。上記で言えば`vm.levels=0`の場合はBAREモード変換だと思われるので、アドレス変換は行わずそのまま仮想アドレスを物理アドレスとして取り扱っていることが分かる。

そこから先がアドレス変換のためのページジャンプで、これは`for`文でPTEを次々と呼び出しながらページをジャンプして最終的なゲスト物理アドレスに到達していることが分かる。

```cpp
  reg_t base = vm.ptbase;
  for (int i = vm.levels - 1; i >= 0; i--) {
    int ptshift = i * vm.idxbits;
    reg_t idx = (addr >> (PGSHIFT + ptshift)) & ((1 << vm.idxbits) - 1);

    // check that physical address of PTE is legal
/* ... 中略 ... */
```

最終的にゲスト物理アドレスに到達したら、おそらく`s2xlate()`によりゲスト物理アドレスからスーパバイザ物理アドレスへの変換を行っているものと思われる。

```cpp
/* ... 中略 ... */
      reg_t page_base = ((ppn & ~((reg_t(1) << napot_bits) - 1))
                        | (vpn & ((reg_t(1) << napot_bits) - 1))
                        | (vpn & ((reg_t(1) << ptshift) - 1))) << PGSHIFT;
      reg_t phys = page_base | (addr & page_mask);
      return s2xlate(addr, phys, type, type, virt, mxr) & ~page_mask;
    }
  }
```

おそらく正解だ。`s2xlate()`はベースアドレスをHGATPレジスタとしているアドレス変換で、変換方式そのものは通常のアドレス変換と大差ない。

```cpp
reg_t mmu_t::s2xlate(reg_t gva, reg_t gpa, access_type type, access_type trap_type, bool virt, bool mxr)
{
  if (!virt)
    return gpa;

  // ベースアドレスレジスタとしてHGATPを使用していることが分かる。
  vm_info vm = decode_vm_info(proc->max_xlen, true, 0, proc->get_state()->hgatp);
  if (vm.levels == 0)
    return gpa;
/* ... 中略 ... */
```

同じようにしてページテーブルのジャンプを繰り返していくことで、最終的な物理アドレスに到達していることが分かった。

```cpp
/* ... 中略 ... */
		break;

      reg_t page_base = ((ppn & ~((reg_t(1) << napot_bits) - 1))
                        | (vpn & ((reg_t(1) << napot_bits) - 1))
                        | (vpn & ((reg_t(1) << ptshift) - 1))) << PGSHIFT;
      return page_base | (gpa & page_mask);
    }
  }
```

