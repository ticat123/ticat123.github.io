:encoding:utf-8

# 记录一次关于 LoongArch gcc-12 tree-optimization 的优化分析 [_记录一次关于_loongarch_gcc_12_tree_optimization_的优化分析]

樊鹏 — 1.0, 2024-09-09

环境信息:

-   系统： euler-2403

-   工具链版本：gcc-12.3, bintuils-2.41

## 问题概述： [_问题概述]

报错：

在上述环境中，p7zip 在 O1 优化级别时出现段错误。其它优化级下别均正常。

复现：

``` bash
wget https://www.7-zip.org/a/lzma2408.7z
make -C CPP/7zip/Bundles/Alone all
./bin/7za x lzma2408.7z -r -o./t1
```

由于C++的代码编译时会产生大量其他调用，导致难以定位分析，因此首先简化测试用例。当然，依据现有报错的测试，可以 构建简单的新测试也是可行的，但是这种需要对原理清楚才能进行，否则构建测试本身就相当麻烦了。这里处于时间考虑，只是 简化了测试用例。

简化后测试用例

``` cpp
// Bcj2Coder.cpp
#include "StdAfx.h"
#include "../../../C/Alloc.h"
#include "../Common/StreamUtils.h"
#include "Bcj2Coder.h"
namespace NCompress {
namespace NBcj2 {
CBaseCoder::CBaseCoder()
{
  for (int i = 0; i < BCJ2_NUM_STREAMS + 1; i++)
  {
    _bufs[i] = NULL;
    _bufsCurSizes[i] = 0;
    _bufsNewSizes[i] = (1 << 18);
  }
}

CBaseCoder::~CBaseCoder()
{
}

CDecoder::CDecoder(): _finishMode(false), _outSizeDefined(false), _outSize(0)
{}

}}
```

## 问题定位 [_问题定位]

段错误其实是最好分析的，目前我遇到的主要是以下几类情况会触发段错误：

-   访问非法内存地址，比如read/write from/to (void \*)0x0

-   内存权限错误：这种比较少见，之前调试一个关于 glibc-ifunc 的问题，对于 IFUNC 而言，会在运行时修改 .text 部分， 正常的话，大多数情况下，.text 部分基本都是read-only的，而在该版本的glibc中，由于修改之前未先将相应的内存权限 修改为可写，导致后续写操作引发段错误。

### GDB 分析段错误的具体位置，确定段错误类型 [_gdb_分析段错误的具体位置确定段错误类型]

``` cpp
#0  0x00007ffff7b25754 in free () from /usr/lib64/libc.so.6
#1  0x00005555555718bc in align_free (ptr=<optimized out>) at ../../../../C/Alloc.c:66
#2  VirtualFree (address=<optimized out>) at ../../../../C/Alloc.c:185
#3  0x0000555555571858 in MidFree (address=0x400040004000400) at ../../../../C/Alloc.c:211
#4  0x00005555555ef594 in NCompress::NBcj2::CBaseCoder::Alloc (this=this@entry=0x5555556cf558, allocForOrig=allocForOrig@entry=true) at ../../../../CPP/7zip/Compress/Bcj2Coder.cpp:46
#5  0x00005555555efc9c in NCompress::NBcj2::CDecoder::Code (this=0x5555556cf520, inStreams=0x5555556d2580, inSizes=0x5555556cbfa0, numInStreams=<optimized out>, outStreams=0x5555556d25b0,
    outSizes=0x5555556cf9a0, numOutStreams=<optimized out>, progress=0x5555556c9390) at ../../../../CPP/7zip/Compress/Bcj2Coder.cpp:365
...

发生在free的时候，显而易见，释放了非法内存块。
```

### 初步分析源码，确定非法地址来源 [_初步分析源码确定非法地址来源]

``` cpp
CPP/7zip/Compress/Bcj2Coder.cpp：

HRESULT CBaseCoder::Alloc(bool allocForOrig)
{
  unsigned num = allocForOrig ? BCJ2_NUM_STREAMS + 1 : BCJ2_NUM_STREAMS;
  for (unsigned i = 0; i < num; i++)
  {
    UInt32 newSize = _bufsNewSizes[i];
    const UInt32 kMinBufSize = 1;
    if (newSize < kMinBufSize)
      newSize = kMinBufSize;
    if (!_bufs[i] || newSize != _bufsCurSizes[i])
    {
      if (_bufs[i])
      {
        ::MidFree(_bufs[i]);    /* 就是这个地方 */
        _bufs[i] = 0;
      }
      _bufsCurSizes[i] = 0;
      Byte *buf = (Byte *)::MidAlloc(newSize);
      _bufs[i] = buf;
      if (!buf)
        return E_OUTOFMEMORY;
      _bufsCurSizes[i] = newSize;
    }
  }
  return S_OK;
}
```

结合该文件，以及根据我对C++的粗浅认知，应该是构造函数：

``` cpp
CBaseCoder::CBaseCoder()
{
  for (int i = 0; i < BCJ2_NUM_STREAMS + 1; i++)
  {
    _bufs[i] = NULL;
    _bufsCurSizes[i] = 0;
    _bufsNewSizes[i] = (1 << 18);
  }
}
```

'\_bufs\[i\] = NULL;' 该语句没有执行的原因，造成分配的内存块没有被清零，结果代码的检查逻辑会free 非零内存块。

## 机理分析 [_机理分析]

在上一步的分析之后，可以确定出问题的现象，接下来需要确认上面的结论和汇编代码是否一致。

``` asm
Disassembly of section .text:

0000000000000000 <NCompress::NBcj2::CBaseCoder::CBaseCoder()>:
   0:   02c0a08c        addi.d          $t0, $a0, 40
   4:   02c0f08f        addi.d          $t3, $a0, 60
   8:   00119004        sub.d           $a0, $zero, $a0
   c:   1400080e        lu12i.w         $t2, 64

0000000000000010 <.L2>:
  10:   002c118d        alsl.d          $t1, $t0, $a0, 0x1
  14:   29fec1a0        st.d            $zero, $t1, -80
  18:   25000180        stptr.w         $zero, $t0, 0
  1c:   2980518e        st.w            $t2, $t0, 20
  20:   02c0118c        addi.d          $t0, $t0, 4
  24:   5fffed8f        bne             $t0, $t3, -20   # 10 <.L2>
  28:   4c000020        ret

000000000000002c <NCompress::NBcj2::CBaseCoder::~CBaseCoder()>:
  2c:   4c000020        ret

0000000000000030 <NCompress::NBcj2::CDecoder::CDecoder()>:
  30:   2980c080        st.w            $zero, $a0, 48
  34:   1a00000c        pcalau12i       $t0, 0
  38:   28c0018c        ld.d            $t0, $t0, 0
  3c:   02c0418d        addi.d          $t1, $t0, 16
  40:   2700008d        stptr.d         $t1, $a0, 0
  44:   02c2218d        addi.d          $t1, $t0, 136
  48:   29c0208d        st.d            $t1, $a0, 8
  4c:   02c3218d        addi.d          $t1, $t0, 200
  50:   29c0408d        st.d            $t1, $a0, 16
  54:   02c4418d        addi.d          $t1, $t0, 272
  58:   29c0608d        st.d            $t1, $a0, 24
  5c:   02c5418d        addi.d          $t1, $t0, 336
  60:   29c0808d        st.d            $t1, $a0, 32
  64:   02c6418c        addi.d          $t0, $t0, 400
  68:   29c0a08c        st.d            $t0, $a0, 40
  6c:   29c32080        st.d            $zero, $a0, 200
  70:   29c34080        st.d            $zero, $a0, 208
  74:   29c36080        st.d            $zero, $a0, 216
  78:   29c38080        st.d            $zero, $a0, 224
  7c:   2903a080        st.b            $zero, $a0, 232
  80:   2903a480        st.b            $zero, $a0, 233
  84:   29c3c080        st.d            $zero, $a0, 240
  88:   4c000020        ret
```

查看该文件的反汇编代码，发现，NCompress::NBcj2::CDecoder::CDecoder() 函数并没有调用NCompress::NBcj2::CBaseCoder::CBaseCoder() 构造函数，而且也没有明显的内联痕迹。因此怀疑源码中实际的调用语句是被删除了的。

同时，对比O0和O2，发现O2会内联展开，O0则是调用跳转。而O1则是既无内联展开，又无调用跳转。此时，基本可以确认就是该调用被GCC给删除了。

### PASS 对比 [_pass_对比]

首先，dump-tree，发现 tree-dce2 的输出中无 CALL CBaseCoder::CBaseCoder() 语义的相关语句。通过详细的信息查看，发现该 PASS 中有关于 该调用语句的消除操作：

    Eliminating unnecessary statements:
    Deleting : NCompress::NBcj2::CBaseCoder::CBaseCoder (_1);
    Deleting : _1 = &this_19(D)->D.27959;

因此，接下来分析该 PASS, 为何对于该条 Gimple 语句标记为无用。

tree-dce2 的关键调用：

``` cpp
perform_tree_ssa_dce()->
    find_obviously_necessary_stmts()->
        Find obviously necessary statements.  These are things like most function calls, and stores to file level variables.
        mark_stmt_if_obviously_necessary()->mark_stmt_necessary()

    ->eliminate_unnecessary_stmts()
        Eliminate unnecessary statements. Any instruction not marked as necessary  contributes nothing to the program, and can be deleted.
```

其实，不难发现，这种策略分析时间较长，因此，此时决定反向分析：根据 Gimple 的基本概念，该 PASS 一定是对函数，然后逐级为基本块、语句进行分析。 因此，可以分析其为何在判断 CALL CBaseCoder::CBaseCoder() 是将其标记为无用，或者说未标记为有用。

进一步分析该 PASS，发现： mark_stmt_if_obviously_necessary() 在判断相应的 Gimple 语句时，gimple_has_side_effects () 返回结果为 False，从而未标记为 ‘useful stmt’ ,后面 对于 ‘unuseful stmt’ 就会进行消除。该函数是判断对应语句是否存在副作用。GOON，发现该语句的标记位多了一个 ECF_CONST，它的含义是无副作用的 函数调用（或者没有读取全局内存）标记。显然，从语义上看，这个标记是不对的。对应的是函数声明的 TREE_READONLY。此时，可以猜到，这个标记是 错误的，而且早于该 PASS 就已经标记错误了。

tree-fixup_cfg3 阶段删除了一个内存引用的语句：.MEM_21 = VDEF \<.MEM_35\>。有点可疑，冥冥之中，和报错现象有所关联。

其实，这个时候已经有些许分析不动了，除了上述的分析外，我还分析了 tree-dse2，该 PASS 是消除 dead-store 语句的。严重怀疑是不是分析方向出错了。 花了半天时间，将之前的分析梳理了一遍，确定这些阶段都没有问题，而是之前的阶段就已经出错了。至此，继续向前推进。功夫不负有心人，

tree-modref2 阶段让我发现了可疑的地方：

``` cpp
void modref_access_analysis::analyze ()
{
  m_ecf_flags = flags_from_decl_or_type (current_function_decl);
  bool summary_useful = true;

  basic_block bb;
  bitmap always_executed_bbs = find_always_executed_bbs (cfun, true);
  FOR_EACH_BB_FN (bb, cfun)
    {
      gimple_stmt_iterator si;
      bool always_executed = bitmap_bit_p (always_executed_bbs, bb->index);

      for (si = gsi_start_nondebug_after_labels_bb (bb);
       !gsi_end_p (si); gsi_next_nondebug (&si))
    {

      if (infer_nonnull_range_by_dereference (gsi_stmt (si),
                          null_pointer_node))
        {
          if (dump_file)
        fprintf (dump_file, " - NULL memory access; terminating BB\n");
          if (flag_non_call_exceptions)
        set_side_effects ();
          break;    /* 在分析内存引用对应的语句时，退出了，直接进入了下一条语句的分析。正常应该调用之后的 anylyze_stmt() */
        }
      analyze_stmt (gsi_stmt (si), always_executed);
...
```

分析 infer_nonnull_range_by_dereference()，walk_stmt_load_store_ops() 的一个回调函数 check_loadstore() 执行略显奇怪, 返回 True，从而在 modref_access_analysis::analyze () 的循环中 break 退出。

check_loadstore() 会检查对指针类型的接引用，正常对应的节点类型应该是 MEM_REF，而此时为 TARGET_MEM_REF。

MEM_REF：表示一个由指针+偏移指向的对象。

TARGET_MEM_REF：表示一个内存访问的操作。具体的地址计算方式为：TMR_BASE + TMR_OFFSET + TMR_INDEX \* TMR_STEP + TMR_INDEX2

### create_mem_ref() & create_mem_ref_raw() [_create_mem_ref_create_mem_ref_raw]

接下来检查 TARGET_MEM_REF 节点的来源。 rewrite_use_address() → create_mem_ref() 调用 create_mem_ref_raw()，返回一个 tree mem_ref，其类型为 TARGET_MEM_REF。 最终生成的 Gimple 语句为： MEM\[(Byte \* \*)0B + \_19 + \_18 \* 1\] = 0B;

MEM\[(Byte \* \*)0B + \_19 + \_18 \* 1\] 节点就是刚刚上面的调用链生成的 TARGET_MEM_REF 节点。

至此，从源头到出错现象，基本梳理完毕。

那么，为何在该场景下，生成 TARGET_MEM_REF 节点就会导致函数调用语句被删除而报错呢？

对与一条简单的C语句, 这个转换过程对于GCC等编译器而言是一个漫长而复杂的过程。对于该问题，一个调用，最重要的就是确定其地址。 \_buf\[i\] = NULL; 的操作同样，对某个内存写入，那么内存地址从何而来。

编译器并不知道什么地址，什么是常量，需要经过一系列的分析、标记、组合。才能将其转换为自己的形式，最后才是汇编代码。同时又要 确保形式不同，语义相同。

create_mem_ref 发生在 pass_iv_optimize，简单来说就是变量优化，比如消除无用变量。

可以看到，在该场景下，这个地址引用的 base 是 0，实际的真实地址应该是保存在 index2 中的，那么是不是可以换个类型，换个计算方式？ 很容易想到，和 MEM_REF 的区别。

此时，对比看了下高版本的代码，发现，高版本 GCC 中 rewrite_use_address() 中调用 create_mem_ref() 之后，会将 TARGET_MEM_REF 类型的节点转换为 MEM_REF。 commit: 13dfb01e5c30c3bd09333ac79d6ff96a617fea67

好吧，看了下记录，人家就是解决 IVOPT 中的 zero-based memory reference 情景的。这个 GCC 的中间形式真是好复杂啊，虽然将出错的地方 和来源联系了起来，但是对其中间形式并不熟练。

``` cpp
_21 = &MEM[(Byte * *)0B + _20 + _19 * 1];
*_21 = 0B;
After:
MEM[(Byte * *)0B + _19 + _18 * 1] = 0B;
```

## 总结 [_总结]

对比X86架构，发现可以正常生成调用指令，发现X86后端会对此类情况进行处理，对地址形式进行转换。 这一块的逻辑还是比较麻烦的，尤其是前面 PASS 的分析结果会影响后面若干 PASS 的执行，同时虽然是前端的分析，但是又涉及了后端的特性支持。 GCC 的中间形式还是挺复杂的，也是借着本次分析，对 ipa 阶段中的一两个 PASS 有了一个较为全面的认识。

当然了，还是存在诸多不足，主要表现 为对其处理逻辑并不熟悉，中间形式的转换逻辑更加不熟悉。但是回过头发现，这种设计、实现，恰恰就是对常见的编译原理的具体实现。
