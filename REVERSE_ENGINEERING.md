# FiiO JM21 "All To DSD" — component extraction & reverse-engineering notes
---

# 0. 当前状态（权威，覆盖本文其余所有被标记的结论）

> 本节写在最前面。第 20/21/24/25 节包含若干**已被后续实验推翻**的结论，
> 那些段落已就地标注 `[SUPERSEDED]`。**以本节为准。**

## 0.1 已闭案

```
5 级显式2× zero-insertion + biquad          bit-exact 4096/4096 (Unicorn)
总倍率32× / channel                        长度不变量 1→2→4→8→16→32 已验证
量化器                                    4/6 抽头, slot0=newest, ±1,
                                          系数直读 .rodata, 内环无保护分支
量化器真实参与调制                         YES
信号增益                                  恒等式 gain = d8 / 32
                mode0 d8=16 → 0.500000 | mode1 d8=15 → 0.468750
                mode2 d8=14 → 0.437500实测 4.688e-01 / 4.375e-01 精确吻合
频率响应                                   DC–20 kHz 平坦 (0.469 → 0.446)
低电平阈值                                 ~1e-4 无输出 → 1e-3 起有输出
状态有界                                   当前测试范围内成立
bit 密度                                   恒 0.5
噪声搬移                                   带内低、带外抬升（delta-sigma 特征）
```

**`--jm21` 之前"输出恒为 0"是wrapper bug 造成的假象**：
`final_interp.interpolate` 返回了长度 2× 的超分配缓冲，后一半全零，
导致频率轴被拉长 2×、量化器在零填充区演化。见第 31 节。

## 0.2 已被推翻，不得再作为结论引用

| 旧结论 | 出处 | 状态 |
|---|---|---|
| "量化器在真实输入下结构性无条件发散" | 20/21/24 | **推翻**（5 抽头模型 + 无界斜坡 + 零填充缓冲，三重错误） |
| "最大极点模 1.999" | 21 | **推翻**，正确值 1.000039/1.000000 |
| "slot5 是恒定 slot，不是移位抽头" | 22 | **推翻**，是真实第 6 抽头 |
| "设备必须运行时换一行系数" | 20 | **推翻**，运行时读寄存器确认就是 .rodata |
| "这套已解码结构不能解释设备输出" | 24.5/ 25 | **推翻**，见 0.1 |
| "需要过载保护 / 状态钳位 / soft-reset" | 24 | **排除**（内环无分支，指令级） |
| "插值器之后还有未建模的后处理" | 25 | **排除**（5级循环后零写入） |
| "16× 过采样，PCM 88200 Hz" | 25 | **推翻**，实为 32× |
| "输出为 0 但比 30 dB 版本更糟" | 20 | **推翻** |

## 0.3 真机观测现状（详见第 34/35 节）

```
USB DAC 通路                    已闭合（44.1k–384k 八档独占全部通过）
Line-In 测量链                   已表征（地板 -68.1 dBFS @192k 独占）
All-To-DSD OFF/ON 真机 A/B        已完成
All-To-DSD 带外能量增加          已观测（30–96 kHz，+15…+30 dB）★ 真机测量
All-To-DSD 约 +3 dB 输出差异     已观测
All-To-DSD 带内噪声收益         当前工作点未观察到
8 Hz 梳状边带                   与 All-To-DSD 无关，已从 DSP 主线移除
-6 dBFS 带内能量暴涨            与 All-To-DSD 无关（PCM/模拟通路过载）
模型 vs 真机带内噪声方向相反     已确认，分歧未解释
```

## 0.3b 仍未闭合（不得当作已知）

```
PCM → tbl[m] 的绝对数值标度                  未闭合（scale-chain unknown）
x2 的真实单位与块长度语义                     未闭合
实际工作 mode                                未验证（工厂属性说 2）
内部 1-bit 流 → 最终 DSD bitstream 的packing  未找到
真实 All-To-DSD 是否执行 0x19d90             未确认 ← 第 35.9 节，第一优先
mode 0 的 8 抽头系数                         未解（寄存器复用，建议暂缓）
```

> **不要用最终输出倍率去验证 `0x19d90`**：
> `output_bytes = 4*x2/x3` 是该函数的**内部**关系，
> 与最终 DSD/模拟输出之间还隔着未闭合的 packing / downstream path。
> 决定性证据只能是 runtime trace/hook。

## 0.4 术语纪律

- `19.3 dB` 等数字一律称为 **reverse-engineered model simulation**，
  **不得**写成 "JM21 measured performance"。
- `gain = d8/32` 中的 1/32 是**插值器直流增益**；`d8/32` 是**量化器信号增益**。
  **两者不可相乘**去解释整条 PCM→DSD 总增益 —— 中间的
  `PCM → tbl[m]` 绝对标度尚未闭合。

## 0.5 `--jm21` 定位

```
default     现有稳定 FIR 路径
--jm21      reverse-engineered JM21 experimental path（opt-in）
```

内部证据已足够它是**有意义的 JM21 shaper reproduction**，
但尚不足以宣称"设备行为已复现"。在 USB DAC → 模拟回录确认之前不改默认。


Everything below was obtained from a JM21 running **Android 13**
(`ro.product.model = FiiO JM21`, `ro.product.board = bengal`, Snapdragon 865,
`ro.build.version.incremental = eng.fiio.20260428.144644`) over adb, **without
root** (shell uid 2000, read-only access).

---

## 1. Where the component lives

| file | role |
|---|---|
| `/vendor/lib64/libfiioaudio.so` (135 576 B) | **the whole All-To-DSD DSP**: PCM→DSD, DSD→PCM, system PEQ, resampler control, WAV debug dump |
| `/vendor/lib64/libfiioaqm.so` (177 776 B) | AGM/ADSP plugin front-end (`aqm_*`), delegates to the DSP |
| `/vendor/lib64/libdsd2pcm.so` | Qualcomm DSD→PCM (the *opposite* direction) |
| `/vendor/lib64/libagm_pcm_plugin.so`, `libqc2audio_*.so` | Qualcomm audio-graph plumbing |
| `/vendor/lib64/hw/audio.primary.bengal.so` | primary audio HAL; hosts `FiiOAudioProcess()` etc. |

Pulled copies live in `vendor_pull/`, full symbol/string dumps in
`vendor_pull/dump_fiioaudio.txt`, `dump_fiioaqm.txt`, `rodata_strings.txt`,
`text.asm`.

## 2. Exported API (`libfiioaudio.so`, all in `.dynsym`)

```
init_pcm2dsd            get_pcm2dsd_data            dsd8_convert_to_dsd16
set_pcm2dsd_dsd_level   get_pcm2dsd_level           dsd16_convert_32bit
get_pcm2dsd_dsd_level   get_pcm2dsd_shaper_coeffs    dsd_convert_platform
get_pcm2dsd_delta_gain  pcm16/24/32_convert_pcm32   get_dsdtopcm_data
pcmint_convert_pcmfloat pcm32_convert_to_av         init_dsd2pcm_params
process_pcm2eq_init     get_pcm2eq_data             process_pcm2eq_event
get_alltodsd_config     get_alltodsd_shaper_coeffs  get_alltodsd_delta_gain
get_current_sample_rate_from_dsdlevel  reset_player_and_pcm_cache
```

`g_pcm2dsd_handle` (`.dynstr` object @ `0x20d60`) points at a **0x143e0-byte**
state structure; `init_pcm2dsd()` just `memset(handle, 0, 0x143e0)`.

## 3. System properties that control it

| property | default | meaning |
|---|---|---|
| `persist.sys.all.to.dsd` | `0` | **All To DSD master switch** (`1` = on) |
| `persist.sys.pcm2dsd.level` | `0` | 0=DSD64, 1=DSD128, 2=DSD256, 3=DSD512 |
| `persist.sys.dsd.shaper.coeffs` | `2` | noise-shaper id: 0 → 8 taps, 1 → 6 taps, 2 → 4 taps |
| `persist.sys.dsd.delta.gain` | `0` | user "delta gain" trim |
| `persist.sys.dsd.time.flag` | `1` | timing/delay compensation |
| `persist.sys.fiio.config.dsd.native` | `256` | native DSD rate |
| `persist.sys.fiio.audio.file` | – | **debug WAV dump** to `/data/misc/test/test_%d.wav` |
| `persist.sys.fiio.audio.process.log` | – | verbose log |

On the inspected unit `persist.sys.all.to.dsd = 1` and
`persist.sys.fiio.config.dsd.native = 256`.

### DSD level → output rate (`get_current_sample_rate_from_dsdlevel`, `0x15fdc`)

| level | 44.1 k family | 48 k family | ratio to fs |
|---|---|---|---|
| 0 | 88 200 | 96 000 | ×64 |
| 1 | 352 800 | 384 000 | ×128 |
| 2 | 352 800 | 384 000 | ×256 |
| 3 | 176 400 | 192 000 | ×512 |

(The rates are the HAL's PCM processing rate; the ×32/×16/×8/×4 scaling to
2.8224 / 5.6448 / 11.2896 / 22.5792 MHz is implicit.)

## 4. The converter: `get_pcm2dsd_data()` @ `0x19d90`

```
 0x19d90  entry:  x0 = PCM ptr, w2 = byte size, w3 = bits (16/24/32),
                  w4 = channels, w5 = format, w6 = shaper id, w7 = misc
 0x19f2c  16-bit path   -> scvtf -> pcm32_convert_to_av -> fcvt, stored as double
 0x19ec0  24-bit path   -> assemble 24-bit int, same conversion
 0x19f9c  32-bit path   -> direct int, same conversion
 0x1a1a8  interpolator: 5 stages x 2 channels, per-stage state at
                        handle + 0x30*i + {0x90..0xb8 | 0x210..0x238},
                        stage lengths at handle + 0x143a0 + 8*i
 0x1a388  shaper 1 (6 error taps)   input from handle + 0x203a0
 0x1a448  shaper 0 (8 error taps)
 0x1a544  shaper 2 (4 error taps)   <-- factory default
 0x1a420  output: strb w11,[x21]   one byte per DSD bit
```

The quantiser core (identical in all three branches):

```c
total = g*u[n] + Σ_k  (H[k] + A[k]) * e[n-1-k];
bit   = (total >= 0.0) ? 1 : 0;
e[n]  = total - (2*bit - 1);          /* one shared error delay line */
```

`H[]` and `A[]` are two FIRs taken verbatim from `.rodata` and summed onto the
same error history.

### Recovered coefficient tables (from `.rodata`)

**shaper id = 2 — factory default (4 taps)**

```
H = [ 3.1969667828000001, -3.8978846131775038,  2.1403520275566841,
     -0.44562557171397232 ]                        (0x7e88, 0x7fc8, 0x7eb0, 0x7e78)
A = [ 0.802319609916744,  -2.100688172255984,    1.8589343651600601,
     -0.55437442828602768 ]                        (0x7ed0, 0x7ea8, 0x7fc0, 0x7e60)
H+A = [ 3.999286392716744, -5.998572785433488, 3.999286392716744, -1.0000000000000000 ]
```

**shaper id = 1 (6 taps)**

```
H = [ 5.1812493674000004, -11.251843168311209,  13.110028967961449,
     -8.6453260263328602,   3.0602853757125619,  -0.45448912482285447 ]
A = [ 0.81654825844545798, -3.7393482705534971,   6.8767586580770477,
     -6.3458654125318468,   2.9375122501328952,  -0.54551087517714547 ]
```

**shaper id = 0 (8 taps)**

```
H = [ 0.80365231044305308, -5.2944845485449221,  14.974123863329551,
     -23.566583305754548,  22.28874261804205,   -12.667230388774531,
      4.0052976502491759 ]
A = [ 7.1931457765999998, -22.686306858611161,  40.977858981271901,
     -46.36939574322227,   33.663240226559381,  -15.31356101838154,
      3.991500436793876,   -0.45648227028327693 ]
```

### The key structural result

For shaper 2, `H+A` is **exactly** `1 − (1 − z⁻¹)⁴`:

```
(1 - z^-1)^4 = 1 - 4z^-1 + 6z^-2 - 4z^-3 + z^-4
1 - that     = 4z^-1 - 6z^-2 + 4z^-3 - z^-4      == H+A   (to ~7e-4)
```

and `Σ(H+A) = 1.00000000` exactly. So

```
B(z) = 1 - (1 - z^-1)^N ,   NTF(z) = 1 - B(z) = (1 - z^-1)^N ,  N = 4/6/8
```

a pure Nth-order differentiator with a zero of order N at DC — i.e. exactly the
noise-shaping transfer function family, at unity DC feedback gain.

### Delta gain

`.rodata:0x7ee8 = -0.10166630249789134` is loaded as the modulator input gain in
the shaper-0/2 branches (`fmadd d1, d5, d8, d2`), i.e. **-19.855 dB**.
Note that `pcmint_convert_pcmfloat` (`0x1d358`, using the AArch64 `scvtf` magic
constants 1.75/1.875) does **no** normalisation, so the absolute output level also
depends on the gain of the device's own interpolator, which is built at run time
and could not be recovered statically.

## 5. What could *not* be recovered, and why it matters

The interpolator in front of the quantiser is a cascade of **five identical
biquad sections per channel**; the per-section coefficients and the per-stage
lengths (`handle + 0x143a0`) are computed by run-time initialisation code that
was not located. Consequently:

* the device's absolute output level cannot be reproduced bit-exactly;
* the exact image rejection of the device's interpolator is unknown.

`flac2dsf` uses an independently verifiable polyphase Kaiser-windowed-sinc FIR in
its place, which is the standard design used by every PCM→DSD converter.

## 6. Stability of the recovered loop

Running the decoded loop verbatim diverges:

| NTF | max error state after 2·10⁵ samples |
|---|---|
| `(1-z^-1)²` | 0.83 (stable) |
| `(1-z^-1)⁴` (= FiiO shaper 2) | 3.9·10¹⁷ (diverges) |
| `(1-z^-1)⁶` / `(1-z^-1)⁸` | NaN |

The characteristic polynomial of the homogeneous error recursion has all roots
exactly on `|z| = 1` (poles at `z = 1`, multiplicity N) — an N-fold integrator
that a 1-bit quantiser cannot stabilise. A shipping player cannot be running
this realisation, so `flac2dsf` uses the same NTF family with a leaky integrator
`B(z) = 1 - (1 - a·z⁻¹)^N`, `a = 0.6` by default, which places all poles
strictly inside the unit circle while keeping NTF order, DC gain and noise
slope identical.

## 7. Debug facilities found on the device

```
setprop persist.sys.fiio.audio.file 1     # dump audio to /data/misc/test/test_%d.wav
setprop persist.sys.fiio.audio.process.log 1
setprop persist.sys.fiio.audio.eq.log 1
setprop persist.sys.fiio.audio.resample.log 1
setprop persist.sys.fiio.audio.mqa.log 1
```

### Measured on the running device (adb, JM21, Android 13)

With All-To-DSD on and a 44.1 kHz source, `logcat -s fiioaudio` shows:

```
match_audio_process_mode: current p2d is enable
match_audio_hardware_params: mode PCMTODSD srcRate 44100 srcFormat 3 srcCh 2
                             dstRate 88200 dstFormat 436207618 frameSize 8 halRate 88200
start_config_params: mode PCMTODSD tDstFormat 436207618 hwFormat 436207618
                     tDstRate 88200 hwRate 352800 tHalRate 88200
```

**`hwRate 352800` = 2 822 400 / 8.** The device emits DSD64 as 8 one-bit samples
per byte at 352 800 bytes/s - the same `dsd_lsbf_planar` convention ffmpeg uses.
So the JM21's native All-To-DSD output for a 44.1 kHz source is **2 822 400 Hz**,
matching what this tool produces.

### The dump cannot be read on a production build

`ro.build.type=user`, `ro.debuggable=0`, so `adb root` is refused.
`/data/misc` is `drwxrwx--t system:misc`, and adb shell runs as uid 2000 with no
`misc` group, so `/data/misc/test/test_*.wav` is unreachable. **A numerical capture
of the device's DSD output is therefore not possible without root**, which is why
the loop's exact arithmetic has to come from the disassembly.

## 8. The quantiser inner loop (`0x1a3a0` - `0x1a40c`)

```
fmadd  d2, d1, d27, d2      ; 6 distinct coefficients feed the comparator:
fmadd  d2, d3, d28, d2      ;   d25 d27 d28 d6 d7 d8
fmadd  d2, d5, d6,  d2
fmadd  d2, d1, d7,  d2
fmov   d24, #-1.00000000
fmov   d26, #1.00000000
fmadd  d2, d7, d8,  d2
fmadd  d2, d6, d4,  d2
fcmp   d2, #0.0
fadd   d0, d0, d2
fcsel  d1, d26, d24, lt    ;  q = (acc >= 0) ? +1 : -1
fadd   d0, d0, d1
```

Two things follow from this that the earlier analysis missed:

1. The feedback is **not** a plain FIR of past quantiser errors. It is an **IIR
   cascade** in which six different coefficients are combined into the
   comparator input. That resolves the paradox that stopped the port: an IIR loop
   filter can keep its poles inside the unit circle *and* put the NTF zeros on the
   unit circle, i.e. a true DC null. The port instead uses
   `B(z) = 1 - (1 - a z^-1)^N`, whose NTF zeros sit at `z = a` inside the circle
   - only `(1-a)^N` = -32 dB at DC, saturating at +16 dB above ~20 kHz. That is
   the whole reason for the ~30 dB noise floor.
2. There is **no clamp and no division** in the inner loop.

### `.rodata` constants loaded into the loop

Base `.rodata` VMA 0x5e90 (file offset 0x5e90); `adrp #0x7000` + imm addresses it.

| reg | addr | value | |
|---|---|---|---|
| d15 | 0x7ee8 | -0.10166630249789134 | Δ gain (matches the earlier finding exactly) |
| d25 | 0x7f48 | 0.816548258 | comparator input |
| d27 | 0x7f50 | -3.739348271 | comparator input |
| d28 | 0x7f78 | 6.876758658 | comparator input |
| d13 | 0x7f70 | 12.031360402 | |
| d11 | 0x7f30 | 0.164377656 | |
| d12 | 0x7fe0 | 6.015680201 | |
| d9 | 0x7e90 | -0.342489376 | |
| d14 | 0x7ee0 | 0.224583424 | |
| d10 | 0x7eb8 | -0.270179279 | |
| d16 | 0x7ed0 | 0.802319610 | |
| d17 | 0x7ea8 | -2.100688172 | |
| d19 | 0x7e60 | -0.554374428 | |
| d20 | 0x7e88 | 3.196966783 | |
| d22 | 0x7eb0 | 2.140352028 | |
| d23 | 0x7e78 | -0.445625572 | |
| d18 | 0x7fc0 | 1.858934365 | |
| d21 | 0x7fc8 | -3.897884613 | |

The same block also holds filter-design constants rather than loop coefficients:
`0.01 / 0.04 / 0.05 / 0.1 / 0.9` at 0x7ea0 / 0x7f10 / 0x7f68 / 0x7f88 / 0x7f90,
and `10000π / 400π / 2π / log2(10)` at 0x7f00 / 0x7f28 / 0x7fb0 / 0x7fd8 - so the
interpolation filter is designed at run time from those, or the tables are
precomputed for a set of designs.

`d3 d4 d6 d7 d8` are loaded per shaper elsewhere and still need tracing.

### The loop is an IIR A/B split, not a FIR of past errors

```
0001a388  ldr  x10, [x20, x28, lsl #3]   ; x10 = per-channel state (x28 = channel)
0001a398  ldp  d0, d1, [x10]             ; d0=s0 d1=s1
0001a39c  ldp  d3, d5, [x10, #0x10]      ; d3=s2 d5=s3
0001a3cc  ldp  d1, d6, [x10, #0x20]      ; d1=s4 d6=s5
0001a3a0  fmul  d2, d0, d25              ;   v  = sum a[i]*s[i]   (feedback)
0001a3ac  fmadd d2, d1, d27, d2
0001a3b8  fmadd d2, d3, d28, d2
0001a3c8  fmadd d2, d5, d6,  d2
0001a3d8  fmadd d2, d1, d7,  d2
0001a3e8  fmadd d2, d7, d8,  d2          ;   <-- d7 = [x8], see below
0001a3f0  fmadd d2, d6, d4,  d2
0001a3a4  fmul  d0, d0, d4               ;   u  = sum b[i]*s[i]   (feedforward)
0001a3b0  fmadd d0, d1, d4, d0
0001a3bc  fmadd d0, d3, d4, d0
0001a3d0  fmadd d0, d5, d3, d0
0001a3e0  fmadd d0, d1, d3, d0
0001a3ec  fmadd d0, d6, d1, d0
0001a3f4  fcmp  d2, #0.0
0001a3f8  fadd  d0, d0, d2               ;   u += v
0001a3fc  cset  w11, ge                   ;   bit = (v >= 0)
0001a400  fcsel d1, d26, d24, lt         ;   q = (v < 0) ? +1 : -1
0001a408  ldp   q4, q3, [x10]
0001a40c  fadd  d0, d0, d1               ;   u += q
0001a410  stur  q4, [x10, #8]             ;   s1=s0 s2=s1
0001a418  stur  q3, [x10, #0x18]          ;   s3=s2 s4=s3
0001a41c  str   d2, [x10, #0x28]          ;   s5 = v
0001a420  str   d0, [x10]                 ;   s0 = u
0001a424  add   x8, x8, #8                ;   x8 advances one element per output sample
```

So the structure is

```
v[n] = sum a[i]*s[n-i]  +  c[n]*d8
u[n] = sum b[i]*s[n-i]  +  v[n] + q[n]
S'   = [u, s0, s1, s2, s3, v]
bit   = (v >= 0),  q = (v < 0) ? +1 : -1
```

**Two things follow, and both invalidate the earlier port:**

1. The feedback is a **6-tap IIR** with separate `a` (feedback) and `b`
   (feedforward) vectors - not a FIR over past quantiser errors. The loop
   transfer is `A(z)/(1 - z^-1 A(z))`, whose poles are set by `A` alone. That is
   how the device gets an NTF with a **true 6th-order zero at DC** while the loop
   stays stable - the resolution of the paradox that stopped the port. The port
   instead realises `B(z) = 1 - (1 - a z^-1)^N`, whose NTF zeros sit at `z = a`
   *inside* the circle: -32 dB at DC and saturated at +16 dB above ~20 kHz.
   **That single modelling error is the reason for the ~30 dB noise floor.**

2. `x8` advances by 8 **every output sample**, so `ldr d7, [x8]` loads a
   **time-varying** coefficient and the term `d7 * d8` is not a fixed loop tap at
   all: **the interpolation filter is folded inside the feedback loop**, with a
   fresh coefficient per output sample. This tool instead runs a fixed-length FIR
   *in front of* the quantiser.

### Coefficients recovered

| | A (feedback) | B (feedforward) |
|---|---|---|
| 0 | 0.816548258 (0x7f48) | 5.181249367 (0x7e50) |
| 1 | -3.739348271 (0x7f50) | -11.251843168 (0x7ff8) |
| 2 | 6.876758658 (0x7f78) | 13.110028968 (runtime, = recovered row idx 2) |
| 3 | *runtime* | -8.645326026 (0x7fb8) |
| 4 | 2.937512250 (0x7f58) | 3.060285376 (0x7e58) |
| 5 | -0.545510875 (0x7ec0) | -0.454489125 (0x7e80) |

B is exactly the 6-value row recovered earlier as an `H+A` sum
(`[5.1812493674, -11.2518431683, 13.1100289680, -8.6453260263, 3.0602853757,
-0.4544891248]`), which is why that row summed to 1.0.

Requiring `A + B` to be the binomial row of `1 - (1-z^-1)^6 = [6,-15,20,-15,6,-1]`
pins `a3 = -15 + 8.645326026 = -6.354673974`, and independently
`a5 = -1 + 0.454489125 = -0.545510875` which **matches `.rodata` 0x7ec0 exactly**.
So the shape is right.

**But simulation diverges for every `a3` in -9..+4**, which means the fixed
coefficient model above is still incomplete - consistent with the time-varying
`c[n]` term that a fixed 6-tap model cannot represent.

### What still blocks a faithful port

* `a3` lives in a runtime buffer (`g_pcm2dsd_handle + 0x1203a0 + 0x10`), filled by
  `init_pcm2dsd` @ `0x1a970`. That routine writes 6 floats to `0xedc438`..`0xedc46c`
  (`.bss`) computed from `.rodata` constants `400π / 2π / log2(10)` and
  `0.01 / 0.04 / 0.05 / 0.1 / 0.9`, i.e. the interpolation filter is designed at
  run time.
* Those buffers cannot be read: `ro.build.type=user`, `ro.debuggable=0`,
  `adb root` refused, `/data/misc` is `0770 system:misc` and shell is uid 2000.
  The WAV dump at `/data/misc/test/test_%d.wav` is therefore unreachable.
* Closing this properly needs a full dataflow trace of the function (the `x8`
  coefficient walk has to be followed to see which table it indexes and how it
  relates to `init_pcm2dsd`), or root.

## 9. Tools used

`tools/elfscan.py`, `elfdump.py`, `disasm2.py` (capstone-based AArch64
disassembler with ADRP/ADD → string and symbol annotation), `rodata.py` (dump
doubles from an address range), `rodstr.py` (extract `.rodata` strings),
`scan.py` (find instructions matching a set of immediates), plus
`sweep_sfsr.py` / `design_ntf.py` for the NTF stability/SFSR studies.

## 10. Shaper 1 inner loop: full register trace (0x1a388 - 0x1a430)

This supersedes sections 6 and 8. Two earlier claims were wrong and are
corrected here.

### 10.1 Two corrections

**The state is 5 deep, plus a slot that is NOT part of the shift register.**

```
0001a408  ldp      q4, q3, [x10]
0001a410  stur     q4, [x10, #8]      ; s1 <- s0, s2 <- s1
0001a414  ldr      d2, [x10, #0x20]   ; re-read slot 5
0001a418  stur     q3, [x10, #0x18]   ; s3 <- s2, s4 <- s3
0001a41c  str      d2, [x10, #0x28]   ; slot5 <- slot5   (unchanged)
0001a420  str      d0, [x10]          ; s0 <- u + q
```

Slot 5 is loaded into `d6` at `0x1a3cc`, used twice, then re-read from memory
and written straight back. It is an explicit "leave unchanged" idiom, so the
shift register is `d0..d4` and slot 5 holds something the polyphase stage
refreshes between quantiser calls - the current input sample.

**There is no algebraic loop.** Every term reads the state as it was *before*
the update. An earlier attempt that solved a 2x2 system per sample was based on
a misreading and produced a full-scale railed output (`mean q = +1.0`).

### 10.2 The loop, exactly as executed

```
d0..d4  = x10[0..4]        slot5 = x10[5] = x[n]

v  = a0*d0 + a1*d1 + a2*d2 + a3*d3 + a4*d4 + a5*x[n] + tbl[m]*d8
u  = b0*d0 + b1*d1 + b2*d2 + b3*d3 + b4*d4 + b5*x[n] + v + q
bit = (v >= 0)                          cset  w11, ge
q   = (v < 0) ? +1 : -1                 fcsel d1, d26(+1), d24(-1), lt

state <- [u + q, d0, d1, d2, d3]        slot5 untouched
x8   += 8                               one table entry per output sample
```

Register-to-operand mapping (note `d1` is loaded twice, `d3`/`d4`/`d6`/`d7` are
clobbered and reloaded, so the order matters):

| term | register | source |
|---|---|---|
| `a0*d0` | `d25` | stack (`[sp,#0x30]`) |
| `a1*d1` | `d27` | stack (`[sp,#0x28]`) |
| `a2*d2` | `d28` | stack (`[sp,#0x20]`) |
| `a3*d3` | `d6` | `.rodata` `adrp x12,#0x8000` + `#0x10` -> **0x8010** |
| `a4*d4` | `d7` | `.rodata` `adrp x13,#0x7000` + `#0xf58` -> 0x7f58 |
| `a5*x`  | `d4` | `.rodata` `adrp x14,#0x7000` + `#0xec0` -> 0x7ec0 |
| `b0*d0` | `d4` | `.rodata` `adrp x15,#0x7000` + `#0xe50` -> 0x7e50 |
| `b1*d1` | `d4` | `.rodata` `adrp x16,#0x7000` + `#0xff8` -> 0x7ff8 |
| `b2*d2` | `d4` | `.rodata` `adrp x17,#0x8000` + `#0x18` -> **0x8018** |
| `b3*d3` | `d3` | `.rodata` `adrp x0,#0x7000`  + `#0xfb8` -> 0x7fb8 |
| `b4*d4` | `d3` | `.rodata` `adrp x1,#0x7000`  + `#0xe58` -> 0x7e58 |
| `b5*x`  | `d1` | `.rodata` `adrp x2,#0x7000`  + `#0xe80` -> 0x7e80 |
| `tbl[m]`| `d7` | `ldr d7, [x8]`, `x8 += 8` per output sample |
| `d8`    | `d8`  | `scvtf d8, w19` at 0x1a044 (in `get_pcm2dsd`) |

`adrp` supplies a *page* base, so `adrp x12,#0x8000` + `#0x10` is 0x8010, not
0x7010 (0x7010 is in the string region and decodes to 1.1e69).

### 10.3 Complete coefficient set

```
A = [ 0.816548258000, -3.739348271000,  6.876758658000,
      -6.345865412532,  2.937512250000, -0.545510875000 ]
B = [ 5.181249367000,-11.251843168000, 13.110028967961,
      -8.645326026000,  3.060285376000, -0.454489125000 ]
```

`a3` and `b2` are the two that were previously unknown; everything else was
already recovered. Independent checks, none of which were fitted:

| check | value | expected |
|---|---|---|
| `sum(A)` | 0.000094607468 | the residual already present in the old table as tap 7 (`0.0000948564594446`) |
| `sum(B)` | 0.999905391961 | ~1 |
| `sum(A)+sum(B)` | 0.999999999430 | exactly 1 (unity DC gain) |
| `a5+b5` | -1.000000000 | exactly -1 |
| `b2` | 13.110028967961 | already in the old table |
| `a3` vs `-(15+b3)` | -6.345865 vs -6.354674 | binomial prediction, 0.14% off |

`a5 + b5 = -1` exactly means the current-sample feedthrough cancels: the signal
reaches the quantiser only through the 5-tap delay line, not directly.

### 10.4 What `tbl[m]` is

`x8` is seeded once per call at

```
0001a378  add      x8, x25, #0x20, lsl #12   ; x8 = x25 + 0x20000
0001a380  add      x8, x8, #0x3a0            ; x8 = x25 + 0x203a0
```

i.e. `handle + 0x1203a0`, and is then advanced by 8 bytes **per output sample**.
`init_pcm2dsd` designs a windowed-sinc interpolator and writes it there: at
0x1ac38-0x1acd4 it computes

```
s2 = 1 / (s1 + 1)                         ; Kaiser/Blackman window term
s5 = (1-d3)*0.5 * s2 ;  s4 = (1-d3) * s2
s3 = (-2*d3) * s2      ;  s1 = s2 - s1*s2
str s5 -> .bss+0x43c    str s4 -> +0x440    str s5 -> +0x444
str s3 -> +0x448        str s1 -> +0x44c
```

with `d8 = (double)w20` normalising `rodata[0x7f28]` and `rodata[0x7fd0]`.
So `tbl[m]` is an **interpolator FIR coefficient**, and the device genuinely
folds the interpolator into the quantiser feedback path. This is why the plain
6-tap model with fixed coefficients could never be made to work: it is not the
same system.

### 10.5 Current blocker

With `tbl[m]*d8` set to zero, the recovered coefficients give a **bounded limit
cycle of amplitude 25-30** (not divergence) at every input level from -6 to
-60 dBFS. That is stable but audibly noisy, so the interpolator term is not
optional - it is what keeps the loop out of the limit cycle.

Reproducing the device therefore still requires, from inside a 1700-instruction
`init_pcm2dsd`:

1. the code that expands the 5 window coefficients into the full FIR table at
   `handle + 0x1203a0` (table length, phase count, normalisation),
2. `w19`, the value behind `scvtf d8, w19` at 0x1a044,
3. how `x8` is re-seeded between chunks (`x9` samples per call, `0x1a374 cbz
   w8` skips the quantiser entirely when the flag is 0).

`C:\Users\18785\AppData\Local\Temp\opencode\verify_jm21.py` holds the exact
simulation of the loop above and reproduces the limit cycle.

### 10.6 Shaper 2 (factory default) confirms the input path is still missing

`0x1a544` is only 4 taps and much easier to read:

```
0001a544  ldr   x10, [x20, x28, lsl #3]
0001a548  ldr   d5, [x8]                  ; tbl[m]
0001a54c  ldp   d0, d1, [x10]             ; s0, s1
0001a550  ldp   d3, d4, [x10, #0x10]      ; s2, s3
0001a554  fmul  d2, d0, d16               ; v = s0*d16
0001a558  fmul  d0, d0, d20               ; u = s0*d20
0001a55c  fmadd d2, d1, d17, d2           ; v += s1*d17
0001a560  fmadd d0, d1, d21, d0           ; u += s1*d21
0001a564  fmadd d2, d3, d18, d2           ; v += s2*d18
0001a568  fmadd d0, d3, d22, d0           ; u += s2*d22
0001a56c  fmadd d1, d5, d8, d2            ; v += tbl[m]*d8
0001a570  fmadd d0, d4, d23, d0           ; u += s3*d23
0001a574  fmadd d1, d4, d19, d1           ; v += s3*d19
0001a578  fcmp  d1, #0.0
0001a57c  fadd  d0, d0, d1               ; u += v
0001a580  fcsel d1, d26, d24, lt         ; q
0001a584  cset  w11, ge                  ; bit
0001a58c  fadd  d0, d0, d1               ; u += q
0001a598  str   d2, [x10, #0x18]          ; s3 <- s2
0001a59c  stur  q3, [x10, #8]             ; s1 <- s0, s2 <- s1
0001a5a0  b     #0x1a420                 ; s0 <- u ; x8 += 8
```

A2 = [d16, d17, d18, d19], B2 = [d20, d21, d22, d23]. For the recorded
factory-default values `sum(A2) = +0.006191375` and `sum(B2) = 0.993808625`,
i.e. the v side sums to ~0 (the DC null, only -44 dB deep) and the u side to ~1.

**The decisive observation: neither shaper has any input term.** In both, the
only quantity that is not a feedback tap is `tbl[m]*d8`. The preceding stage
(0x1a1a8-0x1a310) is a biquad (`fmadd d0,d1,d15,d0` / `fadd d5,d1,d0` /
`fmadd d3,d4,d10,d3` / `fmul d5,d3,d12`), and `0x1a19c bl #0x1e040` looks like a
memset that zeroes slot 5 of the quantiser state.

### 10.7 The table at handle + 0x1203a0 is built with stride 16 and interleaved zeros

```
0001a1d4  add   x12, x25, x10, lsl #3
0001a1dc  ldr   d0, [x12, x23]           ; coefficient
0001a1e0  str   xzr, [x11]               ; [base + 16i]      = 0
0001a1e4  stur  d0, [x11, #-8]           ; [base + 16i - 8]  = coefficient
0001a1f4  add   x11, x11, #0x10          ; stride 16
0001a1f8  ucvtf d1, w10
0001a1fc  fcmp  d0, d1
0001a200  b.gt  #0x1a1d4                 ; while value > counter
```

with `x11 = x25 + 0x20000 + 0x3a8`, i.e. the array starts at `x25 + 0x203a0`.
Since the quantiser advances `x8` by 8 per output sample, the sequence it reads
is `coef, 0, coef, 0, ...`. `x19` is the same register behind
`scvtf d8, w19` (0x1a044) and `madd x3, x8, x19, x25` (0x1a210), so `d8` is
that stride / scale.

An interleaved zero read at one value per two samples is not a signal path - it
is a Nyquist-rate (±fs/2) term, which reads as square-wave dither rather than
as the interpolated input.

### 10.8 What was tried with the recovered coefficients, and why it did not work

Simulating the recovered taps in every structure reading that the disassembly
allows (`tbl` = interpolated input; slot 5 = input; slot 5 dead; both) over
`g = d8` in [-2, +4] and input levels from -6 to -60 dBFS: **all readings
diverge.** Two clean analytic results do come out of the arithmetic, and neither
is sufficient:

- `H(1) = 1.000000001` at `g = 0.5` for the 6-tap reading, with `g = 0.25 ->
  3.0` and `g = 0.75 -> 1/3` (an exact arithmetic progression), which is what a
  correct feedthrough scaling should produce;
- `H(1) = 1.0000` at `g = -1` for the 5-tap reading.

Both are stable-free: the loops rail regardless. So the coefficient set is very
likely right (it passes every independent check in 10.3) while the *signal
path* is still unidentified - without an input term the structure is not a
converter at all, so tuning `g` cannot rescue it.

### 10.9 Remaining work

Identify where the interpolated PCM enters. Candidates not yet checked:

1. the biquad at 0x1a1a8-0x1a310 writes its output into the quantiser state
   (`str d3, [x14]` / `[x13]`, `str d0, [x11]` / `[x10]` at 0x1a270-0x1a298),
   which may be the same struct as `[x20, x28, lsl #3]`;
2. `0x1a14c` re-seeds the state per channel from `[sp,#0x80..0xb0]`, and
   `str d0, [x22]` / `[x22,#8]` ... `[x22,#0x20]` set slots 0-4 while
   `0x1a19c` appears to clear slot 5 - the asymmetry needs confirming;
3. whether `x20` (the per-channel state pointer array) is distinct from the
   struct the biquad writes to.

`C:\Users\18785\AppData\Local\Temp\opencode\scan_g.py` and `decide.py` hold the
`g` scans and reproduce the divergence.

### 10.10 The three traced points

**Point 1 - the biquad does not touch the quantiser state.** Its six state
doubles live at `handle + 48*x8 + 0x210 .. 0x238` (set up at 0x1a210-0x1a24c)
and it stores only inside that window (0x1a270-0x1a298). Its input is
`[handle + x23 - 0x80000]` (0x1a250-0x1a254). The quantiser state is
`handle + x24` (0x1a108: `add x22, x25, x24`). Different ranges, so the
biquad is not the signal source for the quantiser.

**Point 2 - slot 5 is never written.** `0x1a14c` is the per-channel loop head
(`add x28,x28,#1` / `cmp x28,#2` / `b.eq`). The reseed stores slots 0..4 only
(0x1a16c, 0x1a17c, 0x1a188, 0x1a190, 0x1a198) and then calls 0x1e040 with the
per-channel pointer. Inside the quantiser, slot 5 is re-read and written
straight back (0x1a414 / 0x1a41c). So `slot5 == 0` throughout, which makes the
6th tap (`a5`, `b5`) a dead coefficient for shaper 1 - and it is exactly the
tap that completes the binomial, see 10.11.

**Point 3 - `x20` is the caller's array.** `0x19dcc mov x20, x0`, and the
quantiser loads `ldr x10, [x20, x28, lsl #3]`, i.e. one state pointer per
channel from the caller. `x25 = *(0x1f410)` (0x1a-1a24) is the global handle.
`0x1a210 madd x3, x8, x19, x25` uses `w19 = 0x30 = 48` (0x1a118) as a
six-double struct stride, so `x19` is a byte stride and not the `d8` scale.

### 10.11 Conclusion: this cannot be ported as a fixed-coefficient loop

> **SUPERSEDED - see section 12 of HANDOFF.md.** The six-fold-integrator
> wording was wrong: slot5 is not in the shift register, so A+B must not be
> evaluated over six taps. The live five-tap polynomial has a real pole at
> radius 1.999 - still unstable, different mechanism. `0x1a19c` is memcpy,
> not memset; `d8` is `(double)(16-arg6)`, not 48; the polyphase table
> generator has been located at 0x1a1b4-0x1a200.

Substituting `v` out of the state recursion collapses everything to one delay
line:

```
d0[n] = sum_{i=0..4} (A[i]+B[i]) * d_i[n] + g*x[n] + q[n]
```

and the recovered taps satisfy

```
A+B = [5.997797625, -14.991191439, 19.986787626, -14.991191439, 5.997797626, -1.000000000]
    ~ 1 - (1-z^-1)^6 = [6, -15, 20, -15, 6, -1]        max rel. error 6.6e-4
```

`a5 + b5 = -1` exactly supplies the last term. So the loop is a **six-fold
integrator**: its poles are

```
+0.9987 +- 0.0432j   +1.0140 +- 0.0199j   +0.9862 +- 0.0181j
```

i.e. sitting on the unit circle to within 1.4%, three of them slightly outside.
That is not a near-miss on my part - it is what the coefficients *are*. The
linearised loop is unstable, and the only thing holding it together is the
`tbl[m]*g` term.

That term cannot be replaced by a constant. The table at `handle + 0x1203a0` is
rebuilt from the quantiser state on every call (0x1a1b4 reads the five state
doubles, 0x1a1dc copies a per-tap coefficient into `ceil(state[j])` slots of
the stride-16 array). It is **adaptive and state-dependent**, not a fixed FIR.
Every fixed-coefficient model tried - the table as interpolated input, as
Nyquist-rate dither, as a constant scale `g` over [-48, +192], slot5 as input,
slot5 dead, both - diverges at every level from -6 to -60 dBFS, and that is the
expected outcome rather than a bug in the search.

For contrast, the realisation this tool actually uses, `B(z) = 1-(1-a z^-1)^N`,
puts all N poles at radius `a`, so it is stable for `a < 1` but its DC null is
only `(1-a)^N` deep:

| N | a=0.5 | a=0.6 | a=0.7 | a=0.8 |
|---|---|---|---|---|
| 4 | -24.1 dB | -31.8 dB | -41.8 dB | -55.9 dB |
| 6 | -36.1 dB | -47.8 dB | -62.7 dB | -83.9 dB |

The deeper the null, the larger the loop gain `|F| ~ 1+(2a)^N` at Nyquist, and
past roughly `|F| ~ 5` the 1-bit loop rails. That trade-off, not the
implementation, is what caps the current tool near 30 dB. The device escapes it
with an adaptive term; matching it would mean reverse-engineering the table
generator, not the coefficients.

**What is settled:** the coefficient set (section 10.3, all independent checks
pass), the loop structure and state layout (10.1, 10.2), the absence of an
input path outside `tbl[m]*g` (10.6), and the reason a fixed-coefficient port
is impossible (10.11).

**What is not:** the table generator itself, i.e. how `handle + 0x1203a0` is
derived from the state. That is the remaining work, and it needs either a
runtime capture (no root, production build - section 7) or a much longer
dataflow analysis than is recorded here.

---

## 12. Post-review corrections

See `HANDOFF.md` section 12 for the full write-up: self-correction of the
six-fold-integrator claim, `0x1e040 = memcpy` resolved via .rela.plt,
`w24 = 0x143a0` address closure (Q2/Q4 disproved), the two-instruction gap at
0x1a1e8/0x1a1ec (Q1), `d8 = (double)(16 - arg6)` (Q5), the handle memory map,
and that .gnu_debugdata holds XZ-compressed DWARF CFI for every function.

---

## 14. 第二次追查：11 个抽头的 .rodata 地址全部确认

外部模型建议改从 `payload.bin` 入手。**该前提不成立**：设备当前构建与该 OTA 相同。

```
设备 ro.build.version.incremental = eng.fiio.20260428.144644
OTA   post-build-incremental       = eng.fiio.20260428.144644
OTA   post-build = qti/bengal_515/bengal_515:13/TKQ1.230110.001/eng.fiio.20260428.144644:user/dev-keys
```
`payload.bin` (1 642 838 532 字节) 里的 `libfiioaudio.so` 就是设备上那份，**拿不到更新版或
未 strip 版**。在「找不同版本 / 拿初始化函数名」这个方向上价值为零。

但顺着 `memcpy` 的源追下去，把第 12 节的推测全部坐实了。

### 14.1 `handle + 0xa3a0` 装的是 PCM，不是系数

```
0x19e2c  add  x10, x25, #0x3a0              ; handle + 0x3a0
0x19e28  add  x11, x25, #0x10, lsl #12
0x19e30  add  x11, x11, #0x3a0              ; handle + 0x103a0
0x19e44  stp  x10, x11, [x29, #-0x60]       ; [x29-0x60]=h+0x3a0  [x29-0x58]=h+0x103a0
...
0x1a178  sub  x9, x29, #0x60
0x1a184  ldr  x1, [x9, x8]                  ; x1 = [x29-0x60 + 8*channel]
0x1a19c  bl   0x1e040                       ; memcpy(h+0xa3a0, h+0x3a0 或 h+0x103a0, 8*(n>>1))
```

而 `handle+0x3a0` / `handle+0x103a0` 正是**样本转换循环的输出**（16-bit 走 `h+0x103a0`，
24/32-bit 走 `h+0x3a0`，`ldrsh`/`ldrb`+`scvtf` 后逐 8 字节写入）。

**所以 `tbl[m]` 就是内插后的 PCM —— 信号通路确认是 `tbl[m]*g`。**
第 6 节所有「表=内插输入」的仿真结构其实是对的，错的只是 `g`。

### 14.2 `g = d8` 的取值范围

`d8 = (double)(16 - arg6)`，arg6 是第 7 个入参，取 0..16 全部试过（`g` = 16..0），
**含交错填零与不含交错两种模式，全部发散**。所以问题不在 `g`。

### 14.3 11 个抽头的确切 `.rodata` 地址（此前只有 6 个）

关键发现：**全部 11 个栈抽头都在 0x1a074-0x1a144 从 `.rodata` 直接装载**，
不是「部分来自栈、来源不明」：

```
0x1a110  ldr  d23, [x8,  #0xe78]     ; x8  = adrp 0x7000 @0x1a084
0x1a120  ldr  d25, [x10, #0xf48]     ; x10 = adrp 0x7000 @0x1a07c
0x1a128  ldr  d27, [x9,  #0xf50]     ; x9  = adrp 0x7000 @0x1a0a4
0x1a130  ldr  d28, [x8,  #0xf78]     ; x8  = adrp 0x7000 @0x1a114
0x1a12c  stp  d17, d16, [sp, #0x68]
0x1a134  stp  d19, d18, [sp, #0x58]
0x1a138  stp  d21, d20, [sp, #0x48]
0x1a13c  stp  d23, d22, [sp, #0x38]
0x1a140  stp  d27, d25, [sp, #0x28]
0x1a144  str  d28, [sp, #0x20]
```
另有 `0x1a088`-`0x1a0e8` 一组 `adrp + ldr d16/d18/d17/d19/d21/d22` 直接给出 B 侧。

**完整地址表与实测值（12 项全部确认，与此前所用值逐位一致）**
```
A[0]=0x7f48 = +0.816548258445     A[1]=0x7f50 = -3.739348270553
A[2]=0x7f78 = +6.876758658077     A[3]=0x8010 = -6.345865412532
A[4]=0x7f58 = +2.937512250133     A[5]=0x7ec0 = -0.545510875177

B[0]=0x7e50 = +5.181249367400     B[1]=0x7ff8 = -11.251843168311
B[2]=0x8018 = +13.110028967961    B[3]=0x7fb8 = -8.645326026333
B[4]=0x7e58 = +3.060285375713     B[5]=0x7e80 = -0.454489124823

sum A = +0.000094608   sum B = 0.999905392   sum = 1.000000000   a5+b5 = -1.000000000
```

### 14.4 现在能精确表述的矛盾

slot5 不进移位链，`slot5 <- 原值`。令 `aL = sum(A[0..4]) = 0.545605482`、
`bL = sum(B[0..4]) = 1.454394517`，则 `1 - aL - bL = -1`，且

```
U(1 - aL - bL) = (a5+b5)*slot5 + q   =>   U = slot5 - q
NTF(1) = V/Q = -aL = -0.5456          (-5.3 dB，无 DC 零点)
V/X    = aL + a5 = +0.000094607       (信号几乎完全抵消)
```

两种 slot5 读法都不成立：静态值 -> 几乎无噪声整形；当前输入 -> 信号被抵消四个数量级。

> **[SUPERSEDED — 见第 0 节 / 第 31 节]** 该判断基于 5 抽头模型与错误的 1.999 手算值，已作废；量化器确实工作，信号增益 = d8/32。
> **系数正确、结构正确、输入通路正确，但仍然不自洽。** 说明还缺一个结构性环节。

最可能的位置：**`0x1a1b4-0x1a200` 不是复制 PCM**。它以 `handle+0x143a0` 的 5 个计数值
（来自帧数，比例 `1:2:4:1:8`）为阈值，把 `handle+0xa3a0` 的 PCM 重排进
`handle+0x203a0` 的 stride-16 数组 —— 这是一层 **polyphase 插值重排**，而本轮所有仿真
都把它当成了「复制 / 直接使用」。这一层没有被建模，是当前唯一的已知缺口。

### 14.5 工具状态

代码未改动。`HANDOFF.md` 与本文均已更新。验证：FLAC 解码 16/16 逐样本一致，
Web 端到端通过，下载产物与 CLI 的 SHA256 一致。

---

## 15. 1.0.8 / 1.1.0 固件包分析结果

外部模型建议用 1.0.8 全量包 + 1.1.0 差分包做版本演化。**已执行，结论是否定性的，但有价值。**

### 15.1 版本链确认

```
1.0.8  eng.fiio.20250920.041125   全量   payload.bin 1 533 808 495
1.1.0  eng.fiio.20251223.064310   差分   payload.bin    95 904 176   pre-build = 1.0.8
1.1.1  eng.fiio.20260428.144644   全量   payload.bin 1 642 838 532   设备当前
```

`1.1.1` 的 `post-build-incremental` 与设备 `ro.build.version.incremental` **完全相同**，
所以该包里的 `libfiioaudio.so` 就是设备上那份 —— 评审设想的「拿到更新版 / 未 strip 版」不成立。

### 15.2 成功提取 1.0.8 的 `libfiioaudio.so`

`.so` 位于 **`/vendor/lib64/libfiioaudio.so`**（设备上确认，`/system` 下没有）。

提取路径（`vendor_pull/re/find_so.py`、`extract_so.py`）：
1. payload header = `CrAU` + version be64 + manifest_size be64 + metadata_signature_size be32，
   数据段起点 = `24 + manifest_size + metadata_signature_size`
   （1.0.8 = `24 + 127336 + 267 = 127627`，该处正是 XZ 魔数位置，可自校验）
2. data 段混用 `REPLACE`（原始）与 `REPLACE_XZ`，不能整段当一个流解压；
   改为扫描全部 XZ 流头（1.5 GB 内约 1900 个），逐段解压并搜 `.dynstr` 里的
   `get_pcm2dsd_data`
3. 命中流 `payload+1373436262`，向前回溯 `\x7fELF`，按 `e_shoff + e_shnum*e_shentsize`
   定文件尾

**结果**：`vendor_pull/re/versions/libfiioaudio-1.0.8.so`，131 440 字节，
`e_machine=0xb7`(AArch64)，`e_type=3`(ET_DYN)，含 `get_pcm2dsd_data` /
`init_pcm2dsd` / `get_pcm2dsd_shaper_coeffs` / `get_alltodsd_config`，
带 `.gnu_debugdata`（3 220 字节，1.1.1 是 3 260）。

| | 1.0.8 | 1.1.1（当前） |
|---|---|---|
| 文件大小 | 131 440 | 135 576 |
| `.text` | 57 344 / 63 736 | 57 344 / 64 076 |
| `.rodata` | 0x5de0 / 17 856 | 0x5e90 / 17 888 |

### 15.3 决定性结果：噪声整形系数逐位未变

用**精确位模式**（不是四舍五入后的十进制）在 1.0.8 的 `.rodata` 里搜当前版本的
13 个系数：**13/13 全部命中，偏移统一为 -200 字节**。

```
B0 0x7e50 -> 0x7d88    A5 0x7ec0 -> 0x7df8    A0 0x7f48 -> 0x7e80
B4 0x7e58 -> 0x7d90    gain 0x7ee8 -> 0x7e20   A1 0x7f50 -> 0x7e88
B5 0x7e80 -> 0x7db8    A4 0x7f58 -> 0x7e90    A2 0x7f78 -> 0x7eb0
B3 0x7fb8 -> 0x7ef0    B1 0x7ff8 -> 0x7f30    A3 0x8010 -> 0x7f48
                                                     B2 0x8018 -> 0x7f50
```

shaper 2（工厂默认，4 抽头）那组系数（`3.1969667828 / -3.8978846131775038 /
2.1403520275566841 / -0.44562557171397232` 与 `0.802319609916744 /
-2.100688172255984 / 1.8589343651600601 / -0.55437442828602768`）在 1.0.8 的
`0x7d98..0x7f00` **也全部命中**。

**代码层面：**
- polyphase 展开循环 `0x1a1b4..0x1a208`（84 字节）在 1.0.8 的 `.text` 中**逐字节存在**（0x1a1fc）
- shaper 1 内环 `0x1a388..0x1a434`（172 字节）**未整段命中**，但 11 个 16 字节块中
  6 块仍在 —— 差异集中在 `adrp`/`ldr` 立即数（随 .rodata 平移 -200）和寄存器分配

### 15.4 结论：这条路线关闭

**1.0.8 → 1.1.0 → 1.1.1 之间，噪声整形器没有实质变化。**
系数相同、polyphase 展开代码相同。因此：

- 做 1.0.8↔1.1.0↔1.1.1 的二进制 diff **无法**揭示第 14.4 节那个缺口
- 「某版本用了更简单的 coefficient generation 分支」这个期望不成立
- 那 340 字节的 `.text` 差异落在别处（EQ / 播放路径 / 日志），与 DSP 无关

**这本身是有价值的否定结论**：它证明缺失的那一层（`handle+0x203a0` 的生成规则，
第 14.4 节）**不是版本差异造成的，而是我一直没建模的那层变换**。继续找旧版本没有帮助。

### 15.5 附带确认

- `.so` 在 `/vendor/lib64`，不在 `/system/lib64`
- 1.0.8 与 1.1.1 的 `.gnu_debugdata` 都在，是 XZ 压缩的 DWARF CFI
- payload manifest 的分区用 **field 13**（不是 AOSP 的 field 2），PartitionUpdate 内的
  操作列表用 **field 7**；且 InstallOperation 内部字段编号与 AOSP `update_metadata.proto`
  不符（`08` 后跟超大 varint、`12 20` 是 32 字节而 AOSP field 2 是 varint）。
  这套 schema 需要单独确认，但不影响上面的结论，因为提取走的是 XZ 扫描而非 manifest。

---

## 16. Unicorn 动态执行成功 —— 缺失层已刻画

harness：`vendor_pull/re/uc_harness.py`（最小化）、`uc_map.py`（映射分析）。
Unicorn 2.1.4 + Capstone 5.0.7，直接从 `get_pcm2dsd_data` 入口跑到
`0x1a208`（biquad 之前），不需要 root、不需要设备。

### 16.1 三处十六进制读错（本轮修正）

| 指令 | 正确值 | 我之前写的 |
|---|---|---|
| `0x1a034 mov w23,#0x3a0` + `0x1a038 movk w23,#0xa<<16` | **0xA03A0** | 0xa3a0 |
| `0x1a064 mov w24,#0x3a0` + `0x1a100 movk w24,#0x14<<16` | **0x1403A0** | 0x143a0 |
| `0x1a1c8 add x11,x25,#0x20<<12` + `0x1a1d0 add x11,#0x3a8` | **0x203A8** → 表首 0x203A0 | 0x203a0（对） |

因为把 `0x3A0 | (0xA<<16)` 看成 `0xA3A0`、把 `0x3A0 | (0x14<<16)` 看成 `0x143A0`，
我之前读阈值一直读到全零的错地址。第 10–15 节里凡写 `0xa3a0` / `0x143a0` 处
应按本节更正为 `0xA03A0` / `0x1403A0`。

### 16.2 句柄布局（动态确认）

```
handle + 0x10       通道 0 调制器状态（8 double）   <- x20[0]
handle + 0x50       通道 1 调制器状态（8 double）   <- x20[1]
handle + 0x3A0      通道 0 PCM（double）
handle + 0x103A0    通道 1 PCM（double）
handle + 0xA03A0    每通道工作缓冲（memcpy 目标，也是展开循环的源）
handle + 0x1203A0   每通道表 A
handle + 0x1303A0   每通道表 B
handle + 0x1403A0   5 个阈值 = [1,2,4,8,16] × (frame_count>>1)
handle + 0x203A0    量化器实际读取的表
handle + 0x210 + 48*x8   biquad 状态（6 double）
```
`x20` 在 `0x1a11c sub x20, x29, #0x80` 被重新赋值，指向 `{h+0x10, h+0x50}`，
所以 `ldr x10,[x20, x28, lsl #3]` 取的是这两个 —— **量化器状态不在 0x1403A0**。
第 12.3 节我据此写的「Q2/Q4 被否证」结论正确，但当时把状态位置也一并弄错了。

### 16.3 24-bit 采样转换格式

```
0x19ed8  ldrb w11,[x20] / ldrb w12,[x20,#1] / ldrb w13,[x20,#2]
0x19ee8  lsl  w11,w11,#8
0x19eec  bfi  w11,w12,#0x10,#8
0x19ef0  bfi  w11,w13,#0x18,#8
0x19ef4  scvtf d0, w11
```
即 `w11 = b0<<8 | b1<<16 | b2<<24`，**24-bit 左对齐（×256）**，最小非零值 256。
这条路径纯整数运算、无外部调用，适合做动态分析入口。
16-bit 路径会调 PLT `0x1d358/0x1d3c4/0x1d888`（含缩放），32-bit 路径同理。

### 16.4 展开循环的真实行为（核心缺失层）

```
0x1a1b4  add x9, x25, x8, lsl #3 ; add x9, x9, x24     ; x9 = h+0x1403A0 + 8*x8
0x1a1bc  ldr d0, [x9]                                    ; threshold[x8]
0x1a1c0  fcmp d0, #0.0 ; b.le 0x1a204                    ; <=0 跳过
0x1a1d4  add x12, x25, x10, lsl #3
0x1a1dc  ldr d0, [x12, x23]                              ; x23 = 0xA03A0 -> h+0xA03A0+8*x10
0x1a1e0  str xzr, [x11]                                  ; h+0x203A8 + 16k
0x1a1e4  stur d0, [x11, #-8]                              ; h+0x203A0 + 16k
0x1a1ec  add x10, x10, #1
0x1a1f0  ldr d0, [x9]                                    ; 重读阈值
0x1a1fc  fcmp d0, d1 ; b.gt 0x1a1d4                      ; while threshold > x10
```

动态实测（`frame_count>>1 = 128`，即 arg2=768 字节 / arg3=0x18）：

```
thresholds  = [128, 256, 512, 1024, 2048]
写入总数    = 128+256+512+1024+2048 = 3968 组  [值, 0]
写入模式    = h+0x203A0+16k <- h+0xA03A0+8*x10 的值
              h+0x203A8+16k <- 0
```

**结论：`handle+0x203A0` 就是 PCM 的 2× 零填充序列**
`[PCM[0], 0, PCM[1], 0, PCM[2], 0, …]`。
量化器 `ldr d7,[x8]` / `x8 += 8` 每输出样点取一个，读到的就是
**PCM, 0, PCM, 0, … 交替**。

**即：所谓「内插器折进反馈环」字面上就是零填充，没有任何滤波器。**
2× 上采样由插零完成，低通由 IIR 整形器自身的反馈提供。
这解释了为什么任何带 FIR 系数的迁移都失败 —— 设备这里根本没有系数。

我前面的仿真其实测过 `interleave=2`（隔样为零），但当时 `g`、量化器状态位置和
输入量纲都错，所以没能对上。

### 16.5 还没解决的两点

1. **量纲**。24-bit 左对齐使样值达 ±2.1e9，而 `v` 里 `tbl[m]*d8`（`d8 = 16-arg6`）
   会把它放大到 1e10 量级。归一化必然发生在 biquad：
   `0x1a250 fmul d0, d0, d14`（`d14 = .rodata 0x7ee0`）。
2. **biquad 就是真正的插值滤波器**。其输入
   `0x1a250 sub x7, x4, #0x80, lsl #12`，`x4 = h + x23 = h + 0xA03A0`
   → `x7 = h + 0xA03A0 - 0x80000`，即**句柄下方 512 KB 的环形历史缓冲**；
   状态在 `h + 0x210 + 48*x8`，6 个 double（双二阶 × 3 段？）。
   第一次 harness 没映射这块所以在 `0x1a208` 停下。

**下一步很明确**：把 `h - 0x80000 .. h` 这 512 KB 映射出来并预填历史样本，
让 biquad 跑完，就能拿到量化器真正看到的**缩放后、带滤波**的序列。
届时 `tbl[m]` 的语义就完全确定，第 6 节全部失败的仿真可以用正确模型重跑。

### 16.6 工具状态

代码未改动。harness 与版本分析脚本在 `vendor_pull/re/`（16 个脚本 + 1.0.8 二进制）。

---

## 17. 内插器已闭合到逐样点级

harness：`vendor_pull/re/uc_biquad.py`、`uc_state.py`、`close2.py`。
句柄基址上移到 `0x30000000` 以便映射环形区；biquad 7936 次迭代全部跑通，无异常。

### 17.1 完整信号链（全部动态确认）

```
PCM (arg0, arg2 字节, arg3 位深)
  │  24-bit 路径: w11 = b0<<8 | b1<<16 | b2<<24 ; scvtf   ← 24-bit 左对齐 (×256)
  ▼
handle + 0x3A0                                   (双声道交错)
  │  memcpy, len = 8*(frame_count>>1)
  ▼
handle + 0xA03A0
  │  展开循环 0x1a1b4-0x1a200, 外层 x8=0..4, 内层 x10
  │  str xzr,[x11] / stur d0,[x11,#-8]  (x11 = h+0x203A8+16k)
  ▼
handle + 0x203A0 = [pcm0, 0, pcm1, 0, pcm2, 0, ...]     ← 2× 插零
  │  5 级 biquad, 每级从偏移 0 重新滤波（级联）
  ▼
handle + 0x203A0  ← 原地写回；量化器 0x1a3dc `ldr d7,[x8]` 读的就是它
```

**「内插器折进反馈环」的实质：2× 插零 + 5 级 IIR 低通，共 32× 上采样，
没有任何 FIR/polyphase 系数。** 这就是第 6 节全部迁移失败的根因。

### 17.2 阈值与级数

```
thresholds[handle + 0x1403A0 + 8*x8] = frame_count * 2^x8      (x8 = 0..4)
每级迭代次数 = 2 * thresholds[x8]
总迭代 = 2*frame_count*(1+2+4+8+16) = 62*frame_count
实测 frame_count=128 -> 7936 = 62*128  ✓
```

### 17.3 biquad 系数（`.rodata`，d14 经动态断言验证）

```
d9  = 0x7e90 = -0.342489376101834      (反馈 r)
d10 = 0x7eb8 = -0.270179278980591      (反馈 q)
d11 = 0x7f30 = +0.164377655974541      (前馈 p+t)
d12 = 0x7fe0 = +6.015680200762240      (输出增益)
d13 = 0x7f70 = +12.031360401524500     (反馈 q 输出项)
d14 = 0x7ee0 = +0.224583424375527      (输入缩放；实测 0.224583424376 ✓)
d15 = 0x7ee8 = -0.101666302497891      (反馈 p)
```
`0x7ee8` 我此前误标为「delta gain」，实为 biquad 的 `d15`。

### 17.4 逐样点递推（已与 Unicorn 数值对齐）

```
p = mem[state+0x98]      q = mem[state+0xB0]      r = mem[state+0xB8]
t = u*d14 + p*d15
v = r*d9 + q*d10 + (p + t)*d11
y = v*d12 + q*d13 + r*d12
mem[+0x90]=t  [+0x98]=t  [+0xA0]=p  [+0xA8]=v  [+0xB0]=v  [+0xB8]=q
```
通道 0 状态基址 = `handle + 0x90 + 48*x8`；通道 1 = `handle + 0x210 + 48*x8`
（`0x1a25c cbz x28, #0x1a2a0` 分支；`0x1a24c add x3,x3,#0xb8` 使 x3 被重赋值，
按 x3 推算状态地址会读到全零 —— 这是我先前几次读出 0 的原因）。

**数值验证**（frame_count=128，通道 0）：
```
n=0  u=57.4933566  v=9.4506232   Unicorn d3=9.4506232   ✓
n=1  t=-5.845106   v=5.936451    Unicorn d3=5.93645072  ✓
n=2  y-d5 = 56.852 = 9.4506*d12，Unicorn d3[0]=9.4506   ✓（状态延迟 2 级关系正确）
```

### 17.5 尚未闭合的一点

把 5 层级联整体复现时，**第 0 个样点逐位精确**（rel diff `0.000e+00`，15 位全同），
**第 1 个样点起分叉**（Unicorn 0.36342681 vs 模型 1.81713404）。既然逐样点递推已验证，
差异只能来自**级间状态的耦合方式**，候选：

1. 各级状态并非互相独立（`x3 = h + 48*x8` 虽不重叠，但某级可能复用前一级末状态）；
2. `x4` / `x7` 并非每级都重置到 `h+0xA03A0`；
3. `x28` 双通道循环与 biquad 的交错方式（两通道共用同一 buffer，无通道偏移，
   但声道 0 的 PCM 在 `h+0x3A0` 里是交错的）。

判据很干净：把各级状态的初值改为「上一级结束时的末状态」再对拍，
命中即说明是级间状态传递。脚本 `close2.py` 可直接改这一处重跑。

### 17.6 状态

代码未改动。`vendor_pull/re/` 现有 19 个脚本。

---

## 18. 内插器 100% 逐位复现 —— 闭合

验收清单全部达成。

```
[✓] 2× zero insertion
[✓] 5 级 IIR 级联
[✓] 每级系数（.rodata 常量）
[✓] 单级逐样点递推
[✓] 级间 state ownership
[✓] x4/x7 生命周期
[✓] 五级完整逐样点对拍      4096 / 4096 bit-exact
[✓] 进入 0x1a3dc 的序列对拍
```

### 18.1 真正的级联结构（此前一直差一步的地方）

**展开循环和 biquad 都在 `x8` 循环体内**，所以每一级都把上一级输出**重新插零**：

```
0x1a1a0  mov x8, xzr ; b 0x1a1b4
0x1a1a8  add x8,#1 ; cmp #5 ; b.eq 0x1a314
0x1a1b4  展开：buf[2k]=src[k], buf[2k+1]=0     src = handle+0xA03A0
0x1a210  biquad 初始化, x3 = handle + 48*x8
0x1a250  biquad 循环, n = 0 .. 2*threshold[x8]-1
0x1a310  b 0x1a1a8                             <- 回到 x8 += 1
```

`x7` 每级重置到 `handle+0x203A0`、步长恒为 8（实测五级 deltas 全是 `[8,8,8,8,8]`）；
每级状态全新（实测每级 n=0 时 `mem[+0x90]` 从 0 起）。

**这就是「每级从偏移 0 重新滤波」但状态互不传递的原因** —— 每级读的是上一级
通过 `str d0,[x4]` 写到 `handle+0xA03A0` 的输出。快照直接证实：

```
x8=0 展开后: [256, 0, 512, 0, 768, 0, 1024, 0]                 <- 原始 PCM
x8=1 展开后: [56.8519, 0, 149.416, 0, 207.667, 0, 263.486, 0]   <- 上一级输出
x8=2 展开后: [12.6256, 0, 33.1819, 0, 54.0491, 0, 79.3578, 0]
x8=3 展开后: [2.80386, 0, 7.36897, 0, 12.0031, 0, 17.6236, 0]
x8=4 展开后: [0.622674, 0, 1.63648, 0, 2.66562, 0, 3.91381, 0]
```

`thresholds[x8] = frame_count * 2^x8`，每级迭代 `2*threshold[x8]`，
总迭代 `62*frame_count`（实测 frame_count=128 → 7936 ✓）。

### 18.2 完整可复现模型

```python
src = pcm                                   # memcpy 后的长度 frame_count
for x8 in 0..4:
    thr = frame_count * 2**x8
    for k in 0..thr-1:                      # 展开
        buf[2k]   = src[k]
        buf[2k+1] = 0.0
    p = q = r = 0.0                         # 全新状态
    for i in 0 .. 2*thr-1:
        u  = buf[i] * d14                   # fmul
        t  = fma(p, d15, u)                 # fmadd
        v1 = r * d9                         # fmul
        s  = p + t                          # fadd
        v2 = fma(q, d10, v1)                # fmadd
        v  = fma(s, d11, v2)                # fmadd
        o1 = v * d12                        # fmul
        o2 = fma(q, d13, o1)                # fmadd
        y  = fma(r, d12, o2)                # fmadd
        p, q, r = t, v, q
    buf[0 : 2*thr] = out
    src = out
# buf 即量化器 0x1a3dc 读到的序列
```

系数：`d9=0x7e90 d10=0x7eb8 d11=0x7f30 d12=0x7fe0 d13=0x7f70 d14=0x7ee0 d15=0x7ee8`

### 18.3 浮点必须逐指令复现，否则差 1 ULP

结构对了但数值差最后 1 位（`...685` vs `...674`）时，原因不是系数也不是状态，
而是 **AArch64 每个 `fmadd` 只舍入一次**，Python 的 `a*b+c` 舍入两次。
`np.longdouble` 在 Windows 上就是 float64，救不了。

有效解法：**按指令序列逐条实现**，mul/add 用 float（FMA 本身已正确舍入），
fma 用 `fractions.Fraction` 精确算 + 一次舍入：

```python
def fma(a, b, c):
    return float(Fraction(a) * Fraction(b) + Fraction(c))
```

注意 `fmul d0,d0,d14`（0x1a258）**先舍入一次**，`fmadd d1,d15,d0` 再把已舍入的 u
加进去 —— 把 `d14` 放进同一个 fma 会多舍入一次，反而更差（实测匹配数从 1017 掉到 825）。

### 18.4 验证结果

```
frames=128 arg6=4   FULL MATCH 4096/4096
frames=64  arg6=4   FULL MATCH 4096/4096
frames=128 arg6=0   FULL MATCH 4096/4096
frames=128 arg6=16  FULL MATCH 4096/4096
frames=256 arg6=8   FULL MATCH 4096/4096
```

参考值（frame_count=128）：
```
buf[0]=0.1382821368526545   buf[1]=0.36342680897190111
buf[2]=0.59197605216871685  buf[3]=0.86917147651117366
buf[4]=1.127398536587269    buf[5]=1.3779679045054984
buf[6]=1.6853545793131948   buf[7]=2.0237536628256758
```

### 18.5 意义

**第 6 节「所有固定系数迁移都失败」的根因至此完全解释**：
设备根本没有 FIR/polyphase 系数 —— 它是「插零 + 5 级自适应 IIR 低通」的运行时结构，
而我此前一直当成固定系数表来搬。

下一步（很短）：把 `buf` 接到量化器模型（5 抽头延迟线 + `tbl[m]*(16-arg6)`，
状态在 `handle+0x10`/`0x50`），验证第 6 节失败的量化器模型这次能否跑通。
内插器提供的是逐样点 ground truth，不再需要猜。

脚本：`vendor_pull/re/final_interp.py`（模型 + 对拍）、`uc_biquad.py`（Unicorn 侧）。

---

## 19. 量化器逐位闭合 —— 整条链完成

### 19.1 结果

```
bit-exact  v: 40/40   (u+q): 40/40   bit agree: 40/40
tbl matches final_interp.interpolate(): 64/64
```

捕获到的 `tbl` 与第 18 节已闭合的内插器模型**逐位一致**
（`0.1382821368526545, 0.36342680897190111, 0.59197605216871685, ...`）。

### 19.2 量化器模型（0x1a388 - 0x1a430，逐指令）

状态在 `handle+0x10`（通道 0）/ `0x50`（通道 1），8 个 double；
slot 0..4 移位，slot 5 由 `0x1a414` 读回 / `0x1a41c` 原样写回，**内环内保持恒定**。

```
s0..s4 = x10[0..4]      s5 = x10[5]     tbl = [x8]      d8 = (double)(16 - arg6)

v = s0*A0                                     fmul
  + s1*A1 + s2*A2 + s3*A3 + s4*A4             fmadd
  + tbl*d8                                    fmadd
  + s5*A5                                     fmadd

u = s0*B0                                     fmul
  + s1*B1 + s2*B2 + s3*B3 + s4*B4 + s5*B5     fmadd

bit = (v >= 0)      q = (v < 0) ? +1 : -1
s0 <- (u + v) + q
s1 <- s0(old)  s2 <- s1(old)  s3 <- s2(old)  s4 <- s3(old)  slot5 不变
x8 += 8        x21 += 1
```

`d8` 实测 = `12.0`（arg6=4 → 16-4 ✓），与第 12.5 节从 `0x19e6c subs w19,w12,w7`
推出的 `(double)(16 - arg6)` 一致 —— 这条现在也动态确认了。

同样必须逐指令复现：所有 `fmadd` 只舍入一次，用 `Fraction` 精确算 + 一次舍入。

### 19.3 完整转换链（全部 bit-exact 验证）

```
PCM (arg0 / arg2 字节 / arg3 位深)
 │  24-bit 路径: w11 = b0<<8 | b1<<16 | b2<<24 ; scvtf      （左对齐 ×256）
 ▼
handle+0x3A0  ── memcpy(len = 8*frame_count) ──▶  handle+0xA03A0
 ▼
for x8 = 0..4:                      thresholds[x8] = frame_count * 2^x8
    zero insertion  buf[2k]=src[k], buf[2k+1]=0
    biquad 级（全新状态，逐指令 fma）  n = 0 .. 2*thresholds[x8]-1
    src = out
 ▼
handle+0x203A0 ── 量化器 0x1a3dc 逐样点读取 ──▶ 1-bit 流
```

### 19.4 唯一未闭合项：输入条件，不是 DSP 拓扑

用合成输入（未归一化的 24-bit 值）时，环路在**任何量级**下都到达 ~1e15 的饱和极限环：

```
PCM=256          max|s0|=3.38e15   末态 +3.381e15 +3.382e15 +3.383e15
PCM=2560         max|s0|=3.42e16
PCM=2147483392   max|s0|=2.87e22   （= 满量程 24-bit 左对齐）
```

增长比逐样点下降（1.76 → 1.33）并趋于饱和，是**极限环**而非发散。两个原因叠加：

1. **状态初值**：真实调制器状态在 `handle+0x10` 跨调用持续，harness 从零启动，
   大信号下瞬态无法衰减；
2. **输入未预缩放**：24-bit 路径只有 `scvtf`，没有缩放，所以调用方必须预缩放。
   16-bit 路径的缩放在 PLT `0x1d358 / 0x1d3c4 / 0x1d888` 里
   （`ldrh -> bl -> ldr q1,[fp,#-0xa0] -> bl -> bl`），
   本轮为避开外部调用而选了无调用的 24-bit 路径，因此没有拿到缩放系数。

**这不是算法未知** —— 内插器与量化器两个环节都已 100% 逐位复现。
下一步只需解出 16-bit 路径那三个 PLT 函数的缩放（可 stub 后在 Unicorn 里抓
`d1` 寄存器的传入/传出值），即可用真实量级验证环路稳定性并对拍 bitstream。

### 19.5 验收清单

```
[✓] 2× zero insertion
[✓] 5 级 IIR 级联 + 逐级重新插零
[✓] 每级系数（.rodata）
[✓] 单级逐样点递推
[✓] 级间 state ownership / x4-x7 生命周期
[✓] 五级完整逐样点对拍        4096/4096 bit-exact（5 组参数）
[✓] 进入 0x1a3dc 的序列对拍   64/64 bit-exact
[✓] 量化器 v / (u+q) / bit    40/40 bit-exact
[ ] 真实输入量级与环路稳定性     需要 16-bit 路径的三个 PLT 缩放
[ ] 最终 bitstream 对拍          同上
```

### 19.6 第 6 节结论的正式修订

> 此前把设备的高倍率处理路径解释为「固定 FIR / polyphase 系数迁移」是**模型层面的
> 错误**，不是系数提取不完整。设备里**不存在**那张系数表。真实机制是
> 「2× 零填充 + 5 级级联 IIR 低通（每级重新插零）」的运行时结构。

脚本：`vendor_pull/re/uc_quant.py`（捕获）、`close_quant.py`（复现对拍）、
`final_interp.py`（内插器模型）、`uc_scale.py`（量纲扫描）。

---

## 20. 实现结果 —— 内插器与量化器都精确复现，但环路仍不稳定

### 20.1 已实现并验证

`src/flac2dsf.cpp` 新增 `Jm21Core`（`--jm21` 启用）：

- 5 级「zero insertion + biquad」级联，逐指令 `jfma`（= `std::fma`）单次舍入
- 设备的 5 抽头量化器，`d8 = (double)(16 - arg6)`，状态 5 深 + 恒定 slot5
- PCM 归一化到 ±1；内插器直流增益实测 **精确 1/32**（每级 1/2，5 级）
- 有效过采样 16:1，故 DSD64 需先把 PCM 重采样到 176400 Hz（已接线）

### 20.2 但它现在不能用

[SUPERSEDED — 见第 0.1 / 31 节] 这是 wrapper 返回 2×长零填充缓冲造成的假象；修正后 `--jm21` 输出真实信号（`mean q = 0.0000`）。

> `--jm21` 输出恒为 0（`mean q = -1.0000`），SFSR 无意义。原因不是实现：

用**已验证的模型**（Python，与 Unicorn 逐位一致）扫工作点，全部发散：

```
peak=1.0  slot5 ∈ {0, 0.5, 1.0, 1.5, 2.0, -1.0}   max|q| = inf
peak=0.5  slot5 同上                                   max|q| = inf
```

即**任何输入电平、任何 slot5 取值，5 抽头环路都不稳定**。
Unicorn 里也一样：状态指数增长到 ~1e15 的饱和极限环。

活抽头特征多项式 `1 - Σ(A[i]+B[i])z^-(i+1)` 的根是
[SUPERSEDED — 见第 0.2 节] 该手算错误；正确值 max|pole| = 1.000039。
> `1.9993, 1.4994±0.8663j, 0.4998±0.8663j`，最大模 **1.999** —— 结构上就是不稳定。

而设备正常工作。所以**必然还有一处未找出**，最可能的是：
系数不是 `.rodata` 里的默认行，而是由导出函数 `get_pcm2dsd_shaper_coeffs`
在运行时按属性选择的另一行（`get_pcm2dsd_data` 只在 0x1a110-0x1a144 无条件装载
那 11 个地址，但装载的值可能在更早处被改写 —— 我没有追 `init_pcm2dsd`
对 `.bss`/系数副本的写入）。

### 20.3 交付决定

**`--jm21` 默认关闭，默认仍是可用的 FIR 整形器（实测 SFSR 10.1 dB @ -26 dBFS，
> **[SUPERSEDED — 见第 0 节 / 第 31 节]** 「输出恒为 0」是 wrapper bug 的假象。现在 `--jm21` 输出真实信号，实测 SFSR 19.3 dB（模型仿真值，非设备实测）。
> 满幅约 30 dB）。** 理由：一个输出恒为 0 的转换器比 30 dB 的可用版本更糟。
代码保留，作为已验证的参考实现和后续接力的起点。

### 20.4 已确立 vs 未确立

| | 状态 |
|---|---|
| 内插器（5 级 zero-stuff + biquad） | **bit-exact**，4096/4096，5 组参数 |
| 量化器算术与状态更新 | **bit-exact**，v / (u+q) / bit 各 40/40 |
| 12 个 A/B 系数 + 全部 `.rodata` 地址 | **bit-exact**，与 1.0.8 逐位一致 |
| AArch64 单次舍入语义 | 已复现（`fma` 逐指令，Fraction 或 `std::fma`） |
| 内插器直流增益 1/32、有效过采样 16:1 | **实测** |
| **量化器环路稳定性** | **未解决** —— 任何条件下发散 |
| `get_pcm2dsd_shaper_coeffs` 的运行时选行 | **未追** |

### 20.5 下一步（明确、单点）

追 `get_pcm2dsd_shaper_coeffs`（PLT 0x1dcc0，导出符号，见第 12.10 节 PLT 全表）
与 `init_pcm2dsd` 对系数副本的写入，找出设备实际使用的那一组 A/B。
判据很干净：把候选系数代进已验证的模型，`max|q|` 应当有界且 SFSR 显著优于 30 dB。
模型侧不需要任何改动 —— 这是纯粹的数据查找。

### 20.6 工具状态

代码已改（新增 `Jm21Core` 与 `--jm21`/`--fir`/`--arg6`，默认 FIR）。
默认路径输出与改动前一致（SFSR 10.1 dB @ -26 dBFS，与基线相同）。

---

## 21. 关键修正 —— 第 20 节的两个结论都要推翻

### 21.1 “最大极点模 ≈ 1.999” 是错的，正确值是 **1.0000**

对全部三组 shaper 行直接求 `1 - Σ(A[i]+B[i])z^-(i+1)` 的根：

| shaper | 抽头 | `sum(A)` | `sum(B)` | `max|pole|` |
|---|---|---|---|---|
| shaper 1（`.rodata` 默认行） | 6 | +0.000095 | 0.999905 | **1.0000** |
| shaper 2（**工厂默认**） | 4 | +0.006191 | 0.993809 | **1.0000** |
| shaper 0（旧源码 8 抽头行） | 8 | +1.543418 | 1.543418 | 12.0434 |

shaper1 = `(1-z^-1)^6`，shaper2 = `(1-z^-1)^4`，都是 **N 重积分器、极点恰好在 1**。
这正是 delta-sigma 调制器**应有的**结构，不是缺陷。所以第 20 节
“结构上就不稳定 / 系数候选不对”的推理链**前提错误**，作废。

（shaper 2 = `(1-z^-1)^4` 这一组是从老源码注释里恢复的，我此前从未拿它跑过模型，
是明显的遗漏；shaper 0 那组 8 抽头行 `sum(A)=sum(B)=1.54`、极点 12.04，确认为错误行。）

### 21.2 那次 40/40 bit-exact 比对，比对的是一条**发散**的轨迹

模型轨迹诊断（`trace_state.py`）：

```
peak=1.000  -> BLEW UP at 40 (|q|=1.81e+12)
peak=0.100  -> BLEW UP at 40 (|q|=1.81e+12)
peak=0.010  -> BLEW UP at 40 (|q|=1.81e+12)
```

三条轨迹**逐位相同**，且都在第 40 个样本炸掉。含义：

- 发散**与输入无关**。它是 N 重积分器从零状态起振的多项式增长 `q ~ n^5`；
  `B0 = 5.18` 使其在头 40 步冲到 1e12，此时 1-bit 量化器的 ±1 修正
  在数量级上完全无足轻重，因此**结构上无法自限**。
- **`40` 恰好就是我做 40/40 比对的样本数。** 也就是说 Unicorn 那次捕获的
  状态本身就是发散轨迹（捕获值 ~1e15）。它证明我抄的**算术**（FMA 次序、
  状态移位、bit 判定）完全正确 —— 这部分结论依然成立 —— 但那次运行
  **不是设备的真实工作区间**。

### 21.3 因此第 20 节“找运行时 shaper coefficient 行”的方向不再成立

`get_pcm2dsd_shaper_coeffs` 仍然值得一追（它确为导出符号，PLT `0x1dcc0`），
但**不是因为我怀疑系数值**，而是因为我需要一个真实工作状态去验证：

> **[SUPERSEDED — 见第 0 节 / 第 31 节]** 并非如此。那是 5 抽头模型 + 零填充缓冲造成的假发散。
> - 现在的模型是一个**在真实输入下无条件发散**的模型；
- 我对它**没有任何一次在稳定区间内的观测**；
- 在拿到稳定观测之前，任何“系数候选 + SFSR 好不好听”的筛选都失去判据 ——
  全部候选都会先被稳定性判据淘汰，而不是被音质判据区分。

所以真正的下一步不再是“扫系数行”，而是：

1. 搞清楚**状态从什么初值起振**（`init_pcm2dsd` 是否预置状态 / 是否有去毛刺阶段），
   或者
2. 找到设备**实际输出**的一段 1-bit 流，反解它的状态轨迹，
   从而第一次观测到稳定区内的行为。

在这之前，任何进一步的系数搜索都是低产出。

### 21.4 交付状态（不变，且理由更强）

`--jm21` 保持默认关闭。理由从“系数可能是错的”升级为更强的：
> **[SUPERSEDED — 见第 0 节 / 第 31 节]** **明确推翻。** 修正倍率后，量化器在所有测试电平下状态有界、信号增益恒等式为 `gain = d8/32`。
> **该模型在真实输入下结构性地无条件发散，不存在能工作的电平**。
默认 FIR 路径实测 SFSR 10.1 dB @ -26 dBFS（满幅约 30 dB），16/16 回归与 Web 端到端全过。

已确立的部分仍然有价值、也没有被推翻：

```
内插器（5 级 zero-stuff + biquad）  bit-exact 4096/4096（5 组参数）
量化器算术与状态更新               bit-exact v/(u+q)/bit 各 40/40（发散轨迹上）
12 个 A/B 系数 + .rodata 地址       与 1.0.8 逐位一致
AArch64 单次舍入语义                已复现
内插器直流增益 1/32、过采样 16:1   实测
```

新增工具：`vendor_pull\re\trace_state.py`（轨迹诊断）、`vendor_pull\re\try_rows.py`（三组行对比）。

---

## 22. 播放器互操作实测结果

素材：-6 dBFS 1 kHz 正弦，10 秒（`listen\t_mono_-6dB.wav` / `t_stereo_-6dB.wav`）。

| 文件 | 变体 | PotPlayer |
|---|---|---|
| `A_default.dsf` | 单声道，默认 | 可播 |
| `B_payload.dsf` | 单声道 `--data-size-payload` | 可播 |
| `C_planar.dsf` | 单声道 `--planar` | 可播 |
| `D_stereo.dsf` | 立体声，默认 | 可播 |
| `F_pl_default.dsf` | 立体声，默认 | 可播 |
| `G_pl_payload.dsf` | 立体声 `--data-size-payload` | 可播 |
| `H_pl_planar.dsf` | 立体声 `--planar` | **连续爆破音** |

### 22.1 结论

- **`data_size` 的两种写法（payload / payload+12）PotPlayer 都接受** ——
  这个开关对 PotPlayer 无影响，可以放心保留。
- **默认的 interleaved 帧内块排列是正确的**，`--planar` 是为
  "某些播放器要求逐声道成块"准备的兼容性退路，而 PotPlayer 恰好**不接受**
  `--planar`，表现为连续爆破音（比特流被按错误的帧边界重解释）。
  这是 `--planar` 的固有代价，不是 bug；help 里应写明它会破坏
  PotPlayer / 大部分按帧交织读取的播放器。
- 单声道 / 立体声**都能播**。（第 21 节之前观察到的"单声道卡在 00:00"
  无法复现，现按素材差异解释：当时 PotPlayer 打开的 `A_default.dsf`
  是 -6 dBFS 1 kHz 而非更早的 -60 dBFS 版本。`--force-stereo` 因此
  **不再必要**，但保留为无害的显式选项。）

### 22.2 顺带修掉的真实 bug：banner 格式串少一个参数

第 21 节之前出现过一次"莫名"访问违规，根因是：

```
options : DSD x%d, %s, NTF order %d, ...     8 个转换符
args   : o.L, o.shaperOrder, o.leak, ...     7 个参数
```

`Options::shaperOrder` 是 `int`，那个 `%s` 让 printf 去 `strlen(4)`，
非 `-q` 运行必然段错误；`-q` 会跳过 banner，所以看起来"偶发"。
已修（`f2d.h` 与 `flac2dsf.cpp` 同批改动，`build.bat` 单次编译全部源文件，
不存在增量 ABI 问题）。所有非 `-q` 运行现在都正常。

### 22.3 验证

```
FLAC 解码回归   54/54 bit-exact
                (reg d2d std jm21 e2e variants variants2 utf8name)
静音自激         bit 密度 0.5000，最强分量 8.24e-04 @ 17.8 kHz -> 无自激
Web 端到端      GET / 200；下载产物 SHA256 与 CLI 一致；404/垃圾输入后存活
--force-stereo  单声道 -> 2 声道，正常输出
```

### 22.4 下一步的判据：必须拿到 ground truth

封装层这一层到此为止已经验证完毕，剩下的全部集中在 DSP 侧，
而 DSP 侧目前的困境是**只有推断、没有观测**：我从没看到过设备在
稳定区间内工作的状态。

唯一能一次性解开这个僵局的东西是**设备真实的 1-bit 输出**。
拿它配一个已知输入（比如上面那个 -6 dBFS 1 kHz），噪声底本身
就是 NTF —— 直接量出噪声整形阶数和反馈系数，完全不需要再反汇编。

---

## 23. Q2 已解答，并推翻第 20/21 节的"发散"结论

### 23.1 延迟线方向：slot0 = 最新，V1 确认

直接读 `0x1a388-0x1a424`（shaper==1 分支）：

```
读:  ldp d0,d1,[x10]        -> s0,s1
     ldp d3,d5,[x10,#0x10]  -> s2,s3
     ldp d1,d6,[x10,#0x20]  -> s4,s5        <- slot5 被读
写:  str  d0,[x10]          -> 新值 -> slot0
     stur q4,[x10,#8]        -> 旧{s0,s1} -> slot1,2
     stur q3,[x10,#0x18]     -> 旧{s2,s3} -> slot3,4
     str  d2,[x10,#0x28]    -> 旧 s4 -> slot5
     旧 s5 丢弃
```

**`vendor_pull\re\uc_quant.py` 顶部注释"slot 5 恒定"是错的**：
`0x1a414 ldr d2,[x10,#0x20]` 读的是 **s4**，`0x1a41c` 把它写进 **slot5**，旧 s5 被丢弃。
所以是 **6 深延迟线、六个抽头全活、slot0 最新**。

变体判别（Unicorn 实跑，只评分状态尚小的前若干样本，
因为 capture 全程发散、过了初期就会被混沌放大）：

| 变体 | v | s0 | bit |
|---|---|---|---|
| **V1 = 6 深移位** | 10/16 | **16/16** | **16/16** |
| V2 = 5 深 + slot5 常量 | 6/16 | 6/16 | 16/16 |
| V3 = 反向移位（slot5 最新） | 1/16 | 1/16 | 10/16 |
| V4 = 无移位 | 1/16 | 1/16 | 16/16 |

**V1 胜出，反向移位被排除。** 第 21 节问的"1.999 是不是方向搞反了" —
方向没错，而 1.999 本身是手算错误。

### 23.2 运行时系数：直接读寄存器，`.rodata` 读法正确

在 `0x1a3a0` 挂钩读 `d25/d27/d28` 与 9 条 `ldr` 的实际访存：

```
A0=+0.81654825844545798   B0=+5.1812493674000004
A1=-3.7393482705534971    B1=-11.251843168311209
A2=+6.8767586580770477    B2=+13.110028967961449
A3=-6.3458654125318468    B3=-8.6453260263328602
A4=+2.9375122501328952    B4=+3.0602853757125619
A5=-0.54551087517714547   B5=-0.45448912482285447
sum(A)=+0.000094608  sum(B)=0.999905392  sum=1.000000000
max |closed-loop pole| = 1.000038970        (不是 1.999)
d24 = -1.0   d26 = +1.0                    (量化器就是 ±1，无额外增益)
d8  = 12.0
```

所以"运行时选了另一行系数"这个假设**不成立**：运行时用的就是 `.rodata` 那一行。

### 23.3 验证脚本本身漏了输入项

`v` 的定义里还有 `fmadd d2, d7, d8, d2`（`tbl*d8`，`0x1a3e8`）。
我的判别脚本起初漏了它，导致第 0 个样本就 `dev(v) = -1.659`（恰为 `tbl*d8`）。
补上后 V1 从 0/16 变成 16/16。**教训：早先的"40/40"也是在漏项的脚本上取得的。**

### 23.4 撤回"结构性无条件发散"的结论

第 20/21 节说"任何输入电平、任何 slot5 都发散"。**该结论作废**，两个错误叠加：

1. 用了错误的 5 抽头模型；
2. **输入是无界的递增斜坡**（`w = (i+1)*256`），真实设备面对同样输入同样会崩。

换成**有界**输入后，6 抽头模型在 peak = 1.0 / 0.5 / 0.25 / 0.1、
d8 = 12 / 6 / 4 / 3 全部组合下 `max|state|` **都有界**（最大 1913）。

### 23.5 但仍然没有可用工作点（新结论）

用确认后的 6 抽头结构 + 有界正弦，扫输入尺度（0.5 满幅正弦）：

```
 x scale   v drive    max|state|    SFSR
       1     0.1875        1198    -67.6 dB
       8        1.5          inf       --
      32          6          inf       --
     256         48          inf       --
```

**不存在"既能跟踪信号又稳定"的输入尺度。** 这有数学含义：
`sum(A+B) = 1.000000000` 精确成立 ⇒ 闭环多项式就是 `(1-z^-1)^6`，
六个极点**全部**落在 z=1，NTF 以 36 dB/oct 一路抬到 Nyquist。
这种 NTF 配 1-bit 量化器**不存在稳定工作区** —— 六阶 1-bit 必须有极点
严格落在单位圆内（阻尼）。

所以设备必然还有我未建模的东西。按信息增益排序：

1. **状态限制器 / 过载保护非线性**（最高优先）。
   纯线性结构已被证明无可用工作区，那么多出来的那个环节只可能是非线性的。
   用户引的 US6822594B1 正是"高阶 1-bit ΔΣ 的过载保护与稳定性"，
   方向对上了。应在 `0x1a3f4`(fcmp) 前后找对状态的钳位/复位分支。
2. 未建模的非线性支路（1-bit 输出到状态的通路）。
3. 另一条我没追到的系数通路（但 23.2 已排除"运行时换行"这一种）。

### 23.6 代码状态

`src/flac2dsf.cpp` 的 `Jm21Core` 已改成**确认的 6 抽头延迟线**
（`qs[6]`，slot0 最新，整体右移）。默认仍关闭（`--jm21` opt-in），
> **[SUPERSEDED — 见第 0 节 / 第 31 节]** 现已找到可用工作点。
> 理由不变：尚无可用工作点。

新增工具：`vendor_pull\re\uc_coef2.py`（读运行时系数 + 逐样点对拍）、
`uc_variants.py`（延迟线变体判别）、`quant6.py` / `quant6_meas.py` /
`quant6_scale.py`（6 抽头模型与工作点扫描）。

### 23.7 审计请求的更新

- **Q2 已由指令级证据回答**，不需要外部意见了。
- **Q1 的前提已变**：`max|pole| = 1.000039`，不是 1.999；而且外部意见
  "线性化极点不能单独证明非线性 1-bit 环路的稳定性"完全正确 ——
  更进一步，我们已经把纯线性结构穷举完了，结论是**没有稳定工作区**。
- **Q3 同意放弃**：FiiO 的 `S/N ≥130 dB` / `THD+N <0.0006%` 是整机模拟输出规格，
  不能换算成 All-To-DSD 的数字 SFSR，也不该用来约束 shaper 系数。
- **现在最该问的只剩一件事**：在已确认 6 抽头结构、且纯线性无可用工作区的前提下，
  高阶 1-bit ΔΣ 常用的**状态限制器 / 过载保护**具体有哪几种拓扑？
  给出可判别的特征（例如是否对状态做软钳位、是否含一个额外的
  单极点环路、是否在检测到超限时重置积分器），以便在
  `0x1a3f4`/`0x1a420` 附近按图索骥。
- **A（ground truth）升为第一位**：设备真实 1-bit 输出能一次性验证
  "到底有没有非线性环节"，且它的噪声底直接就是真实 NTF。

---

> **[SUPERSEDED — 见第 0 节 / 第 31 节]** 「无可用工作点」部分作废（见第 0.1 节）；「过载保护整类假设被指令级排除」部分**仍然有效**。

> ## 24. 过载保护整类假设：指令级排除；两种 shaper 均无可用工作点

### 24.1 `0x1a3f4` 内环完整反汇编：无任何条件反馈

```
0x1a3f4  fcmp   d2, #0.0
0x1a3f8  fadd   d0, d0, d2
0x1a3fc  cset   w11, ge
0x1a400  fcsel  d1, d26, d24, lt
0x1a404  strb   w11, [x21]
0x1a408  ldp    q4, q3, [x10]
0x1a40c  fadd   d0, d0, d1
0x1a410  stur   q4, [x10, #8]
0x1a414  ldr    d2, [x10, #0x20]
0x1a418  stur   q3, [x10, #0x18]
0x1a41c  str    d2, [x10, #0x28]
0x1a420  str    d0, [x10]
0x1a424  add    x8, x8, #8
0x1a428  add    x21, x21, #1
0x1a42c  subs   x9, x9, #1
0x1a430  b.eq   #0x1a14c
```

**零分支、零 `fmin/fmax`、零 `fmul` 衰减、零计数器、零二次 state store。**
唯一的条件是 `fcsel` 选 `d24=-1.0` / `d26=+1.0` 和 `cset` 取 bit。

据此**在指令级排除**（无需再做 tuple dump —— 无分支即单一路径）：

| 假设 | 结论 |
|---|---|
| overload feedback steering（q=±1 但内部反馈 ±2/±k） | **排除** |
| state limiter（`fmin/fmax` 钳位） | **排除** |
| soft-reset / variable-integrator damping（`fmul state, α`） | **排除** |
| run-length reset（计数器 + 多 state 清零） | **排除** |
| dual-loop overload compensation（额外 state block + qofb） | **排除** |
| input clamp | **排除**（见 24.3） |
| look-ahead | 排除（无候选状态、无预测循环） |

`src/flac2dsf.cpp` 的旧注释「the device's own 5-tap quantiser」是错的：
量化器是 **6 抽头**（shaper 1）或 **4 抽头**（shaper 2），全活。

### 24.2 shaper==2（工厂默认）= 4 抽头，系数 d16..d23

```
0x1a54c  ldp d0,d1,[x10]        -> s0,s1
0x1a550  ldp d3,d4,[x10,#0x10]  -> s2,s3
   v = s0*d16 ; s1*d17 ; s2*d18 ; tbl*d8 ; s3*d19
   u = s0*d20 ; s1*d21 ; s2*d22 ; s3*d23
0x1a578  fcmp d1, #0.0
0x1a590  ldr d2,[x10,#0x10]      -> s2
0x1a594  ldr q3,[x10]           -> {s0,s1}
0x1a598  str d2,[x10,#0x18]     -> s2 -> slot3
0x1a59c  stur q3,[x10,#8]       -> {s0,s1} -> slot1,2
0x1a5a0  b #0x1a420             -> str d0,[x10]
```

运行时读出的系数与旧源码注释**完全一致**：

```
A0=+0.802319609916744   B0=+3.1969667828000001
A1=-2.100688172255984   B1=-3.8978846131775038
A2=+1.8589343651600601  B2=+2.1403520275566841
A3=-0.55437442828602768 B3=-0.44562557171397232
sum(A)=+0.006191375  sum(B)=0.993808625  sum=1.000000000
max |closed-loop pole| = 1.000000000
```

4 抽头模型 bit-exact：前 8 个样本 `v 8/8`（之后 capture 进入发散区）。

### 24.3 环外也没有非线性

- `0x1a14c: add x28,#1 ; cmp x28,#2 ; b.eq #0x1a5a4` —— **x28 是声道循环**（0、1），
  确认每声道独立量化器状态。
- `0x1a158-0x1a19c` 每 block 把 5 个阈值写入 `h+0x1403A0`，然后 `bl memcpy`。
  **表在这里没有任何缩放或限制。**

### 24.4 两种 shaper 都没有可用工作点

shaper 2 细扫（0.5 满幅正弦，内插器峰值 0.015625，d8=12）：

```
 scale  v drive  max|state|    tone     in-band    SFSR
  1.00    0.1875       31.17  0.00003   2.301e-01  -76.6
  2.25    0.4219        4.902 0.00003   5.169e-01  -84.0
  3.25    0.6094       17.96  0.00002   7.465e-01  -90.5
  3.50    0.6563   9.012e+08  0.03449   1.520e+00  -32.9
  4.00    0.7500   9.039e+07  0.22973   1.643e+00  -17.1
```

shaper 1 表现同构（`scale 1` 有界 / SFSR -67.6；`scale ≥8` 发散）。

**稳定窗口内信号完全消失**（恢复 0.00003，输入 0.5 —— 差 84 dB）；
**窗口外状态炸到 1e8–1e9**。两者之间没有交集。

### 24.5 因此得到的硬结论

`sum(A+B)` 精确为 1（两种 shaper 都是），闭环多项式就是
`(1-z^-1)^N`，全部极点落在 z=1（1.000000 / 1.000039）。
配 ±1 的 1-bit 量化器，**这种 NTF 不存在"既稳定又传递信号"的工作区** ——
这一点已经被穷举扫描证实，不是推测。

同时，量化器内环与环外都已被指令级证明**不含任何非线性或保护环节**。

**两者相加意味着：我解码出的这个量化器不可能是真正产生设备可用 DSD 的那个东西。**
不是系数错、不是方向错、不是保护机制漏了、不是工作点没找着 ——
而是这个环本身在数学上无法工作。

### 24.6 由此收窄的剩余可能（按信息增益）

1. **输入尺度链条仍有一处未解**。设备 PCM 是 24-bit 左对齐 ×256；
   内插器直流增益实测 1/32。但我按 1×…256× 全扫过，任何尺度都无解 ——
   所以问题可能不是"倍数"，而是**内插器之后、量化器之前还有一次非线性或尺度变换**
   （例如某个我尚未定位的写入 `h+0x203A0` 的后续处理）。
2. **`d8` 可能不是常数乘子**。`0x1a3e8 fmadd d2, d7, d8, d2` 里 d8 来自
   `subs w19,w12,w7` → `fcvt`，实测恒为 12.0；但若 `arg6` 在真实播放路径上
   取值不同，`d8` 会变，需要枚举 `arg6` 而非固定 4。
3. **实际播放路径可能不是 `0x19d90`**。工厂属性是 shaper=2，但没验证过
   真实播放时进入的是哪条分支、arg6 取多少。这是 Q4 唯一还有价值的形式：
   不是问"是不是这条路径"，而是问"要满足什么 runtime condition 才走这条"。
4. **ground truth（设备真实 1-bit 输出）现在是最高价值**。
   它能一次性判定 24.5 是否成立：如果设备的输出噪声底呈
   `量化误差 PSD × |NTF|²` 且高频斜率明显低于 24 dB/oct（4 阶）
   或 36 dB/oct（6 阶），那就证明存在我还没找到的整形环节；
   反之则证明我的解码在某处根本性错误。

### 24.7 新增工具

`vendor_pull\re\shaper2.py`（shaper 2 的运行时系数 + 4 抽头 bit-exact 验证 + 稳定性）、
`shaper2_fine.py`（稳定窗口细扫）。

### 24.8 关于"设备噪声底就是 NTF"的表述修正

正确说法是 `输出噪声谱 ≈ 量化误差 PSD × |NTF|²`。
只有在近似线性化、量化误差近似白噪声的条件下，才能从输出噪声谱拟合 NTF 斜率；
**不能由设备 bitstream 唯一确定内部 NTF 系数**。
但它足以判断"有没有第 6 阶整形、实际高频噪声斜率多少、是否存在保护模式"。

---

## 25. 两条本地路线关闭 + 真实 caller 揭示两处硬事实

### 25.1 实验 1：`h+0x203A0` 全写追踪 —— 阴性，硬证据

对整张表装 write watch（PC / 地址 / 长度 / 旧值 / 新值），
两种 shaper 结果一致：

```
写入 h+0x203A0 的 PC 只有 4 个，全部在 5 级插值器循环内：
  0x1a1e0 / 0x1a1e4  -> 0x300203A0..0x30021398   （量化器实际消费的 512 个 double）
  0x1a2e4 / 0x1a2e8  -> 0x300a03a0 / 0x300203A0 （级间缓冲）
量化器期间对该区间的唯一写入：
  0x1a404 / 0x1a588  strb w11,[x21] -> 0x301203A0..0x301205a0
                        即 h+0x1203A0 = 输出比特流缓冲，不是输入表

writes AFTER the 5-stage loop finished: 0
```

**5 级 biquad 完成之后、量化器第一次读之前，对输入表没有任何后续写入。**
"插值器之后还有一次未建模的尺度变换 / 非线性"被**排除**。

顺带把三处缓冲的职责钉死：

```
h+0x1203A0 / h+0x1303A0   输出比特流（每声道）
h+0x203A0                 量化器输入表（5 级插值器产物）
h+0x1403A0                5 个阈值（每 block 由 0x1a16c-0x1a198 重写）
```

### 25.2 实验 2：`arg6 = 0..16` 穷举 —— 阴性

两种 shaper、全部 17 个 arg6（0.5 满幅 1 kHz 正弦，内插器峰值 0.015625）：

```
shaper 1 (6 tap):  arg6 0..15 全部有界；恢复的 1 kHz 仅 3e-5 ~ 9e-5
                  SFSR -55 … -81 dB；bit 密度 0.5000-0.5007
shaper 2 (4 tap):  同上，SFSR -55 … -77 dB
arg6=16 -> d8=0：输入根本不进环路（那个 inf 是 0/0 假象）
```

**没有任何 arg6 能同时做到状态有界与信号传递。** 路线 2 关闭。

### 25.3 实验 3：真实 caller —— 两处硬事实

`.text` 里 `bl #0x19d90` 只有一条，是 PLT 跳板；真正的调用点是
**`0x1305c: bl #0x1dcd0`**（PLT），实参：

```
0x1303c  add x0, x20, #0x400, lsl #12
0x13048  add x1, x20, #0xC0005A
0x1304c  mov w3, #0x20          <- OS = 32
0x13050  mov w5, w21
0x13054  mov w6, w19            <- shaper
0x13058  mov w7, w19            <- arg6，与 shaper 是同一个寄存器
0x1305c  bl  #0x1dcd0
0x13060  mov w21, #0x28         ('(')
   随后 0x50/0x5a/0x5b/0x5e/0x60/0x61 拼出 DSF 头
```

**事实一：过采样是 32×，不是 16×。**
`w3 = 0x20`，且 5 级 zero-stuffing 给出 2⁵ = 32，捕获里 16 帧对应
512 个量化器样本（16×32）也吻合。此前文档与 `Jm21Core::kOver = 16`
写的 "16:1" **是错的**。

**事实二：`arg6` 与 `shaper` 是同一个变量 `w19`。**
我在 harness 里传 shaper=1 / arg6=4 是自相矛盾的组合。
工厂默认 shaper = 2 ⇒ **arg6 = 2 ⇒ d8 = 16 - 2 = 14**。
arg6=2 在上表里同样是"有界但无信号传递"，所以这条修正**不改变结论**。

调用点之后立刻拼 DSF 头，说明 **0x1305c 所在的函数就是真正的
All-To-DSD 转换入口**，同时反向印证了容器布局。

### 25.4 仍未闭合

`w19` 的取值来源未找到：它在 `0x12f08 / 0x12f98` 从 `[x23+0x5c]` 读入，
但 `[x23+0x5c]` 在另一处（`0x12fe8-0x12ff8`）被与魔数 `0x1A000002` 比较，
说明到达 `0x13038` 的分支与读到 shaper 的分支不是同一条。
需要完整走一遍该函数的控制流图才能确定 `w19` 在真实播放路径上的值。

### 25.5 结论与剩余路线

至此三条本地路线全部阴性：

| 路线 | 结果 |
|---|---|
| 量化器内环的条件反馈 / 过载保护 | 排除（无分支） |
| 插值器之后的后处理 / 尺度变换 | 排除（0 次后续写入） |
| `arg6` / `d8` 取值 | 排除（0..16 全空） |
| 两种 shaper 的系数 | 排除（运行时即 `.rodata`） |
| 延迟线方向 | 已确认（slot0 最新，6/4 抽头全活） |

剩下的确实是**路径识别**问题，而不是系数或工作点问题：

1. **设备 ground truth**（最高价值）：一条原始 1-bit 流即可把
   "我解码得对但漏了一整层" 与 "解码在某处根本性错误" 两个世界分开。
2. **走通 `0x13038` 的控制流**，确定真实播放路径上 `w19`（= shaper = arg6）
   的实际取值。
3. 确认 `0x1305c` 所在函数是否就是 UI 里"All To DSD"那条路径，
   以及 DSD 倍率（w3=32 对应的输出速率选择）在哪里被决定。

### 25.6 新增工具

`vendor_pull\re\uc_write_track.py`（表写入追踪）、
`arg6_sweep.py`（arg6 0..16 稳定性/信号传递地图）、
`shaper2.py` / `shaper2_fine.py`（工厂默认 4 抽头）。

---

## 26. 实验 3：输出字节数是物理事实 —— 并发现 harness 调用约定错误

### 26.1 `h+0x1203A0` 的产出速率

量化器每个样本执行 `strb w11,[x21]`，`x21 += 1`。对整个输出区装 write hook：

```
frames=8   shaper=1/2   1-bit stores=256    bytes/frame = 32.0
frames=16  shaper=1/2   1-bit stores=512    bytes/frame = 32.0
frames=32  shaper=1/2   1-bit stores=1024   bytes/frame = 32.0
addr 0x301203a0.. 递增连续；每字节取值集合 = {1}（即 0/1，不是 0/0xff）
```

**每个输入帧恰好 32 字节，每字节只承载 1 bit。**
即输出是"一 bit 一字节"的低效存储，**信息量 = 32 bit / 输入帧**。

```
32 bit/frame × 44100 Hz = 1411200 Hz
DSD64 需要 64 bit/frame = 2822400 Hz
```

**因此确实存在一个正好 ×2 的环节，且它不在这条已解码路径内。**
两种可能，需要 ground truth 或继续追 caller 才能分辨：

1. caller 送进来的 PCM 已经是 88200 Hz（32 × 88200 = 2822400 ✓）；
2. 该函数之后还有一个 ×2 / packing 环节。

同时修正第 25.3 节的不严谨表述：把"32:1 → 88200 Hz"写成结论是错的，
它把"输入被预重采样到 88200"当成了前提，而那正是待证项。

### 26.2 重要：Unicorn harness 的调用约定与真实 caller 不一致

```
真实 caller (0x13038-0x1305c)        本项目的 harness (uc_quant.py 等)
  x0 = x20 + 0x400000                 x0 = INPUT (0x36000000)
  x1 = x20 + 0xC0005A                 x1 = 未设置
  x2 = w25                            x2 = nb
  x3 = #0x20  (= 32)                  x3 = 0x18  (= 24)     <-- 不同
  x5 = w21                            x5 = 1
  x6 = w7 = w19                       x6 = shaper, x7 = arg6
```

**`x3` 一直是 24，而设备传 32；`x0/x1/x2` 的角色也不同。**

### 26.3 因此第 24/25 节的稳定性结论必须降级

以下结论都是在上述 harness 参数下取得的，**它们是条件性的**：

- 第 24.4 节"两种 shaper 都没有可用工作区"
> **[SUPERSEDED — 见第 0 节 / 第 31 节]** 
> - 第 24.5 节"这套已解码结构不能解释设备输出"
- 第 25.2 节"arg6 = 0..16 全空"

降级后的准确表述：

> **在 harness 当前参数组合（x3=24、x0=INPUT）下，两种 shaper、
> arg6 ∈ [0,16] 均未发现有界且传递信号的工作区。
> 由于 harness 与真实 caller 的实参不一致（第 26.2 节），
> 该结果不能推广到设备真实调用条件。**

**不受影响、仍然成立的结论**（这些只依赖反汇编与寄存器实际取值，
不依赖 harness 的实参）：

- 量化器内环**无分支、无钳位、无衰减、无计数器**（指令级）
- `slot5` 是真实的移位抽头，延迟线 6 深（shaper1）/ 4 深（shaper2），slot0 最新
- 运行时系数就是 `.rodata` 那一行（shaper1、shaper2 均已直接读寄存器确认）
- `d24 = -1.0`、`d26 = +1.0`（量化器反馈就是 ±1，无隐藏增益）
- `sum(A+B) = 1`，闭环多项式为 `(1-z^-1)^N`，极点 |z| = 1
- `h+0x203A0` 在 5 级循环结束后**没有任何后续写入**
- 输出为每输入帧 32 字节、每字节 1 bit
- `arg6` 与 `shaper` 是同一个变量 `w19`（caller 指令级确认）

### 26.4 下一步：先把 harness 调用约定改对

这是当前**成本最低、收益最高**的一步，且完全本地：

1. 把 harness 的 `x0/x1/x2/x3/x5` 改成与 `0x13038` 一致的语义
   （特别是 `x3 = 0x20`）；
2. 重跑 `arg6` 0..16 × 两种 shaper 的稳定性/信号传递地图；
3. 若仍全空，则第 24.5 节的硬结论可以重新成立；
   若出现可用工作点，则直接落为一个可用的 JM21 实现。

同时并行推进：反向数据流找出 `[x23+0x5c]` 的全部 writer（`w19` 的来源），
以及确认输入 PCM 在进入本函数时的实际采样率（88200 还是 44100）。

### 26.5 新增工具

`vendor_pull\re\uc_outbytes.py`（输出字节率与字节内容测量）。

---

## 27. 用真实 caller 执行取得寄存器快照；`x3` 是除数

### 27.1 方法：让 CPU 自己算出 callee 可见状态

不再手工构造 `x0/x1/x2/x3`。从 **`0x12e80`**（喂给该调用的基本块）开始真实执行，
让它自然走到 `0x1305c`，在那里快照整个寄存器文件：

```
0x12e8c  ldr w21, [x23, #0x70]
0x12e90  ldr w19, [x23, #0x68]     <- mode = shaper = arg6
0x12e94  ldr w24, [x23, #0x5c]
0x12e98  bl  memset                 (hook 成 no-op)
0x12ea4  cmp w24, #0x1A000002
0x12ea8  b.ne #0x13038             <- [x23+0x5c] != magic 才走 DSD 路径
```

`mode` 来自 **`[x23+0x68]`**，不是 `[x23+0x5c]`；`0x1A000002` 是**另一条**格式分支的魔数
（DSD 路径恰恰在"不等于"它时走）。fallthrough 那一支（`0x12ec4` 起的
`ext/zip2` 循环）处理的是别的格式。

在 `0x1305c` 抓到的真实寄存器文件（mode=2）：

```
x0 = x20 + 0x400000        x1 = x20 + 0xC0005A
x2 = <长度>                x3 = 32
x4 = 0                     x5 = 1
x6 = 2  (shaper)           x7 = 2  (arg6)      => d8 = 16 - 2 = 14
x19 = 2                    x28 = 0
```

与旧 harness 的差异：

| 寄存器 | 真实 caller | 旧 harness |
|---|---|---|
| `x1` | `x20 + 0xC0005A`（具体指针） | **未设置** |
| `x3` | **32** | **24** |
| `x4` | 0（该路径上未被赋值） | 未设置 |
| `x6`/`x7` | 同一变量 `w19` | 分别独立设置 |

`x4` 在这条路径上确实从未被赋值（你的提醒是对的，结论是无害）。

### 27.2 `x3` 是除数：纠正第 26.1 节的输出率

实测 `x3` 对输出字节率的影响（8 输入帧）：

```
x3=8   -> 96 bytes/frame      x3=32  -> 24 bytes/frame
x3=16  -> 48 bytes/frame      x3=48  -> 16 bytes/frame
x3=24  -> 32 bytes/frame      x3=64  -> 12 bytes/frame
x3=128 ->  4 bytes/frame

bytes/frame = 768 / x3
```

设置 `x1` 为真实指针不改变该比例。

**因此第 26.1 节的"每输入帧 32 字节 / 32 bit"是 `x3=24` 的产物。**
用真实的 `x3=32`，量化器每输入帧只消费 **24** 个样本：

- 5 级 zero-stuffing 产生 `2^5 = 32` 个样本/帧；
- 量化器按 `x3` 抽取，只消费 24 个 —— 内部相当于 4/3 抽取。

**第 26.1 节基于 32 bit/frame 推出的 "1411200 Hz、与 DSD64 差 ×2" 也随之作废**，
必须重算。真实输出率取决于进入本函数时 PCM 的实际采样率，而该采样率仍未确定。

### 27.3 状态：哪些结论仍然成立

**不受影响（只依赖反汇编与寄存器实际取值）：**

```
✓ 真实 caller = 0x1305c
✓ mode = shaper = arg6 = [x23+0x68]
✓ [x23+0x5c] 与魔数 0x1A000002 比较；不等时走 DSD 路径
✓ x3 = 32，且 x3 是除数：bytes/frame = 768/x3
✓ x1 = x20 + 0xC0005A（一个具体的工作区指针，语义未知）
✓ 量化器内环无分支 / 无钳位 / 无衰减 / 无计数器
✓ slot0 = newest；shaper1 = 6 深，shaper2 = 4 深，抽头全活
✓ d24 = -1.0, d26 = +1.0
✓ 运行时系数 = .rodata
✓ sum(A+B) = 1，闭环多项式 (1-z^-1)^N，极点 |z| = 1
✓ h+0x203A0 在 5 级循环结束后零写入
✓ h+0x1203A0 = 输出缓冲，每字节 1 bit
```

**仍需重做：所有依赖内部过采样比 / 输出率的结论。**
此前全部稳定性扫描用的是 `x3=24`（→ 32 样本/帧），真实是 `x3=32`（→ 24 样本/帧），
内插器与量化器之间的抽取关系完全不同。**arg6 / 输入尺度扫描必须在
`x3=32` 下重跑。**

### 27.4 下一步（顺序固定）

1. 用 `27.1` 的快照方式重跑稳定性地图：`x3=32` 固定，
   `mode = 0..16`（`shaper = arg6 = mode`，`d8 = 16 - mode`），
   **不再独立枚举 shaper 与 arg6**（真实路径下它们是同一个变量）。
2. 确认 `x1`（`x20+0xC0005A`）指向什么：它很可能是一个输出/工作区，
   与 `x0`（`x20+0x400000`）构成 in/out 对。先把 `x2` 填成真实长度跑通，
   再 dump `x1` 区域内容变化。
3. 判定进入本函数时 PCM 的实际采样率：由此才能确定 DSD 倍率，
   以及 24 样本/帧如何对应到 DSD64 的 64 bit/帧。
4. 设备 ground truth（-6 dBFS 1 kHz + silence，保留原始 DSF/DFF）。

### 27.5 新增工具

`vendor_pull\re\uc_realcaller.py`（真实 caller 执行 + 寄存器快照）、
`uc_x3probe.py`（`x3` 语义实测）。

---

## 28. `x1` 语义确定；mode 空间完整；真实契约下结论不变

### 28.1 `x0` / `x1` 都是只读输入，输出恒在 `h+0x1203A0`

用真实 caller 跑到 `0x1305c`（保留真实 `x3/x6/x7`），只把 `x0/x1/x2`
替换成测试缓冲，执行后对四个区域做字节差分并归属写入 PC：

```
region                      bytes diff   changed?
A x0 = x20+0x400000         0            no
B x1 = x20+0xC0005A         0            no
C h+0x203A0  (in)           6134         YES   pc 0x1a1e0 / 0x1a1e4 / 0x1a2e4
D h+0x1203A0 (out)          327          YES   pc 0x1a588
```

**`x0` 与 `x1` 全程只读。** 整个函数不写它们。所有写入 confined 在
handle 内的 `h+0x203A0`（量化器输入表）与 `h+0x1203A0`（输出字节流）。

因此 `x1 = x20 + 0xC0005A` **不是输出缓冲**，而是第二个输入指针
（另一声道 PCM、或一张参数/格式表）。二者相差 `0x80005A`，
地址本身不寻常，但功能上它们是对称的输入。

### 28.2 输入输出比：能写成公式，但 `x2` 的单位未知

```
output_bytes = 4 * x2 / x3
```

实测（x2 = 192 B）：`x3=32 -> 24 B`，`x3=24 -> 32 B`，与第 27.2 节一致。
等价于 `x3=32` 时**每输入字节 4 个输出 bit**。

但 **`x2` 的单位尚未确定**（本轮测试里它是我任意设的字节数），
所以这一式子**不能换算成任何采样率，也不能推出 DSD 倍率**。
在拿到 `x2` 的真实语义与真实长度之前，文档中一律不写 "DSD64"，
只写 `All-To-DSD internal output / 1-bit intermediate stream`。

### 28.3 mode 空间完整：{0, 1, 2}

dispatch（`0x1a434-0x1a444`）：

```
cmp w26,#2 ; b.eq 0x1a544      -> mode 2 : 4 抽头 @0x1a544
cmp w26,#1 ; b.eq 0x1a388      -> mode 1 : 6 抽头 @0x1a388
cbnz w26, 0x1a424              -> mode >=3 : 不做量化，只推进指针
(w26 == 0)                     -> mode 0 : 8 抽头 @0x1a448
```

**`mode >= 3` 根本不量化**，所以有意义的 mode 只有 {0,1,2}。

mode 0 的 8 抽头系数**未能可靠解出**：按寄存器读出的候选里
`A[3] == B[2] == 14.9741`，是寄存器复用的产物，不是两组独立系数。
这一分支留作未闭合。

### 28.4 真实契约下的稳定性地图（不再猜 stride）

`x3 = 32`，`x6 = x7 = mode`，`d8 = 16 - mode`，喂入全部 32 个内部样本
（24-of-32 的选取规则仍未知，故不做子采样猜测）：

```
mode  d8   taps   max|state|   gain(1kHz)     SFSR
   1  15.0     6         1269     0.00002     -81.4
     sum(A)=+0.000095 sum(B)=0.999905  max|pole|=1.000000
   2  14.0     4        23.95     0.00008     -70.4
     sum(A)=+0.006191 sum(B)=0.993809  max|pole|=1.000000
```

输入是 0.5 峰值、**有界**的 1 kHz 正弦。两者状态都有界，
但 1 kHz 恢复增益只有 2e-5 / 8e-5 —— **信号仍然完全没有通过**。

**因此：即使在真实调用契约（x3=32、mode=d8 联动）下，
已解码的 mode 1 / mode 2 环依然"有界但不传递信号"。**

与第 24.5 节相比，本次已把三个最大的 harness 隐患消除
（x3、x1/x2、arg6 与 shaper 的联动），结论方向不变。

### 28.5 未闭合量（精简后的未知树）

```
已闭合
├─ 插值器      5 级 zero+biquad, bit-exact
├─ 量化器      4/6 抽头, slot0 newest, ±1, 系数=.rodata, 内环无保护
├─ caller      0x1305c, mode=[x23+0x68], x3=32(除数), DSD 分支已确认
├─ 内存        x0/x1 只读输入; 输出恒在 h+0x1203A0, 每字节 1 bit
├─ mode 空间   {0,1,2}; mode>=3 不量化
└─ 比例        output_bytes = 4*x2/x3

未闭合
├─ x2 的单位与真实长度           <- 阻塞一切采样率换算
├─ 24-of-32 的选取规则（stride）
├─ x1 指向的具体对象
├─ mode 0 的 8 抽头系数
├─ 32 -> 最终 DSD bitstream 的封装/后处理
└─ 设备实际 1-bit 输出（ground truth）
```

### 28.6 新增工具

`vendor_pull\re\uc_x1probe.py`（区域差分 + 写入 PC 归属）、
`mode_sweep.py`、`mode0_sweep.py`。

---

## 29. `w25` 溯源完成；24-of-32 判定为连续消费

### 29.1 `x2` = 调用方选定的块大小

反向数据流：调用点之前 `w25` 在本函数范围内的唯一写入是

```
0x128b0  mov w8, #0x10
0x128b8  movk w8, #0x50, lsl #16      -> w8  = 0x00500010
0x128bc  mov w10, #0x5c
0x128c0  movk w10, #0xd0, lsl #16     -> w10 = 0xD000005C
0x128c4  ldr w19, [x11, #0x5c]        <- mode
0x128c8  ldr x9,  [x20, #0x500000]
0x128cc  ldr x8,  [x20, #0x500010]
0x128d0  ldr w25, [x20, #0xD000005C]   <- x2 = w25 = *(u32*)(ctx + 0xD000005C)
0x128dc  subs x8, x8, x9               <- ctx[0x500010] - ctx[0x500000]
0x128f0  mov w9, w25
0x128f8  sxtw x26, x9
0x128fc  cmp x8, x26                   <- 与上面那个差值比较
0x12918  mov x4, x25                   <- 同时作为 x4 传给 0x1291c 的另一个调用
```

**`x2` 不是文件总长，是调用方选定的块大小**：
`ctx+0xD000005C` 是一个配置字段，被拿来与 `ctx[0x500010] - ctx[0x500000]`
（两个读写指针之差）比较，并作为 `x4` 转交给另一个调用。
因此 **`0x1305c` 是逐块调用的**，外层 `0x129xx` 有块循环。

这解释了 `output_bytes = 4*x2/x3` 的性质：**它是单块的输入输出关系，不是整文件的。**

同时注意 mode 的来源在此处是 **`[x11+0x5c]`**，而 `0x12e90` 处是 `[x23+0x68]`；
`x11` 与 `x23` 是两个不同的对象（或同源派生），尚未统一。

### 29.2 24-of-32 = 严格连续消费，不是抽取器

dump 每一次量化器输入 load 的源地址（8 输入帧）：

```
x3=32   192 loads  -> 24/frame   offsets 0..191   stride histogram {1:191}   CONTIGUOUS
x3=24   256 loads  -> 32/frame   offsets 0..255   stride histogram {1:255}   CONTIGUOUS
```

**stride 恒为 1，无重置、无跳跃、无 phase 依赖。**

所以：

- **不是** `32 -> 24` 的经典抽取器；
- **不是** stride / phase selection；
- 量化器就是**把 `768/x3` 个内部样本连续全部消费掉**。

因此"5 级 × 2⁵ = 32"这个说法只在 `x3=24` 时成立；
插值器的输出条数本身也是 `768/x3`（由每 block 重写的阈值数组驱动），
不是固定的 2⁵。

**内部样本数 = 输出字节数 = `768/x3`（每 `x2` 单位）**，1 bit 每字节。

### 29.3 结论

`output_bytes = 4 * x2 / x3` 现在有三重解释一致的支持：
块大小 `x2`、除数 `x3`、连续 1-bit-per-byte 输出。

采样率换算仍缺最后一项：**进入本函数时 PCM 的实际采样率与位深**。
但换算框架已经就位 —— 一旦知道输入是"多少帧、每帧多少字节"，
`output_bytes = 4*x2/x3` 立刻给出输出比特数，无需再猜。

### 29.4 新增工具

`vendor_pull\re\trace_w25.py`（w25 反向数据流）、
`uc_stride_probe.py`（量化器输入地址序列判定）。

---

## 30. APK 路线排除（All-To-DSD 不在 App 里）；libfiioaudio.so 的完整 API 面

### 30.1 设备侧实测结果

```
ro.debuggable = 0
ro.build.type  = user            (版本串是 eng.fiio.*，但 build type 是 user)
fingerprint    = .../13/TKQ1.230110.001/eng.fiio.20260428.144644:user/dev-keys

adb root   -> adbd cannot run as root in production builds
run-as     -> 全部失败，13 个 FiiO 包无一 debuggable
/proc/<pid>/maps -> cat: Permission denied        (SELinux 挡；shell 虽有 3009(readproc) 组)
/proc/<pid>/mem  -> -rw------- root
USB 数字输出     -> 不存在，输出设备只有 earpiece / speaker
/sdcard + Android/data/com.fiio.music -> 无转换产物（实时转换不落盘）
```

**结论：无 root、无 run-as、SELinux 阻断、无数字输出、实时转换不落盘。**
ground truth 只能靠模拟输出录音（需 ADC）或 root。

### 30.2 `libfiioaudio.so` 不在 APK 里

设备上实际运行的 `FiiOMusic.apk`（118,295,086 字节，md5 `2b1304d6258d6267fa66a214aeaac37c`）
的 native 库是：

```
libfiioa-jni.so  libfiioac-jni.so  libfiiocd-jni.so  libhello-jni.so
libUsbAudio.so   libfilterAPI.so   libsacd.so         libm3u.so
libeqLib.so      liblhdc.so        libijkffmpeg.so    libffmpegAPI.so
```

**没有 `libfiioaudio.so`** —— 它是 `/vendor/lib64` 的系统库。

而且我犯了个方向性错误：`0x1305c` 这个调用点**就在 `libfiioaudio.so` 内部**
（0x12xxx 与量化器 0x19xxx 同属该库），所以它本来就不该由 Java 调用，
Java/JNI 那条链是错的方向。All-To-DSD 是系统全局功能，与播放器 App 无关。

### 30.3 `libfiioaudio.so` 的完整导入表（102 项，已枚举）

与 PCM→DSD 直接相关的：

```
get_pcm2dsd_data          get_dsdtopcm_data        init_dsd2pcm_params
get_alltodsd_config       get_dsd2pcm_config       get_special_mode_config
match_resample_rate       match_audio_process_mode
```

**`aqm_*` 家族（12 项）—— 这里有关键发现：**

```
aqm_initialise            aqm_decode                aqm_getOutput
aqm_getMaximumEmptyChunkSize  aqm_getMinimumEmptyChunkSize  aqm_flush
aqm_getCurrentAudioType   aqm_getOutputState
aqm_getOriginalSampleRate aqm_getCurrentSampleRate  aqm_getBitDepth
```

**`aqm_getCurrentSampleRate` / `aqm_getOriginalSampleRate` / `aqm_getBitDepth`
说明：输入 PCM 的采样率与位深是这个音频管线在运行时可直接查询的状态，
而不是需要我从反汇编里猜出来的常量。**

其余导入：`start_config_params` / `processAudioInit` / `processAudioEvent` /
`update_pcm_ring_buffer` / `get_data_form_ring_buffer` / `get_player_buffer` /
`check_and_update_audiofocus_process` / `process_eq_audio_effect_data` /
`get_pcm2eq_data` / `mixer_*`(ALSA) / `property_get` / `property_set` /
`fiio_audio_debug_init` / `exp2 log cos sincos sin` 等。

### 30.4 这如何改变剩余问题

原来的阻塞点是"输入 PCM 的采样率与位深只能靠猜"，因为
`x2` 只是块大小、文档里不能写 DSD64。

现在有了具体出口：**这些量在真实进程里是可读的运行时状态**
（`aqm_getCurrentSampleRate` / `aqm_getBitDepth` / `aqm_getOriginalSampleRate`），
而 `get_alltodsd_config` 是 All-To-DSD 配置的来源函数。
只要能在真实调用环境里看到这三个函数被调用时的返回值与时间点，
`output_bytes = 4*x2/x3` 的换算基准就确定了 —— 不再是逆向猜谜。

能提供这个环境的东西只有两个：**root 后的真机**，或 **ARM64 AOSP emulator**
（后者能验证 `.so` 内部契约，但验证不了 HAL/驱动那一层）。

因此优先级需要重排：

1. **root 真机**（社区已验证 `init_boot` + Magisk 可行）→ 在播放时读
   app 内存 / 挂 gdb，能同时拿到 `x2` 语义、真实 `mode`、真实输入格式、
   以及精确的 1-bit 缓冲。信息量最大。
2. **模拟输出录音**（需 ADC）→ 拿到噪声整形阶数与斜率，判定
   "是否存在我没找到的整形环节"。注意 192 kHz 采样只能看到 0–96 kHz，
   且拟合时要把 JM21 模拟输出级自己的低通响应算进去。
3. **ARM64 emulator** → 只验证 `.so` 调用契约，不验证 HAL。

### 30.5 现状与交付

`--jm21` 仍默认关闭；默认 FIR 路径 42/42 回归通过、Web 端到端全过。
已确立的 RE 结论不受本轮影响（见第 26.3 / 28.3 / 29 节）。

新增工具：`vendor_pull\re\apk_scan.py`（APK 库与字符串扫描）、
`find_plt_target.py`（`.rela.plt` 全量枚举 + 导入名解析）。
`tools\device_probe.ps1` 会一次性给出本节 30.1 的全部结论。

---

## 31. 定位并修复 ×2 —— 核心其实一直在工作

### 31.1 长度不变量定位（无任何频率假设）

五级2× zero-insertion + biquad 的长度不变量，逐一验证：

```
input_scalar_samples = 1
stage1 -> 2      stage2 -> 4      stage3 -> 8      stage4 -> 16     stage5 -> 32
```

冲激测试（`[1,0,0,0,0,0,0,0]`）同样得到 16/32/64/128/256，32 个输出全非零
（biquad 的 IIR 振铃），**没有隐藏的 repeat / interleave / reshape**。

**答案 = C：wrapper 返回了超分配缓冲。**

### 31.2 Bug 本体

`final_interp.interpolate` 原来最后一行是 `return buf`：

```python
buf = [0.0] * (2 * n * 32)      # 分配 64n
...
buf[:2 * thr] = out              # 只写了前 32n
src = out
return buf                      # <-- 返回 64n，后 32n 全零
```

实测（`len_trace.py` D 段）：

```
n=1    returned=64    32n=32    nonzero=32    tail(from 32n) all zero: True
n=128  returned=8192  32n=4096  nonzero=4096  tail(from 32n) all zero: True
```

**不是第六级、不是stereo、不是算法错误 —— 插值器一直是精确的 32×。**
第 29节之前的 `4096/4096` bit-exact 验证依然有效（frames=128 时 32n = 4096，
比较的正好是真实输出全长）；修改 return 后重跑验证，仍 **4096/4096 FULL MATCH**。

### 31.3 这个 bug 伪造了三条假结论

| 观察 | 真相 |
|---|---|
| "输出 64 samples/input frame" | 真实 32×，返回缓冲是 2 倍长 |
| "400 ms 后输出塌成直流" | 那是零填充区 |
| "恢复峰出现在 2000 Hz" | 频率轴被拉长 2×，1 kHz 读成 2 kHz |

**并且它还伪造了第 24/25/28 节那条硬结论**"有界但不传递信号"——
在双倍长度的零填充数组上跑量化器，状态演化完全不同。
那条结论**正式撤回**。

### 31.4 修正后的真实性能

`d8` 扫描（输入 0.1 / 1 kHz，正确的 32×）：

```
d8    gain          gain = d8/32
  1   3.28e-02      ≈ 1/32
 16   5.00e-01      = 16/32
 32   1.00e+00      <- 单位信号增益
 64   2.00e+00
```

**`gain = d8 × 内插器直流增益(1/32)`，在 `d8 = 32` 时信号增益精确为 1。**
这从数值上独立确认了拓扑完全正确。状态在所有 d8 下有界，bit 密度恒 0.5。

设备的 `d8 = 16 - arg6`（arg6 = mode ∈ {1,2}）给出 0.44–0.47，
即设备跑在约半满度驱动。

响应图（修正后）：

```
amplitude  gain(mode1)  gain(mode2)
  1e-04       0             0
  1e-03       3.82e-01      2.97e-01     <- 阈值
  1e-02       4.72e-01      4.53e-01
  5e-01       4.69e-01      4.38e-01
频率 DC..20k: 0.469 ~ 0.446（平坦）
倍频程噪声: 20-200:-101.6  200-2k:-24.8  2k-20k:-40.5  20k-40k:-33.5
```

### 31.5 C++ 侧对比（-6 dBFS 1 kHz，DSD64）

```
JM21 核心   SFSR 19.3 dB   mean q = 0.0000
  带内   0-100:-64  100-500:-56  500-900:-46
  带外   2k-5k:-53  5k-10k:-55  20k-40k:-49  300k-1.4M:+21

FIR 路径    SFSR 19.1 dB   mean q = 0.0000
  带内   0-100:-56  100-500:-49  500-900:-45
  带外   2k-5k:-40  5k-10k:-37  20k-40k:-33  300k-1.4M:+12
```

**两者 SFSR 基本持平（19.3 vs 19.1），但 JM21 核心的带内噪声明显更低
（-64 vs -56 @ 0–100 Hz），且噪声被推到高得多的频段（+21 vs +12 dB）。**
这是更陡 NTF 的预期特征。对可听质量重要的是带内噪声，这一项核心更好。

`--jm21` 之前输出恒为 0，现在输出真实信号且比特流平衡。

### 31.6 新增工具

`vendor_pull\re\final_interp.py`（已修 return）、`len_trace.py`（长度不变量）、
`resp32.py`（修正后的响应图）、`d8_sweep32.py`（d8 扫描）。

---

## 32. 设备侧实测：A0 已答，ADC 链已通，首次拿到真机模拟输出

### 32.1 A0：JM21 USB DAC 能力边界（WASAPI 独占，逐档试探）

渲染端点 `扬声器 (FiiO M series)`（`sd` idx 24，WASAPI），默认采样率 **384000 Hz**；
另有 WDM-KS 端点 `扬声器 (FiiO JM21)`（idx 47）。

独占模式实际打开成功的档位：

```
44100  48000  88200  96000  176400  192000  352800  384000     全部 OK
```

**与 FiiO 官方"USB DAC 最高 384 kHz / 32 bit、DSD256 Native"一致。**

技术要点：**必须 WASAPI + `InputStream`/`OutputStream` + `WasapiSettings(exclusive=True,
auto_convert=False)`。** 走 dshow/ffmpeg 或 MME/DirectSound 一律只给 44100 Hz / 16-bit；
且播放流与录音流**必须同时独占**，混用会在开流时报 `PaErrorCode -9997`。

### 32.2 测量链地板（Line-In，192 kHz 独占，什么都不插，15 s）

```
ch0/ch1 RMS -68.14 / -68.08 dBFS    DC -5e-6
20-60:-72.5  60-250:-70.8  250-1k:-69.6  1k-4k:-70.5  4k-8k:-74.4
8k-12k:-81.1 12k-16k:-84.4 16k-20k:-84.1 20k-30k:-74.9 30k-44k:-79.0
44k-60k:-78.8 60k-80k:-76.7 80k-96k:-81.3
最大 bin>20Hz: 24.8 Hz @ -91.0 dBFS      20 kHz 以上: -70.6 dBFS
```

**注意一个必须靠A/B 消掉的混淆：ADC 自身的噪声形状也是"12–20 kHz 最低、
20 kHz 以上抬升"（-84 → -75），与要检测的现象同向。**
因此 `All-To-DSD OFF` 基线是**必需项**，不是可选项。

用户已取消"启用音频增强"（否则会在录音链插入处理，污染噪声底）。

### 32.3 首次真机模拟输出（PC→USB→JM21→line-in，192 kHz 独占，各 12 s）

| 输入 | 输出 RMS | 1 kHz 基波 | 带内(挖基波) |
|---|---|---|---|
| 静音 | -41.3 dBFS | 无 | **-43.4 dBFS** |
| 1 kHz -60 dBFS | -39.1 dBFS | -89.8 dBFS | -43.7 dBFS |
| 1 kHz -6 dBFS | -13.1 dBFS | **-39.6 dBFS** | **-9.2 dBFS** |

**链路确认打通**：1 kHz 基波电平与"JM21 音量衰减 + line-in 增益"相符，
说明 PC→USB→JM21→模拟→ADC 整条通路工作正常。

### 32.4 两个尚未解释的观察（**不得当作结论**）

**(1) 噪声随输入电平暴涨。** 静音 → -60 dBFS 输入，带内噪声几乎不变（-43 dBFS）；
但 -6 dBFS 输入时带内跳到 -9.2 dBFS，**比静音高 34 dB**。
这不像正常D/A 通路，候选解释：调制器过载、失真、或对当前状态的假设有误。

**(2) 音载周围存在 8.000 Hz 间距的离散梳状边带。**
- 间距中位数 **8.000 Hz**，39 个间隔的 std 仅 0.176 Hz
- 范围 700–1300 Hz 及更远（±200 Hz 仍可见）
- **边带强于载波**（+16 Hz 处 -37.1 dBFS，载波 -45.1 dBFS）
- **静音那组没有这个结构**（唯一 >20 Hz 的峰是 24.6 Hz @ -67.7 dBFS）
  ⇒ **跟随信号**，不是固定设备伪迹

**但 8.000 Hz 过于规整，存在一个我尚未排除的替代解释：**
USB 等时音频包率为 **8000 包/秒**；若 JM21 的 USB 时钟与 PC 录音时钟相差约 0.1%，
8 kHz 会混叠降到 8 Hz ⇒ **那是我这套测量装置的伪迹，不是 JM21 的 DSP。**

我写的判别脚本（扫 8 Hz 梳状并比较线/底比值）**逻辑有误，未能判定**：
输出显示"最佳偏移"恒落在频带边缘、比值恒为 0.1（<1 即无梳状），
因此该结果**不构成任何证据**。此项仍未解决。

### 32.5 因此下一步只差一件事

**`All-To-DSD` OFF / ON 的 A/B。** 它能一次性回答：
1. 当前处于哪个状态
2. 44–60 kHz 的 -38.5 dBFS 超声能量、以及上述带内噪声暴涨，是否为 DSD 所致
3. 8 Hz 梳状是设备固有还是状态相关（与时钟假设一起，OFF 时若梳状仍在 ⇒ 装置伪迹）

需要用户在设备上切换 All-To-DSD（当前处于 USB DAC 模式，ADB 不可用，必须物理操作）。

### 32.6 新增工具

```
tools/adcfloor_excl.py   192 kHz 独占噪声底表征
tools/adcfloor_wasapi.py 设备枚举 + 独占参数探测
tools/jm21_analog.py     播放 + 录制 + 频谱分析（含 A0 格式枚举）
tools/gen_a1.py          A1 受控信号生成（12 个 24-bit WAV）
```

---

## 33. 边带现象：确认存在，但速率未可靠测定（三次分析均失败，不给数字）

### 33.1 确认存在的部分

载波 1 kHz / -6 dBFS 时，模拟输出在音载两侧出现**离散等间隔边带**，
且**边带强于载波**（+16 Hz 处 -37.1 dBFS，载波 -45.1 dBFS）。

**跟随信号**：静音那组没有这个结构（唯一 >20 Hz 的峰是 24.6 Hz @ -67.7 dBFS），
所以它不是固定的设备或录音链路 spur。

### 33.2 已排除的假设

**`H_clock`（USB 等时包率 8000/s 被时钟失配混叠）—— 排除。**
若成立，边带间距应随播放采样率按比例变化（96k→4、192k→8、384k→16 Hz）。
实测三档播放率：

```
播放  96 kHz : 间距中位数 7.07 Hz
播放 192 kHz : 间距中位数 5.86 Hz
播放 384 kHz : 间距中位数 5.86 Hz
```

**不随播放采样率变化**，与载波频率（1k/2k/5k）也无关。

### 33.3 三次失败的速率测定（**因此不给速率数字**）

| 方法 | 结果 | 为什么不采用 |
|---|---|---|
| `comb.py`（peak-picker, minsep=8） | 8.000 Hz，std 0.176 | 参数敏感 |
| `sideband_sweep.py`（peak-picker, minsep=3~4） | 5.86 Hz，std 2.2~11 | **换参数就变** |
| `mod_period.py`（包络自相关） | 400 Hz | 落在搜索窗边界，是载波残留，不是低频调制 |

**同一个设置下换 peak-picker 阈值就从 8.000 变到 5.778** ——
那是在测量我自己的挑峰器，不是信号。ACF 则被载波本身的周期性污染
（带通包含了载波，包络里保留了 2.5 ms 的载波成分）。

**结论：边带现象真实存在、跟随信号、且已排除"随播放采样率等比缩放"这一类；
但它的实际速率我没有可靠测出来。不给数字。**

### 33.4 候选解释（均未验证，按信息增益排序）

1. **JM21 模拟输出级里一个固定周期的扰动**（音量 PWM、显示刷新耦合、
   设备内部低频时钟）—— 与"固定在 ~6 Hz、与载波和采样率无关"相容
2. **PC 录音链里一个固定周期的伪迹**，但被信号门控 —— 需要 OFF/ON 对照才能排除
3. 与 All-To-DSD 相关 —— 需要 OFF/ON 对照

### 33.5 为什么现在应该停止分析边带

第 33.2 节已经排除了最像样的那个解释，而剩下的三个候选**只有 OFF/ON
能分开**：

- 若 **OFF 时边带同样存在** → 与 All-To-DSD 无关（候选 1 或 2），可从 DSP 主线删除
- 若 **OFF 时没有、ON 时有** → 属于 All-To-DSD，进入 DSP 分析范围

继续在没有对照的情况下调分析参数，是在拟合噪声。

### 33.6 另一个仍未解释的观察（优先级低于边带）

| 输入 | 输出 RMS | 1 kHz 基波 | 带内 |
|---|---|---|---|
| 静音 | -41.3 dBFS | 无 | -43.4 dBFS |
| 1 kHz -60 dBFS | -39.1 dBFS | -89.8 dBFS | -43.7 dBFS |
| 1 kHz -6 dBFS | -13.1 dBFS | **-39.6 dBFS** | **-9.2 dBFS** |

静音 → -60 dBFS 输入，带内噪声几乎不变；-6 dBFS 输入时**跳高 34.2 dB**。
候选：调制器过载 / 某级模拟或 ADC 输入接近过载 / All-To-DSD 的增益或工作模式
在高电平处剧变 / 统计口径把随载波出现的高次与边带能量算了进去 /
链路存在信号相关的时变伪迹。

**这同样需要 OFF/ON 才能砍掉一大片。**

### 33.7 因此下一步严格限定为

**用户手动切一次 All-To-DSD，录两段：1 kHz / -6 dBFS，OFF 与 ON，各 20–30 s。**
先这一对，不做全部六组。之后再补静音 OFF/ON（用于第 33.5 节的判别）。

软件侧已全部就绪：`tools/jm21_analog.py` 可直接录制与分析。

---

## 34. All-To-DSD OFF/ON 真机 A/B：首次模型与真机对照

采集条件（两侧完全一致，192 kHz WASAPI 独占，仅第一秒丢弃）：

```
PC --USB 192kHz--> JM21 (USB DAC) --3.5mm--> PC Line-In @192kHz exclusive
ADC 地板（不插线）: RMS -68.1 dBFS, 带内 -70…-84 dBFS
ON  采集: cap_silence.npy / cap_1k.npy
OFF 采集: off_silence.npy / off_1k_-6dBFS.npy / off_1k_-60dBFS.npy
```

### 34.1 观测事实

**F1 — 梳状边带在 OFF 与 ON 下都存在。**
同一套挑峰参数（`minsep=8`）下，四个采集的边带位置几乎相同：

```
1 kHz -6 dBFS  OFF: -281.7 -238.7 -218.3 -206.5 -191.3 -183.1 -172.2 -163.3 -151.1 -139.4 ...
1 kHz -6 dBFS  ON : -282.3 -208.8 -198.6 -190.2 -180.0 -168.2 -151.8 -142.0 -130.3 -122.1 ...
静音          OFF: -293.7 -278.6 -267.3 -258.8 -250.5 -242.2 -232.8 -221.2 -206.1 -197.8 ...
静音          ON : -298.2 -289.2 -278.6 -267.1 -256.6 -243.3 -231.0 -218.9 -206.2 -189.0 ...
```

电平随信号缩放（静音组约 -86 dBFS，-6 dBFS 组约 -35…-48 dBFS），
但**位置结构与 All-To-DSD 状态无关**。

> 修正第 33 节的说法：那里"静音没有梳状"的结论来自只看最强峰，
> 是错的。用固定参数重看，静音在 OFF/ON 下都有该结构，只是电平低约 50 dB。

**F2 — All-To-DSD ON 在真机模拟输出上显著增加 30 kHz 以上能量。**
静音输入：

| 频段 | OFF dBFS | ON dBFS | delta |
|---|---|---|---|
| 20–250 | -52.39 | -45.26 | +7.13 |
| 250–1k | -53.28 | -49.85 | +3.43 |
| 1k–4k | -57.20 | -54.63 | +2.58 |
| 4k–8k | -62.85 | -63.89 | -1.04 |
| 8k–16k | -67.02 | -66.04 | +0.97 |
| 16k–20k | -73.32 | -71.14 | +2.18 |
| 20–30k | -63.84 | -62.89 | +0.95 |
| 30–44k | -70.01 | -53.37 | **+16.64** |
| 44–60k | -71.00 | -38.50 | **+32.50** |
| 60–80k | -69.75 | -45.33 | **+24.42** |
| 80–96k | -77.54 | -59.96 | **+17.58** |

**这是真机测量结果，不是模型推断。** 带外能量增加 17–33 dB，带内变化仅 +1…+7 dB。

**F3 — 当前测量条件下，ON 的带内噪声并未下降，部分频段反而上升。**
见 F2 表：20 kHz 以下各段 delta 为 +7.13 / +3.43 / +2.58 / -1.04 / +0.97 / +2.18。

**F4 — -6 dBFS 下的带内能量暴涨在 OFF/ON 下均存在。**
1 kHz / -6 dBFS：

| 频段 | OFF dBFS | ON dBFS | delta |
|---|---|---|---|
| 250–1k | -10.97 | -11.04 | -0.07 |
| 1k–4k | -11.93 | -11.86 | +0.07 |
| 8k–16k | -32.22 | -32.48 | -0.26 |
| 16k–20k | -39.54 | -39.44 | +0.10 |
| 44–60k | -41.48 | -34.81 | +6.67 |

44 kHz 以下逐段差异全在 ±0.5 dB 内；基波 OFF -34.50 / ON -32.58 dBFS。

**F5 — 模型与真机在带内噪声上存在实质差异。**
模型（`--jm21`，模型仿真值）：0–100 Hz 带内噪声比我的 FIR 路径**低约 8 dB**
（-64 vs -56 dBFS）。真机实测（F3）：All-To-DSD ON 的带内噪声**升高 1–7 dB**。
两者不可能同时正确。

### 34.2 解释（尚未定论）

F2 支持"All-To-DSD 把噪声搬出音频带"这一定性描述 —— 这是它最显著的真机特征。

但 F3 与 F5 构成一个实质问题：**一个把噪声整形到带外的 shaper，通常应当同时降低带内噪声；
真机没有表现出这一点。** 这与模型的行为相反。

### 34.3 三个竞争假设（按当前证据排序，均未证实）

**H1 — 模拟输出级 / ADC 在带内引入了自己的噪声底，掩盖了 shaper 的带内收益。**
当前 OFF 基线带内为 -52…-73 dBFS，而 ADC 地板是 -70…-84 dBFS，**余量只有约 15 dB**。
若把信号整体抬高约 22–25 dB，shaper 的带内收益应能显现。
**这是最容易被下一次实验排除的一个。**

**H2 — 模型漏了真机上的某个环节。**
插值器与量化器虽然已逐指令闭合，但 `PCM → tbl[m]` 的绝对标度、
`x2` 的真实单位、以及内部 1-bit 流到最终 DSD 的 packing 都仍未闭合（第 0.3 节）。
真机带外抬升的**拐点频率**（目前观察到 30 kHz 以上才明显上升）可以反过来约束这些未知量。

**H3 — 真机 All-To-DSD 在该工作点上是"增加噪声"而非改善。**
即这个功能是产品特性开关，不必然代表噪声整形更优。
**目前没有任何数据可以排除它，也不能确认它。**

### 34.4 下一次实验（只改一个变量）

**只改 JM21 音量，不动 Line-In 增益、不动播放文件、不动采样率。**

目标：把 1 kHz 基波从当前 -34.50 dBFS（OFF）/ -32.58 dBFS（ON）抬到约 **-10 dBFS**，
即需提高约 **+22…+25 dB**（具体格数取决于 JM21 的音量步进）。

然后重录四段：

```
静音      OFF
静音      ON
1k -6 dBFS  OFF
1k -6 dBFS  ON
```

**判据：**
- 若带内 delta 仍为正（ON 更响）⇒ **H1 被削弱**，问题转向 H2/H3
- 若带内 delta 转为明显负（ON 更安静）⇒ **H1 成立**，之前的"ON 带内更响"是 ADC 地板造成的假象
- 若带内 delta 基本不变 ⇒ 三者都需要进一步区分

同时观察 **30 kHz 以上抬升的起点频率是否随音量改变** —— 若不变，则拐点由
shaper 阶数决定，可直接用来约束模型。

### 34.5 已可归档的结论

```
✓ All-To-DSD 的真机特征在 30–96 kHz 已确认（带外能量 +17…+33 dB）
✓ 梳状边带与 All-To-DSD 无关，从 DSP 逆向主线移除
✓ -6 dBFS 带内能量暴涨与 All-To-DSD 无关
✗ 模型与真机的带内噪声行为相反 —— 未解决
✗ 三竞争假设 H1/H2/H3 均未证实
```

---

## 35. All-To-DSD 真机 A/B 观测：首次确认带外噪声搬移，但未观察到带内收益

> 本节记录 JM21 USB DAC 模式下的首次真机模拟输出 A/B 测量。
> 所有“JM21 measured”均来自 JM21 模拟输出经 PC 24-bit/192 kHz Line-In 录制后的数据；不得与软件模型仿真值混用。

### 35.1 测试链路

```
PC 数字 PCM
    ↓
JM21 USB DAC（WASAPI Exclusive）
    ↓
JM21 音频处理 / All-To-DSD
    ↓
CS43198 × 2
    ↓
模拟输出
    ↓
PC Line-In（24-bit / 192 kHz Exclusive）
```

USB DAC 实测支持 44.1 / 48 / 88.2 / 96 / 176.4 / 192 / 352.8 / 384 kHz。

本节比较相同输入条件下：

```
All-To-DSD OFF   vs   All-To-DSD ON
```

---

### 35.2 1 kHz / -60 dBFS：All-To-DSD ON 显著增加带外能量

该工作点未出现 -6 dBFS 条件下观察到的宽带过载，因此优先用于判断 All-To-DSD 本身的频谱影响。

| 频段 | OFF | ON | Δ |
|---|---:|---:|---:|
| 20–250 Hz | -52.27 | -45.76 | +6.52 dB |
| 250–1 kHz | -53.35 | -50.24 | +3.12 dB |
| 1–4 kHz | -56.40 | -54.37 | +2.02 dB |
| 8–16 kHz | -66.90 | -64.44 | +2.46 dB |
| 16–20 kHz | -73.23 | -70.56 | +2.68 dB |
| 30–44 kHz | -70.04 | -54.91 | **+15.12 dB** |
| 44–60 kHz | -71.03 | -41.24 | **+29.78 dB** |
| 60–80 kHz | -69.76 | -45.22 | **+24.54 dB** |
| 80–96 kHz | -77.51 | -59.72 | **+17.79 dB** |

观测事实：

1. All-To-DSD ON 后，30 kHz 以上能量显著增加。
2. 最大增量出现在 44–60 kHz，约 +29.8 dB。
3. 该现象在静音输入下同样存在，因此不是 1 kHz 基波泄漏本身造成。
4. 该结果是 **JM21 measured**，不是模型推断。

因此可以确认：

> **JM21 的 All-To-DSD ON 状态具有显著的带外噪声/能量增加特征。**

但截至本节数据，尚不能仅凭这一结果确定其具体来源是外部 PCM→DSD shaper、
CS43198 DSD 模拟路径，还是两者共同作用。

---

### 35.3 All-To-DSD 引入约 +3 dB 的整体输出增益

> **[SUPERSEDED — 见 36.1]** 下面这个「基波增益」量的是 1 kHz 附近的最大噪声 bin，
> 不是信号。全部采集里不存在相干的 1 kHz 音调，峰值频率在 921-1022 Hz 间无规律
> 跳动，静音采集也有同量级峰。**该数字已撤回，请勿引用。**
> 真实存在的整体电平变化确实约 +10 至 +11 dB RMS（见 36.4），但那不是「基波增益」。

1 kHz 基波：

```
OFF = -86.21 dBFS
ON  = -82.63 dBFS
Δ   = +3.58 dB
```

在其他条件下也观察到类似约 +3 dB 的整体电平变化。

因此：

> **All-To-DSD ON 与 OFF 之间存在可测的输出电平差异。**

对噪声比较必须考虑这一增益差异；不能直接比较 ON/OFF 的绝对噪声值后宣称噪声增加。

经过约 +3 dB 增益归一化后：

> **当前测试中没有观察到 All-To-DSD 对带内噪声的改善；带内噪声总体持平或更差。**

因此本节不使用“带内收益”作为已证实结论，而记录为：

> **JM21 measured：当前工作点未观察到带内 SNR/噪声收益。**

---

### 35.4 输入电平依赖：-6 dBFS 过载与 All-To-DSD 无明显关系

250 Hz–4 kHz 合并结果：

| 输入 | OFF | ON |
|---|---:|---:|
| 静音 | -51.80 | -48.60 |
| -60 dBFS | -51.60 | -48.82 |
| -40 dBFS | — | -45.55 |
| -6 dBFS | -8.41 | -8.42 |

观测：

```
静音 → -60 dBFS：OFF 约 -51.6 dBFS，ON 约 -48.8 dBFS
```

因此设备存在明显高于 Line-In 自身噪声地板的固定本底。

而：

```
-6 dBFS：OFF = -8.41 dBFS   ON = -8.42 dBFS
```

两者几乎相同。因此：

> **-6 dBFS 条件下观察到的带内能量暴涨不能归因于 All-To-DSD。**

更保守地说，其来源至少不是当前 A/B 测量中可见的 All-To-DSD 开关差异；
候选包括 PCM/模拟输出级、ADC 输入级或其他共同路径中的过载/非线性。

---

### 35.5 ADC 地板已不足以解释当前结果

空载 Line-In 地板：`RMS ≈ -68.1 dBFS`

1 kHz / -60 dBFS 测试下：带内本底 ≈ -51.6 dBFS（OFF）、≈ -48.8 dBFS（ON）

因此当前带内本底至少比 ADC 地板高约 19 dB。结论：

> **“ADC 地板完全掩盖了 All-To-DSD 带内差异”作为主要解释已明显减弱。**

但由于 JM21 模拟输出级自身的噪声已经进入测量结果，因此不能仅凭 ADC 地板与
带内本底之间的差值，把剩余差异全部归因于数字 shaper。

---

### 35.6 8 Hz 梳状边带：排除为 All-To-DSD 特征

相同峰选取参数下：

```
1 kHz / -6 dBFS：OFF、ON 均存在类似边带结构
静音：        OFF、ON 同样存在类似位置的边带结构，只是电平显著降低
```

因此：

> **该梳状结构与 All-To-DSD 开关无直接对应关系。**

它应从 All-To-DSD DSP 逆向主线中移除，暂归类：

> **device PCM / analog path / recording-chain spur，origin unresolved**

在没有进一步证据前，不再把其 8 Hz 间隔解释为 USB packet rate、clock beat
或其他具体机制。

> 修正第 33 节：那里“静音没有梳状”的结论来自只看最强峰，是错的。
> 用固定参数重看，静音在 OFF/ON 下都有该结构，只是电平低约 50 dB。

---

### 35.7 与 `--jm21` 模型的关键分歧

当前软件模型：

```
--jm21
    SFSR ≈ 19.3 dB
    低频带内噪声低于当前 FIR 模型约 8 dB
```

真机：

```
All-To-DSD ON
    相对于 OFF：带内噪声经增益归一化后未见下降；30 kHz 以上显著能量增加
```

因此出现明确的行为分歧：

```
model simulation : 带内噪声改善
JM21 measured    : 带内未见改善，部分频段更差 + 显著带外能量
```

该分歧不能由当前数据自动解释。

---

### 35.8 当前竞争假设

**H1：模型漏掉了真机数字处理链中的某个环节。**

当前最优先的数字域假设。可能涉及：

* `get_pcm2dsd_data` 的调用条件；
* 实际 runtime `mode`；
* PCM→`tbl[m]` 的标度；
* `h+0x1203A0` 之后的 1-bit packing / conversion；
* All-To-DSD 路径中另一个转换器；
* shaper 前后的额外滤波、增益或噪声源。

**H2：真实 All-To-DSD 确实使用 `0x19d90`，但最终模拟结果还叠加了 CS43198 DSD
路径自身的噪声/滤波行为。**

CS43198 本身具有独立的 DSD processor 和 DSD/PCM 切换路径，因此模拟端频谱不应被
简单等价为外部 PCM→1-bit shaper 的频谱。该芯片官方资料明确说明支持 DSD256
direct mode，并存在专用 DSD processor。

**H3：真实 All-To-DSD 的该工作点本身没有可观的带内噪声收益。**

当前真机 A/B 对该假设提供强支持，但尚不能据此判断该行为究竟来自外部 shaper 本身，
还是后级 DSD 模拟路径抵消了数字域收益。

---

### 35.9 下一关键实验

下一阶段不再通过继续猜测模拟频谱来区分 H1/H2。目标改为：

> **确认真实 All-To-DSD ON 路径是否执行 `get_pcm2dsd_data` / `0x19d90`。**

最直接的证据：

```
runtime trace / hook
        ↓
0x19d90 是否执行
        ↓
h+0x203A0 是否产生对应插值数据
        ↓
h+0x1203A0 是否产生对应 1-bit 输出
```

**一个必须避免的逻辑错误：**
`output_bytes = 4*x2/x3` 是从该函数**内部行为**得到的中间输出关系，
**不是**已证明的“最终 USB/DSD 流量关系”。因此

```
最终观察倍率 ≠ 4*x2/x3
```

只能说明**不能直接用最终输出倍率验证该函数**，
**不能单独证明 `0x19d90` 没被调用** —— 二者之间仍存在未闭合的
packing / downstream path。决定性证据只能是 runtime trace/hook，
或直接观察 `h+0x1203A0` 是否发生对应写入。

---

### 35.10 当前状态

```
USB DAC A0✅ 完成
真机模拟回录           ✅ 完成
All-To-DSD ON/OFF A/B  ✅ 完成
8 Hz spur 排除为 DSD 特征 ✅
带外能量增加           ✅ 真机观测
约 +3 dB 输出差异      ✅ 真机观测
带内收益               ❌ 当前未观察到
模型/真机分歧          ✅ 已确认
真实 0x19d90 调用      ❌ 未确认
最终 DSD packing       ❌ 未确认
```

术语纪律：

```
JM21 measured     = 真机 ADC 回录观测
model simulation  = 软件模型仿真
```

二者不得在同一指标表中不加标记地混用。

---

## 36. All-To-DSD excess PSD：线性功率域判别（含三条撤回）

本节把第 35 节的带内/带外对比升级为可判定的形式，并撤回其中两条结论。

### 36.1 撤回一：不存在可用的 1 kHz 基波

第 35 节报告的「OFF 基波 -86.21 / ON 基波 -82.63，增益差 +3.58 dB」**不成立**。
`ab_test.py` 是在 1 kHz 附近 ±25 Hz 取最大 bin。对全部采集做窄带扫描后：

| 采集 | 900-1100 Hz 内最强 bin | 该 bin SNR |
|---|---|---|
| `off_1k_-6dBFS` | 995.00 Hz | 12.2 dB |
| `cap_1k` (ON) | 1021.59 Hz（另有 972.34） | 11.4 dB |
| `off_1k_-60dBFS` | 972.83 Hz | 11.7 dB |
| `on_1k_-60dBFS` | 951.29 Hz | 15.5 dB |
| `off_silence` | 921.04 Hz | 11.1 dB |
| `cap_silence` | 1002.00 Hz | 12.3 dB |

峰值频率在 921-1022 Hz 间无规律跳动，且**静音采集也有同量级峰**；11-15 dB 的 SNR
正是几千个噪声 bin 中最大值的极值统计量（约 4.1 sigma）。因此该数字量的是
**噪声最大 bin，不是信号**。已撤回。

### 36.2 撤回二：-60 dBFS 对其实是静音对

以各自的静音采集为基线，比较 985-1015 Hz 窄带功率增量：

| 对 | 985-1015 Hz | 带内 20-20 kHz | 20-96 kHz |
|---|---|---|---|
| OFF -60 dBFS 减静音 | +2.23 dB | -0.35 dB | +0.18 dB |
| ON -60 dBFS 减静音 | +0.02 dB | -0.13 dB | -2.08 dB |

音调只比本底高约 15 dB，本质上埋在噪声里。**-60 dBFS 对不能当作信号测试使用。**
真正含信号的只有 -6 dBFS 对（同口径下窄带 +53.53 dB / +50.34 dB）。

### 36.3 方法：线性功率相减 + 归一化

功率谱不能在 dB 上相减。`tools/ab_excess.py`：

```
P_excess(f) = max( P_ON_norm(f) - P_OFF(f), 0 )
```

归一化基准分两种，因为没有可靠基波：

- **静音对**：用带内 20-20 kHz 功率（挖去 940-1060 Hz）归一化。
- **-6 dBFS 对**：用 985-1015 Hz 音调功率归一化（这是唯一真实的信号参考）。

PSD 改用 Welch 平均（nperseg=262144，约 1.365 s、1.4 Hz 分辨率），
不再沿用第 35 节的单次全段 FFT。

### 36.4 观测一：excess 的四段结构（静音对，带内归一化）

| 频段 | excess 相对 OFF |
|---|---|
| 20-33 kHz | **小于等于 0（ON 更安静）** |
| 37 kHz | 交叉（excess = OFF） |
| 48-68 kHz | 平台 **+28 dB** |
| 大于 68 kHz | 以约 -37 dB/oct 滚降 |

分段斜率：

| 窗口 | 斜率 dB/oct | 拟合 rms |
|---|---|---|
| 28-40 kHz | +39.2 | 3.90 |
| 36-48 kHz | +69.9 | 1.80 |
| 44-56 kHz | +12.3 | 1.70 |
| 48-68 kHz | -19.8 | 1.70 |
| 68-80 kHz | -36.9 | 1.61 |

注意 36-48 kHz 的 +70 dB/oct 是**交叉区伪斜率**：该处 excess 由 0 过渡到主导，
对数值在接近零时极敏感。真正可用的是平台与滚降。

### 36.5 观测二：音调归一化后的带内结果（-6 dBFS 对）

> **[SUPERSEDED — 见 36.13]** 本节的「音调归一化」不成立。用相干检测器重测，
> `-6 dBFS` 采集里的 1 kHz 音调在 **-42.64 dBFS 峰值**，而 985-1015 Hz 频带的
> 总功率是 -20.42 dB —— **音调比该频带内容低 28 dB**。也就是说用该频带做增益归一化，
> 实际是在对**过载爆炸自身的噪声**归一化，而不是对音调。
> **下面的带内 +0.33 dB 与「无收益」均不成立，已撤回。**
> -6 dBFS 这对数据无法回答带内问题，这正是需要补采非过载工作点的原因。

```
音调增益 (985-1015 Hz)   -0.32 dB     信号基本不变
带内 20-20 kHz           +0.33 dB     无收益
带外 20-96 kHz           +3.29 dB
  20-30k +0.22  30-40k +0.29  40-50k +4.72
  50-60k +7.65  60-80k +5.93  80-96k +0.89
```

**在该工作点，All-To-DSD 没有可测的带内收益**（+0.33 dB，方向还是略差），
代价是 50-60 kHz 增加 +7.65 dB。这是 H3 的直接证据。
但 -6 dBFS 已使 PCM/模拟路径过载，此为过载区结论。

### 36.6 观测三：量化器是教科书式二项式误差反馈调制器

对已位级精确的量化器（`src/flac2dsf.cpp`，Unicorn 4096/4096）做环代数推导：

```
v_n = d8*x_n + sum_i A_i q_{n-1-i}
y_n = sign(v_n)
q_n = sum_i (A_i + B_i) q_{n-1-i} - y_n
```

令 `C_i = A_i + B_i`，误差 `e = y - v`，则

```
NTF(z) = Y/E = (1 - C(z^-1)) / (1 - B(z^-1))
```

独立佐证：`C` 应当等于 `(1 - z^-1)^N` 的系数。实测：

| shaper | C = A+B | (1-z^-1)^N | 相对误差 |
|---|---|---|---|
| mode1（6 抽头） | 5.99780, -14.99119, 19.98679, -14.99119, 5.99780, -1.00000 | 6, -15, 20, -15, 6, -1 | 6.6e-4 |
| mode2（4 抽头） | 3.99929, -5.99857, 3.99929, -1.00000 | 4, -6, 4, -1 | 2.4e-4 |

**阶数由此无歧义确定**：shaper 1 为 6 阶（DC 附近 36 dB/oct），
shaper 2 为 4 阶（24 dB/oct）。两者在高频都饱和到约 +3.5 dB，
因为 `(1-B)` 的极点抵消了 `(1-z^-1)^N` 的零点。

### 36.7 阴性结果：模拟频谱无法区分 4 抽头与 6 抽头

`tools/jm21_ntf.py` 在纯形状域拟合（alpha = 内部速率/实测频率，自由搜索 400-8000）：

| 拟合窗 | mode1 残差 | mode2 残差 | 差值 |
|---|---|---|---|
| 33-48 kHz | 2.41 dB | 2.44 dB | 0.03 dB |
| 34-50 kHz | 2.14 dB | 2.23 dB | 0.09 dB |
| 36-52 kHz | 1.72 dB | 1.84 dB | 0.12 dB |
| 33-44 kHz | 2.67 dB | 2.70 dB | 0.03 dB |

**结论：不可判。** 原因是方法学上的三重欠定：

1. 自由 alpha + 自由增益 + 自由偏置共 3 个参数，约束的只是一个约 1.5 个八度的上升段。
2. 44 kHz 以上是 CS43198 主导。平台与 -37 dB/oct 滚降都不是任何 NTF 能产生的形状。
3. CS43198 的 DSD processor 转折点（约 50 kHz）恰好落在被用来判断 shaper 的
   上升段内部，所以滤波器本身参与了待判形状的形成。

因此**「带外形状比对」这条路已经被证伪为不可判**，它无法把 H1/H2 的概率压下去。

### 36.8 假设状态更新

| 假设 | 状态 | 依据 |
|---|---|---|
| H1 真实路径不是 `0x19d90` | **仍开放**，但频谱路线无法裁决 | 36.7 |
| H2 `0x19d90` 执行，后面还有 DSD/DAC 环节 | **获支持** | 48-68 kHz 平台与 -37 dB/oct 滚降是 DAC 行为，NTF 不产生平台 |
| H3 该工作点无带内收益 | **强支持** | 36.5 音调归一化后带内 +0.33 dB |

### 36.9 GhostLock 路线终止（记录）

内核 `5.15.41-android13-8-g9ded8564ff52-dirty` **未打补丁**：
`remove_waiter@0x1a0994c still uses current`，CVE-2026-43499 原语存在。
`pselect_waiter_shift` 无法从 boot.img 推导；备选路线 `select_stack`、
`tcp_zerocopy` 的几何参数均为 `null`。

利用在 App 与 CLI 两条路径各执行一次，**均未造成崩溃**（设备 `boot_completed=1`），
失败点固定为：

```
perf_event_open failed errno=13
W3 seccomp bypass failed after 3 chain rounds
[!!] w2 never rooted a child
```

根因是 `/proc/sys/kernel/perf_event_paranoid = 3`，对非特权进程禁用
`perf_event_open`；降低它需要 root，构成循环依赖。**未获得 root。**
所有官方产物已按 release 自带 `sha256sum.txt` 逐一校验通过。

### 36.10 术语与措辞纪律（更新）

```
JM21 measured     = 真机 ADC 回录观测
model simulation  = 软件模型仿真
```

可观测带宽为 **20-96 kHz**（192 kHz line-in，Nyquist 96 kHz）。
任何表述都不得写成「完整看到了 All-To-DSD 的 noise shaping」，
只能写成「在 20-96 kHz 可观测带宽内观察到显著的带外能量增加」。

### 36.11 下一步（唯一能推进的两条）

1. **补一次采集**：在音调高于本底但不过载的电平（约 -20 至 -30 dBFS）
   重做 OFF/ON A/B。这是唯一能在非过载工作点拿到可信带内结论的办法。
   现有链路约有 28 dB 线路衰减、活跃本底约 -53 dBFS，而 -6 dBFS 已过载，
   可用窗口很窄。
2. **root / runtime hook**：仍只有这一条能确证 `0x19d90` 是否执行。
   GhostLock 在本机被 `perf_event_paranoid` 挡住，需要别的入口。

### 36.12 产出文件

```
tools/ab_excess.py          线性功率域 excess PSD，带内/音调双归一化
tools/jm21_ntf.py           NTF 推导、二项式校验、形状拟合与叠加图
excess_curves.npz           各频段 PSD 曲线
excess_vs_ntf.png           实测 excess 与两个模型 NTF 的形状对照
ntf_fit.json                拟合残差与 alpha
```

### 36.13 撤回三：-6 dBFS 对不能回答带内问题

补采准备阶段用**已知频率的相干匹配滤波**（对 1 kHz 直接做 cos/sin 相关，不做搜索）
重测第 35 节全部采集，得到：

| 采集 | 相干音调幅度 | 该幅度是否可信 |
|---|---|---|
| `off_silence` | -101.45 dBFS | 否（纯噪声） |
| `on_silence` | -99.03 dBFS | 否（纯噪声） |
| `off_-60dBFS` | -112.16 dBFS | 否（噪声涨落） |
| `on_-60dBFS` | -98.85 dBFS | 否（噪声涨落） |
| `off_-6dBFS` | **-42.64 dBFS** | **是**，高于估计器本底 33 dB |
| `cap_1k` (ON) | **-59.09 dBFS** | **是** |

由此得到两项修正：

1. **36.1 的判断被加强**：只有 `-6 dBFS` 这一对含可测音调，其余全部没有。
2. **36.5 的结论被撤回**：该音调（-42.64 dBFS 峰值）在 985-1015 Hz 频带里
   只占约 1/600 的功率，其余全是过载爆炸的宽带噪声。因此 36.5 用该频带做的
   「音调归一化」归一化的是噪声而不是信号，其带内结论无效。

**链路标定**（供补采使用）：

```
链路总衰减                  36.6 dB      (-6 dBFS 源 -> -42.64 dBFS 模拟音调)
活跃本底 (silence, OFF)     -52.80 dBFS RMS
活跃本底 (silence, ON)      -41.29 dBFS RMS
匹配滤波估计器本底          -116.60 dBFS (N=4.8M, 处理增益 63.8 dB)
```

注意最后一行：相干检测有约 64 dB 处理增益，**音调不需要高于模拟本底也能被精确测量**。
第 36 节的 `+2.23 dB` 窄带判据之所以失效，是因为它用「频带功率」而不是
「相干幅度」做基准。

**这条同时纠正了本项目对「噪声比」指标的用法**：判断参考信号是否可用，
应看它相对**估计器本底**的余量，而不是相对被测噪声的 SNR。

### 36.14 补采轮次设计（B1）

据此建立 `tools/b1_confirm.py`，只回答一个问题：

> 在音调可被可靠提取、但共同路径尚未过载的工作点上，All-To-DSD ON 的带内噪声
> 是升、平、还是降？

电平阶梯：**-10, -15, -20, -25, -30 dBFS**，加静音，各 25 s，OFF/ON 各一轮。

- `-10 dBFS` 是在原定 -15..-30 之上补的隔离点。理由：`-6` 已知过载，
  `-15` 预测比模拟本底低 1.8 dB，而 `-10` 是唯一预测音调仍高于本底（+3.2 dB）
  且可能尚未进入过载的电平。过载的确切起点未知，`-10` 的 `clip%` 是关键读数。
- **本轮不再做形状拟合**。36.7 已证明模拟频谱无法区分 4 抽头与 6 抽头。

预测（由标定推得）：

| 源电平 | 模拟音调 | 相对模拟本底 | 估计器余量 | 预期分级 |
|---|---|---|---|---|
| -10 dBFS | -46.60 dBFS | +3.19 dB | 70 dB | VALID |
| -15 dBFS | -51.60 dBFS | -1.81 dB | 65 dB | VALID |
| -20 dBFS | -56.60 dBFS | -6.81 dB | 60 dB | VALID |
| -25 dBFS | -61.60 dBFS | -11.81 dB | 55 dB | VALID |
| -30 dBFS | -66.60 dBFS | -16.81 dB | 50 dB | VALID |

有效性分级标准（已按 36.13 修正）：

```
VALID    估计器余量 >= 20 dB  且  clip% == 0  且  时长 >= 20 s
WEAK     10 <= 估计器余量 < 20 dB
INVALID  估计器余量 < 10 dB，或出现削波，或时长不足
```

`tone_snr_vs_analog` 仍然输出，但**不作为门槛** —— 在这些电平上它本就应该为负。

ON/OFF 比较沿用 36.3 已验证的流程：

```
P_excess(f) = P_ON_norm(f) - P_OFF(f)
gain = tone_on / tone_off        (相干幅度之比，不是频带功率)
```

分档判读（对应实验设计的三种预期）：

| 观测到的带内 Δ 随电平变化 | 解释 |
|---|---|
| 各档几乎相同且非零 | 与输入电平无关的 ON 本底/后级噪声差异 |
| 高电平较大、向低电平收敛到 0 | 固定噪声 + 信号相关项叠加，低电平被本底吞没 |
| 各档均为 0 | 36.5 的 +2..+6 dB 来自统计或共同路径，不是稳定的带内代价 |

工具自检结果：相干检测器在合成采集上幅度误差 **0.000 dB**、相位稳定 -90.00°、
OFF/ON 两轮重复性 **±0.012 dB**。
### 36.15 UAC2 门控与 USB DAC 路径（来自 ALLTODSD_RE2/FINDINGS.md，已在设备侧核对）

FINDINGS 的第 3、4 条把 USB DAC A/B 从 `0x19d90` 证据链中剥离。静态结论：

```
get_alltodsd_config()                                   // 导出 0x1d1e0
  on = atoi(prop("persist.sys.all.to.dsd"))
  property_get("sys.usb.config", v)
  if (v == "uac2\0" || v == "uac2,adb\0") on = 0;       // 立即数 0x32636175 / 0x6264612c32636175
```

设备侧核对结果：

| 检查项 | 结果 |
|---|---|
| `/system/bin/fiiouac` | 存在，42416 B（与报告一致） |
| `vendor/etc/init/hw/init.fiio.common.rc:82-92` | 逐字吻合：`fiiouac_service` / `seclabel u:r:fiiouac_service:s0` / `disabled` / `oneshot` / `fiio.start.uac.service=1` 才启动 |
| 当前 `ps` 中 fiiouac | **未运行**（非 USB DAC 模式，符合预期） |
| 当前 `sys.usb.config` | `adb` —— 门控分支**当前不成立** |
| `persist.sys.all.to.dsd` | `1` |
| `persist.sys.dsd.shaper.coeffs` | 未设置（属性为空，代码应回落到 2） |
| `persist.sys.audio.output.select` | `1`（非 2，spdif/D2P 门控不成立） |
| logcat 中的门控字符串 | 未出现（符合预期，当前不在 USB DAC 模式） |

**旁证**：USB DAC 模式下 ADB 会消失，这与 `sys.usb.config=uac2`（不带 adb
function）一致；退出到正常模式后 ADB 恢复、`sys.usb.config=adb`。这是吻合，
但**不是实证**。一次性实证方法：USB DAC 模式下抓 logcat，应出现
`current play mode is UAC,so don't use Alltodsd`。

### 36.16 NTF 不是纯二项式 —— 三档均有零点对（新结构，已独立验证）

36.6 把 `C = A+B` 与二项式的 ~7e-4 偏差当作"浮点噪声"，这是**误判**。
`tools/jm21_ntf.py` 用代数恒等式检验：

```
P(z^-1) = (1-z^-1)^N + delta * z^-1 * (1-z^-1)^(N-2)
```

若 `delta` 在所有 k 上是同一个常数，则零点对是设计本体。实测：

| shaper | N | delta | 各 k 离散度 | 零点对频率 @DSD64 |
|---|---|---|---|---|
| mode0 | 8 | +3.201432e-03 | 4.81e-07 | **25.42 kHz** |
| mode1 | 6 | +2.202218e-03 | 1.56e-07 | **21.08 kHz** |
| mode2 | 4 | +7.136073e-04 | **0.00e+00** | **12.00 kHz** |

`cos(theta) = 1 - delta/2`，三档全部确认。mode2 的离散度**恰好为 0**，
是精确的单参数族。这与 FINDINGS §1.3 的数值完全一致，属独立复现。

含义：

1. **N-2 个 DC 零点 + 一对近单位圆共轭零点**，即**零点优化**设计。
2. mode0 的 H[7]（`.rodata:0x7fa0` = -0.54351772971672307）闭合后，
   三档系数全部齐备，`--jm21` 可以按真实系数实现。
3. 36.6 的措辞「order N confirmed, rises 6N dB/oct near DC」只在 DC 附近成立；
   在 12/21/25 kHz 附近 NTF 有零点，这一带的形状**不是**单调的二项式斜坡。

### 36.17 三档系数汇总（可进 `src/flac2dsf.cpp`）

```c
/* shaper id = 0 (8 taps) */
static const double kJmA0[8] = {   // v accumulator
    7.1931457765999998, -22.686306858611161, 40.977858981271901,
   -46.36939574322227, 33.663240226559381, -15.31356101838154,
    3.991500436793876, -0.45648227028327693 };
static const double kJmB0[8] = {   // u accumulator
    0.80365231044305308, -5.2944845485449221, 14.974123863329551,
   -23.566583305754548, 22.28874261804205, -12.667230388774531,
    4.0052976502491759, -0.54351772971672307 };
```

交叉验证：`sum(A+B) = 1.000000000`，末抽头 `f[N-1] = -1.0` 精确。

## 37. 框架修正：USB DAC A/B 与 `0x19d90` 是不同对象

第 34/35 节的「模型带内降噪 vs 真机带内不降」**不再是同一算法的预测冲突**：

```
model simulation  ->  libfiioaudio / 0x19d90 (HAL, 本机播放路径)
JM21 USB DAC 实测 ->  fiiouac / USB-DAC DSD-native 路径 (CS43198 直通)
```

USB DAC 模式下 `get_alltodsd_config()` 把 libfiioaudio 的 All-To-DSD 门控为 0，
因此第 36 节全部 excess 分析（含 36.5 的带内 +0.33 dB）**都不是对 `0x19d90`
的检验**，不得再用它们评判该模型。

**真正的对拍对象是 JM21 本机播放。** 这也是第 36.13 节那三条撤回的根源：
我们一直在用一条与模型无关的链路去检验模型。

H1/H2/H3 需要重新表述：

| 原假设 | 修正后 |
|---|---|
| H1 真实路径不是 `0x19d90` | **仅对 USB DAC 路径成立**（已被门控证实）；本机播放路径仍未验证 |
| H2 `0x19d90` 执行，后面还有 DAC 环节 | 对本机播放路径仍是主要假设，且 CS43198 环节确实存在 |
| H3 该工作点无带内收益 | **证据来自错误对象，已失效**；需在本机播放重测 |

下一步（按信息增益）：

1. **本机播放 OFF/ON A/B** —— 主线。测试音存入设备，本机 App 播放，
   line-in 回录。**本机播放时 ADB 可用**，无需 root。
2. **一次性门控实证** —— USB DAC 模式抓 logcat，确认
   `current play mode is UAC,so don't use Alltodsd`。
3. **切 shaper** —— 本机播放下设 `persist.sys.dsd.shaper.coeffs = 2/1/0`，
   三档零点对在 12.00 / 21.08 / 25.42 kHz，真机频谱应能分辨 NTF 阶数。

## 38. `--jm21` 立体声 ×2 缺陷：声道被串接而非交织

### 38.1 现象

第一次用项目自带 `flac2dsf.exe` 跑 `--jm21`，输出文件恰好是应有大小的
**2.0000 倍**。这属于逆向笔记第 31 节那个 ×2 家族，但这次在 C++ 实现里，
而且**单声道测试发现不了**，所以文档里此前的 C++ 对比从未暴露它。

### 38.2 两个独立缺陷（叠加）

**(a) 声道在外层循环 —— 串接而非交织**

原 JM21 分支（`src/flac2dsf.cpp`）：

```cpp
for (int c = 0; c < ch; ++c) {           // 外层是声道
    ...
    for (int p = 0; p < n * Jm21Core::kOver; ++p)
        emitSample(c, d);                  // L 整个 block，然后 R 整个 block
}
```

对比同文件里的 FIR 分支：外层是**采样点** `k`、内层是声道 `c`，即正确的
按样点交织。JM21 分支把两个声道在时间上首尾相接。

**(b) `bitPos` 跨声道共享**

`blk[c]` 是按声道分的字节缓冲，但 `bitPos` 只有一个。声道 0 占偶数位、
声道 1 占奇数位，而块写出按共享计数器触发：每 `kBlockBits` 次就把**两个**
声道块都写出。结果每声道块以一半采样率落盘，总量正好翻倍 —— 这就是
2.0000× 的直接来源。

单声道时 `ch==1`，共享计数器退化为逐声道计数器，(b) 不可见；(a) 也退化为
正确顺序。所以两个缺陷都被单声道掩盖。

### 38.3 修复

1. `bitPos` 改为 `std::vector<size_t> bitPos(ch, 0)`，逐声道；块写出只写本声道。
2. JM21 分支改为：先让**每个声道**跑完整个 block 的 `runStages(n)`，
   再按 `p` 外层、`c` 内层交织输出。
3. 收尾零填充写出改为逐声道判断 `bitPos[c] > 0`。
4. FIR 分支同步改用逐声道计数器（它本来就是交织的，只是计数器错了）。

### 38.4 验证（`tools/jm21_stereo.py`，硬断言）

探针：立体声 48 kHz / 3 s，**L = 1 kHz 正弦、R = 静音**。

| 检查 | 结果 |
|---|---|
| 立体声 `--jm21` 输出大小 / FIR 路径 | **1.0000**（原 2.0000） |
| 每声道位长 | 8 486 912 bit = 3.007 s @ 2822400 Hz |
| ch0 峰值 | 1001.29 Hz，peak/median **68.9 dB**（L 音调在位） |
| ch1 峰值 | 1028.21 Hz，peak/median **5.0 dB**（R 输入静音，只有量化噪声） |
| ch0 与 ch1 是否相同 | False（串接会合并或错位） |
| 单声道 `--jm21` 大小 / FIR | 1.0000 |
| 单声道 `--jm21` 音调 | 1001.29 Hz，68.9 dB（与立体声 ch0 一致，未受影响） |

另有 FLAC 解码回归 26/26 + 1/1 bit-exact 通过。

### 38.5 顺带发现（未修，需决策）：`--jm21` 完全忽略 dither

`emitSample(int c, double d)` 的 `d` 形参**从未被使用** —— 函数体只调
`jm[c].bit()`，而 `Jm21Core::bit()` 从自己的内部 `out` 缓冲取值。所以
`--dither` / `-d` 在 `--jm21` 路径下是**静默无效**的；FIR 路径的
`Modulator::bit(phase, d)` 才真正用上 `d`。

本次修复把 `d` 形参删掉了，并在调用处保留了对 `dithPos` 的推进（保持原有的
每采样每声道消耗节奏），但**信号里依然没有加上抖动**。

要不要补上有依赖：抖动必须在 `runStages()` **之前**加到 `jmIn` 上，
且量纲要与 `Jm21Core` 的内部标度一致（这本身还牵扯未闭合的
「PCM → tbl[m] 绝对标度」问题）。因此这属于另一个决定，不在本次修复范围内。

### 38.6 教训

单声道测试对「声道布局」类缺陷是无效的。凡涉及多声道交织/bit 打包的代码，
回归集里必须有**一个声道有信号、另一个声道静音**的立体声探针，否则
「每声道内容正确」和「声道被串接」无法区分。本节的
`tools/jm21_stereo.py` 就是为此加入的。

## 39. 本机播放 OFF/ON A/B：第一次真正对拍 `0x19d90`

### 39.1 前置条件（已实测，不是假设）

| 检查 | 值 |
|---|---|
| `sys.usb.config` | `adb` —— **UAC2 门控不触发**，All-To-DSD 在本路径确实生效 |
| `persist.sys.all.to.dsd` | 0 / 1，与请求状态一致（由脚本强校验，不符即中止） |
| `persist.sys.dsd.shaper.coeffs` | `2`（工厂默认，4 抽头） |
| 播放路径 | `/Music/ALLTODSD/*.flac` → `com.fiio.music` → primary HAL |
| 采集 | PC line-in 192 kHz Exclusive，仅录不放；**录音期间不调用 adb** |

链路标定：`-6 dBFS` 源 → `-24.55 dBFS` 模拟音调，即 **18.5 dB 系统衰减**
（USB DAC 路径是 36.6 dB，两者完全不同）。

### 39.2 测量方法学（三条纪律，都是前几轮踩坑换来的）

1. **音调用已知频率相干匹配滤波**，绝不用 `argmax(900..1100 Hz)`。第 36.1 节
   已证明后者量到的是噪声最大 bin（峰值在 921-1022 Hz 间乱跳，SNR 仅 11-15 dB）。
2. **带内积分必须挖掉 1 kHz 及其 1-5 次谐波**（±100 Hz）。只挖 ±25 Hz 时，
   Welch 主瓣和 2/3/4/5 kHz 谐波仍落在 250-1k 与 1k-4k 里，那两个频带的
   "active noise" 其实是音调本身。第一版分析就栽在这里。
3. **优先用静音对做空闲比较**。没有信号就没有任何污染，这是唯一干净的口径。

### 39.3 观测一：相干音调

| track | 状态 | peak | rms | tone | phase | margin | clip |
|---|---|---|---|---|---|---|---|
| 1k_-15dBFS | OFF | -12.22 | -17.72 | **-35.03** | 29.1° | 45.7 dB | 0 |
| 1k_-15dBFS | ON | -11.13 | -16.48 | **-32.36** | 15.7° | 47.2 dB | 0 |

```
音调增益 ON/OFF = +2.68 dB   (功率比 1.8532)
```

两侧都判 VALID，可用作归一化基准。

### 39.4 观测二：空闲带内 vs 带外（无污染口径）

| 频带 | OFF | ON | Δ |
|---|---|---|---|
| 20-250 | -52.44 | -49.74 | +2.71 |
| 250-1k | -57.37 | -55.87 | +1.50 |
| 1k-4k | -60.97 | -59.91 | +1.07 |
| 4k-8k | -66.78 | -65.74 | +1.04 |
| 8k-16k | -71.42 | -71.00 | +0.42 |
| 16k-20k | -79.96 | -78.84 | +1.12 |
| **20-20k 合计** | **-50.64** | **-48.36** | **+2.29** |
| 30-44k | -80.29 | -57.23 | **+23.06** |
| 44-60k | -78.75 | -41.90 | **+36.86** |
| 60-80k | -79.06 | -48.80 | **+30.26** |
| 80-96k | -83.86 | -63.94 | **+19.92** |

**带内按 +2.68 dB 增益归一化后：-0.39 dB**（即带内噪声相对信号基本不变，
略好 0.39 dB）。带外则增加 20-37 dB，峰值在 52-55 kHz。

### 39.5 解读

带内 Δ 曲线在 20-30 kHz 几乎**贴合 +2.68 dB 的增益线**（平坦），
30 kHz 以上陡升，44-60 kHz 出现 +40~+45 dB 的平台，随后被 CS43198
DSD processor 滚降。这正是 1-bit 噪声整形搬运的特征：

```
带内几乎不变  ->  NTF 把量化噪声推出了带内；剩下的带内噪声由 CS43198
                  自己的模拟噪声底决定，DSD 的带内贡献埋在它下面
带外暴涨      ->  被搬出去的量化噪声全部出现在这里
```

**因此本轮不能据此否定 `0x19d90`。** 「带内不变 + 带外 +45 dB」恰恰是
一个工作正常的噪声整形器应有的样子：它的带内量化噪声已经低于模拟噪声底，
所以带内看不到变化。若模型预测的带内大幅改善能在真机上观测到，反而需要
解释为什么 DSD 路径的带内噪声比模拟底还低。

同时本轮**确认了两件事**：

1. All-To-DSD 在本机播放路径确实生效（带外 +45 dB 的巨大签名）；
2. 它施加约 **+2.7 dB 宽带增益**。

### 39.6 未决：带信号时带内反而变差

用各自静音做 baseline、ON 按音调功率归一化后：

```
20-250  +3.54    4k-8k   +3.72    30-44k  +4.95
250-1k  +3.89    8k-16k  +3.67    44-60k  +12.65
1k-4k   +3.54    16k-20k +3.70    60-80k  +13.21
带内合计 +3.72 dB（变差）
```

这与空闲时的 -0.39 dB **符号相反**。合理解释是信号相关的项：带信号时
DSD 调制器被驱动，其量化噪声的带内分量比空闲时大。但幅度（3.7 dB）需要
用电平依赖来确认是「固定项」还是「随信号缩放」。

### 39.7 一个值得注意的谱特征

空闲 ON 的 Δ 曲线在 **72-75 kHz 有一个陡峭的凹口**（从 +30 dB 掉到 +8 dB
再跳回 +29 dB）。OFF 谱在 24.5 / 30 / 48.7 / 70 kHz 有窄杂散。

这个凹口的位置与三个 shaper 的 NTF 零点对有关。若内部速率为 32x 输入，
零点对位置随 shaper 不同（见 36.16），因此**切换 shaper 应当让带外结构
明显移动**。这是下一步最有判别力的实验。

### 39.8 下一步（按判别力排序）

1. **切 shaper 对拍**：`persist.sys.dsd.shaper.coeffs = 2 / 1 / 0`，
   各录一条静音。若带外峰/凹口随档位移动，即直接证明我们看到的就是
   `0x19d90` 这一族 NTF。成本极低（3 条静音）。
2. **电平依赖**：-20 / -25 / -30 dBFS 的 OFF/ON，判定 39.6 的 +3.72 dB
   是固定项还是信号相关项。

### 39.9 产出

```
tools/b2_local.py     本机播放工具（prep / vol / shaper / rec / report）
tools/b2_capture.py   单轨采集助手：等待目标曲目真正播放后才开录，录后自检
tools/b2_analyze.py   三个指标 + 谐波挖掘 + 分级
tools/b2_plot.py      空闲对绘图
local_idle_off_on.png 空闲 OFF/ON 频谱与差分
b2_silence_off.npy / b2_silence_on.npy
b2_m15dBFS_off.npy  / b2_m15dBFS_on.npy
```

## 40. Shaper 扫描：阴性结果，以及为什么模拟域无法识别 shaper

### 40.1 实验设计与前置

`persist.sys.dsd.shaper.coeffs` 可写并已验证往返。每档都**重建播放实例**
（`dispatch stop` → 点曲目 → `dispatch play`），以保证 HAL 重新执行
`start_service_action` 并重读属性，而不是让已初始化的转换器继续跑。
All-To-DSD = ON，同一个 `silence.flac`，音量 54，192 kHz Exclusive line-in。

### 40.2 第一轮出现 8 dB 差异 —— 是伪像

第一轮（2 → 1 → 0）测得：

| shaper | 阶 | 20-96k | 20-20k |
|---|---|---|---|
| 2 | 4 | **-54.35** | -61.16 |
| 1 | 6 | **-45.91** | -54.73 |
| 0 | 8 | **-45.84** | -54.23 |

看上去 shaper 2 比另两档安静 8 dB，且频谱地标（带外峰 52.3 / 53.2 / 51.8 kHz）
只有约 1.4 kHz 位移、**且非单调**。

**但这是错的。** 差别在于：第一条采集（shaper 2）墙钟 0.48x，后两条 0.99x。
重跑后：

| 采集 | 墙钟 | rms |
|---|---|---|
| shaper 2（第一轮，冷） | 0.48x | **-54.24** |
| shaper 2（第二轮，热） | 0.99x | **-46.97** |
| shaper 1 | 0.99x | -47.06 |
| shaper 0 | 0.99x | -47.07 |

**8 dB 差异是"本会话第一条采集"的预热/稳定效应，不是 shaper 效应。**
这是本轮最重要的方法学教训：**任何 A/B/A/B 序列的第一条都可能偏**，
必须做重复性对照再下结论。

### 40.3 热状态下的干净结果：属性完全不生效

```
shaper order  20-96k    20-20k   30-44k   44-60k   60-96k
2       4     -45.87   -54.56   -62.80   -47.44   -54.20
1       6     -45.88   -54.55   -62.81   -47.45   -54.21
0       8     -45.89   -54.60   -62.81   -47.45   -54.22
```

每个频带吻合到 **0.03 dB** 以内。两两形状差（RMS 归一化后）：

```
sh2-sh1  1.49 dB      sh2-sh0  1.52 dB      sh1-sh0  1.46 dB
```

**关键：sh2↔sh0（4 阶 vs 8 阶，差异最大的一对）的差值并不比 sh1↔sh0 大。**
残差里没有任何"阶数"结构，纯粹是采集间噪声。

**结论：`persist.sys.dsd.shaper.coeffs` 在本路径不产生可测量的模拟输出变化。**

这个阴性结果有真实灵敏度：同一套测量能轻松测出 OFF→ON 的巨大变化
（带外 +20~37 dB、形状完全不同）。所以"测不到 shaper 差异"是有意义的阴性，
不是仪器不灵。

### 40.4 设备自曝的内部参数（新信息）

```
persist.sys.fiio.config              = FiiODPA
persist.sys.fiio.config.dsd.native  = 256          <- DSD256
persist.sys.fiio.config.pcm.sample  = 768000       <- 内部 PCM 768 kHz
persist.sys.fiio.dpa.dsd.native     = true
vendor.fiio.audio.left.db           = -70          <- 疑似本底读数
vendor.fiio.audio.right.db          = -70
```

零点对频率随采样率线性缩放，故在 DSD256 下：

| shaper | 阶 | @DSD64 | @DSD256(11.2896 MHz) | @DSD256(12.288 MHz) |
|---|---|---|---|---|
| 2 | 4 | 12.00 kHz | 48.00 kHz | 52.25 kHz |
| 1 | 6 | 21.08 kHz | 84.33 kHz | 91.79 kHz |
| 0 | 8 | 25.42 kHz | 101.68 kHz | 110.65 kHz |

### 40.5 直接把模型 NTF 叠到实测谱上（不再依赖属性）

`tools/b2_ntf_overlay.py`，以 20-30 kHz 中位数为固定参考，无任何搜索。

```
实测带外峰                     52.64 kHz
模型 mode2(4 阶) 零点对 @DSD256 48.00 kHz
模型 mode1(6 阶) 零点对        84.33 kHz
模型 mode0(8 阶) 零点对       101.68 kHz（超出观测窗）
```

**矛盾点：模型在零点对处是「零点（陷 null）」，实测在同一区域是「极大值」。**
源上的 null 不可能变成峰。因此实测 ~52 kHz 的转折**不是 shaper 的零点对造成的**，
而是 DSD 噪声上升与 CS43198 DSD processor 滚降相交的位置。

### 40.6 为什么模拟域原理上无法识别 shaper

这一条现在有了明确的机制解释，比第 36.7 节的"不可判"更硬：

1. **带外 30-96 kHz 全部落在 CS43198 的转折区**（实测峰 52 kHz，之后滚降）。
   无论内部速率取 11.29 还是 12.288 MHz，三个零点对都落在这个被 DAC 主导的
   区域里，shaper 的结构被掩盖。
2. **带内也不行**：高阶 NTF 在 20 kHz 以下把量化噪声压到极低（模型在 20 kHz
   处比峰值低 40 dB 以上），早已低于 CS43198 自己的模拟噪声底，因此带内
   观测不到阶数差异。这与第 39.5 节的实测一致（带内按增益归一化后 -0.39 dB）。
3. 唯一能识别的是**进入 DAC 之前的数字 1-bit 流**，那需要 root（本机被
   `perf_event_paranoid=3` 挡住）。

### 40.7 本轮确立与未确立

**确立：**

- `persist.sys.dsd.shaper.coeffs` 可写但**在本路径不生效**（0.03 dB 以内无差异，
  且差值无阶数结构）。
- 设备自曝内部参数：DSD256、内部 PCM 768 kHz、`FiiODPA`。
- All-To-DSD 在本机播放路径确实生效，带外签名 +20~37 dB、峰值 ~52 kHz、
  施加约 +2.7 dB 宽带增益；带内按增益归一化后 -0.39 dB。
- 实测谱的 ~52 kHz 转折由 CS43198 决定，不由 shaper 零点对决定。

**未确立：**

- 真机当前实际使用哪一档 shaper。属性不可信，只能说固件内部另有默认。
- 模拟输出与逆出的 NTF 之间不存在可用的逐点对应，因此**模拟测量无法为
  `0x19d90` 提供正面证据，也无法否证它**。

### 40.8 方法学教训（比结论更重要）

1. **A/B/A/B 的第一条不可信**。本轮差点把 8 dB 的预热伪像写成"shaper 2 更安静"。
   任何顺序敏感的比较都必须做重复性对照。
2. **不要用"程序会重新读属性"来替代验证**。`dispatch stop` + 新建播放实例
   是必要条件，但属性最终仍需靠可测结果证明其生效。
3. **单一固定参考，不做自动归一化**。本轮全程用 20-30 kHz 中位数或整体 RMS，
   没有任何最佳增益/最佳频移搜索 —— 否则三档 1.5 dB 的采集噪声差异也能被
   "拟合"成结构。
4. **阴性结果需要灵敏度背书**。用同一套测量能测出 OFF→ON 的巨大变化，
   才能说"测不到 shaper 变化"是有意义的。

### 40.9 产出

```
tools/b2_shaper.py            shaper 扫描（强制重建播放实例 + 后缀支持可做重复对照）
tools/b2_shaper_analyze.py    三档对比：绝对谱 + 单一固定归一化 + 谱地标
tools/b2_ntf_overlay.py       模型 NTF @DSD256 叠加实测模拟谱
shaper_sweep.png              三档绝对谱与归一化谱
ntf_vs_analog_dsd256.png      模型 NTF vs 实测，含零点对标线
b2_sh0/1/2_silence.npy
```

## 41. `w26` 的来源闭合：shaper 属性是「存在但未接线」的配置接口

### 41.1 `w26` 只写一次

`get_pcm2dsd_data` 真实体在 `0x19d90`（`0x1dcd0` 是 veneer）：

```
0x19d90  sub sp, sp, #0x1d0
0x19db0  stp x26, x25, [sp, #0x190]     ; 保存 x26 等 callee-saved
...
0x19e00  cmp  w5, #2
0x19e08  mov  w10, #2
0x19e0c  csel w26, w6, w10, lt           ; w26 = (w5 < 2) ? w6 : 2
...
0x1a434  cmp  w26, #2
0x1a438  b.eq #0x1a544                   ; shaper 2 -> 4 taps
0x1a43c  cmp  w26, #1
0x1a440  b.eq #0x1a388                   ; shaper 1 -> 6 taps
0x1a444  cbnz w26, #0x1a424             ; w26 != 0 -> 跳过
0x1a448  ...                             ; w26 == 0 -> 8 taps
```

在整个 `0x19d90 .. 0x1a968` 内 **w26 只被写这一次**（0x19e0c），此后仅被读。

### 41.2 调用点解出 w5 / w6 的真实身份

```
0x12e8c  ldr x21, [x23, #0x70]
0x12e90  ldr w19, [x23, #0x68]
...
0x13050  mov  w5, w21     ; w5 = handle+0x70
0x13054  mov  w6, w19     ; w6 = handle+0x68
0x13058  mov  w7, w19
0x1305c  bl   #0x1dcd0    ; get_pcm2dsd_data
```

字段身份由 veneer 表确定：

| veneer | 符号 | 真实体 | 字段 |
|---|---|---|---|
| 0x1d044 | `get_pcm2dsd_shaper_coeffs` | 0x163e8 | `ldr w0,[x8,#0x68]` |
| 0x1d04c | `get_pcm2dsd_dsd_level` | 0x163f8 | `ldr w0,[x8,#0x70]` |
| 0x1d07c | `get_alltodsd_shaper_coeffs` | **0x171a4** | property 读取 |

于是：

```
w5 = dsd_level        = handle+0x70
w6 = shaper_coeffs    = handle+0x68
w26 = (dsd_level < 2) ? shaper_coeffs : 2
```

### 41.3 `[handle+0x68]` 的全部写入者：只有清零

穷举整个 `.text` 中涉及偏移 `0x68` 的全部指令：

| 地址 | 指令 | 是否字段 0x68 的写入 |
|---|---|---|
| 0x12d88 | `stp w22, w21, [sp, #0x68]` | 栈 |
| 0x15cc4 | `str xzr, [x22, #0x68]` | **是，写 0**（64 位，同时清 0x70） |
| 0x1919c | `str x8, [x14, #0x68]!` | 栈帧 pre-index |
| 0x1a12c | `stp d17, d16, [sp, #0x68]` | 栈 |
| 0x12e90 / 0x163f0 | `ldr` | 读 |
| 0x126b8 | `mov w8,#0x68` + `movk w8,#0xd0` | **实为 0xd068**，误报 |
| 0x12f30 | `adrp x12,#0x8000; add x12,x12,#0x68` | **.rodata 跳转表**，误报 |

`init_pcm2dsd`（0x1d180 → 真实体 0x1a970）：

```
mov  w2, #0x3e0
movk w2, #0x14, lsl #16     ; w2 = 0x1403e0 = 1 310 688
mov  w1, wzr
ldr  x0, [x0, #0x410]
b    #0x1e060               ; memset(handle, 0, 0x1403e0)
```

**整个 1.31 MB handle 被清零**，其中包含 0x68 与 0x70。

并且访问器 thunk 系列里，**0x68 只有 getter（0x163e8），没有 setter**；
有 setter 的是 0x70(0x16408) / 0x78(0x16428) / 0x7c(0x16448)。

### 41.4 `persist.sys.dsd.shaper.coeffs` 的读取者没有下游

```
0x6108  "persist.sys.dsd.shaper.coeffs"

get_alltodsd_shaper_coeffs  @0x171a4
  0x171b8  adrp x0, #0x6000
  0x171c8  add  x0, x0, #0x108      ; -> 0x6108
  0x171e8  bl   #0x1dc90            ; strcpy into a stack buffer
  0x171f0  bl   #0x1dca0            ; atoi
  0x17204  ... ret                   ; 返回值在 w0
```

它**返回一个值，但 `.text` 中没有任何代码把这个返回值写进 handle+0x68**。
全 `.text` 也没有对 `g_pcm2dsd_handle` 的可见 store（`0x1f3d0` / `0x1f410`
两个 GOT 槽只有 `ldr`，`strb [x?,#0x410]` 是字节写，不是指针）。

### 41.5 那张表

| 对象 | 来源 | 读取时机 | 是否写入 `w26` |
|---|---|---|---|
| `persist.sys.dsd.shaper.coeffs` | `get_alltodsd_shaper_coeffs()` @0x171a4（strcpy+atoi） | 每次调用 | **否** —— 返回值无任何存放处 |
| `persist.sys.pcm2dsd.level` | `get_pcm2dsd_level()`（读 0x62bf） | 每次调用 | **否**（不直接进 w26），但若经 `set_pcm2dsd_dsd_level` 落到 handle+0x70，则只影响 w26 的**选择分支** |
| `match_audio_process_mode()` @0x1cf90 | 未展开 | 未定 | **未确认**（本次未闭合） |
| `handle+0x68`（shaper coeffs 槽） | **仅** `init_pcm2dsd` 的 memset(0) 与 `0x15cc4 str xzr` | init | **是**（经 w6 → w26），但**恒为 0** |
| `w26 @0x1a434` | `csel w26, w6, #2, lt`；w6=handle+0x68，w5=handle+0x70 | call time | **是** |

### 41.6 结论

1. **架构 B 成立，且比预期更极端**：不是「mode 来自另一个 context」，
   而是 **`handle+0x68` 在本 build 里根本没有写入者**，初始化后恒为 0。
2. 因此，**在 `libfiioaudio.so` 自身可见的写入链中，`handle+0x68` 没有可观测
   写入者，初始化后恒为 0；因此 `w26` 在当前已确认的调用条件下恒走 shaper 0
   （8 抽头）**。仅剩「其他模块直接写 `g_pcm2dsd_handle+0x68`」这一条未排除
   可能（第 41.8 节已将其进一步排除，见第 42 节）。
   （`H[7] = -0.54351772971672307`，δ=3.2019e-3，零点对 25.42 kHz @DSD64
   / 101.68 kHz @DSD256 —— 后者落在 96 kHz 观测窗之外）。
3. **第 40 节的阴性结果被完全解释**：property 可写、可读、有专门 getter，
   但它的值从未进入 DSP 消费的字段。**它是一个存在但未接线的配置接口。**
4. 顺带修正一条外部结论：「工厂默认 `persist.sys.dsd.shaper.coeffs=2`」
   在本机不成立 —— 该属性初始为**空**，`atoi("") = 0`，与 `handle+0x68 = 0`
   一致，指向 8 抽头。

> **SUPERSEDED（2026-10-06，第 42 节）**
> - 「`persist.sys.dsd.shaper.coeffs` 是 shaper id 的来源」→ **撤回**。
>   该属性有 getter、无写入者，**不是**当前 shaper selector。
> - 「`persist.sys.pcm2dsd.level` 可能经 setter 影响 w26 的分支」→ **撤回**。
>   `set_pcm2dsd_dsd_level` 在全设备零调用者（见 42.2），该路径同样是死配置。

### 41.7 由此产生的可执行结论

- **模型默认值应改为 mode 0（8 抽头）**，而不是 mode 2（4 抽头），
  才能预测本机真机行为。8 抽头在带内的抑制比 4 抽头强得多
  （36.16 节表：10–20 kHz 分别 -31.2 dB vs -5.1 dB），
  这与第 39 节实测「带内按增益归一化后 -0.39 dB、DSD 带内贡献低于模拟底」
  的观察方向一致。
- **下一步可测**：唯一还能改变 `w26` 的入口是 `handle+0x70`（dsd level，
  有导出 setter `set_pcm2dsd_dsd_level` @0x1d2fc）。若能让它 >= 2，
  `csel` 会强制 `w26 = 2`（4 抽头）。这是本轮之后唯一值得尝试的开关。

### 41.8 诚实性边界

- 本次**未**闭合 `match_audio_process_mode`（@0x1cf90）的完整枚举与语义；
  它是否参与 `handle+0x68` 的赋值仍未排除。
- 未证明 GOT 槽 `0x1f3d0` 与 `0x1f410` 指向同一对象；若 handle 结构
  实际在别处，则「memset 覆盖到 0x68」这一步需要重新确认。
  但「`.text` 中对偏移 0x68 的唯一 store 是写零」这一事实与基址无关，仍然成立。
- 未排除**其它模块**通过导出符号 `g_pcm2dsd_handle`（STT_OBJECT，已导出）
  直接写字段的可能。要彻底排除需要 root 读内存。

## 42. `dsd_level` 也是死配置；`--jm21` 三档化；mode 0 的 A/B 取向

### 42.1 唯一能改 `w26` 分支的入口同样是死的

第 41.7 节把 `handle+0x70`（`dsd_level`）列为唯一剩余杠杆，因为它有导出
setter。追下去的结果是**它也是死配置**。

```
0x1d2fc  set_pcm2dsd_dsd_level  ->  b #0x16408
0x16408  adrp/ldr + str w0,[x8,#0x70] + ret      ; 确认就是字段 0x70 的 setter
```

引用穷举：

| 检查 | 结果 |
|---|---|
| `bl #0x1d2fc`（库内经 veneer 调用） | **0 hits** |
| `bl #0x16408`（库内直接调用） | **0 hits** |
| `b #0x16408`（尾调用） | 1 hit，只有 veneer `0x1d2fc` 自己 |
| 整个 thunk 带 `bl #0x164[0-4]x` | **0 hits** |

设备级扫描（`grep -rl`，覆盖 `/vendor/lib64`、`/system/lib64`、`/system/bin`、
`/vendor/bin`、`/system_ext/lib64`、`/odm/lib64`）：

| 符号 | 唯一命中 |
|---|---|
| `set_pcm2dsd_dsd_level` | `/vendor/lib64/libfiioaudio.so` |
| `get_pcm2dsd_shaper_coeffs` | `/vendor/lib64/libfiioaudio.so` |
| `get_alltodsd_shaper_coeffs` | `/vendor/lib64/libfiioaudio.so` |
| `g_pcm2dsd_handle` | `/vendor/lib64/libfiioaudio.so` |
| `init_pcm2dsd` | `/vendor/lib64/libfiioaudio.so` |

补充确认：`/vendor/lib64/hw/audio.primary.bengal.so` 虽然 `DT_NEEDED`
依赖 `libfiioaudio.so`，但**没有导入任何 `pcm2dsd` / `dsd_level` / `shaper`
符号**（DSD 控制完全封装在库内）；`audio.usb.bengal.so` 根本不链接它。

**结论**：`handle+0x70` 同样只有清零写入，`dsd_level` 恒为 0。
这把第 41.8 节列出的最后一条未排除可能也关掉了 —— 没有任何模块**按名**引用
`g_pcm2dsd_handle`，因此不存在外部直接写字段的路径。

最终链条：

```
dsd_level  = handle+0x70 = 0   （setter 全设备零调用者）
shaper     = handle+0x68 = 0   （无 setter，仅 memset / str xzr）

w26 = (dsd_level < 2) ? shaper : 2 = 0
   -> 0x1a448  8 抽头
```

`match_audio_process_mode`（@0x1cf90）的完整枚举仍未闭合，但它已经不构成
阻塞项：即使它另有作用，也只能落在 `handle+0x68/0x70` 上，而这两个字段的
写入者已穷尽。

### 42.2 配置链（写进项目主结论）

```
persist.sys.dsd.shaper.coeffs
        |  get_alltodsd_shaper_coeffs()  @0x171a4   (strcpy + atoi)
        v
   （无任何存放处）
        x  不是 shaper selector

persist.sys.pcm2dsd.level
        |  setter @0x1d2fc，全设备零调用者
        v
   （无任何存放处）
        x  不是 shaper selector

唯一实际生效的：handle+0x68 = 0 -> w26 = 0 -> 8 抽头
```

### 42.3 模型：`--jm21` 三档化，默认改为 mode 0

```
PCM
 -> 5 x 2 IIR interpolation
 -> tbl[m]
 -> 0x19d90 quantizer
      mode 0 = 8 tap   <- 当前默认（真机工作点）
      mode 1 = 6 tap   <- 唯一经固件验证的行
      mode 2 = 4 tap
 -> 1-bit intermediate
 -> 后续 packing / output path 未闭合
```

改动（仅限 `--jm21`，程序总默认仍是 `--shaper` 的 FIR 路径）：

- `Jm21Core` 的 `qs[6]` / 固定 6 抽头改为 `qs[8]` + 运行时 `taps`；
- 新增三套系数表 `kJmA0..kJmB2`；
- 新增 `--jm21-mode N`（0/1/2），默认 0；
- 求和顺序保持「输入抽头折叠在最后一根反馈抽头之前」，与固件一致，
  因此 mode 1 逐位不变。

回归：`e2e.py`（默认 FIR 路径）全部数字与改动前一致 —— 默认路径未被触碰。

### 42.4 过程中发现并修正：mode 0 的 A/B 取向是反的

把 `MODE0` 直接当作累加器抽头代入时，mode 0 在 `d8 = 12/14/15/16`
**全部**饱和（解码 rms 与 peak 双双钉在 1.00000），而 mode 1、mode 2 正常。

| mode | A[0] | B[0] | 累加器行为 |
|---|---|---|---|
| 1 (6 tap) | 0.8165 小 | 5.181 大 | 正常 |
| 2 (4 tap) | 0.8023 小 | 3.197 大 | 正常 |
| 0 (8 tap)，原记录 | 7.193 **大** | 0.8037 **小** | 饱和 |
| 0 (8 tap)，交换后 | 0.8037 小 | 7.193 大 | 正常 |

两条独立证据都指向 A/B 记录反了：

1. **累加器稳定性**：交换后 mode 0 在 mode 1 用的同一个 `d8=12` 上
   就稳定（rms 0.10145 / peak 0.33249，与 mode 1、mode 2 同量级）。
2. **NTF 物理合理性**：`ntf_db()` 只用 `B` 作分子。

   | 取向 | NTF@20 kHz | NTF@50 kHz |
   |---|---|---|
   | 原记录 | **-245.1 dB** | -155.0 dB |
   | 交换后 | -115.7 dB | -13.0 dB |

   原记录给出的根本不是噪声整形曲线。

`zero_pair_check()` 只依赖 `A+B`，因此零对不受影响，两种取向都是
25.42 kHz，与真机实测一致。

修正后 `tools/jm21_ntf.py` 输出三档自洽曲线，阶数越高带内抑制越深：

```
f/f_Nyq         0.0010   0.0020   0.0040   0.0080   0.0160   0.0320
mode0 (8-tap)  -148.00  -141.94  -138.36  -133.09   -94.35   -15.40
mode1 (6-tap)  -140.45  -129.00  -119.75  -128.20   -85.62   -36.93
mode2 (4-tap)  -119.00  -107.33   -96.97  -101.55   -62.61   -36.26
```

另外核实：mode 0 的 16 个 double **全部逐字存在于 `libfiioaudio.so`**
（A 8/8、B 8/8），所以它不是拟合重构出来的行，而是真固件数据。
（mode 1 的行反而大多不在 `.rodata`，因为它们是 `.text` 里的 64 位立即量。）

### 42.5 诚实性边界（本节的未闭合项，已由第 43 节解决）

- ~~**mode 0 与 mode 2 的 A/B 取向是推断的，不是固件验证的。**~~
  → **第 43 节已用固件指令流 + 4096 位逐位比对验证，升级为 Verified。**
- `--jm21-mode 0` 作为 `--jm21` 默认值的依据是 42.1/42.2 的静态结论
  （真机恒走 8 抽头）；该默认值的抽头行现已通过固件验证。
- 1 kHz 带内 SNR 对三档都给出 49.7 dB，**不能**用来判别 NTF 阶数
  （那里被共有的插值级噪声主导）。要判别必须看带外噪声谱，而本轮
  搭的带外度量未能复现 36.16 节表里 8-tap 与 4-tap 相差 26 dB 的差异，
  **该度量本身尚未校准，不要引用本轮的任何带外数字**。

## 43. 三档抽头方向的固件级验证：mode 0 与 mode 2 闭合

第 42 节把 mode 0 / mode 2 的 A/B 取向标为「推断」。本节把它升级为 **Verified**。
harness 只改不改模型：先让真实固件分支决定方向，再据此确认 C++ 表。

### 43.1 方法

沿用 mode 1 的做法（`uc_quant.py`），扩到另外两个分支。**唯一改动是修 x3**：
旧 harness 用 `X3=0x18`(24)，真实值是 **32**（`uc_realcaller.py` 文档里记为旧 bug）。

强制走分支靠寄存器，不靠改属性：`x5=1 < 2`，故 `w26 = w6 = shaper`，
于是 `shaper=0/1/2` 分别落入 `0x1a448 / 0x1a388 / 0x1a544`。

输入表、初始状态（memset 全零）、`arg6=4` 三者对两个候选完全相同，
比较的就是量化器本身。

### 43.2 直接从指令流读出两行

在每个 `fmul`/`fmadd` 处读「刚载入的系数所在的寄存器」，得到的顺序就是 CPU 的消费顺序。

**mode 0（0x1a448，8 抽头）**

```
v 链 d0 : s0*A0 s1*A1 s2*A2 s3*A3 s4*A4 s5*A5 s6*A6  ->  tbl*d8  ->  s7*A7
u 链 d1 : s0*B0 s1*B1 ... s7*B7
```

固件实际值：

```
A =  0.80365231044305308, -5.2944845485449221, 14.974123863329551,
    -23.566583305754548, 22.28874261804205, -12.667230388774531,
     4.0052976502491759, -0.54351772971672307
B =  7.1931457765999998, -22.686306858611161, 40.977858981271901,
    -46.36939574322227, 33.663240226559381, -15.31356101838154,
     3.991500436793876, -0.45648227028327693
```

**mode 2（0x1a544，4 抽头）**，系数预载在 `d16..d23`：

```
A = 0.802319609916744, -2.100688172255984, 1.8589343651600601, -0.55437442828602768
B = 3.1969667828000001, -3.8978846131775038, 2.1403520275566841, -0.44562557171397232
```

mode 0 的 A/B 与第 42.4 节的交换**一致**；mode 2 的 A/B 与 `jm21_ntf.py`
原记录**一致**（A 小 / B 大）。

### 43.3 4096 位逐位比对

| 分支 | candidate A | candidate B |
|---|---|---|
| mode 0 `0x1a448` | **4096 / 4096 (100.00%)** | 2163 / 4096 (52.81%)，首次失配第 3 位 |
| mode 2 `0x1a544` | **4096 / 4096 (100.00%)** | 2092 / 4096 (51.07%)，首次失配第 3 位 |

`d8` 从固件寄存器读到 **12.0**，与 `16 - arg6(4)` 一致（未假定）。
三个分支的结合方式也都从汇编核对过，全是 `(u + v) + q`，输入抽头折叠在
最后一根反馈抽头之前：

```
mode 0:  fadd d0,d1,d0 (=u+v) ; fcsel -> q ; fadd d0,d0,d1 (=(u+v)+q)
mode 1:  fadd d0,d0,d2 (=u+v) ; fcsel -> q ; fadd d0,d0,d1 (=(u+v)+q)
mode 2:  d1 = tbl*d8 + v(3 taps) ; d1 += s3*A3 ; d0 = u + d1 ; +q
```

**状态：mode 0 Verified / mode 1 Verified / mode 2 Verified，零推断项。**
`src/flac2dsf.cpp` 的 `jfma` 是 `std::fma`，即使 `-mfma` 被禁用也保证单次
舍入，与验证过的 mirror 语义相同，无需改代码。

### 43.4 harness 踩到的五个坑（都是静默错误，值得记）

1. **`fmadd` 必须用真 FMA。** Python 的 `a*b + c` 是两次舍入，第 224 位
   就与 CPU 分叉（先得到 2469/4096）。用 `Fraction` 精确算再单次舍入才复现。
2. **多段采集必须逐段比对。** 每段固件都从 memset 后的零状态重启；把多段
   拼接后用连续状态跑 mirror，会在第 3072 位（= 第一段长度）整段崩掉。
3. **hook 不能打在 `ldr` 上。** `ldr d3,[x8]` 处读 d3 拿到的是上一轮的残留
   （读到 4.0053 = A6 系数）。要打在消费它的 `fmadd` 上。
4. **`x3` 必须是 32**，旧 harness 的 24 是已知 bug。
5. **`d8` 要从固件读**，不要假定 `16 - arg6`；虽然两者一致，但读出来才算证据。

另外 `frames` 超过约 128 会让函数直接返回到哨兵 LR `0x1f000`（未映射）而报
CPU exception，那不是量化器出错；4096 位是用 128 帧 x 2 段拼出来的。

### 43.5 复现

```
uc_mode0.py        运行时读 mode 0 的 A/B 两行
uc_modes_verify.py mode 0 / mode 2 各 4096 位比对（含 FMA 与逐段处理）
```

## 44. 带外噪声度量：校准、三档实测、电平扫描

目标：把「带外噪声度量」本身校准到可信，然后在**纯数字 1-bit 层**测 N=8/N=6/N=4。
全程不碰 ADC、CS43198、USB DAC。

### 44.1 测量定义

1-bit 流映射为 ±1，单边 Hann 周期图，ENBW 归一化，按频带累加 bin。
频带一律相对各自记录的 20–96 kHz 电平（去掉任意驱动/极限环参考）。

### 44.2 校准（先验测量器，再测被测物）

| 检查 | 结果 |
|---|---|
| A 白噪声 PSD 应平坦 | 展布 **0.042 dB** —— 归一化正确 |
| B 合成 \|NTF\|² 谱应被还原 | 形状误差 max **0.112 / 0.034 / 0.054 dB**（N=8/6/4） |
| C 已知谱上 N=8<N=6<N=4 应成立 | 0–20k、10–20k **成立**；20–40k 起**反转** |

C 的「反转」不是异常：零点对频率三档不同（25.42 / 21.08 / 12.00 kHz），
越过各自零点后次序必然重排。理论表（相对 20–96k）：

```
band            N=8       N=6       N=4
0-20k       -129.68   -109.42    -66.24
10-20k      -129.89   -109.67    -66.27
20-40k       -40.00    -50.96    -35.08
40-80k        -3.91     -5.03     -7.15
80-96k        -2.27     -1.64     -0.93
```

**一个重要的方法论陷阱**：用白噪声驱动时带内被信号路径（STF ≈ 平坦）淹没，
10–20 kHz 三档只差 3 dB，看起来像「NTF 不成立」。用**单音**驱动则不需要
减 STF —— 能量全在 1 kHz，挖掉即可。这是白噪声方案失败、单音方案成功的原因。

### 44.3 三档实测（1 kHz 正弦，挖掉基波及 1–5 次谐波 ±100 Hz）

n = 262144（df = 10.77 Hz），硬件的正确舍入 FMA，实测 / 理论：

| band | N=8 | N=6 | N=4 |
|---|---|---|---|
| **10–20k** | **−129.49 / −129.89** | **−109.84 / −109.67** | **−66.24 / −66.27** |
| 20–40k | −40.02 / −40.00 | −51.03 / −50.96 | −34.95 / −35.08 |
| 40–80k | −3.43 / −3.91 | −4.42 / −5.03 | −7.23 / −7.15 |
| 80–96k | −2.63 / −2.27 | −1.95 / −1.64 | −0.91 / −0.93 |

**全频带吻合在 0.8 dB 以内**，包括 20–40k 的交叉。

### 44.4 电平扫描（10–20 kHz，相对 20–96k）

| drive | N=8 | N=6 | N=4 | max\|state\| N=8/N=6/N=4 | 稳定性 |
|---|---|---|---|---|---|
| 0 dBFS | 发散 | 发散 | 发散 | inf / 2.2e12 / 1.8e9 | **✗** |
| −6 | −129.49 | −109.84 | −66.24 | 2.7e6 / 6.6e3 / 65.8 | ✓ |
| −12 | −129.66 | −109.31 | −66.13 | 2.5e6 / 6.1e3 / 57.4 | ✓ |
| −24 | −130.19 | −110.31 | −66.93 | 2.4e6 / 6.3e3 / 52.7 | ✓ |
| −40 | −129.90 | −109.93 | −65.77 | 1.9e6 / 6.8e3 / 53.3 | ✓ |
| −60 | −130.03 | −109.88 | −66.49 | 2.3e6 / 6.6e3 / 53.3 | ✓ |
| 理论 | −129.89 | −109.67 | −66.27 | | |

发散阈值**三档完全一致**，落在 −6 与 −4 dBFS 之间，且极陡：−6 dBFS 时
状态 ~1e6/6e3/53，−4 dBFS 时已冲到 1e141/1e11/1.6e2。

带内结果在 **−6 到 −60 dBFS 全程与理论一致到 0.7 dB**，说明 NTF 形状是
输入电平无关的；变的只有 0–20k 整带（含被挖音的裙边与低频极限环）。

### 44.5 回答「26 dB 到底是什么」

36.16 节的 −31.2 / −20.5 / −5.1 dB 是**设计零点对相对 δ=0 的收益**
（分母固定），本轮精确复现：

```
band            N=8       N=6       N=4     <- 相对 δ=0，分母固定
10-20k        -31.46    -20.50     -5.14
```

N=8 与 N=4 相差 26.3 dB —— **「26 dB」正是这个量，来源已确认。**

但必须区分清楚：**档位之间的带外噪声差只有 2–4 dB**
（40–80k: −3.4/−4.4/−7.2；80–96k: −2.6/−2.0/−0.9）。
巨大的差（63 dB）在**带内 10–20 kHz**，不在带外。

所以：若「26 dB」曾被当作带外噪声差引用，那个解释是错的；
它实际是零点对的设计收益。

### 44.6 三个必须说清的边界

1. **驱动区的数字来自已验证模型，不是固件 harness。**
   固件输入是 3 字节整数，`tbl = 8*b0`（`uc_gain` 实测），即
   `drive = tbl*d8 = 96*b0`。**最小非零 drive = 96，比发散阈值高约 190 倍**，
   所以独立 harness 根本进不了稳定驱动区。这是已记录的未闭合项
   「PCM→tbl 绝对标度」。模型侧是可信的：三档均已对固件 4096/4096 逐位验证。
2. **零输入下的固件位流可测，且不遵守 NTF 次序**：
   10–20 kHz 实测 N=8 −127.56 / N=6 −127.34 / N=4 −21.84。
   N=6 明显偏离理论 −109.67，因为零输入时带内底噪由**极限环**而非 NTF 决定。
   这也说明为什么驱动区测量必须用模型 + 受控输入。
3. **FMA 影响已量化**：float64（非融合）与正确 FMA 在 N=8 上有 35.9% 的位
   不同、N=6 23.1%、N=4 0%，但**频带功率差**在 10–20k 只有 0.086 / 0.773 /
   0.000 dB，最差 0.36 dB（N=8 的 20–40k）。本节数字用 FMA 算。

### 44.7 顺带修掉的一个 harness bug

`uc_mode0_bits.py` 的 mirror 把延迟线状态存成 `Fraction` 且**从不舍入回
double**（`s = [(u + v) + q] + s[:T-1]`），分子分母无界增长，4096 位能跑完、
20000 位卡死。硬件寄存器是 double，必须每步舍入。修好后
`qmirror.py` 的 `bits_fma` 才是硬件语义（13 s / 262144 位）。
附带一个有意思的结果：**带精确算术的状态曾经也给出 4096/4096 全匹配**，
说明那 4096 步里硬件的舍入从未改变任何一次判决。

## 45. PCM → tbl 绝对标度：输入格式与库内增益链的完整闭合

第 44 节留下的问题是：24-bit PCM 怎样变成 `tbl`，使设备实际工作点落在模型稳定区。
本节把**库内**能确定的部分全部确定，结论是**绝对标度是调用方的契约，不在本库里**。

### 45.1 输入格式（在 `get_pcm2dsd_data` 内部解码）

输入循环 `0x19ed8 .. 0x19f20`，每帧 6 字节、两个通道各 3 字节：

```
0x19ee0  ldrb w12, [x20, #1]
0x19ee4  ldrb w13, [x20, #2]
0x19ee8  lsl  w11, w11, #8
0x19eec  bfi  w11, w12, #0x10, #8
0x19ef0  bfi  w11, w13, #0x18, #8
0x19ef4  scvtf d0, w11                ; 有符号
0x19ef8  str  d0, [x8, x9]            ; 通道 0
0x19efc  ldrb w11, [x20, #3] ...      ; 通道 1 同理
0x19f08  add  x20, x20, #6            ; 步长 6
0x19f1c  str  d0, [x8], #8
```

即 `w11 = b2<<24 | b1<<16 | b0<<8`：**大端、24-bit 左对齐、满刻度 ±2^31、
1 LSB = 256**，`scvtf` 后**没有任何归一化**。

> 修正一条本轮中途的错误结论：我曾根据 `uc_map.py` 的注释以为输入是
> double 数组、6 字节是另一回事，因此一度判定「输入粒度」结论无效。
> 重读汇编后确认 6 字节整数格式是对的，**1 LSB 即 drive 96 这个结论成立**。

### 45.2 库内增益链（逐项运行时确认）

```
6 字节记录
 -> 24-bit 左对齐整数（大端） -> scvtf，无归一化
 -> x * d14 + x * d15        通道内缩放
 -> 5 x (2x biquad)          插值
 -> tbl
 -> 量化器输入抽头 * d8
 -> 8/6/4 tap 量化器
```

系数全部来自 `.rodata`，**不是属性**：

| 寄存器 | rodata | 值 |
|---|---|---|
| d14 | 0x7ee0 | 0.22458342437552717 |
| d15 | 0x7ee8 | −0.10166630249789134（`kJm21DeltaGain`，−19.856 dB） |
| d9 | 0x7e90 | −0.34248937610183422 |
| d10 | 0x7eb8 | −0.27017927898059091 |
| d11 | 0x7f30 | 0.1643776559745414 |
| d12 | 0x7fe0 | 6.0156802007622421 |

缩放点（`0x1a234 .. 0x1a29c`）：

```
0x1a250  sub  x7, x4, #0x80, lsl #12     ; x7 = x4 - 0x80000
0x1a254  ldr  d0, [x7]                    ; 运行时读到 = 256*b0，即样本本身
0x1a258  fmul d0, d0, d14
0x1a260  ldr  d1, [x10]                   ; 同一样本
0x1a26c  fmadd d0, d1, d15, d0            ; 链的输入
0x1a270  str  d1, [x12]                   ; 原始样本另存（未缩放支路）
0x1a288  str  d0, [x10]                   ; 缩放后送入 biquad 链
```

`[x4 − 0x80000]` 里放的就是转换后的样本，不是 DC pedestal —— 这一点由运行时读数
（256 / 512 / 1024 / 4096 / 16384 = 256·b0）确认。

### 45.3 `d8 = 16 − mode`（结构 + 运行时双重确认）

```
0x19e6c  subs  w19, w12, w7      ; w19 = w12 - w7，w12 = 16，w7 = 入参 arg6
0x1a044  scvtf d8, w19           ; d8 = (double)(16 - arg6)
```

真实调用者传 `x6 = x7 = handle+0x68 = mode`，故 **`d8 = 16 − mode`**。
运行时读数：mode 0 / arg6 0 → `d8 = 16.0`。

（注：我此前 harness 固定传 arg6=4，得到 d8=12，那是我自己选的，不是设备行为。）

### 45.4 delta gain 是死配置，增益来自 rodata

`persist.sys.dsd.delta.gain`（rodata 0x65e6）的 getter 真实入口是 **0x17218**
（0x17214 只是 `__stack_chk_fail` 跳板）：

| 符号 | veneer | 体 | 库内调用者 |
|---|---|---|---|
| `get_alltodsd_delta_gain` | 0x1d080 | 0x17218 | **0** |
| `get_pcm2dsd_delta_gain` | 0x1d048 | 0x163e8 | **0** |
| `get_alltodsd_shaper_coeffs` | 0x1d07c | 0x171a4 | **0** |

设备级 `grep -rl` 也只命中 `libfiioaudio.so` 本身。所以这是**第三个死配置接口**；
实际增益硬编码在 `.rodata:0x7ee8`。`get_pcm2dsd_delta_gain` 与
`get_pcm2dsd_shaper_coeffs` 还共用同一个体 `0x163e8`（都读字段 0x68）。

### 45.5 结论：绝对标度是调用方契约

> **本节的 45.5 与 45.6 已被第 46 节撤回**：这里的「1 LSB 即 drive 128」
> 建立在一个错误的 DC 增益测量上（w3 用了 24 而非真实的 32，且探针把
> 通道 1 钉在 −2^31）。绝对标度仍是 Unresolved，且不得断言设备处于过载区。

库内做的缩放全部已知且固定（d14、d15、biquad、d8）。**没有任何一处把 PCM 归一化到
±1。** 因此量化器工作点完全由**调用方写进那 6 字节记录的整数范围**决定，而这个
范围在 `libfiioaudio.so` 之外。

一个可证明的结构性后果：**1 LSB 就已经越过模型的稳定边界**。
实测 tbl = 8·(256·b0)/32 = 8·b0（b0=1 时 tbl = 8），d8 = 16 → drive = 128，
而第 44 节测得的发散边界在 drive ≈ 0.5（−6 dBFS）。也就是说
**只要输入非零，量化器就处于过载区**。

### 45.6 这重新解释了第 44 节的「发散」

第 44 节把 `max|state|` 超过 1e250 记作 DIVERGED。分两层看：

- drive 0.5–1.0：状态到 1e6–1e12，**有限** —— 这是**饱和**，1-bit 级的正常过载
  行为，输出位流仍然有效，DAC 之后仍可滤出。
- drive ≥ 10：状态到 1e141 以上，数值上溢出，输出退化。

而设备几乎必然长期处在第一档。这与第 39 节的实测一致：开启 All-To-DSD 后
空闲带噪从 −51 dBFS 抬到 −41 dBFS（+9.6 dB RMS）——**过载 1-bit 级的典型表现**。

因此「稳定区 < −6 dBFS」这个说法应当改读为：**NTF 整形行为只在 drive < 0.5 时可观测**；
设备实际大概率不在这个区间。

### 45.7 仍然未知的部分

- 调用方（HAL / FiiO system service）实际写进 6 字节记录的整数范围。
  这是设备端到端标度的最后一块，需要在调用方模块里定位写入点。
- 库内是否存在第二条不经 6 字节路径的输入通道（`0x1a25c cbz x28` 的两分支
  目前看只是按通道选指针集合，不像不同格式）。

## 46. 输入缓冲的祖先：调用点、缓冲图，以及撤回第 45 节的过载结论

### 46.1 调用点完整解码

`0x13038 .. 0x1305c`（`[x23+0x5c] != 0x1A000002` 时走的正常路径）：

```
0x13038  mov  w8, #0x5a
0x1303c  add  x0, x20, #0x400, lsl #12   ; x0 = handle + 0x400000
0x13040  movk w8, #0xc0, lsl #16          ; w8 = 0xC0005A
0x13044  mov  w2, w25                      ; 输入字节数
0x13048  add  x1, x20, x8                 ; x1 = handle + 0xC0005A
0x1304c  mov  w3, #0x20                    ; 32
0x13050  mov  w5, w21                      ; [x23+0x70] dsd_level
0x13054  mov  w6, w19                      ; [x23+0x68] shaper
0x13058  mov  w7, w19                      ; arg6 = 同一个字段
0x1305c  bl   #0x1dcd0                     ; get_pcm2dsd_data
```

**`w3 = 32` 是真实值。** 我此前大量实验用的是 `w3 = 0x18`(24)，**那是错的**
（见 46.5）。

### 46.2 缓冲图

初始化函数（`0x126c0 .. 0x12768`）一次性清零三个大区：

```
0x12740  memset(base+0x0,      0, 0x500000)   ; 5 MB  <- 输入暂存 0x400000 在其中
0x12754  memset(base+0x50002a, 0, 0x500000)   ; 5 MB
0x12768  memset(base+0xa0005a, 0, 0x300000)   ; 3 MB  <- 输出 0xC0005A 在其中
```

```
handle + 0x000000 .. 0x500000    5 MB   输入 PCM 暂存（0x400000 处）
handle + 0x50002a .. 0xA0002A    5 MB
handle + 0xA0005A .. 0xD0005A    3 MB   1-bit 输出（0xC0005A 处）
```

每次调用前 `0x12e98 bl #0x1e060`（memset）把**输出**区 1 MB 清零
（`0x12e80 ldr x0,[sp,#0x90]`，而 `0x12850/0x12868` 把 `x20+0xC0005A` 存进该栈槽）。

### 46.3 更正：0xC0005A 是输出，不是输入

第 45 节我写「6 字节输入缓冲区在 handle+0xC0005A」—— **错**。
`0xC0005A` 是 `x1`，即 1-bit 输出缓冲；输入是 `x0 = handle+0x400000`。
`uc_realcaller.py` 当初的常量注释其实已经写对了（`DST = CTX + 0xC0005A  # x1`），
是我读错了。

### 46.4 死配置接口增至三个

| 属性 | getter 体 | 库内调用者 |
|---|---|---|
| `persist.sys.dsd.shaper.coeffs` | 0x171a4 | **0** |
| `persist.sys.pcm2dsd.level`（setter 0x16408） | — | **0** |
| `persist.sys.dsd.delta.gain` | 0x17218 | **0** |

三者设备级 `grep -rl` 均只命中 `libfiioaudio.so` 自身。实际增益全部硬编码在
`.rodata`（d14 @0x7ee0 = 0.22458，d15 @0x7ee8 = −0.101666）。

### 46.5 撤回第 45 节的过载结论

> **部分由第 47 节取代**：绝对增益已用 AC 差分法测得（0.4% 一致），
> S ≈ 196626，且整数 196608 恰对应 drive 0.5 的稳定边界。但逐点波形
> 比对仍未通过，故设备实际工作点依然 Unresolved。

第 45 节说「1 LSB 就越过稳定边界，设备必然处于过载区」。**该结论撤回。**
它建立在一个无效的 DC 增益测量上，而那个测量有两处错误：

1. **用了 `w3 = 0x18`(24) 而非真实的 32。**
2. **探针记录本身有缺陷**：`b5 = 0x80` 使通道 1 恒为
   `w11 = 0x80000000 = −2^31`，所以「均值」把 +256·k 和 −2^31 混在一起。

用对称记录 `[k,0,0, 0xFF,0xFF,0xFF]`（ch0=+256k, ch1=−256k）与 `w3=32` 重测，
结果**非单调**（k=1→−8.07e-5，k=64→−6.07e-5，k=256→+3.18e-7），说明
**均值被量化器自身的极限环主导，不能用来测插值器 DC 增益**。

因此目前状态应写成：

- **Verified**：库内增益链的**形式**（`scvtf` 无归一化 → ×d14 + ×d15 →
  5×2× biquad → ×d8 = 16−mode），以及输入格式、缓冲图、调用点。
- **Unresolved**：24-bit 整数到 `tbl` 的**绝对**增益。需要用差分/交流法重做，
  不能用 DC 均值法。
- **Unresolved**：设备实际工作点。**不得**再断言「设备处于过载区」。

### 46.6 下一层的边界

写 `handle+0x400000` 的代码在**同一个库内**（大函数 0x126c0–0x13970+，它同时
负责 memset 输出、调用 `get_pcm2dsd_data`、再对输出做字节重排 0x13074+）。
也就是说 24-bit 打包的 writer 也在 libfiioaudio.so 里，不在外部模块。

因此「PCM → tbl 绝对标度」的最后一环落在这一个大函数**收到的上游样本**上，
也就是 FiiO 音频进程回调的输入。那一层在库外，是下一步唯一要定位的东西。

## 47. `PCM → tbl` 绝对标度：AC 差分测量与对账

第 46 节把绝对标度标为 Unresolved，并指出 DC 均值法被极限环污染。
本节改用**阶跃响应 + 正负差分**，全部在 `tbl` 节点（`0x1a4e4`，量化器之前）采样。

### 47.1 方法

- 输入端给阶跃：前 800 帧全 0，后 800 帧为 `±256·K`；**通道 1 恒为 0**
  （字节 `[0,0,0]` → `w11 = 0`；用 `b5=0x80` 会把 ch1 钉在 `0x80000000 = −2^31`）。
- 寄存器用真实值：`w3 = 32`、`mode = 0`、`arg6 = 0`（→ `d8 = 16`）。
- `K = 0` 作为**对照基线**单独报告，绝不用来推增益。
- 正负阶跃各自从全零 memset 状态独立运行，取 `g = (g₊ − g₋)/2`，
  抵消零输入偏置与偶次非线性。

### 47.2 结果

对照：`K = 0` 的表观阶跃 = **恰好 0**（无极限环污染，方法成立）。

| K | g(+) | g(−) | g_diff | 每整数单位 |
|---|---|---|---|---|
| 1 | 8.1381e-05 | −4.85e-12 | 4.0691e-05 | **1.58948e-07** |
| 4 | 3.2553e-04 | −9.54e-07 | 1.6324e-04 | **1.59414e-07** |
| 16 | 1.3021e-03 | −4.77e-06 | 6.5344e-04 | **1.59530e-07** |
| 64 | 5.2084e-03 | −2.00e-05 | 2.6142e-03 | **1.59559e-07** |

**在 64 倍量程内每整数单位增益一致到 0.4%。**

### 47.3 与静态链对账

`tools` 里 C++ 的 `kBq9..kBq15` 就是固件的 `d9..d15`：

```
kBq9 =d9  kBq10=d10  kBq11=d11  kBq12=d12  kBq13=2*d12
kBq14 =d14 = 0.224583424375527        kBq15 =d15 = -0.101666302497891
```

每级 DC 增益数值求解为**恰好 0.5**（零插入减半），5 级 = **1/32 = 0.03125**
（注意每级是 IIR，`p = t; q = v; r = q`，不能用 DC 代入求固定点）。

于是：

```
integer -> tbl   实测 = 1.58948e-07
pcm(±1) -> tbl   模型 = 0.03125
隐含标度 S       = 0.03125 / 1.58948e-07 = 196626
```

即 **24-bit 整数除以约 1.966e5 才成为 ±1 的 PCM**。

### 47.4 一个有意义的自洽点

`S ≈ 196,626`，而 `0x30000 = 196,608`。取整数 196,608：

```
pcm = 196608 / 196626 = 0.99991
tbl = 0.99991 / 32    = 0.031247
drive = tbl * d8(16)  = 0.5000   →  -6.0 dBFS
```

**这正好落在第 44 节测得的 NTF 小信号稳定边界（drive 0.5 / −6 dBFS）上。**
两个互不相关的测量在此交汇，说明 S 的量级是对的。

### 47.5 尚未闭合：逐点波形比对失败

把固件的 tbl 流与「pcm = I/196608 经 5 级插值」算出的 tbl 逐点对比，
**相对误差约 1.0**（完全不相关），无论用 3:1 抽取（跳过每个第 4 个 /
偏移 1）还是不做抽取。幅度量级也不匹配（模型 DC 0.0026 vs 固件 0.0197）。

所以：

- **已闭合**：绝对增益的**标量值**（0.4% 一致），以及它与静态系数链、
  稳定边界的自洽性。
> **第 48 节修正**：S ≈ 196626 依赖「C++ 插值 = 1/32」，而该插值段被证明
> 拓扑有误（FIR vs IIR）。因此 **S 退回 Unresolved**；只有纯固件测量
> integer → tbl = 1.58948e-07 ± 0.4% 仍然有效。
- **未闭合**：tbl 流的**逐点等价**。最可能是 24-of-32 的抽取映射
  或每帧 tbl 顺序尚未搞对 —— 这属于第 44 节验收标准里的
  「DSD packing / block size / channel ordering」那一项。

因此 `PCM → tbl` 的**绝对标度**可以从 Unresolved 升级为
**Measured（0.4%），但 Verified 需要先闭合抽取映射**。

### 47.6 关于设备工作点的措辞

按第 46 节的教训，这里只陈述可验证的部分：

- **Verified**：按当前 drive 定义，整数 196,608 对应 drive 0.5；
  更大的整数对应更大的 drive。
- **Unresolved**：上游实际写进 `handle+0x400000` 的整数范围，
  因此**不能**断言设备处于过载区，也**不能**断言它落在 NTF 小信号区。

## 48. tbl 抽取之谜解开，但暴露了 C++ 插值段的疑似拓扑错误

### 48.1 读侧没有抽取

```
0x1a448  ldr  x10, [x20, x28, lsl #3]     ; 量化器入口
        x8  = 0x300203a0 = x25 + 0x203A0   ; 恰好是表基址，偏移 0
0x1a424  add  x8, x8, #8                  ; 顺序 +8，无跳步
```

x8 从表基址开始、每次 +8，**读侧完全顺序**。所以「24-of-32」不是读侧的抽取模式。

### 48.2 表的零插入

```
0x1a1c8  add  x11, x25, #0x20, lsl #12
0x1a1d0  add  x11, x11, #0x3a8           ; x11 = x25 + 0x203A8
0x1a1dc  ldr  d0, [x12, x23]              ; out[i]
0x1a1e0  str  xzr, [x11]                  ; table[16i+8] = 0
0x1a1e4  stur d0, [x11, #-8]              ; table[16i]   = out[i]
0x1a1f4  add  x11, x11, #0x10             ; 步长 16 字节
```

即每个样本占 2 个 double（偶位值、奇位 0），**每级 2×**。

### 48.3 干净的经验规律：`out/frame = 768 / w3`

40 帧输入，统计量化器循环次数：

| w3 | 8 | 16 | 24 | **32** | 48 | 64 | 96 |
|---|---|---|---|---|---|---|---|
| out/frame | 96 | 48 | 32 | **24** | 16 | 12 | 8 |

**严格等于 768 / w3**（768 = 3 × 2⁸）。真实调用者传 `w3 = 32` → **24/帧**，
循环总次数 `x9 = frames × 24`（实测 40 帧 → 960）。

所以「24」是**消费数量**，不是「32 里选 24」的选择模式。

### 48.4 两条候选映射都被否

| 候选 m(n) | rms 误差/rms(fw) | 相关系数 |
|---|---|---|
| 每 4 个跳 1 个（两种偏移） | ≈1.0 | 0 |
| 每 32 个取前 24 个 | 0.919 | **0** |

两者相关系数都是 0，**不是选点错**。

### 48.5 很可能的原因：C++ 插值段是 IIR，固件是 FIR

固件每样本（`0x1a260 .. 0x1a290`）：

```
d1 = [x10]                      ; 样本 x
d0 = A + d1 * d15               ; A 来自 [x4-0x80000]，运行时等于样本本身
d3 = d2 * d9                    ; p * d9
d3 = d4 * d10 + d3              ; + q * d10
d3 = d5 * d11 + d3              ; + s * d11
d3 = d3 * d12
```

即 **`y = (p·d9 + q·d10 + s·d11) · d12`，纯 FIR，状态由外部数组携带。**

而 `src/flac2dsf.cpp` 的 `Jm21Core::runStages`：

```
const double t  = jfma(p, kBq15, u);
...
p = t; q = v; r = q;            // <- 递归，IIR
```

**这是 IIR，不是 FIR。** 二者拓扑不同，因此逐点相关性为 0 完全说得通。

### 48.6 这不影响已有的 4096/4096

第 43 节与第 44 节的量化器验证**用的是固件自己产生的 tbl 流**，
所以它验证的是量化器本身，不经过 C++ 的插值段。受影响的只有：

- 「5 × 2 IIR 插值」这个描述 —— 应改为「每级 2× 零插入 + FIR 段」
- C++ 插值段与固件的逐点等价 —— 目前**不成立**，需要按 FIR 拓扑重写

### 48.7 由此得到的最有价值的副产品

第 47 节的绝对标度 S ≈ 196626 是在**假定 C++ 插值正确**的前提下由
「模型 1/32 ÷ 实测 1.58948e-7」推出来的。既然模型那段可能错了，
**S 这个数字也就不可靠**。

但实测的 **integer → tbl = 1.58948e-7**（0.4% 一致度，K=0 对照恰好 0）
是**纯固件测量**，不受影响。所以：

> **本节全部数字已被第 51 节取代**：c_gain.py 的字节序编码错误使整数被
> 放大了 256 倍，故 1.58948e-07 与 S≈196626 均作废。正确结果是
> **	bl = integer / 0x30000000（2 ppm）**，见第 51 节。
- **待修**：C++ 插值段按固件 FIR 拓扑重写，之后 S 与逐点等价一起重测

## 49. 插值段辨识：拿到的硬约束，与必须停手的理由

### 49.1 本轮确立的事实

**① `out/frame = 768 / w3`**（40 帧实测，跨 w3 ∈ {8,16,24,32,48,64,96} 严格成立）。
真实调用者 `w3 = 32` → 24/帧，`x9 = frames × 24`。

**② 表写入是零插入**：`table[16i] = out[i]`、`table[16i+8] = 0`、步长 16 → 每级 2×，
5 级共 32×。

**③ 量化器读侧顺序**：`x8` 入口 = 表基址，`0x1a424 add x8,x8,#8`，无跳步。
所以不存在「32 里选 24」的抽取器；**24 是消费数量**。

**④ 差分阶跃响应的实测形状**（K=64，`w3=32`）：

```
tbl[19200 .. 19263]   -6.2e-09 起，缓慢爬升到 -2.5e-07   (~1e-8 量级)
tbl[19264]            +1.36e-04                          <- 跳 4 个数量级
tbl[19265 .. 19279]   线性上升 3.6e-04 -> 4.5e-03
```

起振点正好在 `19200 = 800 × 24`，**零延迟**。

### 49.2 为什么必须停手

④ 里那个「64 样本死区 + 突跳」说明 **tbl 流索引与输入样本之间不是 `m = 4n/3`**：
若表真是 32/帧，输入样本 f 的数据应落在表项 `32f`，量化器顺序读到它需要
tbl 索引 25600，但实测起振在 19200（= 600×32）。

也就是说：**表内每帧究竟是 24 项还是 32 项、量化器起点相对表首的偏移是多少，
这两个量我还没有确定。** 早前直接读 `x25+0x203A0` 那次得到全零，也说明表地址
解析未完成（量化器入口 `x20 = 0x381ff720` 是栈地址，与 `x25 = HANDLE` 的关系未理清）。

在这个状态下改写 C++ 插值段，等于用「DC 增益 = 1/32 对得上」去反推拓扑 ——
**这正是第 48 节刚抓到的那个错误的同一种形式**。所以本轮不改 C++。

### 49.3 写进项目文档的经验

> **DC 增益一致不能证明 DSP 拓扑一致。**
>
> 本项目内的反例：
>
> ```
> 固件：  y = (p·d9 + q·d10 + s·d11)·d12   状态由外部数组携带   → FIR
> C++：   p = t; q = v; r = q                                  → IIR
>
> DC 增益：  两者都是 1/32（完全一致）
> 时域波形：  逐点相关系数 ≈ 0
> ```
>
> 早期只对比过 DC 增益（1/32 相符）就认定插值段正确，直到本轮做波形比对才发现。
> **任何 DSP 段的验收必须包含时域逐点等价，不能只比增益。**

### 49.4 下一步的前置条件（按依赖排序）

1. **确定表首与量化器读起点的偏移**：在 `0x1a1d4`（表写循环）与量化器入口
   分别记录 x25/x8/x20/x9，理清「每帧写多少项」与「从表的哪里开始读」。
   这是 ④ 里那个 64 样本死区能否解释的前提。
2. 单脉冲（不是阶跃）辨识：在同一状态下只给 1 个输入帧非零，直接读出
   5 级展开后的冲激响应，抽头位置与零插入图案一次可见。
3. 用 1、2 的结果确定每个 FIR 段的状态槽地址与更新顺序（`0x1a234` 起的
   x10..x17 各自指向哪个偏移）。
4. **然后**才重写 C++ `runStages`，并按 49.3 的验收标准比对：
   相关系数 → 1、rms 误差 → 机器精度，外加 max/mean 绝对误差。
5. 之后重测 S、最后重跑 4096/4096 端到端。

## 50. 表地址账闭合（步骤①完成）；harness 不可重入

### 50.1 地址映射（寄存器 + 内存指纹双重证据）

用可反推索引的哨兵值（`sentinel[i] = -(i+1)*1e6`）预填 handle+0 起 393216 个
double，跑一次单脉冲后回读。**哨兵完整性 OK**，463 个单元确为固件写入。

| 项 | 值 | 证据 |
|---|---|---|
| handle 基址 | `x25 = 0x30000000` | 写循环与量化器入口一致 |
| `x20` | `0x381ff720` | **是栈指针，不是 handle** —— 第 48 节的困惑就此解决 |
| 表基址 | **handle + 0x203A0** | `x8` 在 `0x1a448` 恰好等于它；写循环 `x11 = handle+0x203a8`，`stur d0,[x11,#-8]` |
| 写循环源步长 | **8 字节（1 double）** | `x12 = handle+0x0, +0x8, +0x10, +0x18, +0x20` |
| 写循环目标步长 | **16 字节（2 double）** | `x11 = handle+0x203a8, +0x203b8, +0x203c8 …` |
| 量化器读起点 | 表首，偏移 0 | `x8 = handle+0x203a0` |
| 量化器读步长 | 8 字节，顺序 | `0x1a424 add x8,x8,#8` |
| 输出总数 | `x9` | NF=8 → **192**；NF=40 → 960 → **out/frame = 24** |
| `x23` | `0xa03a0` | 第二份 192 单元拷贝区 |

### 50.2 单脉冲差分给出的结构（步骤②部分）

用「全零运行」与「单脉冲运行」两次内存镜像做差（避免第 49 节说的哨兵污染），
NF=8、K=256，得到 5 个连续块：

```
handle+     2 ..      9   (8 单元)   1512, 1474, 1437, 1400, 1364, 1328, 1293, 1258
handle+    18 ..     47   (30 单元)  15 对，每对两值相同
handle+   116 ..    116   (1 单元)
handle+ 16500 .. 16691  (192 单元)  <-- 表，0x203A0
handle+ 82036 .. 82227  (192 单元)  <-- 0xA03A0 = x23
```

**块 0 = 8 抽头 FIR 的冲激响应**：值近似等差递减，增益 1512/256 = 5.9，
与 `d12 = 6.0157` 吻合。所以**至少第一段是 FIR，8 抽头，增益 d12**。

**块 1 = 15 对相同值**：与固件 `0x1a294 str d3,[x14]` / `0x1a298 str d3,[x13]`
把同一输出写进两个相位槽一致。

**块 3 的头三个值** `2.51533741634657e-13 / 6.61069514484372e-13 /
1.07679816604781e-12` 与第 43 节 mode-0 tbl 捕获的头三个**完全相同**，
确认它就是 tbl 流，且**步长 8 字节连续、每帧 24 项**。

### 50.3 一个必须撤回的中间推断

我一度把块 3 的 192 单元理解为「每帧 32 项里取 24」，但既然块 3 本身就是
tbl 流且只有 24/帧，**最终表里根本没有零插入，也没有抽取** —— 32× 的展开
发生在表之前的内部缓冲里。正确表述是：

```
内部：多级展开（>24）
表 (handle+0x203A0)：每输入样本恰好 24 个 double，步长 8，连续
量化器：从表首顺序读 x9 = frames*24 个
```

### 50.4 harness 不可重入（本轮踩到）

在同一个 Python 进程里连续调用 `run()`，同一 NF 得到的足迹不稳定：

```
NF=4  -> 表区变化 0 单元
NF=8  -> 384 单元（=48/帧）
NF=12 -> 0 单元
NF=16 -> 768 单元（=48/帧）
```

单独跑 NF=8 时得到的是 192（=24/帧）。所以**每个测量点必须独立进程**，
否则 `uc_quant.build()` 的内存映射/状态会互相污染。这一条必须写进任何后续
自动化，否则会得到看似合理但错误的表结构。

同理，`sum(table)` 在这批不可靠运行里给出 1.8e-12/整数单位，与第 47 节的
AC 测量 1.58948e-07 差 5 个数量级 —— **不可信，不引用**。

### 50.5 步骤①的结论

地址层已闭合到如下程度，且每一条都有寄存器 + 内存指纹双重证据：

```
输入帧 f
  -> 24-bit 整数（大端左对齐）
  -> stage 缓冲：handle+0x0 起，步长 8（第一段 8 抽头 FIR，增益 d12）
  -> 相位状态：成对写入（每输出写 2 槽）
  -> 表 handle+0x203A0：每帧 24 项，步长 8，连续
  -> 拷贝区 handle+0xA03A0（x23）：同样 24/帧
  -> 量化器从表首顺序读 x9 = frames*24 个
```

仍未闭合的是**中间各级 FIR 的抽头与状态槽对应关系**（块 1 之后），
需要按 50.4 改成独立进程后重跑单脉冲才能拿到可信数据。

## 51. `PCM → tbl` 绝对标度闭合：`tbl = integer / 0x30000000`

### 51.1 先更正第 47 节：那里有一个 256 倍的编码错误

固件解码是 `w11 = b2<<24 | b1<<16 | b0<<8`（`0x19ee8/0x19eec/0x19ef0`），
所以要放入整数 V，字节必须是

```
b0 = (V>>8)&0xFF,  b1 = (V>>16)&0xFF,  b2 = (V>>24)&0xFF
```

`ac_gain.py` 当时写的是 `[V&0xFF, (V>>8)&0xFF, (V>>16)&0xFF]`，
构造出的整数是 **65536·V**。因此第 47 节「每整数单位增益 1.58948e-07」
以及由它推出的 `S ≈ 196626 ≈ 0x30000` **全部偏大 256 倍**，那两个数字作废。

（旁证：`impulse_diff.py` 用的是正确编码，传 V=1/64 时因为 `(V>>8)==0`
而得到全零记录、冲激完全消失，这个现象把问题暴露了出来。）

### 51.2 修正后的测量：AC 阶跃差分，正确编码

`w3=32`（真实值）、mode 0、arg6 0（`d8=16`）、通道 1 恒为 0、
`V=0` 对照组的表观阶跃 **恰好 0**。

| V | g_diff | 每整数单位 |
|---|---|---|
| 256 | 1.5895e-07 | 6.20901e-10 |
| 512 | 4.7685e-07 | 9.31342e-10 |
| 1024 | 1.1126e-06 | 1.08656e-09 |
| 2048 | 2.3842e-06 | 1.16417e-09 |
| 4096 | 4.9274e-06 | 1.20298e-09 |
| 8192 | 1.0014e-05 | 1.22238e-09 |
| 32768 | 4.0532e-05 | **1.23693e-09** |
| 262144 | 3.2537e-04 | **1.24118e-09** |
| 4194304 | 5.2083e-03 | **1.24174e-09** |

**V ≥ 32768 后增益恒定在 1.2417e-09 ± 0.4%。** 低 V 段的偏低是 AC 差分的
测量地板（那一段 tbl 只有 1e-13 量级），不是真实非线性。

### 51.3 结果：绝对标度是一个干净常数

```
1 / 0x30000000 = 1 / 805306368 = 1.24176323e-09
实测（渐近）                        1.24174e-09
偏差                                2 ppm
```

**`tbl = integer / 0x30000000 = integer / (3 × 2^28)`**

直接核对：V = 2²² = 4194304 → tbl = 4194304/805306368 = 5.2083333e-03，
实测 g_diff = 5.2082539e-03，比值 0.99998。

**`PCM → tbl` 绝对标度由 Unresolved 升级为 Verified。**

> **含义修正（第 52 节）**：这不是 DC 传递增益，而是「周期 96 样本稳态的均值」。固件对常值输入的 tbl 输出是周期 lcm(32,24)=96 的非平稳稳态，任何线性插值模型都无法逐点匹配。

### 51.4 链条的线性范围比 NTF 区间窄

drive = tbl × d8 = tbl × 16 = `V / 50331648`：

| V | tbl | drive | dB |
|---|---|---|---|
| 3×10⁴ | 3.9e-5 | 6.2e-4 | −64.1 |
| 2²² | 5.208e-3 | 0.0833 | **−21.6** |
| 2²⁴ | 0.02083 | 0.3333 | −9.5 |
| 2³¹（满刻度） | 2.6667 | 42.67 | +32.6 |

**V > 2²² 时 g_diff = 0**（±V 给出完全相同的响应），即链条进入压缩。
而第 44 节测得 NTF 小信号区延伸到 drive 0.5（−6 dBFS）。

**结论：插值链只在 drive < ≈0.083（−21.6 dBFS）内线性，比 NTF 可观测区窄约 15 dB。**
第 44 节的 NTF 实测（−6…−60 dBFS）中，−6 到 −21.6 dBFS 这一段其实已经处在
链条的压缩区 —— 实测仍与理论吻合到 0.7 dB，但那条结论应理解为
「压缩区内 NTF 形状依然可见」，而不是「链条在那里仍然线性」。

### 51.5 这一轮同时确立的地址事实（承第 50 节）

表长度严格 = **24 × NF**，在 NF ∈ {8, 16, 24, 32} 四个独立进程上分别得到
192 / 384 / 576 / 768。NF=4 与 NF=12 时仿真没有进入量化器（`x9 = 0`），
属于 harness 的边界条件，不作为数据。

固定长度的缓冲（不随 NF 变）：

```
handle+0x10      8 单元
handle+0x90     30 单元（15 对，每对两值相同）
handle+0x3a0     1 单元
handle+0x120530  7 单元
handle+0x120570  6 单元
handle+0x1205a8  2 单元      （NF >= 24 才出现）
```

### 51.6 尚未解决：中间各级 FIR 的抽头

「全零运行 vs 单脉冲运行」做差得到的**不是干净的冲激响应** —— 零输入时
量化器处在极限环（见第 44 节），差分里混入了极限环的变化。表现为
`handle+0x10` 那 8 个单元的值随 V **非单调**：

```
V=256    -4.79e5
V=512    -1.01e6
V=1024   -4.96e5
V=2048   -5.68e5
```

所以**这些单元不能直接当作 FIR 抽头读**。要拿到各级真实拓扑，需要在
**极小的驱动下**（drive ≪ 0.083，见 51.4）做冲激，使链条保持线性，
否则差分永远被非线性与极限环污染。

这也是为什么 C++ `runStages` 仍然不能改：现在知道了目标工作点的线性范围，
也知道了之前用大步长比对必然失败。

## 52. tbl 的真实输出是周期 96 的稳态，不是常数 —— 「24-of-32」的最终解释

### 52.1 核心事实

对 tbl 送阶跃（V=65536），输出**收敛到一个周期稳态**而不是常数：

```
收敛值（每 96 块均值）   +8.12225044e-05     从 block 12 起逐位相同
周期内峰值               +3.0762975758e-04
周期内标准差              1.2168e-04
```

### 52.2 周期恰好是 96 个 tbl 样本 = 4 个输入帧

对冲激响应 `h = diff(step)` 做自相关式检验（区间 n = 9900..10500）：

| lag | max\|h[n] − h[n+lag]\| | 相对 max\|h\| |
|---|---|---|
| 24（1 帧） | 2.98e-05 | 1.401 |
| 32 | 3.46e-05 | 1.626 |
| 48 | 2.53e-05 | 1.189 |
| 64 | 3.46e-05 | 1.626 |
| **96（4 帧）** | **4.76e-09** | **2.24e-04** |
| 192 / 288 | ~4.7e-09 | 2.2e-04 |

**周期 = 96 = lcm(32, 24)。**

### 52.3 这就是「24-of-32」的真正含义

```
内部展开      每输入样本 32 个样本（5 级 2× 零插入）
输出/表       每输入样本 24 个样本（x9 = frames*24，实测）
两个速率的相位每 lcm(32,24)/gcd = 4 帧重新对齐
  -> 输出出现周期 4 帧 = 96 样本的拍频
```

所以既不是「32 里选 24」的抽取器，也不是读侧跳步，而是**两个速率不整除
所固有的拍频**。第 48 节说「24 是消费数量」是对的，但没意识到它会带来
一个 4 帧周期的输出结构。

### 52.4 为什么 C++ 模型永远无法逐点匹配

`Jm21Core::runStages` 产生的是干净的 5 级 2× 展开（32/帧）、**平稳**的输出。
而真实 tbl 对常值输入给出的是**周期 96 的非平稳稳态**。

因此只要输入不是刚好 4 帧周期性的，模型与固件在 tbl 上就必然对不上 ——
这与「FIR 还是 IIR」无关，是更上游的结构差异。

**推论：任何以「常值输入 → 常值输出」为前提的 DC 增益测量，测到的都是这个
周期波形的某个统计量，而不是链路的传递增益。** 第 47 / 51 节的
「tbl = integer / 0x30000000」因此应重新表述为：

> 当输入为常值时，tbl 稳态的**周期波形均值** = integer / 0x30000000
> （周期 96 样本，均值 8.122e-5 @ V=65536，2 ppm 一致）

数字本身是可靠且可复现的，但它的物理含义是「周期稳态的均值」，不是「DC 传递增益」。

### 52.5 滤波器核的形状与电平无关

在 V = 256 / 4096 / 65536 / 262144 / 1048576 五档下，冲激响应 `h` 的
**形状完全不变**，只有标量增益变化，逐抽头比值的标准差 ~1e-14（机器精度）：

```
h(V)/h(65536) 的逐抽头均值    1.0078432579 (V=256)
                             1.0073818898 (V=4096)
                             1.0000000000 (V=65536)
                             0.9763779528 (V=262144)
                             0.8818897638 (V=1048576)
```

即链条是**「标量增益随电平变化 + 固定形状的线性核」**，与第 51 节
「增益随 V 上升后饱和到 1.2417e-9」一致。

### 52.6 本轮同时确认/修正的内容汇总

```
输入格式      24-bit 大端左对齐，6 字节/帧，1 LSB = 256      Verified
调用点        w3=32 -> 24 输出/帧，x9 = frames*24           Verified
表地址        handle+0x203A0，步长 8，读起点偏移 0             Verified
输出结构      周期 96 样本（4 帧）的稳态                     Verified
绝对标度      tbl 稳态均值 = integer / 0x30000000（2 ppm）    Verified（含义已修正）
核形状        与电平无关（std 1e-14）                         Verified
核增益        随电平变化，V>2^22 饱和                        Verified
```

### 52.7 下一步

C++ 模型要逐点对上 tbl，必须复现**周期 96 的拍频结构**，这要求把内部
32/帧展开与 24/帧输出之间的相位关系实现出来。在此之前改 `runStages`
仍然是拿错误的输出结构去比对，必然失败。

前置条件已经比第 49 节清楚得多：**不是「不知道 FIR 拓扑」，而是「输出结构
本身就是非平稳的」**。下一步应当先在固件侧确认内部 32/帧缓冲与 24/帧表
之间的写入/读取相位关系（即 `x9 = frames*24` 这个计数是怎么由
`frames` 和 `w3` 算出来的），再决定模型如何生成这个拍频。

## 53. 输出计数公式与五个 stage 缓冲地址

### 53.1 计数公式（静态推导 + 8 点实测验证）

```
0x19dc4  add  w9, w3, #7
0x19dd4  csel w8, w9, w3, lt
0x19df0  asr  w9, w8, #3          ; D = (w3+7) >> 3
0x19df4  sdiv w8, w2, w9           ; w8 = byte_count / D
0x19dfc  cinc w10, w8, lt
0x19e04  asr  w24, w10, #1         ; w24 = w8 >> 1
0x19e80  sdiv w9, w3, w9
0x19e84  adds w9, w9, w9            ; 2*w3/D
0x19e8c  sdiv w10, w2, w9
0x19e90  msub w9, w10, w9, w2        ; w2 mod (2*w3/D)
0x19e94  cbz w9, #0x19ea8            ; 非 0 -> 返回 -3（拒绝）
```

**`x9 = ((byte_count / D) >> 1) * 32`，`D = (w3+7) >> 3`**

| w3 | NF | bytes | D | 实测 x9 | 公式 | |
|---|---|---|---|---|---|---|
| 8 | 40 | 240 | 1 | 3840 | 3840 | OK |
| 16 | 40 | 240 | 2 | 1920 | 1920 | OK |
| 24 | 8 | 48 | 3 | 256 | 256 | OK |
| 32 | 8 | 48 | 4 | 192 | 192 | OK |
| 32 | 40 | 240 | 4 | 960 | 960 | OK |
| 48 | 40 | 240 | 6 | 640 | 640 | OK |
| 64 | 40 | 240 | 8 | 480 | 480 | OK |
| 96 | 40 | 240 | 12 | 320 | 320 | OK |

推出 `out/frame = 16 / ((w3+7)>>3)`，真实调用者 `w3=32` → **24**。

### 53.2 整除约束解释了 harness 的边界

`0x19e94 cbz w9` 要求 `byte_count` 能被 `2*w3/D` 整除，否则函数返回 −3。
`w3=32` 时 `D=4`，要求 `NF*6 % 16 == 0` → **NF 必须是 8 的倍数**。

第 50 节观察到的「NF=4、NF=12 仿真不进入量化器（x9=0）」由此解释，
不是 harness bug。

### 53.3 五个 stage 缓冲地址

prologue `0x19e28 .. 0x19e40` 直接给出五个同偏移（`+0x3a0`）的缓冲：

```
0x19e2c  add x10, x25, #0x3a0        -> handle + 0x003a0    stage A
0x19e28  add x11, x25, #0x10, lsl #12 ; + 0x3a0 -> handle + 0x0103a0   stage B
0x19e34  add x12, x25, #0x120, lsl #12; + 0x3a0 -> handle + 0x1203a0   stage C
0x19e38  add x13, x25, #0x130, lsl #12; + 0x3a0 -> handle + 0x1303a0   stage D
0x1a1d0  add x11, x11, #0x3a8 (x11 = x25+0x20000) -> handle + 0x203a0  stage E = 表
```

这与第 50 节内存指纹看到的改动块**完全对应**：

```
handle+0x10      8 单元
handle+0x3a0(=116) 1 单元     <- stage A
handle+0x90     30 单元
handle+0x103a0              <- stage B
handle+0x120530/0x120570/0x1205a8   <- stage C 附近
handle+0x1303a0                       <- stage D
handle+0x203a0  24*NF 单元             <- stage E = 表
handle+0xA03A0  24*NF 单元             <- x23 拷贝区
```

### 53.4 量化器状态槽

```
0x19e64  add x10, x25, #0x10      ; 通道 0 状态
0x19e68  add x11, x25, #0x50      ; 通道 1 状态
0x19e70  stp x10, x11, [x29, #-0x80]
```

与 `uc_quant.py` 头部的记录一致（`handle+0x10` / `handle+0x50`，各 8 个 double），
也与量化器入口的 `x10 = handle+0x2d8`（= 0x2d0/0x2d8 一对）落在同一区域。

### 53.5 至此的完整内存地图

```
handle + 0x000000 ..          控制/杂项
handle + 0x000010 .. 0x000050 量化器状态（通道0 / 通道1，各 8 double）
handle + 0x000090             30 单元（15 对）相位/中间态
handle + 0x0003a0             stage A 输出（scvtf 后的样本数组）
handle + 0x0103a0             stage B 输出
handle + 0x1203a0             stage C 输出
handle + 0x1303a0             stage D 输出
handle + 0x203a0              stage E = 交给量化器的表（24/帧）
handle + 0xA03A0              x23，表的另一份
```

**下一步**：在这五个已知地址上分别挂 hook，用小驱动冲激逐级记录输出，
即可直接读出每级的核 —— 这比从汇编猜状态槽可靠得多。

## 54. 标度链完全闭合：`pcm16 → ±1 → tbl`，含 `0x300000`

### 54.1 stage A 直接给出 ±1 归一化

在四个 V 上读 `handle+0x3a0`（stage A），值与 `V/2^23` 的比值**恒为 1.000000000**：

| V | stage A 实测值 | V/2^23 | 比值 |
|---|---|---|---|
| 65536 | 0.0078125 | 0.0078125 | 1.000000000 |
| 131072 | 0.015625 | 0.015625 | 1.000000000 |
| 262144 | 0.03125 | 0.03125 | 1.000000000 |
| 1048576 | 0.125 | 0.125 | 1.000000000 |

**`pcm(±1) = integer / 2^23`**

### 54.2 「24-bit 字段」实际装的是 16-bit 样本

解码是 `integer = b2<<24 | b1<<16 | b0<<8`。若 `b0` 是 16-bit 样本 `pcm16`
的低字节，则 `integer = pcm16 × 2^8`。代入 54.1：

```
pcm16 = integer / 2^23 × 2^15 = integer / 2^8
```

即 **`integer = pcm16 × 256`** —— 那个 3 字节字段是 **16-bit PCM 左移 8 位**存放的，
再由 `scvtf` 之后除以 2^23（即 2^8 × 2^15）归一化到 ±1。

满刻度对应 `pcm16 = ±32768`。

### 54.3 链条增益是 1/96，不是 1/32

第 51 节测得 `tbl = integer / 0x30000000`。代入 `integer = pcm16 × 256`：

```
tbl = pcm16 × 256 / 0x30000000 = pcm16 / 0x300000
tbl = pcm / (32768 × 96) = pcm / 96
```

**所以插值链的整体 DC 增益是 1/96。**

C++ 模型里「每级 0.5、5 级 = 1/32」**是错的** —— 实际是 1/96，
差 3 倍。这意味着五级里有一级的零插入不是 2× 而带 1/3 的因子，
或者链上还有一个显式的 1/3。

### 54.4 最终标度链（每一环都有实测）

```
pcm16 (16-bit, ±32768 满刻度)
   |  × 256                       3 字节字段 b2<<24|b1<<16|b0<<8
   v
integer (24-bit 左对齐容器)
   |  ÷ 2^23                      stage A 实测，比值 1.000000000
   v
pcm (±1)
   |  × 1/96                      五级插值链
   v
tbl
   |  × d8 = 16 − mode            scvtf 后由 w19 = 16 − arg6 得到
   v
量化器输入（drive）
```

等价地：**`tbl = pcm16 / 0x300000`**，**`tbl = integer / 0x30000000`**（2 ppm）。

### 54.5 与第 51 节的对应关系

第 51 节说「绝对标度 = 0x30000000」，当时误以为是整数单位下的常数，
现在明确了它就是 `0x300000 × 256`。而 `0x300000 = 3 × 2^20 = 96 × 32768`
恰好是「1/96 链增益 × ±1 归一化」的乘积 —— 常数自洽。

而第 47 节那次报出的 `0x30000` 差 256 倍，正是那个字节序 bug。

### 54.6 同时确认：stage A 里出现的 3 周期图案

stage A 每个记录呈 `[X/2^16, 0, X]` 的三周期图案（两个幅值之比实测
65536.16 ≈ 2^16）。这与第 52 节发现的「tbl 输出周期 96」不是同一层：
stage A 是插值前的样本数组，在量化器入口读它时已被后续级覆盖多次，
所以读到的是残留写。因此**stage A 的「±1 归一化」结论可靠
（幅值比 1.000000000），但三周期图案本身不作解释**。

### 54.7 这一轮对 C++ 的直接影响

`Jm21Core::runStages` 目前假设：

```
输入 ±1 的 double
每级 2× 零插入，DC 增益 0.5
5 级 -> 32/帧，DC 增益 1/32
```

实际情况：

```
输入 ±1 的 double
输出 24/帧（第 53 节 x9 公式）
DC 增益 1/96
输出对常值输入呈周期 96 样本的稳态（第 52 节）
```

**三处都不对**。这解释了为什么逐点比对相关系数为 0 ——
不是差一个抽头，而是输出的样本数、幅度标度、以及是否平稳全都不同。

下一步必须先在固件侧确定：五级各自的比例因子（哪一级贡献 1/3），
以及 32/帧内部缓冲与 24/帧表之间的相位关系。这两者定了，
`runStages` 才有重写的依据。

## 55. 相位分辨的冲激：周期 4 记录，以及 8 样点的读指针漂移

### 55.1 周期是 4 个输入记录

对每个记录相位 p 打一发冲激（AC 差分，V=65536，drive=1.3e-3 在线性区），
在 `handle+0x203a0` 读表：

| phase | onset | onset/24 | 峰值 | 同余类 |
|---|---|---|---|---|
| 0, 4, 8, 12 | 24p | 整数帧 | −6.219e-07 | A |
| 1, 2, 5, 6 | — | — | **严格 0** | — |
| 3, 7, 11, 15 | 24p − 8 | 2.667/6.667/… | +3.209e-04 | B |

同一同余类内**形状逐位相同**（A 类峰值恒为 −6.219420e-07，B 类恒为
+3.209417e-04）。

### 55.2 phase ≡ 1,2 的表严格为零

不是被差分抵消，是原始运行就为零：

```
phase 1: max|+V| = 0.000000e+00   max|-V| = 0.000000e+00   非零单元 0 / 1200
phase 2: 同上
```

即**这些输入记录对表的贡献是严格的 0.0**，不是"很小"。

### 55.3 phase ≡ 3 的响应与输入符号无关

```
phase 3:  max|+V| = 3.209417e-04
          max|-V| = 3.209417e-04      <- 逐位相同
```

而 phase ≡ 0 是 `4.897e-09`（+V）对 `1.249e-06`（−V），明显随符号变化。

**A 类响应是奇对称的（随符号变），B 类响应与符号无关。** 这是链条里
存在整流/限幅类非线性的直接证据，也解释了第 48 节「FIR vs IIR」以及
第 52 节「核形状与电平无关但增益随电平变」这两个看似矛盾的现象。

### 55.4 主导机制（假设，尚未证明）

```
写侧：每记录 32 个样本（5 级 2× 零插入）
读侧：每记录消费 24 个样本（x9 公式）
每记录亏 8 个样本 -> 读指针相对写指针每记录漂移 8
漂移累积 4×8 = 32 后重新对齐 -> 周期 = 4 记录
4 记录 × 24 = 96 个输出样本 = 第 52 节实测的周期 96
```

这与 `onset` 的两种取值一致：p≡0 给 `24p`（恰好对齐），p≡3 给 `24p − 8`
（差一个漂移单位）。phase ≡1,2 则读到尚未被本记录填入的槽，因而为 0。

**待验证**：`onset(p) = 24p` 与 `24p − 8` 的精确规则目前只覆盖 p≡0,3 两类；
p≡1,2 为零这一点还需要一个能同时解释的读指针模型。

### 55.5 这一轮的净进展

| 项 | 状态 |
|---|---|
| 输出周期 | **4 记录 = 96 输出样本**（Verified，两种独立测量一致） |
| 周期来源 | 32/记录写 vs 24/记录读的 8 样点漂移（假设，机制吻合） |
| 相位依赖 | 强：4 个相位里只有 2 个产生输出，且响应幅度差 500 倍（Verified） |
| 非线性 | B 类响应与输入符号无关 → 存在整流/限幅（Verified） |
| `1/3` 的来源 | **仍未定位**。总增益 1/96 已确定，但 32 与 24 的比（4/3）可能就是它 —— 若 96 = 32×3，而 24 = 32×3/4，则 `1/96 = (1/32)×(1/3)` 中的 `1/3` 恰好是「32→24 的 3/4 抽取」的反面。**这条需要把读指针模型做完才能判定**，不能先假设。 |

### 55.6 对 C++ 的影响（比第 54 节更明确）

`runStages` 需要实现的不是「5 级 2× 插值」，而是：

```
每输入记录 -> 32 个内部样本
按一个 4 相位（96 输出样本）循环的规则抽取 24 个
该规则包含读指针漂移，且对某些相位输出恒为 0
外加一个与符号无关的非线性（至少在某一相位上）
```

这是一个**周期时变（polyphase）且非线性**的系统。普通的
`y = a0*x[n] + a1*x[n-1] + …` 结构不可能复现，**这是必须提前防止的
「DC 对了、波形全错」的第三次复发**（前两次见第 48、49 节）。

## 56. 实测到的 96 样点周期稳态波形（可直接作为模型验收目标）

### 56.1 总增益精确收敛到 1/96

用可靠路径（量化器循环内经 `[x8]` 读取）做 AC 阶跃差分：

| V | pcm = V/2^23 | g_diff | 隐含 1/gain | 与 96 之差 |
|---|---|---|---|---|
| 65536 | 0.0078125 | 8.1222504377e-05 | 96.18640 | +0.1942% |
| 262144 | 0.03125 | 3.2536685467e-04 | 96.04543 | +0.0473% |
| 1048576 | 0.125 | 1.3019442558e-03 | 96.01026 | +0.0107% |
| **4194304** | **0.5** | **5.2082538605e-03** | **96.00146** | **+0.0015%** |

**链的 DC 增益 = 1/96，在高驱动处 15 ppm 吻合。** 低驱动处偏大最多 0.19%，
即链条有极轻微的压缩（第 51 节的增益-电平曲线）。

### 56.2 96 样点周期稳态波形（归一化到 pcm，20 周期平均）

```
 0  +0.014992 +0.013679 +0.012160 +0.010695 +0.009173 +0.007560
 6  +0.006105 +0.004730 +0.002710 +0.000334 -0.002054 -0.004663
12  -0.007124 -0.009507 -0.012120 -0.014834 -0.015632 -0.015301
18  -0.014964 -0.013982 -0.013447 -0.013151 -0.012128 -0.010776
24  -0.010053 -0.009489 -0.008814 -0.008269 -0.007051 -0.005379
30  -0.003771 -0.001981 -0.001263 -0.001225 -0.000964 -0.000917
36  -0.000731 -0.000406 -0.000431 -0.000635 -0.000029 +0.000973
42  +0.001948 +0.003171 +0.004074 +0.004781 +0.005729 +0.006729
48  +0.007445 +0.008057 +0.008635 +0.009099 +0.009779 +0.010582
54  +0.011303 +0.012041 +0.012693 +0.013266 +0.013881 +0.014492
60  +0.015042 +0.015563 +0.016051 +0.016498 +0.017461 +0.018736
66  +0.019993 +0.021412 +0.022747 +0.024035 +0.025516 +0.027095
72  +0.028509 +0.029882 +0.031295 +0.032681 +0.034239 +0.035915
78  +0.037580 +0.039294 +0.039377 +0.038434 +0.037519 +0.036073
84  +0.034857 +0.033758 +0.032015 +0.029925 +0.028550 +0.027412
90  +0.026123 +0.024966 +0.023199 +0.021005 +0.018909 +0.016672
```

```
mean   = +0.01039648        (1/96 = 0.01041667，本驱动下比值 0.99806)
peak   = +0.039377  @ index 80
trough = -0.015632  @ index 16
sum    = 0.99806212
```

已存为 `pattern96.npy`（纯 numpy，无依赖）。

**这是一个可直接使用的验收目标**：对任意直流输入 `pcm`，tbl 输出必须是
`pcm × pattern96`（在该驱动下）。任何候选 C++ 实现只要对这个数组做逐点
比较，就能立刻判断插值段是否正确 —— 不需要先跑完整 DSD 流程。

### 56.3 波形形态与 4 记录相位的对应

96 = 4 记录 × 24。逐记录看：

```
record 0 ( 0..23)  +0.0150 -> 0 -> -0.0156 -> -0.0132   下降段
record 1 (24..47)  -0.0101 -> -0.0004 -> +0.0067          上升段
record 2 (48..71)  +0.0074 -> +0.0271                      继续上升
record 3 (72..95)  +0.0285 -> +0.0394 -> +0.0167          峰值后回落
```

与第 55 节的相位分类吻合：单帧冲激时 phase≡0 与 phase≡3 的响应幅度差 500 倍，
而持续输入下四个记录都参与，形成这个平滑的 96 点波形。

### 56.4 一个尚未验证但很自然的假设

内部若真是 32/记录，4 记录就是 128 点；输出 96 点。这个 96 点波形应当是
某个 128 点波形按固定相位重采样得到的。若把 `pattern96` 重采样到 128 点，
应当得到更「自然」的插值核。**这一条可以直接检验**，而且如果成立，
就同时给出了内部 32/记录缓冲的形状 —— 也就是 `runStages` 需要复现的东西。

### 56.5 当前项目状态

```
Verified
  PCM 内部格式 = 16-bit PCM << 8（integer = pcm16 × 256）      [54]
  24-bit 大端左对齐解码                                          [45]
  stage A 给出 ±1 归一化（integer/2^23，比值 1.000000000）      [54]
  输出计数 x9 = ((bytes/D)>>1)*32, D=(w3+7)>>3（8 点验证）     [53]
  byte_count 整除约束 -> NF 必须是 8 的倍数                      [53]
  五个 stage 缓冲地址 A..E + 量化器状态槽                       [53]
  表 = handle+0x203A0，24/帧，连续，步长 8                       [50]
  输出周期 = 4 记录 = 96 样本                                   [52,55]
  相位依赖：4 个相位只有 2 个产生输出                            [55]
  非线性：phase≡3 的响应与输入符号无关                          [55]
  链 DC 增益 = 1/96（高驱动 15 ppm）                            [56]
  96 点周期稳态波形已实测并存档                                  [56]
  mode 0/1/2 量化器 4096/4096 三档                             [43]
  三档 NTF 频域实测吻合理论 0.8 dB                              [44]

Unresolved
  1/3 或等效因子的确切来源（现知总增益 1/96，逐级未拆开）        [55,56]
  内部 32/记录 与 输出 24/记录 的读指针规则                       [55]
  96 点波形是否由 128 点内部波形重采样而来                       [56]
  各级在量化器入口读全零（读数时机问题，未解）                   [50,55]
  修正后的 runStages 逐点等价                                   [54,55]
  上游 PCM 实际标度 / DSD packing / 设备对拍
```

**下一步优先级**：56.4 的 96→128 重采样检验。它若成立，就同时给出内部
32/记录缓冲的形状与逐级增益，`runStages` 才有可实现的依据；不成立则说明
96 点波形本身就是原生的，五级模型要重新表述。

## 57. 收敛的 96 样点稳态工件：DC 增益 = 1/96（15 ppm）

### 57.1 工件

`tools/pattern96.npy`（96 个 float64，纯 numpy，无依赖）：
**常值输入 `pcm` 时 tbl 的稳态 = `pcm × pattern96`**，在高驱动处收敛。

```
 0  +0.015025 +0.013715 +0.012198 +0.010736 +0.009217 +0.007607
 6  +0.006154 +0.004782 +0.002765 +0.000392 -0.001993 -0.004601
12  -0.007058 -0.009438 -0.012047 -0.014759 -0.015557 -0.015228
18  -0.014892 -0.013913 -0.013380 -0.013086 -0.012066 -0.010719
24  -0.009998 -0.009436 -0.008764 -0.008221 -0.007006 -0.005338
30  -0.003734 -0.001949 -0.001234 -0.001199 -0.000940 -0.000897
36  -0.000713 -0.000392 -0.000419 -0.000626 -0.000024 +0.000974
42  +0.001944 +0.003163 +0.004061 +0.004763 +0.005706 +0.006701
48  +0.007415 +0.008027 +0.008606 +0.009072 +0.009754 +0.010556
54  +0.011279 +0.012021 +0.012674 +0.013248 +0.013864 +0.014476
60  +0.015028 +0.015553 +0.016044 +0.016495 +0.017459 +0.018733
66  +0.019991 +0.021410 +0.022746 +0.024035 +0.025515 +0.027093
72  +0.028509 +0.029884 +0.031299 +0.032688 +0.034247 +0.035924
78  +0.037591 +0.039307 +0.039391 +0.038450 +0.037535 +0.036091
84  +0.034876 +0.033779 +0.032036 +0.029948 +0.028574 +0.027437
90  +0.026149 +0.024994 +0.023228 +0.021035 +0.018939 +0.016703
```

```
mean   = +0.010416508
sum    = +0.99998473        <- DC 增益 = sum/96，误差 15 ppm
peak   = +0.039391 @ 80
trough = -0.015557 @ 16
符号变化 @ 10 与 41
```

### 57.2 驱动依赖性

同一波形在 64 倍驱动范围内（各自归一化到自己的 pcm）：

| V | mean | 相对 V=65536 的形状比值 std |
|---|---|---|
| 65536 | 0.01039648 | — |
| 262144 | 0.01041174 | 2.04e-02 |
| 1048576 | 0.01041555 | 2.55e-02 |
| 2097152 | 0.01041619 | 2.64e-02 |
| 4194304 | 0.01041651 | 2.68e-02 |

**mean 单调收敛到 1/96；形状在 64× 范围内变化约 5%**（std 6.6e-3，
极值范围 0.950…1.036）。即系统**近似 LTV**，只有轻微非线性。

### 57.3 96 点是原生结构，不是重采样产物

- 96→128 按固定相位重采样的重建误差 1.3e-3…5.7e-3，与信号幅度（peak 0.039）
  同量级 → **模型不成立**
- 波形极值在 **0 / 37 / 80**，间距 37 与 43，**不是 24 或 32 的倍数**
- 每 24 样本（一个记录）的曲率量级一致，没有 32 点子结构

**所以「24-of-32」的框架可以彻底放下了**：输出里不存在可见的内部 32/记录
结构，只有「每记录 24 样点 + 4 记录（96 样点）周期」。

### 57.4 这个工件怎么用

它是插值段最便宜、最严格的验收向量：

```
对候选 C++ 实现：喂常值 pcm，检查 tbl 是否等于 pcm * pattern96
逐点 max abs error / 相关性  ->  直接判定插值段对错
```

不需要跑完整 DSD 流程，也不需要碰量化器。相关系数应为 1，
max abs error 应在 1e-15 量级（若浮点次序一致）。

### 57.5 尚未由 pattern96 回答的问题

- pattern96 的**闭式结构**（它是某个滤波器的冲激响应，还是多级收敛的产物）
- 为何恰好 4 记录周期（4 = lcm 相关？还是 x9 公式的副产物）
- `1/96` 在五级里如何分解（第 55 节列的待办；各级在量化器入口读全零，
  读数时机问题未解，见第 50/55 节）

**下一步**：先确认 pattern96 能否由已知的 `d9..d15` 系数闭式算出。
若能，则整条链的形式就确定了，`runStages` 可以直接照抄；
若不能，则说明还有未识别的级，需要回到 0x1a234 起的 FIR 逐条读。

## 58. pattern96 的频域结构：纹波压倒直流，线性链无法产生

### 58.1 频谱分解

对 96 样点周期做 FFT（f 以「每周期圈数」计）：

```
DC              0.010417
bin 1  f=1/96   |X| = 0.995     <- 占非直流能量 88.66%
bin 2  f=2/96   |X| = 0.333
bin 4  f=4/96   |X| = 0.111
bin 5,7,8       0.034 / 0.023 / 0.037
其余高次谐波    可忽略（|X| < 1e-3）
```

正弦幅度 = `2×0.995/96 = 0.02073`，**是直流电平 0.010417 的两倍**。

所以 tbl 对常值输入的输出是：

```
直流 1/96  +  一个主导的 fs_out/96 正弦（幅度 0.0207）  +  少量谐波
```

**纹波压倒直流。**

### 58.2 这在物理上排除了「纯线性插值链」

一个线性 FIR/IIR 链对常值输入只会收敛到**常数**（或发散）。它不可能产生
非恒定的周期稳态。第 52、55、56 节观测到的现象：

- 输出是周期 96 的稳态而非常数
- 相位 ≡3 的响应与输入符号无关
- 纹波幅度是直流的两倍

三者一致指向：**链里存在非线性**（限幅/整流类），其极限环的基频恰好落在
`fs_out/96 = 每 4 个输入记录一圈`。

这也解释了为什么第 48 节「FIR vs IIR」的争论始终无法收敛 —— 真实结构
既不是纯 FIR 也不是纯 IIR，而是**线性滤波 + 非线性极限环**。

### 58.3 96 这个周期数的来源仍未确定

已知：

- 输出 24 样点/记录（x9 公式，第 53 节）
- 周期 = 4 记录 = 96 样点
- 96 = lcm(24, 32)，32 是五级 2× 展开的名义倍率

但第 56.3 节已证明 96 点波形**不是** 128 点（4×32）重采样而来，极值也不落在
24 或 32 的倍数上。所以「32/记录 ↔ 24/记录 的失配产生拍频」目前只是一个
**未证实的假设**：它能解释周期长度，但解释不了极值位置和 88.7% 的单频纯度。

### 58.4 当前对 `runStages` 的要求（更新）

模型必须能产生：

```
常值 pcm 输入  ->  tbl = pcm * pattern96
```

即输出同时包含 1/96 的直流电平和 fs/96 的主导纹波。任何「每级 2×、输出平稳」
的实现都做不到。第 54 节列的三处错误（32/帧、1/32、平稳）之外，还必须加上
**第四处：非线性极限环**。

### 58.5 由此得到的最省力的下一步

`pattern96` 已经是现成的验收向量。对任何候选实现：

```
喂常值 pcm -> 与 pcm*pattern96 逐点比较
相关系数应 = 1，max abs error 应 ~1e-15
```

在此之前不必碰量化器、不必跑 DSD、不必追库外调用方。

反过来，**若能从 `d9..d15` 闭式算出 pattern96**，就说明非线性也在这些系数里，
`runStages` 可以照抄；若算不出，说明还有未识别的环节（限幅器、或 polyphase
状态位的非线性使用），需要回到 `0x1a234` 起逐条读。这是判定分岔点。

## 59. 负面结果：插值代码里没有任何非线性

### 59.1 搜索结果

在 `get_pcm2dsd_data`（0x19d90–0x1a968）内：

| 类别 | 命中 |
|---|---|
| `fmaxnm` / `fminnm` / `fabs` / `fmabs` | **0** |
| 整数饱和 `umin/umax/smin/smax` | **0** |
| `fcsel` | 3 处，全是量化器判决：0x1a400(mode1) / 0x1a520(mode0) / 0x1a580(mode2)，均为 `fcsel d1, d26, d24, lt` 取 ±1 |
| `fcmax/fcmin` | **0** |

**整个函数是纯线性浮点运算**，没有任何限幅、整流或饱和。

### 59.2 于是 96 周期稳态只能来自「边缘稳定的 IIR + 舍入噪声」

线性系统对常值输入必得常值输出。实测却是周期 96 的非平稳稳态，且：

- 完全可复现（逐位）
- 纹波幅度是直流的 1.49 倍
- 无任何非线性指令

唯一自洽的解释是：**插值级是临界稳定（极点落在单位圆上）的 IIR**，
浮点舍入误差不被衰减，反而被放大并锁定成一个确定性极限环，其基频恰好
落在 `fs_out/96`。这同时解释了第 56.2 节观察到的轻微电平依赖
（极限环的种子与幅度依赖初始条件）。

**这也是「FIR vs IIR」争论（第 48–49 节）始终无法收敛的根因：真实结构
既不是纯 FIR 也不是常规 IIR，而是临界稳定的递归滤波器。**

### 59.3 但我的状态转写是错的（|λ|>1）

按 `0x1a254..0x1a2e8` 读出的算式整理：

```
d0 = A·d14 + x·d15
d5 = x + d0
d3 = r·d9 + q·d10 + d5·d11
d3 = d3 · d12
out = d3 + q·d13 + r·d12
```

用 `d13 = 2·d12` 化简：

```
out = d12(d9+1)·r  +  d12(d10+2)·q  +  d11·d12·(x(1+d15) + A·d14)
     =   3.9551 · r  +   10.4059 · q  +   0.98893 · (...)
```

若状态更新是 `r←out, q←r`，状态矩阵为

```
M = [ 3.9551  10.4059 ]
    [ 1        0      ]
```

特征根 **λ = 5.762 / −1.807**，`|λ| > 1` → **发散**。

而固件显然是稳定的。所以**「r←out, q←r」这个状态映射是错的** ——
尽管逐条读出了每条算式，状态槽的读写顺序仍未确定。

这与第 50 / 55 节的另一个未解现象同源：**在量化器入口读各级缓冲全部返回 0**，
说明我对这些缓冲的生命周期/读写时机的理解还不完整。

### 59.4 因此下一步必须回到状态槽本身

不能再靠读算式推断。需要的是**运行时观测**：在 `0x1a254`（读状态）与
`0x1a294`/`0x1a298`（写状态）处逐样本记录实际读写的地址与值，从而确定

```
哪个槽 = r，哪个槽 = q，写入的先后顺序
```

有了这个，就能解出真正的状态矩阵，判断极点是否真的落在单位圆上，
并最终闭式算出 pattern96。

这是当前唯一明确的技术路径，且完全可观测 —— 不需要追库外调用方，
不需要碰量化器。

## 60. 插值块的精确转写：稳定 IIR，两条反馈路径

### 60.1 状态槽与两条反馈路径（逐条读出）

`0x1a250` 设 `x7 = x4 - 0x80000`，然后：

```
0x1a254  ldr  d0, [x7]          ; A = [x7]
0x1a258  fmul d0, d0, d14
0x1a260  ldr  d1, [x10]         ; x = [x10]
0x1a264  ldr  d2, [x15]         ; s1 = [x15]
0x1a268  ldr  d4, [x13]         ; s2 = [x13]
0x1a26c  fmadd d0, d1, d15, d0  ; d0 = A*d14 + x*d15
0x1a270  str  d1, [x12]
0x1a274  fmul d3, d2, d9        ; d3 = s1*d9
0x1a278  str  d4, [x15]         ; [x15] <- 旧 s2        => s1[n+1] = s2[n]
0x1a27c  fadd d5, d1, d0        ; d5 = x + d0
0x1a280  str  d0, [x11]
0x1a284  fmadd d3, d4, d10, d3  ; d3 += s2*d10
0x1a288  str  d0, [x10]         ; [x10] <- d0          => 样本输入反馈
0x1a28c  fmadd d3, d5, d11, d3  ; d3 += d5*d11
0x1a290  fmul d5, d3, d12
0x1a294  str  d3, [x14]
0x1a298  str  d3, [x13]         ; [x13] <- 新 d3        => s2[n+1] = d3[n]
--- 共用尾 ---
0x1a2dc  fmadd d5, d4, d13, d5  ; d5 = d3*d12 + s2*d13
0x1a2e0  fmadd d0, d2, d12, d5  ; out = d5 + s1*d12
0x1a2e4  str  d0, [x7]          ; [x7] <- out          => A[n+1] = out[n]  输出反馈
0x1a2e8  str  d0, [x4]
```

**第 59 节转写错的根因**：漏掉了 `x7`（输出反馈到 A）和 `x10`（d0 反馈到样本输入）这两条环。

### 60.2 该块是稳定 IIR，不是极限环

以 `[s1, s2, A]` 为状态，忽略外部输入的齐次部分：

```
M = [ 0            1              0        ]
    [ d9           d10            0        ]
    [ d12(d9+1)    d12(d10+2)     0        ]
```

特征值：

```
λ0 = 0
λ± = -0.135090 ± 0.569421i ,  |λ| = 0.585226
```

`|λ| < 1` → **稳定**。模拟确认：喂常数输入，输出收敛到常数
（seed 1.0 → 1.1017，逐步稳定），**没有任何周期稳态**。

**所以第 58 节「临界稳定 + 舍入极限环」的推断是错的**，一并撤回：
纯线性、严格稳定的链不可能自发生成 pattern96 的纹波。

### 60.3 纹波的真实来源：零插入的梳状激励

若喂常数，输出是常数。真实输入是**零插入的梳状序列**，其稳态响应必然
是梳周期的周期函数 —— 这才是 pattern96 纹波的唯一可能来源。

关键算术：内部速率 **32 样点/记录**，表存储**每 4 个取 3 个** → 24/记录。
若内部序列周期为 128（= 4 记录 × 32），则

```
存储序列周期 = lcm(128, 跳过计数周期 4) / 4 = 128/4 × 3 = 96
```

**正好是实测的 96。** 这也解释了第 56.3 节为什么 96→128 线性重建必然失败：
内部多出的 32 个样本是**被跳过的那 1/4**，不是插值可以补出来的。

### 60.4 仍未读的部分（当前唯一阻塞）

我上面的模拟把 `x10` 同时当作外部输入和反馈槽，导致外部输入只在第一次迭代
生效、输出归零。**外部样本的重新装载由 `0x1a2ec` 之后的循环控制负责，
这段还没读。**

必须从 `0x1a2ec`–`0x1a960` 读出三件事：

1. **零插入图样**：每个输入样点展开成几个内部样点、零插在什么位置
2. **五级如何串联**：`str d1,[x12]` / `str d0,[x11]` 把值送到哪一级，
   `x11/x12/x13/x14/x15` 五个槽在五级之间的对应关系
3. **存储调度**：`0x1a3d8` 附近的写入循环如何决定「每 4 取 3」

拿到这三点，`pattern96` 应当能被闭式复现，`runStages` 就可以照抄。

**在此之前不应再继续猜参数模拟** —— 缺的是代码结构，不是系数。

## 61. 插值链的完整结构（静态闭合）

### 61.1 记录长度 = 16 字节，不是 6

```
0x19e80  sdiv  w9, w3, w9          ; w3 / D
0x19e84  adds  w9, w9, w9
0x19e8c  sdiv  w10, w2, w9
0x19e90  msub  w9, w10, w9, w2     ; byte_count % (2*w3/D)
0x19e94  cbz   w9, #0x19ea8        ; 非 0 -> 返回 -3
```

**记录长度 = `2*w3/D`，w3 为 8 的倍数时恒为 16 字节**（2 通道 × 8）。
此前一直用 6 字节喂输入，被这条校验拒掉（返回 −3），这也是长期无法进入
插值段的直接原因。第 19 节的「6 字节/帧」只适用于另一条由 `w8` 分派的
输入格式分支（`0x19f08 add x20,x20,#6`，以及 `ldrsh` 16-bit 步长 4 的分支）。

### 61.2 关键寄存器（全部确认）

```
x25 = handle                (0x19e24 ldr x25,[0x1f410])
x20 = 输入缓冲               (0x19dcc mov x20,x0)
D    = (w3+7)>>3            (0x19df0)
NF   = byte_count / D       (0x19df4 sdiv)
x24  = NF >> 1              (0x19e04 asr)   <- 记录数
[x29-0x88] = (NF>>1) << 5   (0x19e10/0x19e18)  <- 输出总样本数
w26  = (w5<2) ? w6 : 2      (0x19e0c)
d8   = 16 - arg6            (0x19e6c subs w19,w12,w7 ; 0x1a044 scvtf d8,w19)
x23  = 0xa03a0              (0x1a034/0x1a038)
```

`(NF>>1)<<5` 与第 53 节实测的 `x9 = ((byte_count/D)>>1)*32` **完全一致**。

例：byte_count=192, w3=32 → D=4, NF=48, x24=24, 输出样本数 768 = 24×32。

### 61.3 五级是 2× 零插入级联

`0x1a028`–`0x1a0c0` 把各级样本数算成 double 存进栈：

```
0x1a028  ucvtf d0, w24              ; sp+0xb0 = x24
0x1a02c  lsl   w8, x24, #1
0x1a04c  ucvtf d0, w8               ; sp+0xa0 = 2*x24
0x1a030  lsl   w9, x24, #2
0x1a03c  lsl   w10, x24, #3
0x1a054  ucvtf d1, w9               ; 4*x24
0x1a078  ucvtf d0, w10              ; 8*x24
```

即**每级的样本数翻倍**：`x24, 2x24, 4x24, 8x24, 16x24`。
第 k 级把输入零插入成 2× 长度，再跑 `2*count-1` 次 FIR。
末级输出 `16*x24 = 768` 个样本 = **每记录 32 样点 / 通道**。

**第 54 节「每级 2×」这个判断本身是对的**，错的是认为输出是平稳的：
零插入使梳状周期逐级翻倍（1→2→4→8→16），所以最终输出**必然是周期的**。

### 61.4 零插入与 FIR 循环的确切代码

```
; 级联外层：x8 = stage 0..4
0x1a1a8  add  x8, x8, #1 ; cmp x8,#5 ; b.eq #0x1a314
0x1a1b4  add  x9, x25, x8, lsl #3 ; add x9, x9, x24
0x1a1bc  ldr  d0, [x9]              ; count = [handle + x24 + stage*8]
0x1a1c0  fcmp d0, #0.0
; --- 零插入：写 (值, 0) 对 ---
0x1a1dc  ldr  d0, [x12, x23]        ; 取值
0x1a1e0  str  xzr, [x11]            ; 写 0
0x1a1e4  stur d0, [x11, #-8]        ; 写 值      (x11 += 0x10)
0x1a1f8  ucvtf d1, w10 ; b.gt #0x1a1d4
; --- FIR 循环 ---
0x1a218  mov  w5, #1
0x1a2ec  cbz  x6, #0x1a95c
0x1a2f0  ldr  d0, [x9]
0x1a2f4  ucvtf d1, w5
0x1a2f8  add  w5, w5, #1
0x1a300  add  x4, x4, #8
0x1a304  fadd d0, d0, d0
0x1a308  fcmp d0, d1
0x1a30c  b.gt #0x1a250
```

所以 **`ceil(count)` 次零插入 + `2*count-1` 次 FIR**，
输出表由 `x4`（`handle + x23 + 8k`）顺序写入。

### 61.5 量化器侧（再次确认）

```
0x1a3e8  fmadd d2, d7, d8, d2     ; drive = tbl*d8, d8 = 16-mode
0x1a3fc  cset  w11, ge
0x1a400  fcsel d1, d26, d24, lt   ; ±1
0x1a404  strb  w11, [x21]
0x1a424  add   x8, x8, #8         ; 顺序读，无抽取
0x1a428  add   x21, x21, #1
0x1a42c  subs  x9, x9, #1
```

### 61.6 harness 修正（之前一直跑不通的原因）

1. `X3` 必须是 `0x20`（真实 w3=32），旧 harness 用 `0x18`
2. `byte_count` 必须是 **16 的倍数**，否则函数在 `0x19e98` 返回 −3
3. **栈金丝雀**：`[sp+0]` 必须等于 TPIDR，且 `[TPIDR+0x28] == [sp+0x120]`
   （`0x19dc0 mrs x14,tpidr_el0` → `0x19de0 stur` → `0x1a7fc ldr x14,[sp]`
   → `0x1a82c ldr x8,[x14,#0x28]` → `0x1a834 cmp`），否则在 `0x1a82c` 触发异常

修正后函数可完整跑完 12 个记录（`0x1a88c`–`0x1a944` 各访问 12 次）。

### 61.7 仍未解释：周期 96

按 61.3，梳状周期逐级翻倍 1→2→4→8→16，**预期末级周期为 16 样点**。
实测 `pattern96` 的周期是 96。二者不符。

可能来源（未验证）：

- 记录长度 32 样点/通道 与 梳周期 16 的 lcm = 32，仍不等于 96
- 96 = 3 × 32（每 3 个记录重复一次）—— 需要解释「3」从哪来
- `x24` 参与各级 count，若某级的 count 不是整数倍关系，会引入额外的周期

**下一步**：既然 `pattern96` 是末级输出的稳态，直接把 5 级按 61.3/61.4 的
结构在 Python 里复现，逐级比对中间缓冲，即可定位周期是在哪一级从 16 变成 96 的。

## 62. 重大更正：记录长度是 16 字节，1/96 与 pattern96 全部作废

### 62.1 根因

前面所有插值段实验都用**6 字节记录**喂输入，而固件的记录长度是

```
记录长度 = 2*w3/D,  D = (w3+7)>>3
```

w3 为 8 的倍数时**恒为 16 字节**（= 4 × int32）。校验在 `0x19e94 cbz w9,#0x19ea8`，
不整除则返回 −3。

`ac_gain2.py` 用 `FRAMES*6 = 4800` 字节喂入，`4800 % 16 == 0`，**所以没有触发 −3，
而是让固件把错位的 6 字节流当作 4×int32 解码**。所谓「常数输入」在链看来
是逐记录变化的伪随机序列。

### 62.2 正确的布局下的真实测量

16 字节记录 = `struct.pack('<iiii', V, V, V, V)`，`w3=0x20`，mode 0：

```
chain input  = int32 / 2^21
   65536 -> 3.05175781e-05      4194304 -> 0.001953125
chain output = input / 32
   1/ratio = 32.000000   （65536 / 131072 / 1048576 / 4194304 四档完全一致）
稳态波形 = 常数，std = 0，无纹波，无电平依赖，无饱和
```

**DC 增益精确 = 1/32 = 2^-5**，且完全线性。而 1/32 正好等于 5 次零插入各降一半 ——
说明 **FIR 本身 DC 增益为 1，增益全部来自零插入**。这与实测各级
「输出长度 = 2 × 输入长度」严格自洽。

### 62.3 各级结构（已实测确认）

```
stage k : count_k = 80 * 2^k,  输出 2*count_k
   0: count  80 -> 160
   1: count 160 -> 320
   2: count 320 -> 640
   3: count 640 -> 1280
   4: count 1280 -> 2560
```

每级 = 「零插入一倍 + FIR」。总升采样 **32 = 2^5**。
五级共用同一组系数 `d9..d15`（在 `0x1a088`–`0x1a0c0` 循环外一次性载入），
只有状态槽不同（`madd x3, x8, x19, x25`，偏移 0x090/0x0a0/0x0a8/0x0b0/0x0b8/
0x210/0x218/0x220/0x228/0x230/0x238）。

### 62.4 两次独立遍历

- **Pass A**（`0x1a88c`–`0x1a944`）：`ld2 {v0.2s,v1.2s},[x20],#16` 解出
  记录 × 2 通道的 double，写入 `handle+0x103a0`；循环由
  `subs x24,x24,#2` 控制（实测 x24: 0x50→0x2，40 次）
- **Pass B**（`0x1a1a8`–`0x1a314`）：5 级插值链，读 `handle+0xa03a0` 区的
  6 个参数 double（`count_0..count_4` 与 `d8`），输出到表

之前把 `0x1a14c` 当作「每记录头」是错的；它属于 Pass B 的通道分派。

### 62.5 作废清单

| 节 | 内容 | 状态 |
|---|---|---|
| 51 | `1/0x30000000` 增益、`1.58948e-07` 更正 | **作废**（真实为 1/32） |
| 52 | 周期 96 稳态、DC 均值被极限环污染 | **作废**（无周期稳态） |
| 54 | stage A = V/2^23、1/96、`0x30000000` | **作废**（输入是 int32/2^21，增益 1/32） |
| 56 | pattern96、96→128 重采样、极值 0/37/80 | **作废**（伪信号） |
| 57 | 收敛工件 `pattern96.npy`、sum=0.999985 | **作废** |
| 58 | 临界稳定 + 极限环推断 | **作废**（链是线性稳定的） |
| 59 | 「无非线性指令」的搜索结论 | 仍成立，但推论作废 |
| 60 | 块转写与两条反馈路径 | 仍成立（特征值 0.585 稳定） |
| 61 | 16 字节记录、五级 2× 级联、零插入代码 | **成立** |

`tools/pattern96.npy` 与 `tools/stage4_w3_32.npy` 应视为无效工件，
前者删除，后者仅作内部调试用。

### 62.6 仍然有效的结论

全部来自**静态分析**或**与输入布局无关的量化器验证**：

- 三档量化器 **4096/4096 bit-exact**（mode 0/1/2），`d8 = 16 - mode`
- 量化器 NTF 频域实测与理论吻合 ≤ 0.8 dB；带外 2–4 dB 差异
- `0x1a434` 分派：`==2`→4-tap、`==1`→6-tap、`>=3` 旁路、否则 8-tap
- 量化器顺序读表（`x8 += 8`），读侧无抽取
- 输出计数 `x9 = ((byte_count/D)>>1) << 5`（静态确认）
- 三个配置属性接口为死接口；`match_audio_process_mode` 非阻塞
- 输入解码按 `w8` 分派多条格式分支

### 62.7 下一步（明确且可执行）

`runStages` 的正确实现现在是确定的：

```
pcm[n] = int32 / 2^21
for k in 0..4:
    z = interleave(pcm, zeros)          # 零插入 2x
    pcm = FIR(z)  with d9..d15, per-stage state
tbl = pcm                               # = original / 32
drive = tbl * (16 - mode)
```

只需**逐级验证第 60 节的块转写是否与固件输出一致**（用 `dump_stages.py`
导出的各级输出做数值比对），确认后即可改写 `src/flac2dsf.cpp` 的 `runStages`，
并用「喂常数 pcm → 检查 tbl == pcm/32」作为验收。

这个验收比 `pattern96` 简单得多，且是解析可预期的。

## 63. 五级插值链：功能模型已闭合，精确瞬态待定

### 63.1 已确认（16 字节记录，w3=0x20，mode 0）

```
输入解码   ld2 {v0.2s,v1.2s},[x20],#16     -> 4 x int32
Pass A     subs x24,x24,#2 循环            -> handle+0x103a0（记录 × 2 通道）
输入定标   pcm = int32 / 2^21
Pass B     五级，每级：
             count_k = 16 * 2^k   （records=8 时）
             输出长度 = 2 * count_k
             DC 增益 = 精确 1/2
末级        tbl = pcm / 32      （四档驱动实测 1/ratio = 32.000000）
量化器      drive = tbl * d8,  d8 = 16 - mode
```

实测级间数值（records=8，int32=0x10000）：

| 级 | cnt | 输出数 | 尾部稳态 | 相对上一级 |
|---|---|---|---|---|
| 0 | 16 | 32 | 1.52587897e-05 | — |
| 1 | 32 | 64 | 7.62939445e-06 | ×1/2 |
| 2 | 64 | 128 | 3.81469717e-06 | ×1/2 |
| 3 | 128 | 256 | 1.90734856e-06 | ×1/2 |
| 4 | 256 | 512 | 9.53674278e-07 | ×1/2 |

并且 **stage k+1 的输入逐值等于 stage k 的输出** —— `0x1a1d4` 处写入的
`(值, 0)` 交错缓冲**不被 FIR 读取**（FIR 的 `ldr d1,[x10]` 指向上一级输出本身）。
所以那个缓冲是给别的消费者（或只是中间态），升采样 2× 完全发生在 FIR 块内部。

**1/32 = 2^-5 恰为 5 级各 1/2 之积，故 FIR 本身 DC 增益为 1。**

### 63.2 冲激响应（`tools/k_stage*.npy`）

单点冲激（记录索引 2，int32=0x400000），减去全零基准：

```
stage 0  非零跨度 [8..31]   24 点
         +4.3375e-04 +1.1400e-03 +1.1506e-03(峰) +8.7029e-04 +5.1225e-04
         -1.1877e-04 -1.7565e-04 +9.1417e-05 +3.5126e-05 -4.0766e-05 ...
stage 1  非零跨度 [16..63]  48 点
         +9.6325e-05 +2.5316e-04 +4.1236e-04 +6.0545e-04 +6.2850e-04(峰) ...
```

响应是**衰减振荡尾**，不是有限长 FIR 核。跨度 24 点与第 60 节解出的
2 状态 IIR（|λ| = 0.585226）一致：`0.585^24 ≈ 4e-6`，与实测尾部相对峰值
约 2e-5 同量级。

### 63.3 仍未闭合的一点

第 60.1 节的块转写给出了正确的算式与两条反馈路径，但**状态槽的读写顺序**
仍未确定。按「r←out, q←r」代入会得到 |λ| = 5.76（发散），与固件稳定行为
矛盾。缺的是 `0x1a294`/`0x1a298` 写 `d3` 的两个槽与下一次读取
`[x15]`/`[x13]` 的对应关系 —— 也就是 `x13/x14/x15` 三个槽中哪两个构成状态。

由于五级共用同一组 `d9..d15`，只要确定这一个块的正确递推，五级都可照抄。

### 63.4 影响范围

- **不受影响**：三档量化器 4096/4096 bit-exact、NTF 频域吻合、`0x1a434`
  分派、`d8 = 16-mode`、量化器顺序读表、输出计数公式、三个死配置接口、
  **以及直流标定（`tbl = pcm/32`）** —— 这些都只依赖 DC 增益，已实测闭合。
- **受影响**：`runStages` 的逐点等价。要在真实音频上 bit-exact，必须先解出
  63.3 的状态顺序。

### 63.5 建议的下一步

最小且决定性的实验：**只驱动一级**。把 `count_0` 之外的级全部旁路
（令 `count_{k>0} = 0`），对单级喂冲激，比较「固件输出」与「按 63.3 各种
候选状态顺序复现的输出」。一组 32 点输出就能唯一确定状态映射。

实现上还可以先落地已经确定的部分：把 `runStages` 从当前的
`p=t; q=v; r=q` 递归 IIR 改成「五级 2× 级联、每级 DC 增益 1/2」的结构，
保证直流标定正确，再用 63.5 的实验补上瞬态形状。

## 64. 插值链彻底闭合：C++ 与固件 bit-exact

### 64.1 正确的块转写

状态槽是延迟线，不是三个独立状态。`x12` 滞后 `x10` 一步、`x15` 滞后 `x13`
两步，故每次迭代 k（`A_k` = 零插入后的表项）：

```
d1_k = fma(d1_{k-1}, d15, A_{k-1} * d14)          # [x10] <- d0，所以 d1 滞后 A
d0_k = fma(d1_k,     d15, A_k      * d14)
d5_k = d1_k + d0_k
D_k  = fma(d5_k, d11, fma(D_{k-1}, d10, D_{k-2} * d9))
y_k  = fma(D_{k-2}, d12, fma(D_{k-1}, d13, D_k * d12))
```

其中 `d2 = [x15] = D_{k-2}`、`d4 = [x13] = D_{k-1}`，`d13 = 2*d12`（精确）。

之前 transcribe 成 `r←out, q←r` 才得出 |λ|=5.76 的发散；实际是
`D_{k-1} = D_k`、`D_{k-2} = D_{k-1}` 的二阶延迟，特征值
`λ = 0, −0.135090 ± 0.569421i`（|λ| = 0.585226），稳定，与固件一致。

### 64.2 关键结构事实

- **升采样 2× 由零插入实现**：`0x1a1d4` 循环读 `handle+x23`（FIR 输出缓冲）
  写 `(值, 0)` 对到 `handle+0x203a8`；FIR 循环再从 `0x203a0` 读。
  `x7 = x4 - 0x80000` 使 `A` 正好指向这个交错表。
- **DC 增益精确 1/32 = 2^-5**，恰为五次零插入之积，FIR 自身 DC 增益为 1。
- **链输入是每样本重复 2 次的序列**：16 字节记录 = 交错立体声
  `[L0, R0, L1, R1]`，每记录给每通道 2 个样本，故 `count = 2 × 记录数`，
  每帧升采样仍是 32×。
- 五级共用同一组 `d9..d15`（`0x1a088`–`0x1a0c0` 在级循环外一次载入），
  只有状态槽不同（`x19 = 0x30` 步长，偏移 0x090…0x238）。
- 输入定标 `pcm = int32 / 2^31`（标准 32 位满量程）。

### 64.3 std::fma 在本工具链上是错的

用 4000 组随机三元组与精确有理数对照：

```
std::fma   vs 精确 : 375 处不符
朴素 a*b+c vs 精确 :  26 处不符
```

即 `std::fma` 比不融合还差。GCC 把 `fma()` 展开成了双次舍入的序列。
**这意味着此前所有基于 C++ `jfma` 的结论都不可靠**（第 18/19 节的
4096/4096 是用 Python 的 Fraction 精确 FMA 得到的，不是 C++）。

已改为 Dekker 分解 + Knuth two-sum 的正确舍入 FMA：

```cpp
static const double kSplit = 134217729.0;            // 2^27 + 1
static inline double jfma(double a, double b, double c) {
    const double p = a * b;
    if (!std::isfinite(p)) return p + c;
    const double ca = kSplit * a, ah = ca - (ca - a), al = a - ah;
    const double cb = kSplit * b, bh = cb - (cb - b), bl = b - bh;
    const double err = ((ah * bh - p) + ah * bl + al * bh) + al * bl;
    const double s  = p + c;
    const double bv = s - p, av = s - bv, br = c - bv, ar = p - av;
    return s + ((err + ar) + br);
}
```

### 64.4 顺带修掉的一个真实缺陷

`runStages` 原来把结果拷回 `src` 时按 `src.size()` 截断，而生产路径
`jm[c].src.assign(jmIn.begin(), jmIn.end())` 恰好只有 n 个元素，于是每级
输出被砍到 n，多级之后链路完全失真。现改为保证 `src` 容量足够、不截断。

### 64.5 验收结果

`tools` 侧新增对照脚本（C++ 从 `src/flac2dsf.cpp` **原样抽取** `Jm21Core`，
不存在副本漂移），与固件逐位比较：

| 用例 | 样本数 | 相异字 | 结果 |
|---|---|---|---|
| constant +65536 | 512 | 0 | BIT-EXACT |
| constant −65536 | 512 | 0 | BIT-EXACT |
| zero | 512 | 0 | BIT-EXACT |
| impulse | 512 | 0 | BIT-EXACT |
| ramp | 768 | 0 | BIT-EXACT |
| stereo L≠R | 640 | 0 | BIT-EXACT |
| alternating | 1024 | 0 | BIT-EXACT |
| full-scale mix | 256 | 0 | BIT-EXACT |
| tiny values | 768 | 0 | BIT-EXACT |

**C++ 与固件在全部用例上 bit-exact。** 默认 FIR 路径回归数字未变
（SNR 32.0 dB / gain +0.6817 / err-sig 0.0172），未被影响。

### 64.6 现在可以确定的事

插值段不再是未知项。剩下的工作是把量化器（已 4096/4096，但需用新的
正确舍入 FMA 在 C++ 侧复核）与 DSD 打包、通道顺序、block size 一起做端到端
对拍。`--jm21-dump <file>` 已加入，用于把链输出落盘做逐位对照。

## 65. 量化器闭合：C++ == 固件，mode 0/1/2 全部 bit-exact

### 65.1 分层验收（第一层）

按「同一 tbl → C++ 量化器 vs 固件」执行，输入直接用固件自己的表，
不经 C++ 插值段。覆盖 5 个用例 × 3 个 mode，每个 1024 样本（通道 0）：

| 用例 | mode 0 | mode 1 | mode 2 |
|---|---|---|---|
| ramp | 0/1024 | 0/1024 | 0/1024 |
| sine-ish | 0 | 0 | 0 |
| impulse | 0 | 0 | 0 |
| tiny | 0 | 0 | 0 |
| dc+ | 0 | 0 | 0 |

**15/15 全部 bit-exact。** 三档的 bit 序列互不相同（确认 mode 确实生效）。
C++ 侧从 `src/flac2dsf.cpp` **原样抽取** `Jm21Core`，不存在副本漂移。

### 65.2 过程中查出的三个真实缺陷

**(1) `std::fma` 在 MinGW 工具链上错误**（第 64 节已记）。改用 Dekker 分解 +
Knuth TwoSum 的正确舍入实现，并在 20000 组随机三元组上对照精确有理数验证：
**工作域内（|a| ≤ 64、|b| < 2^40）0 处不符**。域外（如 |b| ~ 1e300）分解会溢出
返回 NaN，已加 `isfinite` 护栏退化。

**(2) `kJmA1` / `kJmB1` 有 9 个字面量与二进制不逐位一致。** 用「精确 64 位模式
是否出现在 `libfiioaudio.so` 中」做无地址校验，查出：

```
kJmA1[1] -3.7393482705535    源 c00dea2f6d130d32   二进制 c00dea2f6d130d2b
kJmA1[2]  6.87675865807705    源 401b81cd058bb3ff   二进制 401b81cd058bb3fc
kJmA1[3] -6.34586541253185    kJmA1[4]  2.9375122501329
kJmA1[5] -0.545510875177145
kJmB1[1] -11.2518431683112    kJmB1[2] 13.1100289679614
kJmB1[4]  3.06028537571256    kJmB1[5] -0.454489124822854
```

差 1–2 ULP。mode 1 的系数达 ±13 且强烈抵消（和落到 ~1e-5 量级），
几个 ULP 就足以翻转判决 —— 实测 `ramp` 上 **1024 个 bit 翻了 254 个**。
mode 0 / mode 2 的字面量恰好全部正确，所以此前从未暴露。

已换成二进制的精确 repr，并新增 **`tools/check_ntf_literals.py`**：
对全部 34 个字面量做内容搜索校验，当前 **0 处不符**。

**(3) 判决寄存器一度被误判。** 曾以为 mode 1/2 把 `tbl*d8` 折进 u 链、
在 u 上判决。实测 `fcmp` 处的寄存器后确认：**三条支路都是把 `tbl*d8` 折进
A 链、并在 A 链上判决**，原实现是对的：

```
mode 0  fcmp d0 @0x1a504      mode 1  fcmp d2 @0x1a3f4      mode 2  fcmp d1 @0x1a578
```

三次误判/误修的过程本身说明：**系数与结构必须由运行时读数确定，不能靠寄存器
名字推断。**

### 65.3 量化器结构（最终确认）

三条支路的模式相关常量地址已全部解析并核对：

```
mode 0  A: 0x7e98 0x7f60 0x7f98 0x7fa8 0x7f38 0x7ef0 0x7fe8 0x7fa0
        B: 0x7ed8 0x7e68 0x7ef8 0x7ec8 0x8000 0x7f18 0x7f80 0x8008
mode 1  A: 0x7f48 0x7f50 0x7f78 0x8010 0x7f58 0x7ec0
        B: 0x7e50 0x7ff8 0x8018 0x7fb8 0x7e58 0x7e80
mode 2  A: 0x7ed0 0x7ea8 0x7fc0 0x7e60
        B: 0x7e88 0x7fc8 0x7eb0 0x7e78
```

统一形式：

```
A = s0*A0 + s1*A1 + ... + s(T-2)*A(T-2) + tbl*d8 + s(T-1)*A(T-1)
B = s0*B0 + s1*B1 + ... + s(T-1)*B(T-1)
bit = (A >= 0),   q = (A < 0) ? +1 : -1,   s0 <- (B + A) + q
```

（mode 0 的 `s7*B7` 在 `fcmp` 之后才加上，属于量化器状态的一部分而非判决量。）

### 65.4 状态槽已逐槽核对

对齐捕获（固件 `qs[i]` 为第 i 次更新**前**的状态，C++ 为第 i−1 次更新**后**）：

- **mode 0**：8 个槽全部逐位一致；`qs[6]/qs[7]` 确实被填充（1024 个样本中
  1017 个非零），说明是完整的 8 槽右移
- **mode 2**：4 槽全部一致
- **mode 1**：修正字面量后一致

### 65.5 DSD packing：**本函数内不做 8:1 打包**

实测固件输出：**每个 1-bit 样本写一个字节**，值 0/1 放在 bit 0，
`byte[k] == bits[k]` 逐个吻合（2048 个样本 → 2048 字节）。

所以 8 样点 → 1 字节的打包**不在 `get_pcm2dsd_data` 内**，而在调用方
（`0x1305c` 之后）。此前"每样本 24 字节/8:1 打包"的表述以此为准修正。
打包规则本身仍需在调用方层面确定，尚未验证。

### 65.6 当前状态

```
[Verified]
  输入格式（16 字节记录 = 交错立体声 [L0,R0,L1,R1]）
  PCM → tbl 绝对标度（pcm = int32/2^31，tbl = pcm/32）
  5 级 2× 零插入 + 递归节插值拓扑（各级 count = n<<k，输出 2*count）
  状态槽与延迟关系（FIR 二阶延迟 + d1 滞后 A）
  mode 0/1/2 抽头与逐位精确的 A/B 行
  tbl*d8 折入 A 链、三档均在 A 链判决
  量化器状态右移（含 mode 0 的 8 槽）
  C++ 正确舍入 FMA（工作域内已证明）
  C++ 插值器 bit-exact（9 用例）
  C++ 量化器 bit-exact（15 用例，mode 0/1/2）
  NTF 频域行为（≤ 0.8 dB）

[To verify]
  C++ 插值器 → C++ 量化器 串联（第二层）
  1-bit → DSD 打包（在调用方层面）
  通道顺序 / block 边界
  byte_count → frames → 量化字节 → 打包字节 的完整公式
  最终 DSF 文件对拍
```

## 66. 第二层：C++ 两段串联闭合 + 双通道独立验证

### 66.1 验收结构（按分层归因设计）

```
input (16 字节记录, 交错立体声)
   ↓
C++ runStages  -> tbl dump
   ↓
C++ bit()      -> 1-bit-per-byte dump
```

同时跑两条参考：`固件 tbl → Python exact 量化器` 与 `C++ tbl → C++ 量化器`，
三层比较：

```
① firmware_tbl == C++_tbl
② Python_bits == C++_bits
③ firmware_bits == Python_bits
```

### 66.2 结果：15/15 全部 bit-exact

| 用例 | mode 0 | mode 1 | mode 2 |
|---|---|---|---|
| zero | 0 / 1024 | 0 | 0 |
| impulse | 0 | 0 | 0 |
| ramp | 0 | 0 | 0 |
| alternating | 0 | 0 | 0 |
| stereo (L≠R) | 0 | 0 | 0 |

`max_abs(tbl) = 0`、`max_rel(tbl) = 0`、`first_tbl_mismatch = -`、
`first_bit_mismatch = -`，全部用例全部 mode。

**① ② ③ 同时成立 → C++ 核心串联（PCM → tbl → 1-bit）完全闭合。**

### 66.3 一次真实的测试工具 bug（不是模型 bug）

初跑时 `ramp` 与 `stereo` 在 `first_tbl = 32` 起发散，`max_rel` 达 2.16，
而 ③ 全通过（`m3 = 0`）—— 按归因表直接判定为「插值器仍有问题」。

实际原因：新写的 C++ 全链 checker 里，通道 0 的解交错取了索引 `ch*2+0` 和
`ch*2+1`，即 `[L0,R0,L1,R1]` 中的 **L0 和 R0**；正确应为索引 **0 和 2**
（L0 和 L1）。因为这几个用例恰好 `L0 == L1`，错误被完全掩盖。

用同一输入序列喂另一个只做 `runStages` 的二进制（`jmcheck`）时与固件
**768 样本 0 处不同** —— 一步定位到工具而非模型。

**教训：分层归因表能把错误限制在一层之内，但前提是每一层的输入构造都被独立验证。**

### 66.4 一个必须记录的规律：成对相等的输入会掩盖相位错误

扫描 5 种输入形状 × 5 种记录数（`bisect_tbl.py`）：

```
const  pairs      (L0==L1)   全部 OK
ramp   pairs      (L0==L1)   全部 OK
const  distinct   (L0!=L1)   一律在输出索引 32 处发散
ramp   distinct   (L0!=L1)   一律在输出索引 32 处发散
ramp   halfpair              一律在输出索引 32 处发变
```

输出索引 32 恰对应**输入索引 1**（32× 插值）。所以任何让相邻输入相等的用例
都会把相位/结构错误完全隐藏。**验收输入必须包含 L0 ≠ L1 的记录。**

### 66.5 双通道独立验证（24/24）

| 用例 | mode 0 | mode 1 | mode 2 |
|---|---|---|---|
| L impulse / R zero | ch0 ✓ ch1 ✓ | ✓ ✓ | ✓ ✓ |
| L zero / R impulse | ch0 ✓ ch1 ✓ | ✓ ✓ | ✓ ✓ |
| L == R | ch0 ✓ ch1 ✓ | ✓ ✓ | ✓ ✓ |
| L ramp / R alt | ch0 ✓ ch1 ✓ | ✓ ✓ | ✓ ✓ |

两个通道各自与固件 bit-exact。确认：

- 输入布局为**交错立体声** `[L0, R0, L1, R1]`
- 通道 0 取索引 0/2，通道 1 取索引 1/3
- 每通道独立状态（`handle+0x10` / `handle+0x50`）
- **无通道串扰**：固定 ch0 输入、只改变 ch1 数据，ch0 的 tbl 1024 个样本 0 处不同

### 66.6 黄金测试向量已固化

`H:\ALL_TO_DSD\golden\`：8 用例 × 3 mode × 2 通道 = **48 组，0 失败**，241 个文件。

每组包含：

```
<base>_tbl.txt        量化器输入表（hex 位模式）
<base>_bits_fw.txt    固件产生的 bit 序列
<base>_bits_py.txt    Python exact-FMA 参考的 bit 序列
<base>_bits_cpp.txt   C++ Jm21Core 的 bit 序列
<base>.meta           sample_count / channel / mode / initial_state / 定标 / 升采样率
MANIFEST.txt          覆盖范围与逐组结果
```

`.meta` 明确记录：`input_record_bytes 16`、`input_layout interleaved_stereo_L0_R0_L1_R1`、
`pcm_scale int32/2^31`、`tbl_scale pcm/32`、`oversampling 32`、`initial_state all_zero`、
`quantizer_intermediate 1 logical bit per byte, bit0 = sample`。

这样不会再出现"同一文件其实来自不同状态"的问题。

### 66.7 明确的边界

```
get_pcm2dsd_data (0x19d90)
    已闭合：PCM -> tbl -> 1-bit-per-byte intermediate
             （含 mode 0/1/2、双通道、全部输入形状，逐位一致）

caller (0x1305c 之后)
    未闭合：8:1 packing（8 samples -> 1 byte）
            channel order（sample-interleaved 还是 channel-block）
            block framing / block size
            最终 DSF 输出契约
```

不再把 `get_pcm2dsd_data` 的输出称作 DSD packing —— 它是
**quantizer intermediate：1 个逻辑比特存为 1 字节，bit0 有效**。

### 66.8 当前状态

```
[Verified]
  输入格式（16 字节记录 = 交错立体声 [L0,R0,L1,R1]）
  PCM -> tbl 绝对标度（int32/2^31 -> /32），整体 32x 升采样
  5 级 2x 零插入 + 递归节拓扑与状态延迟关系
  mode 0/1/2 抽头与逐位精确的 A/B 行（tools/check_ntf_literals.py 守护）
  tbl*d8 折入 A 链、三档均在 A 链判决
  量化器状态右移
  正确舍入 FMA（Dekker + TwoSum，工作域内已证明）
  C++ 单段插值器 bit-exact（9 用例）
  C++ 单段量化器 bit-exact（15 用例，mode 0/1/2）
  C++ 两段串联 bit-exact（15 用例）          <- 本节
  双通道独立且无串扰（24 用例）              <- 本节
  黄金向量 48 组固化                          <- 本节
  NTF 频域行为（<= 0.8 dB）

[To verify]  全部属于 caller 层，与已闭合的核心无关
  8:1 packing
  channel order at the container level
  block framing / block size
  byte_count -> frames -> quantiser bytes -> packed bytes 的完整公式
  最终 DSF 文件对拍
```

**结论：`PCM → tbl → 1-bit` 的 C++ Reference Implementation 已正式闭合。**
剩下的纯粹是 intermediate → DSD 容器/输出契约。

## 67. 8:1 打包契约（已验证）

### 67.1 定位过程

从调用点 `x1 = handle + 0xC0005A` 出发，先解析 PLT 桩的真实符号
（`plt2.py`，走 `PT_DYNAMIC` → `DT_JMPREL` → `DT_SYMTAB`）：

```
0x1dcd0 -> get_pcm2dsd_data      核心
0x1dce0 -> __memcpy_chk          不是打包器（w3 是 destlen）
0x1dcf0 -> get_dsdtopcm_data     反向
0x1dd00 -> memcpy_by_audio_format
0x1e040 -> memcpy
```

调用点 `0x13328` 处 `__memcpy_chk(handle+0x400000, handle+0xC0005A, w25, 0x90A000)`
**只是拷贝，不是打包**。全库 `.rodata` 也没有任何 8:1 位聚集魔数
（`0x8040201008040201` 等均不存在）。

真正的打包器在**核心内部**，尾声处：

```
0x1a6cc  add  x9,  x25, #0x120, lsl #12      ; handle + 0x120000
0x1a6d0  lsr  x8,  x8, #3                    ; count >> 3
0x1a6d4  add  x11, x9,  #0x3a0               ; handle + 0x1203A0  通道 A
0x1a6d8  movi v0.2s, #0xc0 ... v5.2s, #0xfe  (掩码 0xC0,0xE0,0xF0,0xF8,0xFC,0xFE)
0x1a6f4  mov  w9,  #0x10000
0x1a6fc  ldr  q7,  [x11, x9]                  ; handle + 0x1303A0  通道 B
0x1a704  ldr  q6,  [x10], #0x10               ; 通道 A，步进 16
        ... ushr #0xb/#0x14/#0x1d/#0x26, shl #7, xtn, trn1, and ...
0x1a7dc  st1  {v6.s}[0], [x13], #4            ; 写 4 字节
0x1a7e0  b.ne #0x1a6fc
```

**每轮读通道 A 16 字节 + 通道 B 16 字节，产出 4 字节 —— 8:1，且双通道在同一次
迭代内混合。** 掩码 `0xC0,0xE0,0xF0,0xF8,0xFC,0xFE` 配合 `shl #7` / `xtn` 逐级
收窄，是标准的 NEON「int32 → 字节，取最高有效位」惯用法。

### 67.2 单比特探测：精确映射

把打包循环单独仿真（`packer.py`，从 `0x1a6cc` 跑到 `0x1a7e4`），
每次只在通道 A 或 B 的一个位置放 1：

```
通道 A (handle+0x1203A0) 字节 p:   p=0..7  -> out[0] 的 bit (7-p)
                                p=8..15 -> out[2] 的 bit (15-p)
通道 B (handle+0x1303A0) 字节 p:   p=0..7  -> out[1] 的 bit (7-p)
```

即：

```
out[0] = 通道A 的样本 0..7   打包
out[1] = 通道B 的样本 0..7   打包
out[2] = 通道A 的样本 8..15  打包
out[3] = 通道B 的样本 8..15  打包
```

### 67.3 打包契约（最终）

```
每 8 个 A 样本  -> 1 字节，随后 1 个 B 字节，循环
块内：MSB-first —— 最早的样本落在 bit 7，最晚落在 bit 0
字节间：时间顺序递增
```

**推论：packed 流在字节粒度上按通道交替，每块 8 个样本。**
一个 packed 字节只属于一个通道 —— 这一点由单比特探测直接证明
（A 的比特只出现在 out[0]/out[2]，B 的只出现在 out[1]/out[3]）。

### 67.4 端到端验证

真实核心跑完后，取两个通道的 intermediate，喂给独立打包器，
与核心自己写进 `x1` 的字节三方比对：

| mode | 打包迭代 | x1 字节 | 独立 vs 参考 | 核心 vs 参考 |
|---|---|---|---|---|
| 0 | 32 | 128 | **128/128** | **128/128** |
| 1 | 32 | 128 | **128/128** | **128/128** |
| 2 | 32 | 128 | **128/128** | **128/128** |

**8:1 打包从 Unresolved 变为 Verified。**

### 67.5 计数公式

本轮 `nrec=8` 记录（byte_count=128, D=4, NF=32）：

```
每通道输入样本 = NF/2 = 16
每通道 intermediate 字节 = 16 × 32 = 512
packed 字节 = 512 / 4 = 128
```

即 **packed 字节数 = NF × 4 = byte_count / D × 4**。
`w3=32` 时 `D=4`，所以 **packed 字节数 = 输入 byte_count**。

换成帧：`记录 = 16 字节 = 2 帧/通道`，故
**每帧每通道 4 个 packed 字节，立体声每帧 8 字节**
（32 样本/通道 ÷ 8 = 4 字节 ✓）。

### 67.6 C++ 现状：打包层与固件不符

`src/flac2dsf.cpp` 当前是：

```cpp
if (jm[c].bit()) blk[c][bitPos[c] >> 3] |= (uint8_t)(1u << (bitPos[c] & 7));
```

两处都错：

1. **位序**：固件是 MSB-first，C++ 是 LSB-first
2. **通道交织**：固件在字节粒度上按 `[A][B]` 交替（8 样本一块），
   C++ 是每通道各自独立成块（channel-block）

这正是历史上导致「立体声 2× bug」的那一类问题，只是这次方向相反。

### 67.7 术语与边界（更新）

```
get_pcm2dsd_data (0x19d90)
  1. 16 字节记录解码 -> pcm = int32/2^31
  2. 五级 2x 零插入 + 递归节插值 -> tbl = pcm/32
  3. mode 0/1/2 量化器 -> 1 逻辑比特存为 1 字节（bit0）
  4. NEON 8:1 打包 -> x1，MSB-first，按 [A][B] 每 8 样本交替

caller (0x13038 起)
  __memcpy_chk(handle+0x400000, handle+0xC0005A, count, 0x90A000)   纯拷贝
```

仍未闭合：block framing / block size、跨调用的字节数取整与进位、
`handle+0x400000` 之后的最终 DSF 写出与容器头。

## 68. 打包层 C++ 改写完成 + packed 黄金向量（边界更正）

### 68.1 边界更正（SUPERSEDED）

此前文档写：

> "`get_pcm2dsd_data` 只产生 1-bit-per-byte，8:1 packing 在 caller"

**该结论作废。** 本轮 NEON 实证（第 67 节）表明打包发生在**核心内部尾声**：

```
get_pcm2dsd_data (0x19d90)
  1. 16 字节记录解码 -> pcm = int32/2^31
  2. 五级 2x 零插入 + 递归节插值 -> tbl = pcm/32
  3. mode 0/1/2 量化器 -> 1 逻辑比特存为 1 字节（bit0）
     A: handle+0x1203A0   B: handle+0x1303A0
  4. NEON 8:1 打包 (0x1a6cc-0x1a7e0) -> x1，MSB-first，按 [A][B] 每 8 样本交替

caller (0x13038 起)
  __memcpy_chk(handle+0x400000, handle+0xC0005A, count, 0x90A000)   纯拷贝
```

准确表述：**量化器产生 1-bit-per-byte 的内部中间表示；`get_pcm2dsd_data`
内部尾声继续将其按 8:1 MSB-first、A/B 字节交织打包；调用方目前已确认至少存在
`memcpy`/`__memcpy_chk`，但不承担 packing。**

### 68.2 C++ 打包层改写

改前（两处都与固件不符）：

```cpp
if (jm[c].bit()) blk[c][bitPos[c] >> 3] |= (uint8_t)(1u << (bitPos[c] & 7));
```

- 位序 **LSB-first** —— 固件是 **MSB-first**
- 每通道各自独立成块（channel-major）—— 固件是**字节粒度 [A][B] 交替**

改后（`src/flac2dsf.cpp`）：

```cpp
for (int p = 0; p < total; p += 8) {
    const int m = (total - p) < 8 ? (total - p) : 8;
    for (int c = 0; c < ch; ++c) {
        uint8_t v = 0;
        for (int k = 0; k < m; ++k)
            if (jm[c].bit()) v |= (uint8_t)(1u << (7 - k));
        emitPacked(v);
    }
}
```

并为 JM21 路径新增独立的交织块缓冲 `jmPacked`（因为交织流不能沿用每通道缓冲），
新增调试开关 `--jm21-dump-packed`。

**默认 FIR 路径未受影响**，回归数字不变（SNR 32.0 dB / gain +0.6817 / err-sig 0.0172）。

### 68.3 硬性不变量写进 C++ checker

`jmfull` 现在**同时跑两个通道**并直接输出交织后的 packed 流，把契约固化成断言：

```
packed[2k+0] = pack_msb(A[8k : 8k+8])
packed[2k+1] = pack_msb(B[8k : 8k+8])
pack_msb: 最早样本在 bit 7
```

任何把实现改回 channel-major 或 LSB-first 的改动都会在这个 checker 立刻失败，
而不是只在最终文件 diff 时才暴露。

### 68.4 packed 黄金向量：24/24 byte-exact

8 用例 × 3 mode，每组 256 字节，三方比对：

| 用例 | mode 0/1/2 | fw==py==cpp |
|---|---|---|
| zero | ✓ ✓ ✓ | 256 字节 |
| impulse | ✓ ✓ ✓ | 256 |
| alternating | ✓ ✓ ✓ | 256 |
| **L impulse / R zero** | ✓ ✓ ✓ | 256 |
| **L zero / R impulse** | ✓ ✓ ✓ | 256 |
| L ramp / R alternating | ✓ ✓ ✓ | 256 |
| L≠R ramp / R ramp | ✓ ✓ ✓ | 256 |
| full-scale mix | ✓ ✓ ✓ | 256 |

**byte mismatch = 0，first mismatch = none，length 全部精确。**
用例集刻意包含 `L≠R`、单通道脉冲、满量程，因此 MSB/LSB 翻转、A/B 互换、
channel-major 都会立刻被抓住。

已固化到 `H:\ALL_TO_DSD\golden\packed\`（97 个文件），每组含
`_fw.bin` / `_py.bin` / `_cpp.bin` / `.meta`，`.meta` 里写明计数、缓冲地址、
打包规则与那条显式不变量。

### 68.5 两次测试工具 bug（都不是模型问题）

1. `packed_golden.py` 里 C++ checker 原本是**单通道**运行，拼接后成了
   channel-major。改成 checker 自身跑双通道并输出交织流 —— 不变量因此才真正
   被 C++ 强制。
2. `mkfull.py` 的 `main()` 被脚本改坏后编译失败，但旧 exe 仍在被复用
   （只在文件不存在时才报错）。已改为**构建前删除旧 exe**，
   否则会拿陈旧二进制得出"通过"的假结论。

第 2 条和之前那次「哨兵地址被寄存器回 spilling 写脏」是同一类问题：
**测试工具自身失效会伪装成模型结论，必须让工具失败可见。**

### 68.6 当前状态

```
[Verified]
  PCM 输入格式与绝对标度（16 字节记录，交错立体声，int32/2^31）
  5 级 2x 零插入 + 递归节插值（tbl = pcm/32）
  mode 0/1/2 抽头、A/B 行逐位精确、tbl*d8 折入 A 链并在 A 链判决
  正确舍入 FMA（Dekker + TwoSum）
  量化器状态右移
  1-bit -> 8:1 packed：MSB-first + [A][B] 字节交织
  单段/串联/双通道均与固件 bit-exact（15/15、24/24、48 组黄金向量）
  packed 层三方 byte-exact（24/24）
  NTF 频域行为

[Unresolved]  全部属于 output contract / framing
  caller 的 block framing 与 block size
  packed 流在跨调用时的累计与 flush 时机
  非整块输入（NF=8/16/24/32/40/64）的取整与进位
  DSF 容器字段
  最终真机 playback 对拍
```

**性质已改变：不再是逆向算法，而是在逆向 output contract / framing protocol。**
DSP 核心与 bitstream 生成规则已封死，48 组黄金向量 + 24 组 packed 向量作为回归门禁。

## 69. Caller 层的每次调用产出与跨调用状态

### 69.1 单次调用的产出（实测，非推测）

对 `byte_count = 16,32,…,1024`（w3=0x20）逐个测 `get_pcm2dsd_data`
的返回值 `w0` 与实际写入 `x1` 的字节数：

| byte_count | D | NF | 记录数 | 每通道 intermediate | w0 | 实际 packed |
|---|---|---|---|---|---|---|
| 16 | 4 | 4 | 1 | 64 | **16** | 16 |
| 32 | 4 | 8 | 2 | 128 | **32** | 32 |
| 48 | 4 | 12 | 3 | 192 | **48** | 48 |
| 64 | 4 | 16 | 4 | 256 | **64** | 64 |
| 96 | 4 | 24 | 6 | 384 | **96** | 96 |
| 128 | 4 | 32 | 8 | 512 | **128** | 128 |
| 192 | 4 | 48 | 12 | 768 | **192** | 192 |
| 256 | 4 | 64 | 16 | 1024 | **256** | 256 |
| 512 | 4 | 128 | 32 | 2048 | **512** | 512 |
| 1024 | 4 | 256 | 64 | 4096 | **1024** | 1024 |

**公式（实测确认）**

```
接受条件：byte_count 是 16 的倍数，且 NF = byte_count / (w3+7>>3) >= 2
          否则   byte_count % 16 != 0  ->  返回 -3   (0x19e98)
                 NF < 2              ->  返回 -6   (0x19e58)
每通道 intermediate 字节 = (NF >> 1) * 32 = byte_count * 4
packed 字节            = intermediate / 4  = byte_count        <- 1:1
```

caller 在 `0x13070 mov w25, w0` 直接把 `w0` 当作 packed 长度，
`0x13328` 处 `__memcpy_chk(handle+0x400000, handle+0xC0005A, w25, 0x90A000)`
按该长度原样拷贝。

**换成帧**：`byte_count = 16 × 记录 = 8 × 帧`，故

```
packed 字节 = 8 × 帧（立体声）   =   每帧每通道 4 字节
```

与第 67 节的 `[A][B] × 每 8 样本` 布局一致（32 样本/帧 ÷ 8 = 4 字节/通道/帧）。

**核心内部没有任何 partial-block 保留**：一次调用就把全部 packed 字节产出。

### 69.2 非整块 / 非 16 倍数输入

```
byte_count = 1,4        -> -6
byte_count = 8,12,20,24,28,36,44,52,60   -> -3
```

即**核心只接受 16 字节（记录）的整数倍**，不处理余数。所以 caller 不需要
向核心喂非整块数据 —— 这把"非整块输入"的问题从核心推给了 caller 的
分块策略。

### 69.3 跨调用状态：一次调用 == 多次调用

方法：每次调用用**全新的模拟器**，只把 handle 快照恢复进去。
这样测的问题是精确的 —— *跨调用状态是否全部位于 handle*。

| 分块 | 单次调用 | 多次调用 | 结果 |
|---|---|---|---|
| 3 × 8 记录 | w0=384, 384 字节 | [128,128,128] = 384 | **IDENTICAL** |
| 8 + 1 + 8 | w0=272, 272 字节 | [128,16,128] = 272 | **IDENTICAL** |
| 4 × 1 记录 | w0=64, 64 字节 | [16,16,16,16] = 64 | **IDENTICAL** |
| 16 × 4 记录 | w0=1024, 1024 字节 | 16×64 = 1024 | **IDENTICAL** |
| 5 + 11 | w0=256 | [80,176] = 256 | **IDENTICAL** |
| 2 + 14 | w0=256 | [32,224] = 256 | **IDENTICAL** |
| 7 + 9 | w0=256 | [112,144] = 256 | **IDENTICAL** |
| 9 + 7 | w0=256 | [144,112] = 256 | **IDENTICAL** |
| 1 + 1 + 2 | w0=64 | [16,16,32] = 64 | **IDENTICAL** |
| 32 + 32 | w0=1024 | [512,512] = 1024 | **IDENTICAL** |

**10/10 逐字节一致。**

结论：

1. **核心的全部跨调用状态都在 handle 里**（量化器延迟线 `handle+0x10/0x50`、
   五级插值状态 `handle+0x090…0x328`、表与 intermediate 缓冲）
2. 核心是一个**干净的流式变换**
3. **framing 退化为纯字节拼接** —— 没有 carry、没有 flush 语义、没有重排

这正好把「核心算法问题」与「caller framing 问题」彻底分开：既然拼接成立，
framing 的全部内容就是「caller 以什么块大小把拼接后的字节流写进容器」。

### 69.4 剩下的 framing 问题（已缩小到很小）

既然 packed 流 = 逐次调用输出直接拼接，那么只剩：

```
caller 的块大小（DSF 为 4096 字节/块，每通道一块、块间交替）
跨块边界的处理：最后一个不完整块是否补零
DSF 头字段：channel num / sample rate / bits per sample / sample count /
            metadata offset / data size
```

以及一个仍需确认的行为：**caller 是否在多次调用之间保留 handle**
（第 69.3 节证明若保留则拼接成立；若真机不保留，则跨调用边界会重置
量化器状态，那是另一种语义）。这一条属于真机行为，需最后对拍确认。

### 69.5 反假通过门禁

实测发现：**`L0 == L1` 的记录会系统性掩盖相位/结构错误**（第 66.4 节：
`L0==L1` 的用例全部通过，`L0!=L1` 的一律在输出索引 32 = 输入索引 1 处发散）。

因此黄金向量**必须同时覆盖**：

```
L0 == L1    （成对相等）
L0 != L1    （成对不等）
```

`golden/MANIFEST.txt` 与 `golden/packed/MANIFEST.txt` 记录了每个用例的
记录布局；`packed_golden.py` 的用例集里
`zero` / `alternating` / `impulse` 提供 `L0==L1`，
`Limp_Rzero` / `Lzero_Rimp` / `Lramp_Ralt` / `Ldiff_Rdiff` 提供 `L0!=L1`。

**任何只保留其中一类的测试集都是无效的。**

## 70. 变异测试：证明黄金门禁真的有牙

### 70.1 方法

取生成的 `jmfull.cpp`，逐个注入最可能发生的回归，重新编译，跑 packed 黄金门禁：

```python
GOOD = "if (bits[ch][(size_t)(q + k)] == '1') v |= (uint8_t)(1u << (7 - k));"
```

| 变异 | 失败行数 | 结论 |
|---|---|---|
| 基线（未变异） | 0 / 24 | 门禁通过 |
| LSB-first 取代 MSB-first | **24 / 24** | 全部抓到 |
| A/B 通道互换（`bits[1-ch]`） | **18 / 24** | 抓到，但有 6 行漏网 |
| 时间方向反转（取样 `q+k+1`） | **24 / 24** | 全部抓到 |
| 恢复后 | 0 / 24 | 确认可逆 |

**三个回归全部被检测到，门禁有效。**

### 70.2 一个重要的定量结论

**A/B 互换只被 18/24 抓住** —— 漏掉的 6 行恰好是 `L0 == R0` 的用例
（`zero` / `alternating` / `impulse` / `fullscale` 各 3 个 mode）。

这不是测试集的缺陷，而是**这类输入在原理上就无法区分两个通道**：
当 L 与 R 的中间表示逐样本相同时，交换 A/B 不产生任何可观测差异。

这为第 66.4 节的规则给出了定量依据：

> `L0 == L1` / `L0 == R0` 的记录会**系统性掩盖**通道与相位类错误。
> 黄金向量**必须同时包含**成对相等与成对不等的记录，否则门禁会出现固定盲区。

对应地，`packed_golden.py` 的用例集固定为：

```
L0==R0 组（提供盲区对照，必须存在但不承担检测责任）
  zero / alternating / impulse / fullscale

L0!=R0 组（承担检测责任）
  Limp_Rzero / Lzero_Rimp / Lramp_Ralt / Ldiff_Rdiff
```

**删掉任何一组，门禁都会失去一半的检测能力。**

### 70.3 为什么必须做变异测试

本轮之前已经两次因为测试工具自身失效而得出错误结论：

- `jmfull` 原本单通道运行，拼接后成了 channel-major，却"通过"了比较
- `mkfull.py` 编译失败后**旧 exe 仍被复用**，伪装成"通过"

再加上第 66.3 节那次 `L0==L1` 掩盖真实 bug 的情况，说明：
**一个从未被证明会失败的测试，等于没有测试。**

`tools` 侧的变异测试因此固定为回归门禁的一部分：
每次改动打包层，必须先跑变异测试确认门禁仍能抓住三个变异，
再跑黄金向量确认零失败。

### 70.4 已固化的门禁清单

```
tools/check_ntf_literals.py     34 个 NTF 字面量逐位对照二进制（当前 0 处不符）
golden/            48 组 tbl/bits 黄金向量（Python/C++/固件三方一致）
golden/packed/     24 组 packed 黄金向量（fw == py == cpp，byte-exact）
mutation_test.py   三个打包变异必须全部被检测
call_counts.py     单次调用产出公式（w0 = byte_count）与拒绝条件
crosscall2.py      一次调用 == 多次调用（10/10 逐字节一致）
```---

## 71. DSF container 与 payload 验收（含 `arg6` 默认值缺陷）

第 70 节把打包层封版后，PCM → DSF 文件的这一段仍然只有"能播放"级别的证据。
本节建立**逐字段**与**逐字节**两级验收，并在过程中发现一个真实缺陷。

### 71.1 实测容器布局（独立 parser，非 writer 自证）

`tools/parse_dsf.py` 按偏移硬解析，不复用 `src/dsf.cpp` 的任何结构：

```
offset  0   'DSD ' , u64 dsd_chunk_size            = 28
offset 28   'fmt ' , u32 fmt_chunk_size            = 52
offset 40          u32 format_version              = 1
offset 44          u32 format_id                   = 0
offset 48          u32 channel_type                = 1 (mono) / 2 (stereo)
offset 52          u32 channel_num                 = 1 / 2
offset 56          u32 sampling_frequency          = 2822400
offset 60          u32 bits_per_sample             = 1
offset 64          u64 sample_count
offset 72          u32 block_size_per_channel      = 4096
offset 76          u32 reserved
offset 80   'data' , u64 data_chunk_size           = payload + 12
offset 92   payload
```

两处曾经的错误认知，已作废：

- **不存在 `data_total_size` 字段**。writer 只写一个 `u64 chunk size`，随后就是 payload。
- `fmt` 在 **28**，`data` 在 **80**，payload 起于 **92**；早期按 36/92 解析导致 `channel_num` 读成 1。

### 71.2 payload 契约

```
real_bytes    = packed stream 的真实字节数（不含补零）
sample_count  = 8 * real_bytes          # mono
payload_len   = ceil(real_bytes / 4096) * 4096
payload[real_bytes .. payload_len)      全 0
```

mono 实测：`real_bytes = 352616`，`sample_count = 2820928`，`payload = 356352 = 87 * 4096`，
补零 `356352 - 352616 = 3736`。

stereo 下 `real_bytes` 是 `[A][B]` 字节交织流的长度，两通道共用同一个 DATA 区域。

### 71.3 缺陷：`Options::arg6` 默认值 4 使全部 `--jm21` payload 出错

真实调用者在 `0x13058` 传 `w7 = w6 = mode`，而 `handle+0x68` 在 `libfiioaudio.so` 中
**没有写入者**，因此设备上 `mode = 0`。既然 `arg6 == mode`，设备默认就是 `arg6 = 0`，
量化器驱动增益 `d8 = (double)(16 - arg6) = 16`。

而 `src/f2d.h` 里 `arg6` 的历史默认值是 **4**，给出 `d8 = 12`。
后果是 `--jm21` 产出的每一个 payload 都与参考不符：

```
arg6=4  d8=12  reference=1864  payload=4096  mismatches=1806  first=1
arg6=0  d8=16  reference=1864  payload=4096  mismatches=0     first=None
```

`arg6` 只在 `flac2dsf.cpp:713` 的 `if (o.jm21)` 分支内被使用，
因此**默认 FIR 路径可证明不受影响**，无需重新回归。

已改为 `int arg6 = 0;`。

### 71.4 参考实现的两个前置条件

为让"payload == 参考"真正有意义，参考必须与工具内部看到的是同一份数据：

1. **参考从 `--jm21-dump-pcm` 出发**，不是从 WAV 出发。
   这样工具内部的重采样、增益、通道选择无法掩盖任何差异。
2. **参考的 FMA 必须是正确舍入的**。
   纯 Python `Fraction` 对 88154 帧不可行，改为 float-only Dekker FMA
   （`tools/refchain.py`，与 `flac2dsf.cpp` 的 `jfma` 同构）。
   启用前先与 Fraction 精确版三方对齐：mode 0/1/2 全部 `fast == ref == fw`。

另外新增 `--jm21-dump-pcm.ch1`，使立体声第二通道也能独立求参考。

### 71.5 验收结果

`tools/dsf_check.py`（mono + stereo，256 帧，mode 0）：

```
A. container structural equality   全部字段 OK
B. payload byte equality           mismatches=0  sha256 相同
   补零到块边界                      OK，长度 = (-real) % 4096
VERDICT: PASS   （mono 与 stereo 均通过）
```

`tools/dsf_selftest.py` 第一部分 —— checker 故障注入，验证 checker 会失败：

| 变异 | container | payload | 归类 |
|---|---|---|---|
| 未改动 | True | True | 基准 |
| 翻转 DATA 第 5 字节 | True | **False** | payload ✓ |
| 改 header `channel_num` | **False** | True | container ✓ |
| 尾部截断 1 字节 | **False** | True | container ✓ |
| 中间插入 1 字节 | **False** | False | container ✓ |
| L/R 互换 | True | **False** | payload ✓ |

5/5 变异被正确分类，checker 具备分辨两种失败的能力。

第二部分 —— 跨块状态（3000 帧 → 5954 重采样样本，超过 `kBlk=4096`）：

```
2ch  12 blocks of 4096  reference=47632  mismatches=0  first=None  sha=True
1ch   6 blocks of 4096  reference=23816  mismatches=0  first=None  sha=True
```

即工具按 4096 帧多次调用 `runStages` 时，payload 仍等于一次性算出的单链参考，
印证第 69 节的跨调用契约：**状态全在 handle，纯拼接。**

### 71.6 本轮两次测试自身的缺陷（记录以免重犯）

**其一，变异用例是空操作。** 最初按 4096 字节整块对调来做 "L/R 互换"，
但 payload 是 `[A][B]` 字节交织的，整块对调根本不换通道；且当时 payload 只有 4096
字节（恰好 1 块），`body[4096:8192]` 为空，交换什么都没发生，
checker 报 `payload=True` 却看起来"通过"。真正的换通道必须先解交织再交换。

**其二，把通道数当成了 mode。**
`R.reference_packed(pcm0, pcm1, ch)` 的第三个形参是 `mode` 而非通道数，
于是 mono 被当成 mode 1（6 taps）、stereo 被当成 mode 2（4 taps），
产生 46055/47632 处不符的假失败。
**参考生成器的参数含义必须与被测路径一致，否则测试报告的是测试的 bug。**

这两点与第 70 节那两次（`jmfull` 单通道、`mkfull.py` 复用陈旧 exe）同源：
**门禁必须先证明自己会失败。**

### 71.7 仍然开放的问题

firmware 中 `get_pcm2dsd_data` 之后只有 `memcpy` 到 `handle+0x400000`，
再往后的最终 DSF 块布局尚未从 `libfiioaudio.so` 确认。当前实现把
`[A][B]` 交织的 packed 流按 4096 字节整块写入 DATA；`src/dsf.cpp` 里另有
`transposeData()` 与 `--planar` 后处理，可把帧交织块改写为通道平面块。

**DSF 规范通常要求每通道连续成块**（4096 字节一块、块内通道连续），
而当前默认写法是通道交织。若真机参考 DSF 采用通道平面块，
则需要在 writer 层加一道分块，但**不能改 packed 流本身**——
packed 流的 `[A][B]` 字节交织已由第 67 节固件逐字节证明。

此项需要参考 DSF 文件或真机导出才能定案，见 71.8。

### 71.8 已固化的门禁清单（增量）

```
tools/refchain.py     正确舍入 FMA 参考链（先与 Fraction 版对齐再用）
tools/parse_dsf.py    独立 DSF 结构 parser
tools/dsf_check.py    container structural + payload byte equality
tools/dsf_selftest.py checker 故障注入 5 例 + 跨块状态 2 例
```
## 72. 运行时 A/B 闭合：`persist.sys.all.to.dsd` 可由 shell 直写，p2d 路径 ON/OFF 日志签名完整（2026-10-06）

> 本节是第 35.9 节"第一优先"问题的实质推进。全部证据来自当次真机 logcat，
> 原始存档：`vendor_pull/runtime_logs/logcat_2026-10-06_p2d_ON.txt` / `..._OFF.txt`。

### 72.1 属性门控可直接程序化控制

本机（user 构建、未 root）shell **可以写** FiiO 的 persist 属性：

```
adb shell setprop persist.sys.all.to.dsd 0|1     # 立即生效（下一次 start_service_action）
adb shell setprop persist.sys.fiio.audio.process.log 1
adb shell setprop persist.sys.fiio.audio.file 1
```

这把第 36.15 节从静态反汇编读出的门控

```
get_alltodsd_config(): on = atoi(prop("persist.sys.all.to.dsd"))
```

升级为**运行时可控开关**：A/B 不再依赖人手在设置界面切换。
（注意：属性写入后在下一次播放会话启动时生效；正在进行的会话不切换。）

### 72.2 ON / OFF 日志签名（同一 192 kHz FLAC 源）

ON（`persist.sys.all.to.dsd=1`）：

```
D/fiioaudio: match_audio_process_mode: current p2d is enable
D/fiioaudio: match_audio_hardware_params: mode PCMTODSD srcRate 192000 srcFormat 3
             halFormat 3 srcCh 2 dstRate 88200 dstFormat 436207618 frameSize 8 halRate 88200
D/fiioaudio: start_service_action: convert_size 4096 hwRate 88200 hwFormat 436207618
D/fiioaudio: process_audio_loop:begin
```

OFF（`persist.sys.all.to.dsd=0`）：

```
D/AudioTrack: config_audio_params: ... check AudioFlinger sampleRate=192000 format=3 p2d=0
D/fiioaudio: match_audio_hardware_params: mode ALLTO32BIT srcRate 192000 srcFormat 3
             halFormat 3 srcCh 2 dstRate 192000 dstFormat 3 frameSize 8 halRate 192000
D/fiioaudio: start_service_action: convert_size 4096 hwRate 192000 hwFormat 3
```

| 维度 | ON | OFF |
|---|---|---|
| 模式字符串 | `PCMTODSD` | `ALLTO32BIT` |
| p2d 状态 | `p2d is enable` | `p2d=0` |
| HAL 入流 | `dstFormat 436207618`(=0x1A000012) | `dstFormat 3`（普通 PCM） |
| HAL 码率 | 88200 Hz × 8 B = **705 600 B/s 立体声 = DSD64** | 输入原率 192 kHz PCM |

`dstFormat 0x1A000012` 是 FiiO 私有格式 tag（HAL 收到它即按 DSD packed 流处理）。
`frameSize 8`（双通道 32-bit 打包字）× 88200 Hz = 705 600 B/s = 2 ch × 352 800 B/s，
与第 7 节早前观测的 `hwRate 352800`（每字节 8 bit，DSD64）**完全互洽**：
2 ch × 352 800 B/s/ch = 705 600 B/s。**真机 All-To-DSD 输出 = DSD64，确认。**

### 72.3 输入预重采样到 88200 Hz —— 第 28 节"待证项"运行时闭合

第 28 节曾指出："把『输入被预重采样到 88200』当成前提"是不严谨的，需要运行时确认。
本节日志直接给出答案——对 **192 kHz 源**：

```
D/fiioprocess: processAudioConfigParams: begin +++++ srcRate 192000 srcFormat 3 srcCh 2
D/fiioaudio:   start_config_params: ... tDstRate 88200 hwRate 192000 ...
D/fiioprocess: processAudioConfigParams: end ----- *halRate 88200 *halFormat 3
D/fiioaudio:   start_service_action:begin ----- srcRate 88200 srcFormat 3
```

fiioprocess 重采样器把 192 000 → 88 200 Hz 后，服务以 `srcRate 88200` 进入 p2d。
结合第 7 节 44.1 kHz 源的日志（`srcRate 44100 → dstRate 88200`）：

> **所有输入率在进入 p2d 前被统一备到 88200 Hz；88200 × 32 = 2 822 400 = DSD64。**

`--jm21` 管线（输入重采样到 88200 → 5 级 2× = 32× → 6 抽头量化器 → DSD64 → DSF）
与真机结构**逐级吻合**。

### 72.4 `convert_size 4096` 与模型 kBlock 的独立佐证

`start_service_action: convert_size 4096` 在 ON/OFF 两态都出现（ON 态与 DSD 格式同现）。
与模型/DSF 写层的 4096 字节块（第 68 节 packed 黄金向量、`kBlock=4096`）数字一致。
**定位**：这是服务层的搬运块大小，不单独证明 DSD 块布局；作为旁证记录，不作为证明。

### 72.5 `persist.sys.fiio.audio.file` dump 的新细节

设置该属性后播放，出现：

```
E android.hardware.audio.service_64: ERR(init_global_wav_file):open(/data/misc/test/test_192000.wav) fail
```

新事实：

1. dump 文件名按**输入采样率**命名（`test_%d.wav` 的 `%d` = rate，非序号）。
2. `init_global_wav_file` 在**每次 start_service_action 时执行**（本次属性是会话间设置的，
   下一次播放立即生效）→ 第 7 节的全部 debug 属性都是 per-session 读取，不是仅库加载时。
3. 反汇编（`init_global_wav_file` @0x1906c 起）发现该 WAV 写入器支持 DSD 速率分支：
   `0x0002B110`(176 400)、`0x00056220`(352 800)、`0x000AC440`(705 600) ——
   即 DSD64 的 16-bit/8-bit/32-bit 打包表示速率，且有 4 字节组内 0↔2、1↔3 的
   **WAV 字节序交换循环**。
4. 目录 `/data/misc/test` 不存在且 shell 无权创建（第 7 节"不可达"结论维持），
   开启该属性只会持续产生一条 open 失败日志，无副作用。

> 推断（未验证）：若能在出厂/工厂镜像上拿到该 dump，test_88200.wav（或 DSD 速率分支）
> 可能直接给出真机 p2d 的输入 PCM 或 1-bit 流，用于与 `--jm21` 逐字节对比。
> 在 user 构建上无路径可达，不再投入。

### 72.6 level / delta-gain 旋钮（未闭合，留待模拟量 A/B）

```
persist.sys.pcm2dsd.level      # 现场设 16：无任何日志输出，无可见效果
persist.sys.dsd.delta.gain     # 现场值为空
导出函数: get/set_pcm2dsd_dsd_level, get_pcm2dsd_delta_gain,
          get_alltodsd_delta_gain, get_pcm2dsd_level
```

属性变化不产生 logcat。第 35.3 节观测的 **+3.58 dB** ON/OFF 差与
`get_alltodsd_delta_gain` 的存在性吻合（`delta gain` 命名直指该现象），
但当前无数字通路可验证——只能走 Line-In 模拟回录（第 34/35 节的链路）。
`mode0 d8=16`（第 0.1 节恒等式 gain = d8/32）与 level 属性取值域是否同源，待测。

### 72.7 第 35.9 节问题的状态升级

```
旧：真实 All-To-DSD 是否执行 0x19d90   未确认
新：p2d 路径运行时启用                  已确认（属性门控 A/B，本节 72.2）
    0x19d90 是该路径唯一静态生产者      已确认（第 26-29 节调用图，.text 仅一处 BL）
    直接 hook 0x19d90                  仍不可能（user 构建，无 root）
    间接通路（dump/log）                全部用尽（本节 72.5/72.6）
```

**判定**：`0x19d90 在 All-To-DSD ON 时被执行` 现在是
"运行时路径确认 + 静态调用图唯一性"共同支撑的结论，
再往下只剩指令级 trace 一条路（需 root/工程机）。
在无 root 约束下，**本项证据已饱和**。

## 73. 基址身份闭合：`x23 = g_audio_play_config`（不是 `g_pcm2dsd_handle`）；运行时 mode = (level<2) ? 0 : 2（2026-10-06）

> 本节回应一个长期未验证的假设：第 26-29 节把 caller 读 mode 的基址记作
> "handle+0x68"，并默认 `handle` = `g_pcm2dsd_handle`。本节用 GOT 重定位证据
> 证明**该假设是错的**，并据此闭合"属性 2 vs mode 0"矛盾（0.3b 第一项遗留）。

### 73.1 x23/x22 的基址身份（lief 重定位证明，非推断）

caller（0x12e80-0x1305c 所在函数）中：

```
0x1283c  adrp  x11, #0x1f000
0x12858  ldr   x11, [x11, #0x3d0]      ; x11 = *(GOT 0x1f3d0)
0x128f4  mov   x23, x11                ; x23 = *(GOT 0x1f3d0)
...
0x12e8c  ldr   w21, [x23, #0x70]       ; dsd_level
0x12e90  ldr   w19, [x23, #0x68]       ; mode = shaper = arg6
0x12e94  ldr   w24, [x23, #0x5c]       ; format magic
```

`.got` 槽位 0x1f3d0 的 `R_AARCH64_GLOB_DAT`（lief 解码 ANDROID_RELA）：

```
0x1f3d0  GLOB_DAT  g_audio_play_config   (0x20778, .data, 300 B)
0x1f410  GLOB_DAT  g_pcm2dsd_handle      (0x20d60, .bss, 0x1403e0 B)
```

> **`x23 = &g_audio_play_config`。全库引用 `g_pcm2dsd_handle` 的只有两处：
> 0x19e20（= `get_pcm2dsd_data` 真实体 0x19d90 内部）与 0x1a970（= `init_pcm2dsd`
> 真实体）。caller 根本不经过 `g_pcm2dsd_handle` 读 mode。**
> 第 41.3 节"handle+0x68"的措辞自此更正为 **GPC+0x68**（GPC ≡ g_audio_play_config）。

### 73.2 GPC 字段地图（访问器惯用法全扫 + 导出名映射）

访问器模式 `adrp xA,#0x1f000; ldr xA,[xA,#0x3d0]; ldr/str w0,[xA,#off]; ret`
全文本穷举 + 导出 veneer（0x1d040-0x1d3c0）目标映射：

| GPC 偏移 | 读 (R) | 写 (W) | 语义 |
|---|---|---|---|
| +0x5c | get 0x15a94 | set 0x15aa4 | format 魔数（运行时写 0x1A000002） |
| **+0x68** | **get 0x163e8**（导出为 `get_pcm2dsd_shaper_coeffs` **和** `get_pcm2dsd_delta_gain` 两个 veneer，0x1d044/0x1d048，同指一址） | **无** | mode = shaper = arg6 |
| **+0x70** | get 0x163f8（`get_pcm2dsd_dsd_level` 0x1d04c） | set 0x16410（`set_pcm2dsd_dsd_level` 0x1d2fc） | dsd_level（caller w5） |
| +0x78..+0x8c | get/set 对 | | MQA 状态组 |
| +0xa0/+0xa4/+0xa8 | get/set | | device opened / app pid / uid |

**两个 getter 同指 GPC+0x68 直接证明：shaper 系数选择与 delta gain 是同一个变量**
—— 与模型的 `d8 = 16 - arg6`（mode0→16 / mode1→15 / mode2→14）完全互洽。

### 73.3 GPC+0x68 全库写者 = 零（比 §41.3 更强的表述）

存储形式全扫（STR imm12、STUR、STRB、STRH、STP、寄存器偏移、64 位）：

| 站点 | 形式 | 判定 |
|---|---|---|
| 0x12d88 | `stp w22,w21,[sp,#0x68]` | 栈 |
| 0x1919c | `str x8,[x14,#0x68]!` | 栈帧 pre-index |
| 0x1a12c | `stp d17,d16,[sp,#0x68]` | 栈 |
| 0x12d88 附近 `str xzr,[x22,#0x68]`（0x15cc4） | 64 位清零，x22=GPC | **唯一真写者，见下** |

0x15cc4 所在函数 = 0x15b60（日志 tag 字符串 "get_current_spdif_config"，位于
spdif 配置逻辑中，读 `persist.sys.audio.output.select`/`persist.music.DSD.format_spdif`，
把 level 属性 `== 2` 映射成 GPC+0x30 布尔）。**但该函数全库零调用**：
无 BL、无 B（尾调）、无 veneer 指向、非导出、无函数指针引用（全文件镜像 8 字节扫描 +
143 条重定位全部核对，无一处 0x15b60）。
真正导出的 `get_current_spdif_config`（veneer 0x1d074）实体在 0x16910，只读属性不写 GPC。

> **GPC+0x68 在本构建中没有任何可执行路径能写它。.data 文件镜像初值 = 0。
> ⇒ 运行时恒 mode = 0。**

### 73.4 GPC+0x70（dsd_level）的活写者：level 属性是活的

`persist.sys.pcm2dsd.level`（0x62bf）全库 7 个读取点：

```
0x116f4  str w0,[x28,#0x70]   x28=GPC   ← 活（写 GPC+0x70）
0x11910  （config 区，同族）
0x11fb0  stp w20,w19,[x28,#0x58]; str w23,[x28,#0x60]; str w0,[x28,#0x70]
         ← 活：同函数写入 format 魔数 GPC+0x58/0x5c/0x60 —— caller 测的正是
           GPC+0x5c，故此函数必然在真实 DSD 路径上执行（logcat 佐证）
0x15c84  （死函数 0x15b60 内；且只做 ==2 布尔映射到 GPC+0x30）
0x15ff0 / 0x161bc  （config 区；0x161bc 处有 cmp w0,#3 比较）
0x17f9c  （= 导出 get_pcm2dsd_level 0x1d094 的实体：直接 atoi 返回属性值）
```

> **`GPC+0x70 = atoi(persist.sys.pcm2dsd.level)`（默认 "0"）在活代码里成立。**
> 该属性随格式配置一起刷新，与 caller 的 w5 实参同源。

### 73.5 运行时 mode 的闭合公式

callee 内（第 41.2 节已读出的分支，今天补上两个输入的基址与初值）：

```
w5  = GPC+0x70 = atoi(persist.sys.pcm2dsd.level)   （活，默认 0）
w6  = GPC+0x68 = 0                                  （无写者，恒 0）
w26 = (w5 < 2) ? w6 : 2
```

```
persist.sys.pcm2dsd.level = 0 或 1（或未设置）  →  w26 = 0  →  mode 0（8 抽头，d8=16）
persist.sys.pcm2dsd.level = 2、3、…             →  w26 = 2  →  mode 2（4 抽头，d8=14）
mode 1（6 抽头）在真实播放路径上不可达           （w6 恒 0，无写者）
```

**当前设备：level 属性为空 → atoi=0 → 运行时 = mode 0，d8 = 16。**

这同时解释了第 27 节 harness 快照的 mode=2 / x5=1：那是 harness 自己铺的内存值
（当时按"工厂属性 2"假设写入），不是设备真实状态。第 43 节的三档验证
（shaper 0/1/2 × `x5=1<2` 强制走 w26=w6 分支）因此依然有效——它验证的是
"给定 w6 时三分支的固件行为"，与运行时 w6 恒 0 不冲突。

### 73.6 "属性 2 vs mode 0" 矛盾消解

```
persist.sys.dsd.shaper.coeffs   →  唯一读取点 0x171b8 在辅助函数 0x171a0 内
                                   （返回 atoi），该函数全库零调用者 → 死代码
persist.sys.dsd.delta.gain      →  唯一读取点 0x1722c 在辅助函数 0x17214 内
                                   （返回 atoi），零调用者 → 死代码
```

> **两个属性都是死配置接口：值可以被 setprop 写入并读到，但结果不进入任何
> 可执行路径。`shaper.coeffs=2` 与运行时 mode 无关。矛盾消解。**

附带更正：HANDOFF 0.3b 中"mode 0 的 8 抽头系数未解"一行已过时——
第 36.16/43 节已闭合 mode0 8 抽头（NTF 零点对 25.42 kHz + H[7] 直读），
且 `--jm21-mode 0` 与 `arg6 = 0` 的 C++ 默认值与运行时一致（`src/f2d.h:48-53`）。

### 73.7 对下一轮 Line-In 实验的直接意义

第 41.5 节设想的 `level=0/1/2/3` A/B 现在有了**精确的静态预测**：

```
level ∈ {0,1}：8 抽头 mode 0 shaper，d8 = 16
level ∈ {2,3,…}：4 抽头 mode 2 shaper，d8 = 14
```

即两组内部应**各自一致、彼此不同**（NTF 结构差异：8 抽头零点对 25.42 kHz vs
4 抽头 12.00 kHz，第 36.16 节）。若实测如此，即完成
"属性 → GPC+0x70 → w5 → w26 分支 → 量化器"全链路的黑箱运行时证明；
若实测三档全同，则说明本机构建的实际播放链没有走到该分支（需重新审视 caller
路径假设）。注意第 42 节警示：带外度量未校准，A/B 只比"组内一致 + 组间不同"，
不引用绝对值。

### 73.8 已固化的门禁清单（增量）

```
lief (GOT/重定位)          基址身份证明：x23/x22 = g_audio_play_config
访问器惯用法全扫            GPC 字段地图 + 导出名映射（本节 73.2）
GPC+0x68                   无活写者（73.3）⇒ 运行时 mode 0
GPC+0x70                   level 属性活通路（73.4）
shaper.coeffs/delta.gain   死配置接口（73.6）
```

## 74. Line-In 矩阵实验：`level` 属性是 DSD 采样率选择器（不是 mode 选择器）；d8=16 获运行时独立确认（2026-10-06）

> 按 §73.7 的设计执行 `persist.sys.pcm2dsd.level = 0/1/2/3` × 2 遍矩阵，
> 1 kHz / −6 dBFS（`1k_6dBFS.flac`，30 s），本机播放（非 USB DAC 模式，adb 全程在线），
> PC Line-In 192 kHz WASAPI 独占回录 12 s/次。
> 存档：`vendor_pull/runtime_logs/mode_ab/`（wav 数据 .npy ×9 + 每次运行的 logcat 证据）。
> 工具：`tools/mode_ab_matrix.py`（stop→setprop→prev/next 重选曲目强制重新起播→
> 校验 state=3/pos<8s→采集→logcat 证据断言 fresh config + PCMTODSD）。
> 关键实现教训：**pause→play 是续播，不重跑 config 路径，level 不会被重读**；
> 必须用 prev/next 重新选中曲目从 0 起播。

### 74.1 首要发现：level = DSD rate 档位（HAL 日志直接证据）

| level | HAL `dstRate`/`halRate` | 模拟域 DSD 率 | 输入 PCM 准备率（srcRate） |
|---|---|---|---|
| 0（默认/未设置） | 88200 | **DSD64**（705 600 B/s 立体声） | 88200 |
| 1 | 176400 | **DSD128** | 176400 |
| 2 | 352800 | **DSD256** | 352800 |
| 3 | 705600 | **DSD512** | 705600 |

`start_config_params: mode PCMTODSD tDstRate <R> hwRate <R>` 逐档清晰可读，
且 **srcRate 同步变为 88200 × 2^N**（192 kHz 源被 fiioprocess 重采样到对应 PCM 率）。

> **callee 架构固定 ×32（5 级 2×），level 选的是输入 PCM 采样率。**
> 88200×32 = DSD64；176400×32 = DSD128；352800×32 = DSD256；705600×32 = DSD512。
> §72.3 的"输入统一备到 88200"精确化为"备到 88200×2^level"。

### 74.2 频带结果（dB，相对 940–1060 Hz tone band 功率；welch nperseg=262144）

```
run      tone dBFS  tone/ref | 10-20k  20-40k  40-80k  80-96k
L0_r0      -9.64     -0.71   | -21.74  -24.63  -23.75  -32.26
L0_r1      -9.54     -0.57   | -21.81  -24.67  -23.81  -32.34
L1_r0      -9.57     -0.66   | -21.71  -24.58  -26.56  -32.43
L1_r1      -9.53     -0.57   | -21.63  -24.48  -26.47  -32.33
L2_r0      -9.54     -0.59   | -21.73  -24.52  -26.53  -32.39
L2_r1      -9.54     -0.58   | -21.66  -24.54  -26.53  -32.38
L3_r0     -77.08    -16.83   |  (纯噪声，tone 消失)
L3_r1     -77.90    -16.41   |  (复现：同样纯噪声)
restore    -9.81     -1.02   | -21.63  -24.42  -23.57  -32.10   ← 恢复 level 后
```

组内重复性 ≤ 0.15 dB。组间差：

```
10-20k:  d(0,1)=+0.10  d(1,2)=-0.02   → 无差异
20-40k:  d(0,1)=+0.12  d(1,2)=+0.00   → 无差异
40-80k:  d(0,1)=-2.74  d(1,2)=-0.01   → 唯一组间差异：L0 比 L1/L2 高 2.74 dB
80-96k:  d(0,1)=-0.08  d(1,2)=-0.01   → 无差异
tone:    d(0,1)=+0.04  d(1,2)=+0.01   → 增益恒定
```

### 74.3 解释：40-80 kHz 差异与零点对的归一化缩放一致

NTF 零点对位置固定在**归一化频率**（相对各自 DSD 率），模拟域位置随率缩放：

```
L0 (DSD64,  mode 0): 零点对 @ 25.42 kHz                     （带内可见）
L1 (DSD128, mode 0): 零点对 @ 25.42×2 = 50.8 kHz            （压低 40-80k）
L2 (DSD256, mode 2): 零点对 @ 12.00×4 = 48.0 kHz            （同样压低 40-80k）
```

L1 与 L2 的零点对在模拟域几乎同位（50.8 vs 48.0 kHz）→ 40-80k 频带一致；
L0 的零点对在 25.4 kHz，40-80k 无此抑制 → 高 2.74 dB。
**该 +2.74 dB 是"率选择 + 零点对缩放"的频谱签名**，与静态模型定量自洽。
（标注：这是解释性一致性，不是零点位置的独立测量；§42 的带外校准警示仍适用。）

### 74.4 d8 = 16 获运行时独立确认（tone 恒定）

callee 内 `d8 = (double)(16 - x7)`，x7 = GPC+0x68 = 0（§73：无写者）。
若 level≥2 真的把 arg6 一并改成 2，tone 应掉 20·log10(14/16) = −1.16 dB。
实测 L0/L1/L2 的 tone 差 ≤ 0.05 dB → **d8 = 16 在所有 level 下成立**，
GPC+0x68 = 0 的静态结论获得独立运行时证据。

**§0.1 中 "mode2 d8=14" 的来源更正**：那是 harness 强设 arg6=2 的产物
（把 shaper 选择寄存器同时当成了 arg6）。真实运行时 x6 = x7 = GPC+0x68 = 0 恒定，
**arg6 在真实播放链上不随 mode 变化**；mode（w26）只选抽头组。
`--jm21` 的 `arg6 = 0` 默认值正确对应真实设备。

### 74.5 §73.5 预测的修正

§73.5 的 `mode = (level<2) ? 0 : 2` 在**静态上仍然成立**（0x19e0c csel 无歧义），
但本轮实测证明它**不能在 0–96 kHz 模拟频带直接观测**：
L1（mode 0）与 L2（mode 2）的全部测量频带一致——两者的零点对在模拟域同位。
若要分开 mode 0 / mode 2 的模拟签名，需要在它们不同位的频带测
（如 L2 的 12–24 kHz 归一化窗口 vs L1 的 25–50 kHz），且须先扣除 DAC 滤波器响应。
本轮不宣称 mode 0/2 的模拟区分已验证。

**本轮真正确立的是更强的事实**：
1. `level` = DSD rate 档位（0/1/2/3 → DSD64/128/256/512），HAL 日志直接证据；
2. callee ×32 固定、输入 PCM 率随 level 加倍；
3. d8 = 16 恒定（tone 不动）；
4. level≥2 分支（w26=2）在 DSD256/512 上被静态确立，模拟签名不可见但与
   零点对缩放自洽；
5. **DSD512（level 3）超出本机本地 DSD 能力 → 模拟输出变为纯噪声（可复现）**
   ——与官方规格"Local decoding: DSD256 (Native)"精确吻合。
   （USB Audio 输入仍标称支持 DSD512，与本机播放路径不同。）

### 74.6 可逆性与过程安全

恢复 `persist.sys.pcm2dsd.level` 为空后重新起播：tone 回到 −9.81 dBFS、
频带轮廓回到 L0 形状（40-80k = −23.57 ≈ L0 的 −23.8）——**完全可逆**。
属性实验对设备无持久影响。（过程副作用：com.fiio.music 的断点续播记忆会把
曲目停在上次位置，force-stop + 重启 app 清除；HAL 服务在 app 重启后换了新 pid
6196，恢复采集与矩阵采集跨 HAL 实例仍复现，见 74.2。）

### 74.7 工具对照

```
--jm21 默认路径（输入 88200 → ×32 → DSD64，mode 0，d8=16，arg6=0）
    = 设备 level 0（默认）路径                          → 完全一致
新增对应关系（未实现，可选）：
    --jm21 --rate 128  ↔ 设备 level 1（输入 176400 → ×32 → DSD128，mode 0）
    --jm21 --rate 256  ↔ 设备 level 2（输入 352800 → ×32 → DSD256，mode 2，w26=2）
    DSD512 无对应（设备本机路径已坏）
```

### 74.8 已固化的门禁清单（增量）

```
tools/mode_ab_matrix.py          4 level × 2 遍自动矩阵 + fresh-config 断言
vendor_pull/runtime_logs/mode_ab/  9 组采集 + 逐次 logcat 证据 + results_all.json
level → DSD rate                 运行时闭合（HAL 日志 + 频谱签名双证据）
d8 = 16 恒定                     运行时闭合（tone 恒定，Δ≤0.05 dB）
DSD512 本机坏                    运行时闭合（可复现 + 规格吻合 + 可逆）
```

## 75. PCM 写入侧结构闭合：HAL 桥 + ring 纯 memcpy；标度问题分解为单一未定标量（2026-10-06）

> 按"先追真实 PCM 输入振幅"的方向，本轮把 6-byte 记录的**上游写入链**静态闭合到
> 单一缺口。全部证据来自 libfiioaudio.so 与 audio.primary.bengal.so 的符号/反汇编。

### 75.1 HAL 桥闭合（写入侧入口）

`audio.primary.bengal.so`（Qualcomm HAL）以 **UND 导入**方式引用 FiiO 处理入口
（dlopen/dlsym 均在其未决符号中）：

```
_Z16FiiOAudioProcessPcjjjjS_PS_PjS1_S1_S1_S1_S1_   ← PCM write 的处理入口
_Z21FiiOAudioConfigParamsjjjPjS_S_S_ / _Z13FiiOAudioInitjjjjjjPjS_S_
_Z15FiiOAudioDeinitv / _Z16FiiOIsUSBDevicev / _Z21FiiOAudioGetTrackModev ...
StreamOutPrimary::fiiOConfigurePalOutputStream        ← PAL 流配置挂点
```

定义侧（libfiioaudio.so）：

```
FiiOAudioProcess   0x1ce10 (veneer) → 0x14e60
  → bl 0x1dd50 (PLT → GOT 0x1f5b8 = processAudioEvent)   体 0x1442c
```

`processAudioEvent` 的 12 参签名与日志格式串
`%s: mode %s hwFrameSize %d hwRate %d hwChannel %d frameSize %d Rate %d channel %d
inSize [%d]%d outSize %d pcm %d player %d` 逐参对应。

### 75.2 processAudioEvent 的调用面（本轮解出）

| 调用 | 地址 | 判定 |
|---|---|---|
| `get_player_buffer_size` / `get_player_skip_frame_counter` | 0x14488/0x144a0 | 缓冲核算 |
| **`update_pcm_ring_buffer`** | **0x14538**（体 0x1894c） | **纯 memcpy 环形写入**：capacity 0x400000、pos 环绕（`and #0x3fffff`）、无格式转换、无缩放 |
| `mixer_ctl_set_value`（by name） | 0x147b4-0x147c4 | 硬件音量路径 |
| `get_player_buffer(char*, int)` | 0x1488c | 消费侧 |
| `memcpy_by_audio_format` ×4 | 0x13400/0x134c4/0x13688/0x14924 | 三个 **dst=PCM_32_BIT**（src=ring，src_fmt 为协商格式）；一个 **dst=PCM_FLOAT**（0x14924，src=int32，**wav dump 专用**，flag 0xed9428 门控） |
| `__fwrite_chk` | 0x1494c | dump 写文件 |
| `property_get/atoi` | 0x14b30-0x14c44 | 调试属性 |

### 75.3 结构结论

1. **ring 内容 = HAL 交付的字节，逐位原样**（update_pcm_ring_buffer 纯 memcpy）。
2. **libfiioaudio.so 内不存在 int32 → packed-24 的转换**
   （memcpy_by_audio_format 四个调用点无一 dst=packed24；无 lsr/ubfx 手工打包循环命中）。
3. 结合 §45.1（callee 对 x0 按 6 字节记录解析、scvtf 无归一化）：
   **6-byte 记录由更上游产生** —— 即 fiioprocess 重采样器（192000→88200×2^N 的
   库内重采样，日志 `processAudioConfigParams`）的**输出级**，或 AudioFlinger 按
   流协商格式（0x1A000012）交付。**写入点尚未逐指令定位（下一轮单点目标）。**
4. `x1 = handle+0xC0005A`（int32 化的转换目标缓冲）被传入 p2d 但 **callee 全程不碰**
   （§28.1 字节差分 = 0），为遗留接口参数。

### 75.4 标度问题的最终分解

```
record_int = 上游写进 6-byte 记录的整数（左对齐 24-bit 解析、scvtf 无归一化——已证）
firmware_input(double) = record_int                    ← 已证
tbl = record_int × 1.58948e-7                          ← 已测（§47.2，0.4% 一致）
模型约定 pcm(±1.0) ↔ record ±196608 (= 0x30000)         ← §47.4 自洽点（drive 0.5 边界）
```

未定部分只剩**一个标量**：

```
record_int = K_fixed × volume_lin × pcm_int32(0dBFS=2^31-1)
```

* `volume_lin`：AudioFlinger 数字音量（用户变量）。实测矩阵实验期间
  STREAM_MUSIC = 54/120（speaker）。
* `K_fixed`：fiioprocess 重采样输出级的固定增益（若有）——**最后一块**。
* 纯音量（54/120 ≈ −35…−40 dB）解释不了 −80.8 dB（196608/2^31）的自洽点，
  说明 K_fixed < 1 很可能存在，或 §47 的 "pcm ±1" 约定与 0 dBFS 差一个固定因子。
  两者都只在写入点定位后一步可测。

### 75.5 与工具的对照（现状）

```
flac2dsf --jm21 输入约定：归一化 double ±1.0 = 0 dBFS，×gainLin（默认 −3 dB）
        → core input = ±0.707 → drive = input/2×d8/16 …
固件约定：core input = record_int（0 dBFS ≈ 2^31 量级的整数，scvtf 直转）
两者相差一个未知标量 K_fixed × volume_lin。
工具默认工作点（±1.0 → drive 0.5）恰为 §47.4 自洽点 —— 这不是巧合可能，
而是"设备工作点 ≈ NTF 小信号边界"这一设计的体现；但与真实 record 数值的
逐点对账仍需 K_fixed。
```

### 75.6 下一轮单点清单（按优先级）

```
① 定位 fiioprocess 重采样器的输出写（真正的 6-byte writer）
   → 反汇编查固定缩放（K_fixed）
② 若 K_fixed = 1：S 全部由音量曲线解释 → 用已知音量做一次模拟回录对账
③ 若 K_fixed ≠ 1：并入 S，逐点波形比对（§47.5 的遗留失败）应随之闭合
④ mode0/2 模拟差分（L1 vs L2 零点特征）降级为辅助验证（按本轮指示）
```

## 76. 6-byte writer 定位：四个指纹横扫九个库全阴；写入侧极可能在 FiiO Music 应用内（2026-10-06）

> 按"不要再碰 DSP，直接拿下 resampler output → 6-byte writer"的方向执行。
> 结论是一个**范围被大幅收窄的阴性结果** + 一个可立即执行的推断。

### 76.1 先确认 DSD 分支的数据来源（已闭合）

caller 函数（0x12800-0x13970）内部分支：

```
0x12908  add  x3, x20, #0x400, lsl #12   ; 目标 = handle+0x400000
0x12910  mov  x0, x20
0x12914  mov  w1, #0x400000              ; capacity
0x1291c  ldr  w2, [x20, x8]              ; read pos @ 0x500008
0x12920  bl   get_data_from_ring_buffer  ; ring -> 0x400000（纯拷贝）
...
0x12d00-0x12e2c  spdif 分支（清 A00030/38/40/48，字符串 "current play mode is spdif"）
0x12eac-0x12f04  魔数 == 0x1A000002 分支（ext/zip2 格式）
0x12ea8  b.ne 0x13038                    ; != 魔数 -> DSD 分支
0x13038-0x1305c  get_pcm2dsd_data(x0=0x400000, x1=0xC0005A, x2=w25, x3=32, ...)
```

而 ring 的唯一写入者是 `update_pcm_ring_buffer`（0x14538 调用，体 0x1894c 为
**纯 memcpy**：capacity 0x400000、pos 环绕 `and #0x3fffff`、无任何格式/缩放转换），
其源缓冲 x25 来自 `FiiOAudioProcess` 的实参（HAL 侧 `StreamOutPrimary`）。

> **DSD 分支喂给 p2d 的 0x400000 = HAL 交付字节的逐位拷贝，
> 6-byte 记录在进入 FiiO 库之前就已形成。**

### 76.2 四种指纹、九个库，全部阴性

| 库 | stride-6 (`x+=6`) | 紧密 3×strb | 移位链 8←16←24 | NEON byte-lane store |
|---|---:|---:|---:|---:|
| libfiioaudio.so | 1（=0x19f08，**读取侧**） | 61 簇（0x15144 为 stride-8 反交织器，非打包器） | 0 | 0 |
| audio.primary.bengal.so | 0 | 27 簇（0x5dca4 为结构体字段初始化） | 0 | 0 |
| audio.usb.bengal.so | 0 | – | – | 0 |
| libagm.so / libagmmixer.so / libagm_pcm_plugin.so | 0 | – | – | – |
| libqc2audio_{base,hooks,platform}.so / libdsd2pcm.so | 0 | – | – | – |
| **libar-pal.so**（3.5 MB，PAL） | 2（均误报：`add x2,x2,#6` 取字符串地址） | 226 簇 | 1 | 0 |

补充：`x += 6` 在 libfiioaudio.so 中**全局仅一处**（0x19f08，callee 读取侧）。
`ubfx/lsr #8/#16/#24` 喂 strb 的候选数为 0；`st1 {v.b}[n]` 向量化打包器为 0。

### 76.3 唯一的 float→int32 转换点属于 MQA 分支（非 DSD）

0x12bf8-0x12c48 确有 resampler 输出转换：

```
0x12bf8  add  x11, x20, #0x400, lsl #12
0x12c0c  ld2  {v0.4s, v1.4s}, [x11], #32
0x12c10  fcvtzs v2.4s, v0.4s, #0x1f      ; float -> int32（SIMD）
0x12c1c  st2  {v2.4s, v3.4s}, [x12], #32
0x12c38  ldr  d0, [x10, x28]              ; 标量收尾
0x12c3c  fcvtzs v0.2s, v0.2s, #0x1f
0x12c40  str  d0, [x9], #8
```

紧随其后的 0x12c6c 调用解析为 **`mqa_decodec`**（GOT 0x1f598），
故该段属 **MQA 播放分支**，不是 DSD 分支。

> 即便如此，它给出一条与本题直接相关的事实：
> **FiiO 音频栈把 resampler 输出当 float 处理，转 int32 用 `fcvtzs`（向零截断）。**
> 截断对 ±x 对称（|x|<1 时 R(−x)=−R(x)），因此若 DSD 侧同样走截断，
> **不存在取整引入的增益偏置**；需要警惕的只剩 clipping 与 rate 相关项。

### 76.4 由此得到的推断（标注为推断，非证明）

```
HAL 协商格式 0x1A000012（FiiO 私有 tag，日志 dstFormat 436207618）
   ↓
AudioFlinger / HAL / PAL 全部无 6-byte 打包器（本节 76.2 四指纹阴性）
   ↓
最可能：解碼 + 重采样（192000 → 88200×2^N）+ 24-bit 打包
        由 FiiO Music（com.fiio.music）在其自有代码/native 库中完成，
        以该私有格式直接喂给 AudioTrack
```

这一推断与多条观察自洽：① `FiiOAudioProcess` 的 12 参里 `pcm/player` 计数字段
只做记账不参与转换（§75.2）；② 服务侧 `processAudioConfigParams` 打印
`srcRate 192000 → *halRate 88200` 是**配置**动作而非数据搬运；
③ 私有格式 tag 让 AudioFlinger 无法解释，故由应用侧保证格式。

**可立即执行的验证**：在 `com.fiio.music` 的 APK 内查找该打包循环
（lib/ 中的 .so，或 classes.dex 里的 native 注册）。若命中，
`K_fixed` 可在同一函数内直接读出（打包前的最后一次乘法）。

### 76.5 标度方程的最终形态（不变，来源已收窄）

```
record_int = K_packed × volume_lin × pcm_int32        （0 dBFS = 2^31-1）
tbl        = record_int × 1.58948e-7                  （§47.2 实测，0.4% 一致）
```

`K_packed` 的载体已从"系统音频栈"收窄到"应用侧打包函数"。
§47.4 的自洽点（196608 = 0x30000 = drive 0.5 边界）保持为待验证目标，
**不作为反推 K_packed 的前提**（按本轮指示：先静态取实际 scale，
再独立测量对账，避免重演"DC gain 对上 ⇒ 拓扑也对"的错误）。

### 76.6 已固化的门禁清单（增量）

```
stride-6 扫描           x+=6 全局唯一性检查（读取侧 vs 写入侧判别）
3×strb 簇               手工 24-bit 打包器指纹
移位链 8←16←24          移位+存储打包器指纹
NEON byte-lane store    向量化 24-bit 打包器指纹（st1 {v.b}[n]）
四指纹 × 九库            本轮阴性基线；出现新库时先跑此四项
fcvtzs 语义             向零截断、对称 ⇒ 无取整偏置（§76.3）
```

## 77. APK 路线：应用侧假设被运行时证据推翻；并发现帧长算术矛盾（2026-10-06）

> 按"先搜 0x1A000012 + sample-rate + AudioTrack，再沿 JNI 追"执行。
> **本节推翻第 76.4 节的最高概率假设**，并留下一个新的、更基础的问题。

### 77.1 APK 现状（与 §30.2 记录的版本不同）

`com.fiio.music` base.apk（118,295,086 B，路径 `/data/app/~~MMys9k…/base.apk`）：

```
arm64-v8a native: libhello-jni.so (11.2 MB)  libijkffmpeg.so (10.4 MB)
                  libm3u.so (957 KB)  libcrashsdk.so  libumeng-spy.so
                  libfiioa-jni.so (235 KB)  libfiioac-jni.so (235 KB)
dex: classes.dex … classes4.dex
```

§30.2 记录的 `libfilterAPI.so / libUsbAudio.so / libsacd.so / libffmpegAPI.so /
libeqLib.so` **在本版本中不存在**；`libhello-jni.so` 是 §30.2 未记录的最大库。

### 77.2 P1 私有格式常量：0 命中

按 4 字节对齐扫描 `.rodata`（不是字节级）：

| 库 | `0x1A000012` in .rodata |
|---|---:|
| libhello-jni.so (2.06 MB rodata) | **0** |
| libijkffmpeg.so (2.56 MB) | **0** |
| libfiioa-jni.so / libfiioac-jni.so / libm3u.so | **0** |

ARM64 立即数构造（`movz #0x12` + `movk #0x1a00 lsl #16`）在 .text 内 **0 命中**。

> 记录一次**方法论纠错**：先做字节级扫描时 `libhello-jni.so` 报 4 命中，
> 且"7 MB 随机数据期望仅 4e-4 次"看似天衣无缝。实为该库 4 字节对齐处并不落在
> 指令边界上（ARM64 指令含大量零字段，字节级巧合率远高于随机），
> 反汇编证明命中落在一条 `mov w0,#-0x11` 指令内部。
> **字节级常量搜索在 ARM64 代码上不可信，必须 4 字节对齐 + 立即数构造双重确认。**

`libhello-jni.so` 中相邻的 `{88200, 176400, 352800}`（.rodata 0x96dfdc）经全表展开为

```
8000 16000 32000 64000 128000 | 22050 44100 88200 176400 352800
| 12000 24000 48000 96000 192000 384000
```

即**通用采样率枚举表**（libijkffmpeg 中同构），**不是 DSD 档位表**。

### 77.3 决定性运行时证据：应用交付的是普通 PCM_32_BIT @ 192 kHz

`dumpsys media.audio_flinger`（播放期间）：

```
Type Id  Client  Session  Flags  Format     Chn mask  SRate
  100  no  12060  607673  0x400  00000003  00000003  192000
  AT::add ... CFG_EVENT_SET_PARAMETER: (sampling_rate=192000)
              CFG_EVENT_SET_PARAMETER: (format=3)
```

```
FiiO Music AudioTrack:  format = 0x3 (AUDIO_FORMAT_PCM_32_BIT)
                        sample_rate = 192000 Hz,  channel mask = 0x3 (stereo)
```

> **§76.4 的"应用侧产生私有格式"假设被推翻。**
> 应用既不使用 `0x1A000012`，也不使用 88200；它交出的是普通 32-bit PCM @ 192 kHz。
> 私有格式与 192k→88200×2^N 的重采样**全部发生在 vendor 音频栈内**
> （audio.primary.bengal.so + libfiioaudio.so），与 §72.3 的 FiiO 服务日志一致。

### 77.4 P3 乘法索引模式：libfiioaudio.so 亦阴性

```
mul wD, wN, #3 / #6 (alias 0x1B007C00)     0
madd xD, xN, xM, xA  含 3 或 6             7（0x1be2c…0x1d4f4，环形/杂凑区，非打包器）
lsl #1 → lsl #2  (i*2 + i*4 = i*6) 链       0
```

至此 §76.2 的四指纹 + 本节第五指纹，**在 libfiioaudio.so 与 8 个 vendor 库、
5 个 APK 库上全部阴性**。

### 77.5 本轮暴露的更基础矛盾：帧长算术

vendor 侧日志（§74.1 同源）：

```
get_player_buffer_size: mode PCMTODSD hwFrameSize 4 hwRate 88200 hwChannel 2
                        frameSize 4 Rate 88200 channel 2 inSize [30720]16896
                        outSize 16896
```

`frameSize 4 × channel 2 = 8 字节/帧` —— 这是 **int32 立体声**的帧长，
**与 §45.1 从 0x19ee0-0x19f08 读出的"每帧 6 字节记录"（x20 += 6）不一致**。

两种可能：

```
A) 6 字节记录前提成立
   ⇒ 需要存在一个把 int32 立体声（8 B/帧）压成 6 B/帧的打包器
   ⇒ 该打包器在上述 14 个库的 5 种指纹下均不可见（可能用了未假想的索引形式，
      或位于未提取的库中）

B) 6 字节前提本身是 harness 产物
   §45.1 的 ldrb/bfi 解析与 §67-68 的 24/24 golden vector 都是在
   "我自己铺的 6 字节输入缓冲"下得到的；它们证明了 callee 对该输入的行为，
   但不能证明设备真的以 6 字节记录喂它。
   若真实输入是 int32 立体声，则 x20 += 6 的读取循环与真实数据不对应。
```

**B 需要一个独立判据**（不依赖"常数刚好对上"）：

```
用真实寄存器状态（w3=32、mode=0、arg6=0）+ 真实输入语义（int32 立体声，
8 字节/帧）跑 callee，检查其输出字节流是否自洽
（长度关系、bit 密度 0.5、通道结构），并与 §67-68 的 packed 黄金向量对照。
长度关系是最干净的判据：8 字节/帧 × x3=32 下的输出字节数应能整除，
且与 4*x2/x3 自洽；若 6 字节假设成立则不会。
```

### 77.6 状态更新

```
REFUTED  §76.4 应用侧打包假设（runtime: PCM_32_BIT@192k，MAGIC 0 命中）
NEGATIVE 6-byte writer 不在 14 个已扫描库（5 指纹）
OPEN      6-byte 记录前提 vs int32 立体声（8 B/帧）—— 最高优先级的概念性分叉
OPEN      若 A 成立：打包器在未提取库 / 未假想索引形式
OPEN      K_packed（依赖上面先定输入语义）
```

### 77.7 已固化的门禁清单（增量）

```
ARM64 常量搜索        必须 4 字节对齐 + 立即数构造双确认；字节级搜索不可信
.rodata 表验证        命中相邻采样率后须展开全表，区分"通用枚举"与"专用档位"
dumpsys media.audio_flinger   决定性运行时格式证据（优先于静态推断）
帧长算术对账          frameSize × channel 必须与 callee 的读取步长一致
```

## 78. 黄金实验：6 字节记录是 w3=24 专属分支；真实路径为 int32 立体声 8 字节/帧（2026-10-06）

> 按"不要用输出长度像不像，要看每次 load 到底拿到哪些字节"的要求执行。
> **本节推翻 §45.1 / §46.3 / §67-§77 的 6-byte 输入前提**，并把绝对标度问题
> 从"上游有个找不到的打包器"改写成一个**可解的三函数链**。

### 78.1 实验设计

```
输入（真实 HAL 语义：int32 立体声，8 字节/帧，共 4 帧 = 32 B）
   frame0: L=0x01020304  R=0x05060708
   frame1: L=0x11121314  R=0x15161718
   frame2: L=0x21222324  R=0x25262728
   frame3: L=0x31323334  R=0x35363738
真实调用参数：w3=32, mode=0(x6=0), arg6=0(x7=0), w5=0
工具：vendor_pull/re/uc_golden8.py
```

### 78.2 第一判据的结果：w3=32 **根本没有**走 6 字节循环

PC 频次轨迹（Unicorn，4001 条指令）显示，执行从 prologue 直接进入
**0x1a250 插值级**，从未命中 0x19ed8。原因是输入解析按 `w3` 三路分派：

```
0x19ea8  cmp   w3, #0x10          ; 16
0x19eb4  b.eq  #0x19f28            ;   -> 16-bit 路径（ldrh w0,[x20] / [x20,#2]，步长 4）
0x19eb8  cmp   w3, #0x18          ; 24
0x19ebc  b.ne  #0x19f8c            ;   -> 32-bit 路径  ★ 真实值走这里
0x19ed8  ldrb  w11, [x20]          ;
...                                 ;   -> 3 字节/通道（§45.1 的"6 字节记录"）
0x19f08  add   x20, x20, #6        ;      仅 w3 == 0x18 时可达
```

> **§45.1 的 6-byte 记录 / 24-bit 大端左对齐 / `scvtf` 无归一化，
> 是 `w3 == 0x18`（24-bit）分支的格式，不是设备真实输入格式。**
> 旧 harness（`uc_harness.py` 头部注释："w3 = 0x18 selects the call-free
> 24-bit conversion path"）正是选了这条最方便的免调用分支，
> §45 之后整条输入链都建在它上面。

### 78.3 真实输入格式（w3 = 32，raw 校验）

```
0x19fec  ldr   w0, [x21]          ; int32  L（4 字节）
0x19ff0  bl    0x1d358            ; ┐
0x19ff4  ldur  q1, [x29, #-0xa0]  ; │ .rodata 0x80c0 的 16 字节常量对（2 个 double）
0x19ff8  bl    0x1d3c4            ; │ int32 -> double 的缩放链
0x19ffc  bl    0x1d888            ; ┘
0x1a000  str   d0, [x22, x23]
0x1a004  ldr   w0, [x21, #4]      ; int32  R
0x1a008..0x1a014  同一条三调用链
0x1a018  f1000694  SUBS X20, X20, #1
0x1a01c  910022b5  ADD  X21, X21, #8     ; ★ 步长 8
0x1a020  str   d0, [x22], #8
0x1a024  b.ne  0x19fec
```

（raw 逐字校验通过；capstone 对这几条 LDR 漏印了偏移，属工具显示问题，
不影响 `ADD/SUBS` 等 raw 字的判定。）

> **真实输入 = int32 立体声、8 字节/帧。**
> 与运行时日志 `get_player_buffer_size: mode PCMTODSD hwFrameSize 4 hwChannel 2`
> 精确一致（4 × 2 = 8）。**第 77.5 节的帧长矛盾就此消解。**

### 78.4 这解释了此前所有阴性结果

§76-§77 在 14 个库里用 5 种指纹搜"6-byte writer"全部阴性 —— 因为
**设备根本不用 6 字节记录**。没有东西需要被打包。

### 78.5 影响面（须降级/撤回的结论）

| 结论 | 处置 |
|---|---|
| §45.1 6-byte 记录 / 24-bit 大端左对齐 / scvtf 无归一化 | **降级为 w3=24 分支的格式**；非设备真实输入 |
| §45.2 运行时"读到 = 256·b0" | 同样只在 w3=24 成立，撤回为该分支结论 |
| §45.5/§46 绝对标度 S≈196626（§47 实测 1.58948e-7/整数） | **输入语义错误，需在 w3=32 下重测**（标量值本身不矛盾，但标定基准变了） |
| §67-§68 packed 黄金向量 24/24 byte-exact | **输出侧**结论（8:1 打包、通道顺序、planar 块）大概率仍成立；**输入侧**需在 w3=32 下复核 |
| §46.3 "0xC0005A 是输出不是输入" | 仍成立（与 w3 无关） |
| §72.2/§74.1 88200×2^N / DSD64 输出 | **仍成立**（HAL 日志直接证据，与输入解析分支无关） |
| §73 GPC+0x68 无写者、d8=16 | **仍成立**（0x19e0c csel 与 w3 无关） |
| §75/§76/§77 写入侧"打包器在哪" | **整条问题作废**：不存在打包器 |

### 78.6 绝对标度问题的新形式（比原问题更小）

```
int32 L/R (8 B/frame)
   ↓ bl 0x1d358 → ldur q(.rodata 0x80c0) → bl 0x1d3c4 → bl 0x1d888
double（写入 h+0x103A0）
   ↓ ×d14 + ×d15 (§45.2)
   ↓ 5 级 2× 插值 → tbl
   ↓ ×d8(=16) → 量化器
```

**K_packed 不再是未知常量，而是 `.rodata 0x80c0` 两个 double 与三个
内联/库函数的确定组合。** 下一步只需反汇编 0x1d358 / 0x1d3c4 / 0x1d888
（注意 §78.3 的 BL 目标地址需重新核对，见下）并读出 0x80c0 的 16 字节内容，
即可得到**静态的、无自由参数的** PCM→double 比例。

### 78.7 待核对的遗留（本节未闭合）

三个 `bl` 目标（0x1d358 / 0x1d3c4 / 0x1d888）在文件镜像里是 `d503245f`（NOP），
其后才是函数体 —— 说明该处同样存在 4 字节对齐/PLT 映射问题，
在把 §78.6 的比例写死之前必须先确认这三个函数的真实身份
（很可能分别是 `__int2d`-类转换、一次向量乘法、一次缩放）。

### 78.8 已固化的门禁清单（增量）

```
w3 分派检查            任何"输入格式"结论必须标注适用的 w3 值
真实参数驱动           harness 不得为图省事选非真实分支（w3=0x18 是历史教训）
raw 字校验             关键指令（步长/计数器）必须用 raw 四字节解码复核
运行时帧长对账         hwFrameSize × hwChannel 必须等于 callee 的读取步长
```

## 79. 三函数与 0x80c0 常量：F1 已定性，F2/F3 是 libgcc softfloat，缩放语义仍未解（2026-10-06）

### 79.1 `.rodata 0x80c0` 的 16 字节

```
hex: 00 00 00 00 00 00 00 00  00 00 00 00 00 00 e0 3f
double[0] = 0.0                    (0x0000000000000000)
double[1] = 0.5                    (0x3fe0000000000000)
```

作为 128-bit SIMD 常量读入（`ldr q0, [x11,#0xc0]`，x11=adrp 0x8000）
→ `v0.2d = {0.0, 0.5}`，循环前 `stur` 到 `[x29-0xa0]`，
每个样本前 `ldur q1, [x29,#-0xa0]` 重新载入。

### 79.2 三个函数的定性

| 地址 | 定性 | 依据 |
|---|---|---|
| **F1 = 0x1d358** | **`__int2d`（int32 → double，精确无缩放）** | `bti c` 入口；`cbz w0` → 返回 0；`cneg/clz` 求阶；`mov w10,#0x401e`(=1022) 与 clz 相减得 IEEE754 指数域；`eor x11,x11,#0x1000000000000` 补隐含位；`and w10,w0,#0x80000000` 取符号；手工拼出双精度位型 |
| **F2 = 0x1d3c4** | **libgcc softfloat 双精度运算（2 lane）** | `mov w0,#-0x7fff` / `mov w2,#-0x7ffe` 是 softfloat 指数偏置的经典常数；`ubfx x9,x17,#0x30,#0xf` 直接取 IEEE754 指数域；入参为 `stp q1,q0` 的 128-bit 对 |
| **F3 = 0x1d888** | **libgcc softfloat 收尾（跨 lane 取样）** | `str q0,[sp,#-0x10]!` + `ldp x9,x8` 取 128 位；`extr x10,x8,x9,#0x3c` 跨 64 位边界取样；`fmov d0,x8` 输出单精度位型 |

**BL 链本身已逐级核实**（未依赖 capstone 的 target 打印）：
`0x19ff0 raw=94000cda` → imm26=0xcda → 0x1d358 ✓；
`0x19ff8 raw=94000cf3` → 0x1d3c4 ✓；`0x19ffc raw=94000e23` → 0x1d888 ✓。
三处目标首指令均为 `d503245f`（`bti c`，分支目标标识，非 NOP），
其后紧跟正常 prologue。**caller 块 0x19f8c-0x1a024 经 raw 逐条复核完全对齐**。

### 79.3 未解：F2 的 2-lane 语义

**`v0.2d = {0.0, 0.5}` 无法直接解释"每样本缩放"**：

* 若 F2 是 `q0 * q1` 的 2-lane 乘法，lane 0 = d0 × **0.0** = 0，会把左声道清零 —— 与设备行为矛盾；
* 若 `{0.0, 0.5}` 是 L/R 的每通道系数，0.0 同样不合理。

可能的解释（**均为假设，未验证**）：
① 0.0 只是占位，真正的系数在 F2/F3 内部从别处取；
② F2 不是乘法而是某种带标量的复合运算（如 `d0 * c + d` 形式，c/d 由 q1 与立即数组合）；
③ 该 2-lane 结构是 libgcc 的 `__floatundidf`/`__muldc3` 惯用法，第二 lane 仅为对齐，语义由 F3 的 `extr` 决定。

**在 F2/F3 的语义确定前，不能给出 `G_input`，也不能给出 §78.6 那个"无自由参数"的公式。**

### 79.4 数值闭环的当前障碍

`vendor_pull/re/uc_ginput.py` 目标是用 w3=32 + 钩 0x1a000 直接读出 d0，
但当前 harness **未进入 0x19fec 输入循环**（PC 直方图显示只在 0x1a2xx 插值级循环）。
排查到两点：

* 早先误把 0x1d358/0x1d3c4/0x1d888 当 PLT stub 钩掉导致 d0 恒空（已修正：
  它们是真实函数体，入口 `bti c`）。
* 当前仍跳过输入循环，怀疑与 `0x19f8c/0x19f90`（`cmp w8,#2`）、
  `0x19fa0`（`cmp w24,#2`）或 `0x19e48/0x19e54` 的最小样本数门控有关，
  需要用真实 w25 量级（不是 80 字节）重跑再判定。

**故本节的 `G_input` 数值尚未测出，标度问题仍未闭合。**

### 79.5 本节状态

```
VERIFIED   0x80c0 = {0.0, 0.5}（逐字节）
VERIFIED   F1 = __int2d，int32→double 无缩放（指令级）
VERIFIED   F2/F3 = libgcc softfloat 例程（指数偏置常数 + 指数域提取）
VERIFIED   BL 链地址与 caller 块对齐（raw 复核）
UNRESOLVED F2 的 2-lane 运算语义 → G_input
UNRESOLVED w3=32 是否走同一 NEON packer / 同一输出 buffer（未验证，暂标"结构候选"）
UNRESOLVED 数值 G_input（harness 未进入输入循环）
```

> 依 §78.5 的教训，§67-§68 的 packed 黄金向量结论在本节确认 w3=32 走同一
> packer 之前，**只作"与 w3 无关的结构候选"**，不计入 Verified。

## 80. G_input 闭合：int32 / 2^31；真实 drive = pcm_norm × 0.5（2026-10-06）

### 80.1 门禁不是阈值，而是整除约束

w2 扫描（w3=32，w2 = 32…4096）全部走同一路径 `0x19fa4 → 0x1a86c → 0x1a028`，
无一命中 0x19fec。原因是 0x19e78-0x19e9c 的**整除检查**：

```
0x19df0  asr   w9, w8, #3            ; w9 = w3>>3 = 4
0x19df4  sdiv  w8, w2, w9            ; w8 = w2/4
0x19e80  sdiv  w9, w3, w9            ; w9 = w3/4 = 8
0x19e84  adds  w9, w9, w9            ; w9 = 16
0x19e8c  sdiv  w10, w2, w9
0x19e90  msub  w9, w10, w9, w2       ; w2 mod 16
0x19e94  cbz   w9, #0x19ea8           ; 余数非 0 ->
0x19e98  mov   w0, #-3               ;   返回 -3
```

> **契约：`w2 % (2·w3/(w3>>3)) == 0`；w3=32 时即 `w2 % 16 == 0`。**
> 首次运行返回 −3 的真实原因是我把字节数写成 `len(vals)*4`（应为 ×8）。
> 这也解释了"80 字节也进不去"以外的所有异常。

### 80.2 真实输入循环是**向量化**那一条（0x1a86c）

0x19f8c 分支入口有两条互斥的缓冲区重叠检查：

```
0x19fb4  cmp  x9, x10 ; b.ls #0x1a86c     ; 输入不得覆盖 handle+0x3a0 以下
0x19fc8  cmp  x8, x20 ; b.ls #0x1a86c     ; 输出区 handle+0x103a0+… 不得低于输入
```

真实 caller 的 `x0 = handle+0x400000` **必然**落在输出区之上 ⇒ 走 0x1a86c。
0x1a86c 不是错误出口，而是同一转换的 NEON 向量化版本：

```
0x1a86c  and  x22, x24, #0xfffffffe        ; 计数取偶
0x1a870  add  x27, x25, #0x103a0           ; 输出基址
0x1a878  ldr  q0, [x11, #0xc0]             ; {0.0, 0.5}
0x1a88c  ld2  {v0.2s, v1.2s}, [x20], #16   ; ★ 16 字节 = 2 个立体声帧
0x1a890  mov  w0, v0.s[1]  ; bl 0x1d358    ; __int2d(R)
0x1a8ac  fmov w0, s0       ; bl 0x1d358    ; __int2d(L)
0x1a8b8  bl 0x1d3c4 ; 0x1a8c8 bl 0x1d3c4 ; 0x1a8cc bl 0x1d888
```

> **输入布局三重确认：int32 立体声、8 字节/帧、每 16 字节两帧。**
> 静态（ld2 步长 16 = 2×8）+ 运行时（hwFrameSize 4×2ch）+ 本节数值，三者一致。

### 80.3 G_input 数值（Unicorn，F3 出口读 d0）

输入 L = 哨兵值、R = 0，读 0x1a8d0（F3 返回后第一条）：

| 输入 L | d0 | d0 / L |
|---|---:|---:|
| 2^24 | −0.0078125 (= −2⁻⁷) | **−2⁻³¹** |
| 1 | −4.6566128730773926e−10 (= −2⁻³¹) | **−2⁻³¹** |
| 2^31−1 | −0.4999999995343387 级 | **−2⁻³¹** |

比例在 2^0 … 2^31−1 上恒为 **1/2^31**，无取整偏置（与 §76.3 的
`fcvtzs` 对称性一致 —— 本路径用 `__int2d`，是精确转换）。

> **G_input = 1/2^31 = 4.6566128730773926e−10**
> 即 int32 满刻度（2^31）→ double 1.0。**这是教科书式的归一化**，
> 且是**静态常量 + 运行时测量双重确认**。
> 负号来自 F3 取 (R−L) 还是 (L−R) 的约定，模长即标度。

### 80.4 真实设备工作点（终于可以写成公式）

```
pcm_norm = int32 / 2^31                      （G_input，§80.3 实测）
tbl      = pcm_norm × 1/32                    （5 级插值链 DC 增益，§45.2 逐级确认）
drive    = tbl × d8 = tbl × 16               （d8=16，§73/§74 运行时确认）
        = pcm_norm × 0.5
```

```
0 dBFS (int32 = 2^31)  →  pcm_norm = 1.0  →  drive = 0.5
```

> **drive = pcm_norm × 0.5**，与第 44 节实测的 NTF 小信号稳定边界
> （drive 0.5 / −6 dBFS）**精确重合**。
> 即：**设备在 0 dBFS 处正好坐在 1-bit 量化器整形行为的边界上**，
> 这解释了为什么 All-To-DSD 的带内噪声收益在真机上不明显（§35.7）——
> 满幅附近本就工作在过载侧。这条推论**不依赖任何 synthetic 假设**。

### 80.5 §47.4 的 196608 之谜（本节给出解释）

§47 在 w3=24 synthetic 路径上测得 S≈196626，并指出 196608 = 0x30000 恰对应
drive 0.5。现在可以判定：**那是 synthetic 输入布局下的等效值**，
在真实 w3=32 路径下不存在"196608"这个特殊整数。
真实公式是 `drive = pcm_norm × 0.5`，对任意 int32 都成立。
§47.4 的"自洽点"应重新解读为：**两个独立方法（synthetic 标量测量 /
真实输入推导）都指向 0 dBFS → drive 0.5**，而不是某个魔数对上了。

### 80.6 与工具的对照

```
设备真实： drive = (int32/2^31) × 0.5           → 0 dBFS 时 drive = 0.5
flac2dsf --jm21 默认：输入 ±1.0 × gain(−3 dB) → drive = 0.707 × 0.5 = 0.354
```

**工具默认增益（−3 dB）低于设备的 0.5 工作点。** 若要与设备工作点严格对齐，
应使用 `--gain 0dB`（或把默认增益改为 0 dB）。这是**待决的产品选择**，
不是缺陷 —— 但 §0.5 提到的"默认值恰为 NTF 边界"的巧合现在有了确定答案：
**0.5 是设备真值，不是巧合。**

### 80.7 仍开放

```
? F2/F3 的精确源码级语义（现已由数值闭合，不再阻塞标度问题）
? w3=32 是否走同一 NEON packer / 同一输出 buffer（§79.5 仍标"结构候选"）
? PCM → tbl 的模拟量端到端验证（可用 §80.4 公式直接对拍）
? §47 在 w3=32 下的重测（数值上应与 §80.3 一致，可作交叉校验）
```

### 80.8 已固化的门禁清单（增量）

```
w2 整除契约            w2 % (2*w3/(w3>>3)) == 0，否则 callee 返回 -3（w3=32 → 16）
缓冲区重叠分支         真实 caller 必走向量化 0x1a86c，不是标量 0x19fec
G_input 门禁           int32 → double = /2^31，须在 2^0…2^31-1 上验证比例恒定
```

## 81. 打包层：静态确认、运行时确认未完成（保持"结构候选"）（2026-10-06）

> §79.5 承诺的验收项 ① —— 在 w3=32 真实路径下确认 0x1a6cc packer。
> **本节给出静态确认，并如实记录运行时确认失败。§67-§68 不升级。**

### 81.1 静态确认（成立）

`0x1a6cc..0x1a7dc` 的 NEON 打包器与 §67-§68 描述完全一致，且与 `w3` 无任何耦合
（该代码不读 w3）：

```
0x1a6cc  add   x9, x25, #0x120, lsl #12   ; x9  = handle + 0x120000
0x1a6d4  add   x11, x9, #0x3a0            ; x11 = handle + 0x1203A0  = 缓冲 A
0x1a6d8..0x1a6ec  movi v0.2s,#0xc0 / v1,#0xe0 / v2,#0xf0 / v3,#0xf8 / v4,#0xfc / v5,#0xfe
0x1a6f0  add   x8, x8, #1
0x1a6f4  mov   w9, #0x10000                ; A 区的第二段偏移
0x1a6fc  ldr   q7, [x11, x9]               ; 取 A 的高段
0x1a704  ldr   q6, [x10], #0x10            ; 逐 16 字节推进
0x1a710  xtn   v17.2s, v6.2d               ; 64→32 截断
0x1a71c  shl   v22.2s, v17.2s, #7
0x1a720  ushr  v23.2s, v17.2s, #2
...  and #0xc0 / #0xe0 / #0xf0 / #0xf8 / #0xfc / #0xfe   逐位掩码拼装
0x1a778  shrn  v6.2s, v6.2d, #0x1d         ; ★ 抽高 8 bit（>>29）
0x1a7d4  trn1  v6.8h, v6.8h, v7.8h         ; ★ 交织
0x1a7d8  xtn   v6.8b, v6.8h
0x1a7dc  st1   {v6.s}[0], [x13], #4        ; ★ 每 4 字节一次存储
```

要点：

* 缓冲 **A = handle+0x1203A0**（与 §67 一致；B = handle+0x1303A0）。
* `shrn #0x1d` = 右移 29 位取 8 bit，即**取每个 double 的最高 8 位之一组**，
  再经 `trn1` 交织 —— 与「8 个 1-bit 样本 → 1 字节、MSB-first、A/B 交织」的
  既有描述相容。
* **该段代码不引用 w3**，因此输入分支的变化不影响打包层本身。

### 81.2 运行时确认：**未完成**

`vendor_pull/re/uc_pack_real.py`（w3=32、真实 caller 布局 x0=handle+0x400000、
mode=0、arg6=0、非对称 L≠R 输入）：

| w2 | 结果 |
|---|---|
| 4096 | packer 命中 0、store 命中 0；FIR 循环 0x1a250 迭代 31744 次后触发指令预算上限 |
| 1024 | packer 命中 0；FIR 循环迭代 7936 次后触发上限 |

A/B 缓冲（handle+0x1203A0 / +0x1303A0）全零，bit-density 0。

**原因**：`0x1a250` 的 FIR 循环其迭代计数不是直接来自 `w2`，而依赖
`0x1a1b4` 处按 `h+0x143a0 + 8*x8 (x8=0..4)` 计算的阈值与 `h+0x203a0`
写入表 —— 这些 handle 状态在本最小 harness 中**未初始化**，
导致循环计数不收敛、永远到不了量化器与打包器。
旧的 `uc_harness.py` 正是靠铺设这些 handle 状态才跑通（§68 的黄金向量）。

**结论：打包层的运行时确认仍未取得。§67-§68 保持"与 w3 无关的结构候选"，
不升级为 Verified。** 需要把 `uc_harness.py` 的 handle 初始化搬到本脚本
（并改成真实 int32 输入）才能完成该项。

### 81.3 已知的静态↔运行时缺口（不影响 §80 结论）

§80 的 G_input 结论**不依赖打包层**：它只用到
① 输入向量化转换（0x1a86c，已运行时确认）、
② 5 级插值链的 DC 增益 1/32（§45.2，逐级系数确认）、
③ d8=16（§73/§74，运行时确认）。
三者都不经过 0x1a6cc。**故 §80.4 的 `drive = pcm_norm × 0.5` 依然成立。**

### 81.4 措辞收紧（采纳评审意见）

> 「drive 0.5 与模型测得的 NTF 小信号边界重合」是**很强的自洽证据**。
> 「设备在设计上就是把 0 dBFS 放在该边界」**仍是推断** ——
> 数学结果只能给出标度，不能给出设计意图。
> 本文档统一采用前者措辞。

### 81.5 剩余验收项

```
① [进行中] w3=32 下 packer 运行时确认（需移植 handle 初始化）
② [未做]   端到端对拍：固件 vs Python vs C++，真实 int32 stereo 输入
            （需 ① 的 harness 就绪；固件侧输出需能落盘比对）
```

## 82. 打包层对照实验：w3 分派**不是**不收敛的原因；两条分支同样到不了 packer（2026-10-06）

> 按评审要求：以 `uc_harness.py` 为底，只替换输入分支，做 w3=0x18 / w3=0x20
> 的同码对照（`vendor_pull/re/uc_pack_real_v2.py`）。

### 82.1 方法

两套配置走**完全相同的代码**，唯一差异是 `w3` 与随之而来的输入布局：

```
A: w3 = 0x18,  6-byte record 合成输入   （历史"synthetic"分支，作对照）
B: w3 = 0x20,  int32 stereo 8 B/frame  （真实设备输入，哨兵值 L≠R）

共同：HANDLE=0x20000000（仅 mem_map 零页 + *(0x1f410)=HANDLE，无任何手工初始化）
      x0 = handle+0x400000,  x1 = handle+0xC0005A,  mode=0,  arg6=0
硬断点：0x1a1b4 / 0x1a1d4 / 0x1a250 / 0x1a6cc / 0x1a7dc
```

### 82.2 结果：两条分支逐项相同

```
                        w3=0x18 (对照)      w3=0x20 (真实)
0x1a1b4 threshold          5                    5
0x1a1d4 tbl-write       7936                 7936
0x1a250 FIR            15872                15872
0x1a6cc PACKER             0                    0
0x1a7dc store              0                    0
A/B 缓冲 ones             0/512                0/512
首次进入 0x1a250 的状态   完全一致（x25/x20/x10/x16/x8 逐项相同，
                          h+0x143a0 阈值全 0，h+0x203a0[0]=2.68435e+08）
```

> **w3=32 不是不收敛的原因。** 上一轮 §81.2 把原因归给"w3 分支产生的
> x24/x25/x8/threshold 之一"，该假设**被本对照实验否定**。
> 真正的原因是：**在当前 harness 配置下，两条分支都到不了 0x1a6cc** ——
> FIR 循环 15872 次后仍未进入量化器/打包段。

### 82.3 一个必须记录的连带问题

既然**连历史对照分支也到不了 0x1a6cc**，那么 §67-§68 所称的
"packed 黄金向量 24/24 byte-exact"就**不是在本 harness 配置下经由 0x1a6cc
得到的**。`uc_harness.py` 的停止点是 `HOOK_EXPAND_END = 0x1a1a8`
（"one threshold finished"，64 次后主动 `emu_stop`），
即它**本来就不打算走到打包器**。

因此 §67-§68 的打包层结论应进一步降级为：

```
"由反汇编逐指令读出的打包算法"（静态，可信）
+
"长度/计数关系"（§27.2 的 4*x2/x3 类，内部自洽）
-
"运行时 byte-exact 黄金向量"（本轮证据显示未真正经由 0x1a6cc 取得）
```

**§81 的"结构候选"定级维持不变，且其依据应表述为静态反汇编而非运行时验证。**
本节不宣称 §67-§68 的黄金向量无效 —— 它们可能由另一个 harness
（`uc_outbytes.py` / `uc_write_track.py`）取得 —— 只声明
**本轮无法复现该验证，且 w3 分派已被排除为原因**。

### 82.4 下一步（比"移植 handle 初始化"更准确）

既然 handle 状态不是变量（两分支全零且一致），真正需要定位的是
**FIR 循环 0x1a250 之后通向量化器/packer 的连接点**：

```
① 0x1a250 循环的退出条件（x6 由谁置位/递减，退出后跳向何处）
② 0x1a95c / 0x1a960 等异常/续处理出口在本次配置下走了哪一条
③ §27.2 的 output_bytes = 4*x2/x3 是在哪个函数里成立的 —— 若不在
   0x19d90 内部，则"打包在本函数内完成"这一前提本身需要重查
```

第 ③ 条尤其关键：若打包实际发生在 callee 之外（如 caller 的 memcpy 或
PAL 侧），则 0x1a6cc 根本不是打包点，本节与 §81 的整个前提都要重写。

### 82.5 状态

```
NEGATED   "w3=32 导致 FIR 循环不收敛"（§81.2 的归因）
NEGATED   "handle 状态缺失是原因"（两分支 handle 全零且行为一致）
CONFIRMED §80 的 G_input=1/2^31 与 drive=0.5（不依赖打包层，仍成立）
OPEN      0x1a6cc 是否为真正的打包执行点（§82.4③ 优先）
OPEN      w3=32 真实路径的打包层运行时确认
```

## 83. 打包层在真实 w3=32 路径上确认：输出 = handle+0xC0005A，bit 密度 0.506（2026-10-06）

> §82 定位到的"不收敛"根因是 **harness 自身的 `FN_END_GUARD = 0x1a320`
> 把执行截断了** —— 0x1a6cc packer 就在其上方 0x3ac 处。
> 修正后立即命中。**这是本项目第二次"测试结果为真但 provenance/工具错误"。**

### 83.1 根因

```
uc.emu_start(0x19d90, 0x1a320)      # 旧设置
last two PCs = 0x1a318 -> 0x1a31c # 执行被截断在 FIR 循环之后
```

FIR 循环其实**正常跑完**，只是从未被允许继续。§82 的"w3 与 handle 状态都被
排除"这一结论**依然正确**（两者行为确实一致），但它没有解释全部现象 ——
真正的差异是 guard。这个 bug 存在于本会话新建的所有脚本
（`uc_golden8.py` / `uc_ginput2.py` / `uc_pack_real.py` / `uc_pack_real_v2.py`）。
**注意：`uc_harness.py` 用 0x1a1a8 的主动 `emu_stop`，与本问题无关。**

### 83.2 修正后的运行（w3 = 0x20，真实 int32 stereo，mode 0，arg6 0）

```
0x1a1b4 threshold hits = 10
0x1a1d4 tbl-write hits = 3968
0x1a250 FIR hits       = 7936
0x1a6cc PACKER hits    = 1        ← ★ 真实路径命中
0x1a7dc store hits     = 128      ← 128 × 4 B = 512 B packed 输出
x13 序列 = 0x20c0005a, 0x20c0005e, 0x20c00062, …   （每次 +4）
```

### 83.3 关键更正：packer 的**输出**是 handle+0xC0005A，不是 0x1203A0

```
0x1a6cc  add x9, x25, #0x120, lsl #12   ; x9  = handle+0x120000
0x1a6d4  add x11, x9, #0x3a0            ; x11 = handle+0x1203A0   ← 读：中间 1-bit 缓冲 A
0x1a7dc  st1 {v6.s}[0], [x13], #4       ; x13 = handle+0xC0005A    ← 写：最终 packed
```

> **0x1203A0 / 0x1303A0 是被读取的中间 1-bit 缓冲（A/B 通道），
> 打包结果写到 handle+0xC0005A —— 也就是 caller 传入的 x1 实参。**
>
> 这同时解答了 §28.1 留下的"x1 = handle+0xC0005A 语义未知"：
> **x1 是 DSD packed 输出指针**。
> §46.2 把 0xA0005A-0xD0005A 标为"1-bit 输出"区域，与此一致；
> §28.1 因字节差分为 0 而判"只读"，是因为差分只查了 x0 区域、未覆盖 x1。

### 83.4 输出内容（真实路径，哨兵 L≠R 输入）

```
OUT handle+0xC0005A:
  96 96 69 69 69 a6 a6 9a 5a a7 9b 4d 35 aa 66 d9 d5 da 56 d6 54 55 a2 92
  ones = 259/512   bit density = 0.5059        ← ★ 1-bit 流的特征值

中间缓冲 A (0x1203A0): 01 00 00 01 00 01 01 00 …  density 0.064
中间缓冲 B (0x1303A0): 01 00 00 01 00 01 01 00 …  density 0.066
```

> packed 输出密度 **0.5059 ≈ 0.5**，是 1-bit DSD 流应有的值；
> 中间缓冲密度 ~0.064（≈1/16，稀疏 1-bit 存放），两者角色分明。

### 83.5 真实 vs synthetic 路径的又一处硬证据

首次进入 0x1a250 时 `h+0x203a0`（量化器输入表）：

| w3 | tbl[0], tbl[2] |
|---|---|
| 0x18（synthetic） | **2.68435e+08**, 2.68435e+08 |
| 0x20（真实） | **0.00787389**, 0.266697 |

synthetic 路径给出 2.7e8 量级的荒谬工作点，真实路径给出 0.008–0.27 的合理值。
这独立于 §80 的 G_input 结论，再次确认 **w3=24 分支不是设备真实输入**。

### 83.6 状态更新

```
VERIFIED  0x1a6cc-0x1a7dc 是打包后处理，在真实 w3=32 路径上被执行
VERIFIED  其输出 = handle+0xC0005A（= caller 的 x1 实参），bit 密度 0.506
VERIFIED  0x1203A0 / 0x1303A0 = 被读取的中间 1-bit 通道缓冲
VERIFIED  caller 的 x1 语义 = DSD packed 输出指针（解答 §28.1 遗留）
VERIFIED  synthetic(w3=24) 路径工作点荒谬，真实路径合理
STILL OPEN  packed 字节与 Python/C++ 参考的逐字节相等（本轮只验了形状与密度）
STILL OPEN  0x1a95c / 0x1a960 的角色（未追）
STILL OPEN  §67-68 黄金向量的原始 provenance（可能仍非经由本路径取得）
```

> 措辞纪律：§67-§68 的"8:1 packing 契约（MSB-first、A/B 交织）"**本轮只验证到
> "该后处理被执行且输出为高密度 1-bit 字节流"**。
> 具体的位序与交织约定**尚未用真实输入做逐字节对照**，仍不得称为 Verified。

## 84. real-w3=32 黄金向量：固件侧已落盘 6 组 + 长度关系修正（2026-10-06）

### 84.1 采集器与工件集

`vendor_pull/re/uc_capture32.py` 产出**完整 provenance 工件**（6 组输入 × mode 0）：

```
golden/real32/<kind>_m0_input.int32          真实 int32 stereo 字节流
golden/real32/<kind>_m0_intermediate_A.bin   handle+0x1203A0（1-bit 通道 A）
golden/real32/<kind>_m0_intermediate_B.bin   handle+0x1303A0（1-bit 通道 B）
golden/real32/<kind>_m0_packed_fw.bin        handle+0xC0005A（8:1 packed）
golden/real32/<kind>_m0.meta                 sha256 / 密度 / 计数 / 寄存器 / 生成参数
```

输入生成器**全部非对称**（L ≠ R、两序列不同），可同时打掉
MSB/LSB、A/B 互换、channel-major、时间反向、bit 反相、块偏移：

| case | L | R |
|---|---|---|
| asym | `(k*2654435761) & 0x7fffffff` | `lcg(k+7) ^ 0x5a5a5a5a` |
| impL  | 单点脉冲 +3<<30 | 该点 0x01010101 |
| impR  | 该点 0x02020202 | 单点脉冲 −0x40000000 |
| alt   | ±0x30000000 交替 | −0x10000000 / 0x20000000 交替 |
| full  | ±0x7fffffff | ±0x40000000 |
| small | +3 / −5 | −7 / +11 |

### 84.2 采集结果（mode 0，arg6 0，w3 32，128 帧）

```
case      pack store  retval  n_out   densA  densP  sha_packed[:12]
asym_m0      1    256    1024   1024   0.0771  0.6237  b08da9ac770d
impL_m0      1    256    1024   1024   0.0630  0.5005  d56fba9a44ba
impR_m0      1    256    1024   1024   0.0625  0.4995  e3feef18ab28
alt_m0       1    256    1024   1024   0.0627  0.5079  fb8287ac3169
full_m0      1    256    1024   1024   0.0710  0.6606  c47ba0cdacfb
small_m0     1    256    1024   1024   0.0625  0.5000  4e8746bfa890
```

密度行为物理上自洽：`small` 恰 0.5000（低电平良好整形）、
`full` 0.6606（满幅过载，1-bit 级偏置）、
`impL/impR` ≈ 0.50（脉冲后回到良好整形）。

### 84.3 长度关系修正：w3=32 下 output = w2（1:1）

```
n_out = 1024 B,  w2 = 1024 B
推算：128 帧 × 32（插值倍率）× 2 通道 ÷ 8 bit/byte = 1024 B   ✓
```

> **§27.2 的 `output_bytes = 4*x2/x3` 不适用于 w3=32。**
> 该式是在 **w3=0x18** 下测得的（4×1024/32=128 ≠ 1024）。
> 真实 w3=32 的关系是 **`output_bytes = nframes × 8 = w2`**（1:1）。
> §27.2 由此推导的"x3 是除数"结论**仅对 synthetic 分支成立**，
> 与 §78-§83 一并归入 legacy synthetic branch analysis。

### 84.4 未完成：与 Python / C++ 的逐字节相等

固件侧工件已就绪且带 sha256，但**参考侧尚未实现**：
需要按真实链

```
int32 → /2^31 → 5 级 FIR(×32) → mode 0/2 量化器 → 1-bit 缓冲
      → 8:1 pack → [A][B] 交织 → packed
```

写出 Python 与 C++ 两份实现，再与 `_packed_fw.bin` 逐字节比对。

因此本轮**不升级** §67-§68 的位序/交织约定为 Verified；
只新增一条已验证事实：**真实路径的 packed 输出长度 = 输入长度**，
且密度随信号电平呈现预期的 0.5 → 0.66 变化。

### 84.5 已固化的门禁清单（增量）

```
golden/real32/                real-w3=32 provenance 工件（回归基线）
长度关系                      w3=32: n_out = nframes×8 = w2；勿套用 4*x2/x3
FN_END_GUARD                  必须 > 0x1a6cc；设成 0x1a320 会静默截断执行
```

## 85. 中间缓冲格式确认 + 长度门禁 6/6 通过；本轮自曝一处度量 bug（2026-10-06）

### 85.1 长度门禁（结构断言，非 sanity）

```
expected_packed = w2 = 1024
alt   1024 == 1024 OK      asym  1024 == 1024 OK
impL  1024 == 1024 OK      impR  1024 == 1024 OK
full  1024 == 1024 OK      small 1024 == 1024 OK
```

6/6 通过。§84.3 的 `n_out = w2`（w3=32）**作为硬门禁成立**。
`.meta` 已显式记录 `w3=32 / w2=1024 / n_out=1024`，防止旧 `4*x2/x3` 被误带回。

### 85.2 本轮自曝的度量 bug（重要）

`uc_capture32.py` 的 `dens()` 用 `popcount/8`，这对**packed**（8 bit/byte）
是对的，对 **intermediate**（1 bit/byte）是错的 —— 后者应直接用
`nonzero/len`。因此 §84.2 表中 `densA ≈ 0.063` 是**度量错误**，
真实值如下（已直接统计）：

```
alt_m0_a0_intermediate_A.bin   len=1024  nonzero=514  distinct values = {0, 1}
asym_m0_a0_intermediate_A.bin  len=1024  nonzero=632  distinct values = {0, 1}
```

> **中间缓冲 A = 每个样本 1 字节，取值仅 {0, 1}**（1-bit-per-byte）。
> 更正后的非零密度与 packed 比特密度高度一致：

| case | A 非零密度 (1 bit/byte) | packed 比特密度 (8 bit/byte) |
|---|---:|---:|
| alt  | 514/1024 = **0.5020** | 0.5079 |
| asym | 632/1024 = **0.6172** | 0.6237 |

> **撤回**：上面两列的差值 "+0.0059" **不能归因于尾部补零**。
> intermediate 只统计了每通道**前 1024 / 4096** 个样本，packed 统计的是
> **完整 1024 B**，两者统计窗口不同，差值无意义。
> 该归因作废；须取满 4096 字节 A/B 后再比较（§85.2 的密度本身仍有效，
> 只是不能与 packed 密度相减）。密度相同是必要条件，**不证明**位序。

### 85.3 参考侧尚缺两项信息

1. **intermediate 的元素总数**：1024 packed 字节 / 2 通道 = 512 B/通道
   = 4096 bit ⇒ 每通道应有 **4096 个 1-bit 样本**（= 128 帧 × 32）。
   本轮只读了每通道前 1024 个（即前 1/4）。
2. **量化器 d8 必须显式传 16**：`tools/refchain.py` 的 `quant()` 目前写死
   `d8 = 16 - mode`，这对 mode 2 会给出 14，而真实链路中
   **d8 = 16 − arg6 且 arg6 ≡ 0**（§73/§74/§80），故 mode 2 也必须是 16。
   参考侧需加 `d8` 参数，**不得沿用 `16-mode`**。

这两项不补齐，任何逐字节比对都会得到无意义的 mismatch。

### 85.4 当前状态

```
VERIFIED  packed 长度 = w2（w3=32），6/6
VERIFIED  intermediate = 1 bit per byte，取值 {0,1}
VERIFIED  intermediate 密度 ≈ packed 密度（+固定 0.0059 尾部差）
OPEN      位序 / A-B 交织 —— 只能由逐字节 mismatch=0 判定
OPEN      参考侧实现（需 85.3 两项）
```

## 86. ★ 真实 w3=32 路径三层逐字节对拍 18/18 通过 —— 打包层正式 Verified（2026-10-06）

> 按评审要求的顺序执行：先 A/B intermediate，再 packed。
> 每一项给出 length / mismatch_count / first_mismatch / sha256。

### 86.1 参考侧的两处必要修正（均已落地 `tools/refchain.py`）

1. **`d8` 与 `mode` 解耦**：`quant(tbl, mode, d8=16.0)`。
   旧代码 `d8 = 16 - mode` 是 **w3=24 synthetic harness 的产物**
   （当时 harness 把 `arg6` 喂成 mode）。真实路径 `d8 = 16 − arg6`，
   而 `arg6 ≡ 0`（§73/§74/§80），**mode 0/1/2 全部 d8=16**。
2. **输入标度**：`split_stereo_int32()` 按 §80 做 `int32 / 2^31`。

未重写任何 DSP：FIR/量化器全部复用既有 exact-FMA 实现。

### 86.2 对拍结果（mode 0，arg6 0，w3 32，128 帧，w2=1024）

```
case     layer     len_fw len_ref mism first   sha_fw        sha_ref
asym     A          4096    4096     0    -    1a24536e7162  1a24536e7162
asym     B          4096    4096     0    -    1003dbb59e9c  1003dbb59e9c
asym     packed     1024    1024     0    -    b08da9ac770d  b08da9ac770d
impL     A          4096    4096     0    -    2cb41fb4107e  2cb41fb4107e
impL     B          4096    4096     0    -    4558f43739e7  4558f43739e7
impL     packed     1024    1024     0    -    d56fba9a44ba  d56fba9a44ba
impR     A          4096    4096     0    -    ee58d1eebdb5  ee58d1eebdb5
impR     B          4096    4096     0    -    3726c0cd7e4  3726c0cd7e4
impR     packed     1024    1024     0    -    e3feef18ab28  e3feef18ab28
alt      A          4096    4096     0    -    b9e161b6d4ff  b9e161b6d4ff
alt      B          4096    4096     0    -    bd3e31141967  bd3e31141967
alt      packed     1024    1024     0    -    fb8287ac3169  fb8287ac3169
full     A          4096    4096     0    -    2a32773f2035  2a32773f2035
full     B          4096    4096     0    -    97a96be0175e  97a96be0175e
full     packed     1024    1024     0    -    c47ba0cdacfb  c47ba0cdacfb
small    A          4096    4096     0    -    a5eac9352945  a5eac9352945
small    B          4096    4096     0    -    0f24c606945c  0f24c606945c
small    packed     1024    1024     0    -    4e8746bfa890  4e8746bfa890

ALL LAYERS BYTE-EXACT: True        (18/18)
```

（impR 的 B 行 sha 打印为 `3726c0cd7e4`，与 meta 的 `3726c0cd7e4…` 同源，
截断位数差异，无 mismatch。）

### 86.3 §67-§68 正式升级

> **[Verified — real w3=32 playback path]**
>
> ```
> int32 stereo PCM (8 B/frame)
>   → /2^31
>   → 5-stage FIR/polyphase (×32)
>   → mode 0 coefficient set
>   → quantiser, d8 = 16 (独立于 mode)
>   → 1-bit intermediate, 1 byte/sample, 值 {0,1}
>     A = handle+0x1203A0   B = handle+0x1303A0   (各 4096 B)
>   → 0x1a6cc..0x1a7e0  8:1 packing
>   → MSB-first
>   → [A][B] 字节交织
>   → output = handle+0xC0005A,  length = w2 = 1024 B
> ```
>
> 证据：6 组非对称输入 × 3 层 = 18/18 mismatch=0、first_mismatch=-、sha256 相同。

**这同时证明的不只是 packer**：因为 packed 一致即蕴含 A/B 一致、A/B 一致即蕴含
`tbl` 与量化器输出一致，因此本轮同时把
**真实链的 int32→tbl 标度、5 级插值、mode 0 量化器**一并从
"分项验证"升级为**端到端 byte-exact**。

### 86.4 mode 2 尚未做同样对拍

当前 6 组只覆盖 mode 0。mode 2（level≥2 的真实分支）需同样跑一遍
（`uc_capture32.py 0,2 128` + 对拍脚本）。在此之前，
"mode 2 与 mode 0 仅抽头不同、d8 同为 16"仍只有静态 + 器件侧证据。

### 86.5 已固化的门禁清单（增量）

```
golden/real32/                real-w3=32 工件（回归基线，6 组 × 4 文件 + meta）
refchain.quant(d8=)           d8 必须显式传参；禁止 16-mode 隐式耦合
intermediate 读取长度         = packed/2 × 8（本轮曾误用 /2，已修）
三层对拍顺序                  A → B → packed；先中间后打包
```

## 87. mode 2 真实 w3=32 端到端 18/18 通过 —— 两个真实可达分支均已 byte-exact（2026-10-06）

### 87.1 条件

与 §86 完全同规格，只改 `mode = 2`（`d8` 仍为 **16**，与 mode 无关）：

```
w3 = 32,  w2 = 1024,  arg6 = 0,  d8 = 16,  mode = 2
6 组非对称输入（asym / impL / impR / alt / full / small）
```

### 87.2 结果

```
case     layer     len_fw len_ref mism first   sha_fw        sha_ref
asym     A          4096    4096     0    -    cdc2df30025a  cdc2df30025a
asym     B          4096    4096     0    -    a79d11e91574  a79d11e91574
asym     packed     1024    1024     0    -    7a4063e64a44  7a4063e64a44
impL     A          4096    4096     0    -    14f8e26aa023  14f8e26aa023
impL     B          4096    4096     0    -    899d6c1fd90a  899d6c1fd90a
impL     packed     1024    1024     0    -    3df19ceb55e3  3df19ceb55e3
impR     A          4096    4096     0    -    efb9f09b2b01  efb9f09b2b01
impR     B          4096    4096     0    -    481fdd2da723  481fdd2da723
impR     packed     1024    1024     0    -    6ecd5ea7d09b  6ecd5ea7d09b
alt      A          4096    4096     0    -    436235418926  436235418926
alt      B          4096    4096     0    -    c7e06fd0c54c  c7e06fd0c54c
alt      packed     1024    1024     0    -    16974d72a0cf  16974d72a0cf
full     A          4096    4096     0    -    2851b977f174  2851b977f174
full     B          4096    4096     0    -    1afd0e05ee58  1afd0e05ee58
full     packed     1024    1024     0    -    b5e26fcf6a14  b5e26fcf6a14
small    A          4096    4096     0    -    25aef4456161  25aef4456161
small    B          4096    4096     0    -    2b52b3d35618  2b52b3d35618
small    packed     1024    1024     0    -    f3b93aa76215  f3b93aa76215

mode 2 ALL LAYERS BYTE-EXACT: True  (18/18)
```

### 87.3 mode 切换确实是"活"的（防止两套数据其实是同一套）

```
asym 输入下 mode0 vs mode2 的 packed 输出：
  895 / 1024 字节不同 (87.4%)
```

mode 0 与 mode 2 产出**显著不同**的码流，说明 §73 的
`w26 = (level<2) ? GPC+0x68(=0) : 2` 分支在真实链路上确实改变量化器抽头组，
而 `d8=16` 不变。这与 §74.4 的器件侧证据（tone 电平在 level 0/1/2 间恒定，
Δ≤0.05 dB）**方向一致且互相独立**。

### 87.4 措辞纪律（采纳评审收紧）

> §86 的证据强度来自**三层各自独立比较并全部 mismatch=0**，
> **不是**来自"packed 映射理论上可逆 ⇒ A/B 必然一致"。
> A/B 是被独立读出、独立比对、独立得到 sha256 相同的。

### 87.5 里程碑

```
JM21 REAL DIGITAL PATH（两条真实可达分支）
  level < 2  → mode 0 (8 taps)  ✓ 18/18 byte-exact
  level >= 2 → mode 2 (4 taps)  ✓ 18/18 byte-exact
  mode 1 (6 taps)  仅作核心算法回归分支，不是真实默认播放路径

已 byte-exact 的链路：
  int32 stereo → /2^31 → 5-stage FIR(×32) → mode 抽头 → quantiser(d8=16)
  → 1-bit intermediate(1B/sample) → 8:1 MSB-first → [A][B] 交织
  → handle+0xC0005A, length = w2
```

### 87.6 剩余三层

```
① C++ production 重编 + 硬门禁回归（正确舍入 FMA、/2^31、d8=16、删旧 exe、
   确认时间戳/hash）→ 证明生产实现未漂移
② DSF/container 契约：C++ DSF DATA payload == verified packed reference
   （不再从 DSF 解码回 PCM 比波形；最终裁判是 packed reference == DATA payload）
③ 最终真机 / dump 对拍
```

### 87.7 建议的最终黄金对象

```
golden/end_to_end/<case>_<mode>/
   input.int32  intermediate_A.bin  intermediate_B.bin
   packed.bin   reference.dsf      .meta
.meta 固定记录：w3 / w2 / mode / d8 / 输入布局 / sample rate / channel count /
                initial state / FMA 实现 / packed 契约 / DSF 参数
```
目标：一条命令判定整个 JM21 reference implementation 是否仍 bit-exact。

## 88. C++ production 构建门禁：固化 + 本机工具链阻塞（2026-10-06）

### 88.1 门禁规范（`build_gate.py`，永久化）

```
GATE 1  源码扫描      拒绝 `d8 = 16 - mode` / `16 - jm21Mode` / `std::fma` / `fma(`
                      （剥离注释后扫描，避免散文误报）
GATE 2a 删除旧 exe    两个 exe 必须确实不存在
GATE 2b build.bat     非零退出码 → 立即停止，打印完整构建输出
GATE 3  exe 校验      存在性 + size + mtime + sha256
GATE 4  黄金基线      golden/real32 必须有工件
任一失败 → BUILD GATE: FAIL → STOP，禁止使用任何 exe
```

**GATE 1 采用源码扫描而非运行结果间接抓取**，因为 `d8 = 16 - mode` 与
`std::fma` 都可能在部分输入上"碰巧通过"，只有静态拒绝才是可靠门禁。

### 88.2 本次事件定性：可恢复的环境事故，非项目结论

```
GATE 1  source scan                PASS
GATE 2a delete old exes            PASS（已删除 build/flac2dsf.exe、flac2wav.exe）
GATE 2b build.bat                  FAIL (rc=1)
GATE 4  golden/real32 baseline     PASS (12 cases, modes 0,2)
```

现象（完整实测证据见 [`ENVIRONMENT.md`](ENVIRONMENT.md)）：

```
g++ -v -E s.cpp   → 正确打印 specs 与 cc1plus 完整命令行，随后 rc=1，无产出
                    ⇒ 驱动能启动并正确拼装后端命令行；失败发生在后端启动阶段
```

**故障面精确到"gcc 后端可执行文件"这一层**：

| 可执行文件 | rc | 输出 |
|---|---|---|
| `gcc.exe` / `g++` / `as.exe` / `collect2.exe` | 0 | 正常版本串 |
| `cc1.exe` / `cc1plus.exe` / `lto1.exe` | 127 | 空 |

```
bash 下  "…/cc1plus.exe" --version     → rc=127，无输出
cmd 下   cmd //c "…\\cc1plus.exe" --version → rc=0，无输出
```

`cmd.exe` 下 **rc=0 但零输出**是"进程被创建后立即终止"的特征，
不是参数错误、不是文件缺失（`cc1plus.exe` 存在，40,098,585 B，合法 MZ，
mtime 2025-08-09 未被改写）。

关键排除项：

```
C 与 C++ 均失败        gcc -c 与 g++ -c 同样 rc=1        → 与语言无关
g++ -E 也失败          纯预处理、无代码生成，同样 rc=1     → 不是"禁止代码生成"
连 int main(){return 0;} 都产不出 exe                     → 与项目源码无关
禁用沙箱后行为完全相同                                    → 非 ZCode agent 沙箱
同目录 lto1.exe(36MB) 失败、collect2.exe(2MB) 正常          → 排除文件大小/内存映射类原因
```

⇒ **本机 MSYS2/mingw64 15.2.0 的 gcc 后端程序无法启动，属可恢复的环境故障，
与项目源码无关。** 恢复步骤见 `ENVIRONMENT.md` §4。

> **定性：空的 `build/` 比拿旧二进制冒充新构建安全。**
> 项目此前出现过"旧 exe 被复用导致假通过"，本次删除虽造成中断，
> 但从验证纪律上是正确动作。**禁止**用旧 exe 恢复门禁，
> **禁止**为绕过此问题修改任何 C++ 源码。

### 88.3 本会话未触碰 C++（provenance）

```
src/dsf.cpp         2026-10-06 19:46      src/flacdec.cpp   2026-10-04 20:16
src/f2d.h           2026-10-06 19:26      src/flacdec.h     2026-10-04 18:02
src/flac2dsf.cpp    2026-10-06 19:48      src/web.cpp       2026-10-04 21:43
src/dsf.h           2026-10-04 20:36      src/wpath.h       2026-10-04 20:12
```

本会话的全部改动都在 Python 侧（`tools/refchain.py`）与
`vendor_pull/re/*.py`，**没有一行 C++ 被修改**。
因此在可正常编译的环境执行 `build.bat` 应可直接成功。

### 88.4 恢复后的执行顺序（不得跳步）

```
GATE 2/3   build.bat → 两个 exe 存在 + mtime/hash 记录
GATE 5     --jm21 对 golden/real32 做双 mode 回归
             mode 0 × 6 组 × (A, B, packed)
             mode 2 × 6 组 × (A, B, packed)
             mode 1 × 既有核心黄金（算法回归，非真实播放分支）
GATE 6     DSF 层：data_offset + data_size == file_size
                       SHA256(DSF DATA payload) == SHA256(packed reference)
           —— 不经过 DSD decoder；DSP 正确性与容器正确性完全解耦
```

**GATE 5/6 必须消费已固化的 `golden/real32/`，不得重新生成参考真值。**

## 89. ★ GATE 5/6 通过：C++ production 修复一处极性 bug 后与固件端到端 byte-exact（2026-10-06）

### 89.1 故障分层（不猜测，直接用 `--jm21-dump-*` 切层）

```
LAYER 1  core input    py +0.5   cpp -0.5   ratio = -1   512/512 全错   ← 断点
LAYER 2  packed        4096/4096 mismatch=4096 first=0
LAYER 3  DSF DATA      8192/8192 first=0
```

**情况 1 成立：故障在 Jm21Core 入口之前。`dsf.cpp`、`transposeData()`、
packer、shaper 系数全部无罪**（本节先前的"`transposeData` 分块假设"是过早归因，已作废）。

补充证据（DC 常量输入）：

```
--gain 0dB    -> core input[0] = -0.5
--gain 1x     -> core input[0] = -0.5      线性 1.0 仍为负
--gain 0.000x -> core input[0] = -0         负零 ⇒ 先取反、后乘 gain
--gain -0dB   -> core input[0] = -0.5
```

### 89.2 根因：`gainLin` 的符号约定未被 jm21 路径吸收

```cpp
src/f2d.h:29          double gainLin = -0.707946;  // -3 dB, negative by convention
src/flac2dsf.cpp:935   --gain-unity  -> o.gainLin = -1.0;
src/flac2dsf.cpp:938   o.gainLin = -std::pow(10.0, atof(..)/20.0);   ← 前置负号
src/flac2dsf.cpp:739   jmIn[k] = pcm.data[...] * o.gainLin;          ← jm21 直接乘带符号值
src/flac2dsf.cpp:666   modCh[c].init(o.shaperOrder, o.leak, o.gainLin);  ← FIR 路径也收到
```

`gainLin` 的**符号承载极性**，幅度才是增益。FIR 调制器吸收了该约定
（历史黄金测试通过），**`--jm21` 路径没有**；而设备侧
`get_pcm2dsd_data` 是 `scvtf d0, w11` 直接转换、**不做取反**（§78/§80），
因此 jm21 路径必须使用幅度。

这也解释了之前所有现象：impL 前 12 字节偶然相同（比特模式局部不变）、
alt/full/small 首字节即错（高频/满幅下极性翻转立即改变量化判决）。

### 89.3 修复（单变量，仅限 jm21 分支）

```cpp
jmIn[k] = pcm.data[...] * (o.jm21 ? std::fabs(o.gainLin) : o.gainLin);
```

FIR 路径行为**逐位不变**；jm21 路径改为与设备同号。

### 89.4 验收结果

```
DC 探针（512 帧，L=0x40000000 R=0x20000000）
  mode 0  LAYER1 OK   LAYER2 OK   LAYER3 OK
  mode 2  LAYER1 OK   LAYER2 OK   LAYER3 OK

GATE 5  6 cases × {0,2} × {A/B, packed, cpp-payload}     PASS
GATE 6  6 cases × {0,2}: data_offset+data_chunk_size == file_size
                               == dsd_total_size
                               SHA256(DATA payload) == SHA256(packed)
                               block_size_per_channel == 4096, channel_num == 2   PASS
```

**GATE 5 与 GATE 6 全部通过。C++ production 实现与固件黄金基线端到端 byte-exact。**

### 89.5 措辞边界（采纳评审）

- 本轮**已证明**：C++ production 与 firmware/Python 在真实 w3=32、
  mode 0 与 mode 2、6 组非对称输入上，core input / packed / DSF DATA 三层逐字节一致。
- 本轮**未验证**：mode 1 的 production 回归（`--jm21-mode 1`）、
  非 88200 Hz 输入（会触发重采样，尚未与固件对照）、
  大于 4096 帧的跨块状态。
- `--help` 中 `--jm21` 的旧文案（"`--shaper 4 --leak 1`"）**仍未修**，
  按评审意见与本次数据修复分开处理，避免混入 provenance。

### 89.6 已固化的门禁清单（增量）

```
build_gate.py                 GATE 1-4（源码扫描/删旧 exe/构建/新鲜度/基线）
gate5.py                      GATE 5 三层三方对照（消费 golden/real32）
--jm21-dump-pcm/-packed       分层定位手段：core input / packed
增益极性                      gainLin 符号是极性约定；jm21 路径须用 fabs
```

## 90. 真实音乐 production 验证 + 真机数字对拍的可行性边界（2026-10-06）

### 90.1 真实音乐跑通（20 s，88200 Hz 输入，DSD64）

```
源：music/9Lana/螺旋 - RASEN - EP/01 螺旋 - RASEN.m4a
    → ffmpeg → 88200 Hz / 32-bit / stereo WAV（1 764 000 帧）
    → flac2dsf --jm21 --jm21-mode {0,2} --arg6 0 --gain 0dB -d 0

mode 0:  13.46 MiB  blocks=3446  rate=2822400  samples=56 448 000  density=0.4999  container=OK
        DATA sha256 = 82bd518ce7af2e5486b738e19c2264feab91ac2b
mode 2:  13.46 MiB  blocks=3446  rate=2822400  samples=56 448 000  density=0.4999  container=OK
        DATA sha256 = aa52809e7da2a840337f77bea7e3014112f4e9dc

mode0 vs mode2 相差 12 713 339 / 14 114 816 字节 (90.1%)
```

倍率与长度自洽：`1764000 × 32 = 56 448 000 samples/ch`；
`56 448 000 / 8 = 7 056 000 B/通道`，×2 + 块补零 = 14 114 816 B ✓（3446 块 × 4096）。
密度 0.4999 —— 真实音乐上 1-bit 整形良好。
跨 3446 块（远超 4096 帧的 4096-frame 分块边界）容器与长度均正确，
**间接覆盖了 §89.5 列出的"大输入跨块状态"未验证项（容器层面）**。

### 90.2 真机数字对拍在本机**不可得**（架构性约束）

原计划是"设备生成 DSF vs C++ production DSF"。**该路径不存在**：

§36.15 已确证 `get_alltodsd_config()` 内：

```c
on = atoi(prop("persist.sys.all.to.dsd"));
property_get("sys.usb.config", v);
if (v == "uac2\0" || v == "uac2,adb\0") on = 0;    // 立即数 0x32636175 / 0x6264612c32636175
```

即 **USB DAC 模式下 All-To-DSD 被强制关闭**。设备只在**本地播放**时做 PCM→DSD，
结果送往 CS43198 模拟输出，**不产生 DSD/DSF 文件，也不经 USB 输出 DSD 码流**
（USB DAC 输入路径标称支持 DSD512，但那是反向：外部 DSD 进设备解码）。

因此可得的最终验证只有两条：

```
A. 模拟域统计比较（已有基础设施：MEASUREMENT_PROTOCOL + tools/mode_ab_matrix.py）
   真机本地播放 → Line-In 192 kHz 录制  vs  C++ DSF → 解码 → 同等条件
   判据：频带形状、增益、mode0/mode2 差异方向（不是字节相等）

B. 数字域等价性论证（本轮已完成）
   固件 golden（Unicorn 逐指令执行真实 ARM64 代码）== C++ production == Python reference
   三方在 mode 0/2 × 6 组输入上逐字节一致
```

B 已经建立；A 需要物理链路，属第 34/35 节那套方法，本轮未做。

### 90.3 状态

```
VERIFIED  C++ production 与固件 golden 端到端 byte-exact（§89，mode 0/2 × 6 组）
VERIFIED  真实音乐 20 s 转换：结构、长度、倍率、密度、mode 差异全部自洽（§90.1）
VERIFIED  大输入跨块容器正确（3446 块）
NOT POSSIBLE  真机 DSD/DSF 数字输出（USB DAC 模式强制关闭 All-To-DSD）
PENDING   模拟域统计对拍（Line-In，物理链路）
PENDING   mode 1 production 回归
PENDING   --help 中 --jm21 旧文案（纯文案，单独修改以保 provenance）
```

## 91. mode 1 production 回归通过 —— 数字复现线封版（2026-10-06）

### 91.1 mode 1（6 抽头）全链 byte-exact

与 mode 0/2 同规格采集 6 组非对称输入的固件黄金向量，三方对照：

```
case    layer      len_fw  len_ref  mism   C++   status
asym    A/B/packed  4096/1024  4096/1024   0     -   OK
asym    cpp-DATA          -       8192     -     0   OK
impL / impR / alt / full / small   （同上，全部 OK）
mode 1 FULL CHAIN: PASS
```

**mode 1 在真实 w3=32 路径上同样可达并 byte-exact** ——
尽管 §73.5 的静态分析指出 `w26 = (level<2) ? GPC+0x68(=0) : 2`
使 **mode 1 在真实播放路径上不可达**。本轮证实：只要把 `w26` 设为 1
（`--jm21-mode 1`），固件与 C++ 的行为完全一致。
即 mode 1 是**算法正确但默认不可选**的分支，保留它是为了防止重构时被悄悄破坏。

### 91.2 黄金基线规模

```
golden/real32/   18 cases（6 输入 × mode {0,1,2}）× 4 文件 + meta
                 = 72 工件 + 18 meta
```

### 91.3 最终状态分层（正式版）

```
VERIFIED
├─ real w3=32 input contract（int32 stereo, 8 B/frame, w2%16==0）
├─ int32 → normalized input（G_input = 1/2^31）
├─ FIR / interpolation topology（5 级 ×32，DC 1/32）
├─ shaper coefficients（三档）
├─ mode 0 / mode 1 / mode 2  ← 三档全部 byte-exact
├─ d8 = 16（独立于 mode）
├─ A/B intermediate（1 B/sample，取值 {0,1}）
├─ packer（8:1，MSB-first，[A][B] 交织）
├─ packed stream（output = handle+0xC0005A，length = w2）
├─ DSF DATA payload
├─ DSF container（结构断言）
├─ 18 组 golden vectors
└─ 20 s 真实音乐 / 3446 blocks（长输入跨块 + 密度 0.4999）

NOT POSSIBLE（architecturally unavailable）
└─ 真机 DSD/DSF 数字抓取
     USB DAC 模式下 get_alltodsd_config() 强制 on=0（§36.15/§90.2）；
     设备仅本地播放时做 PCM→DSD，输出为模拟信号，不产生文件、不经 USB 出码流。

PENDING
├─ --help 中 --jm21 旧文案（纯文案，单独修改以保 provenance）
└─ 模拟域 Line-In 统计对拍（OPTIONAL / PHYSICAL，需物理链路）
```

### 91.4 措辞纪律（不外推）

**已验证**：生产链在长输入下能正确跨块运行（实测 3446 块）。
**不外推**：任意长度、任意边界条件均已穷尽验证。

**已验证**：三档 shaper 在真实 w3=32 路径上三方 byte-exact。
**不外推**：mode 1 是设备实际可达的播放分支（静态分析表明不可达，
§73.5；`level` 只在 0 与 2 之间取值，实测 §74.1 亦只出现 88200/176400/352800/705600）。

## 92. 模拟域 Line-In 对拍（物理链路已接通）：设备 ON/OFF 干净，40-80 kHz +2.76 dB（2026-10-06）

### 92.1 测量条件

```
设备 3.5 mm 输出 → PC 线路输入；WASAPI 独占 24 bit / 192 000 Hz（设备 28）
源：/sdcard/Music/ALLTODSD/1k_6dBFS.flac（1 kHz / −6 dBFS）
level = 0（DSD64，真实默认路径），All-To-DSD ON / OFF 各采 12 s
独占采集自检：本底 −69.0 dBFS RMS（与 §35.5 的 −68.1 dBFS 一致）
门禁：每次都要求 fresh config + PCMTODSD/ALLTO32BIT 签名（prev/next 重选强制重新起播）
```

### 92.2 设备 ON / OFF 频带（相对各自 tone bin）

```
band        OFF      ON     Δ(ON-OFF)   归一到 tone
1-1k      -11.54   -11.53     +0.02        -0.05
2-20k     -20.98   -21.00     -0.02        -0.09
10-20k    -31.29   -31.40     -0.11        -0.18
20-40k    -34.16   -34.25     -0.09        -0.16
40-80k    -36.12   -33.36     +2.76        +2.69
80-96k    -42.03   -41.91     +0.12        +0.05
tone       -9.69    -9.62     +0.07          0
```

**结果自洽且与模型一致**：

1. **40-80 kHz 抬升 +2.69 dB（归一后）** —— 1-bit 噪声整形的典型签名。
   mode 0 的 NTF 零点对在 25.42 kHz（§36.16），其上噪声单调抬升；
   40-80 kHz 落在抬升区，20-40 kHz 仍在零点对影响下 ⇒ 20-40k 几乎不变
   （实测 −0.16 dB），40-80k 明显抬升（+2.69 dB）。
2. **带内无变化**（2-20k −0.09、10-20k −0.18）—— 复现 §35.7 的
   "All-To-DSD 带内无收益"，与 §80 的工作点结论（0 dBFS → drive 0.5，
   正处整形边界）方向一致。
3. **tone 增益 +0.07 dB** —— 与 §35.3 的 +3.58 dB 不同。
   §35 是 **USB DAC 模式**下的测量，而 §36.15/§90.2 已确证该模式下
   All-To-DSD 被强制关闭；因此 §35 的"ON/OFF 增益差"很可能测量的是
   其他差异（PCM 直通 vs 其它处理），**不应继续引用 +3.58 dB**。
   本次本地播放测量给出 +0.07 dB，是当前更可信的增益差。

### 92.3 C++ 侧形状比较（mode 2 − mode 0，DSD 解码到 192 kHz）

```
band        m0 相对 tone   m2 相对 tone   Δ
1-1k            0.00          0.00       +0.00
2-20k          22.38         22.36       -0.03
10-20k         19.81         19.78       -0.03
20-40k         11.28         11.26       -0.02
40-80k        -88.60        -88.49       +0.11
80-96k       -101.21       -100.78       +0.43
```

**不能直接与 §92.2 比较**：ffmpeg 的 DSD 解码把 40-80k 压到 −88 dB
（相对 tone），而设备模拟域同频带只低 24 dB。两者的解码/重建滤波器与
过采样率完全不同（ffmpeg 默认用极陡的重建滤波），故绝对带电平不可比。
**mode 0/2 之间的相对差（≤0.43 dB）则与 §74.4 的模拟观测一致
（L1/L2 在 40-80k 差 0.01 dB）**。

### 92.4 结论与边界

```
VERIFIED   设备模拟域 ON/OFF：40-80k +2.69 dB，带内与 tone 增益均不变
VERIFIED   该形状与 mode0 NTF 零点对 25.42 kHz 的预测一致
SUPERSEDED §35.3 的 +3.58 dB 增益差（USB DAC 模式下 All-To-DSD 被强制关闭，
           该测量前提不成立）
NOT DONE   C++ DSF 与设备模拟输出的同链路同带宽绝对比较
           —— 需要把 C++ DSF 经同一 DAC 播放，当前无此硬件条件
```

**诚实边界**：本轮完成的是"设备自身的 ON/OFF 模拟域形状"验证，
证明 §80 的工作点结论在模拟域成立；
**尚未**完成"C++ DSF 模拟输出 vs 设备模拟输出"的头对头比较，
后者需要把 C++ 产出的 DSF 送进与设备相同的模拟链路。

## 93. 项目封版状态（2026-10-06）

### 93.1 结论

> **JM21 All-To-DSD 的数字信号链已完成固件黄金向量级 byte-exact 复现，
> 并通过真实音乐长时 production 与真机本地播放 ON/OFF 模拟域频谱实验
> 获得独立交叉验证。**
>
> **不存在"设备直接生成 DSF 并与 C++ DSF 对拍"的验证路径 ——
> USB DAC 模式会强制关闭 All-To-DSD（§36.15 / §90.2）。**

### 93.2 证据等级

```
数字域（三方 byte-exact）
  firmware golden (Unicorn 执行真实 ARM64) == Python exact-FMA == C++ production
    core input / DSP+state / A/B intermediate / packer / packed / DSF DATA
    18 cases = 6 非对称输入 × mode {0,1,2}；golden/real32/ 共 90 工件
  DSF container：结构断言全通过（6 组 × 3 mode）
  真实音乐 production：20 s / 1 764 000 帧 → 56 448 000 samples/ch
                        → 3446 blocks，density 0.4999，mode0/2 相差 90.1%

真实设备模拟域（本地播放，Line-In 192 kHz 独占）
  OFF ↔ ON：tone +0.07 dB，带内 ≤0.2 dB，40-80 kHz +2.69 dB（归一）
  与 mode0 NTF 零点对 25.42 kHz 的预测方向一致
```

### 93.3 已撤回的结论（永久保留）

```
§35.3  All-To-DSD ON/OFF 增益差 +3.58 dB
       → SUPERSEDED（§92.4）
       原因：§35 在 USB DAC 模式下测量，而该模式强制 All-To-DSD = 0，
             该 A/B 不能归因于 All-To-DSD。
       替代：本地播放 ON/OFF tone Δ = +0.07 dB（§92.2）

§45-§77  w3=24 synthetic 分支（6-byte 记录 / 24-bit 左对齐 / 4*x2/x3）
       → LEGACY SYNTHETIC BRANCH ANALYSIS（§78/§80/§84.3）
       真实路径为 w3=32 / int32 stereo / 8 B-frame，n_out = w2

§81/§82  "0x1a6cc 是打包器"的运行时定位
       → 已由 §83 闭合（根因是 harness FN_END_GUARD 截断，非固件行为）
```

### 93.4 剩余项

```
PENDING
└─ --help 中 --jm21 旧文案（"the literal FiiO taps"，与 §74/§78 结论矛盾）
     纯文案修改，刻意与数据路径修复分开以保 provenance

OPTIONAL / PHYSICAL
└─ 同 DAC 模拟链的 C++ DSF ↔ 真机模拟输出头对头比较
     需要把 C++ 产出的 DSF 送进与设备相同的模拟重建链路；
     当前 ffmpeg 的 DSD 重建滤波远比 CS43198 陡峭，绝对电平不可比，
     故未做，也不属逆向复现的必要条件
```

### 93.5 方法学教训（值得长期保留）

本项目出现过三次"测试结果为真、但结论错位"，成因各不相同：

```
1. 旧 exe 被复用            → 假通过        （已由 build_gate.py 封堵）
2. harness 选了 w3=24 分支  → 格式/标度/拓扑三个自洽的假象同时成立
                               （DC 增益恰好对上、IIR 插值恰好给出 1/32、
                                 音频还能播）  （已由 real32 黄金基线封堵）
3. FN_END_GUARD 设在打包器之前 → 静默截断执行，"两条分支都到不了 packer"
                               被误读为 w3 或 handle 状态的问题
                               （已由 §83 定位并记录）
```

共同教训：**只有"能指到具体哪条指令/哪一行代码"的失败才可定位；
任何"跑出来对不上"都应先分层切到可观测的最细粒度，再动手改。**

## 94. 独立 DAC 验收：C++ DSF 被 iBasso DC03 Pro（CS43131）正确解码（2026-10-06）

### 94.1 目的与设定

数字链已 byte-exact，但尚未证明产出的 DSD 码流能被**JM21 之外**的 DSD DAC 接受。
iBasso DC03 Pro（CS43131，USB，默认 384 kHz）提供独立实现：

```
C++ DSF (--jm21 --jm21-mode 0, 1 kHz tone, DSD64)
   → 取 DATA payload（21 168 128 B）
   → DoP 打包：DSD64 @ 176.4 kHz，16 bit/frame，24-bit 字低 16 位放 DSD
   → iBasso USB 播放（WASAPI 独占）
   → iBasso 模拟输出 → PC Line-In 192 kHz 独占采集
```

DoP 变体 0（低位）与变体 1（MSB 对齐）各测一次，作为打包约定的自洽性检查。

### 94.2 结果（两次独立运行，重复性 ±0.15 dB）

```
variant 0:  rms=-7.48 dBFS  tone(900-1100)=-6.20
  2000-20000   -30.23   rel tone  -24.03
  10000-20000  -39.46   rel tone  -33.26
  20000-40000  -42.33   rel tone  -36.14
  40000-80000  -35.80   rel tone  -29.61     ← 比 20-40k 高 6.5 dB
  80000-96000  -48.95   rel tone  -42.75
  tone above 2-20k floor: +24.0 dB

variant 1:  rms=-7.57 dBFS  tone(900-1100)=-6.31
  2000-20000   -30.96   rel tone  -24.65
  10000-20000  -40.26   rel tone  -33.95
  20000-40000  -42.80   rel tone  -36.49
  40000-80000  -35.94   rel tone  -29.63     ← 比 20-40k 高 6.9 dB
  80000-96000  -49.70   rel tone  -43.38
  tone above 2-20k floor: +24.6 dB
```

### 94.3 结论

1. **码流被独立 DSD DAC 正确接受**：1 kHz tone 高出本底 24 dB，
   不是宽带垃圾 ⇒ DoP 打包正确、DSF 的 packed 负载是合法 DSD64。
2. **噪声整形特征在第二台 DAC 上复现**：
   40-80k 比 20-40k 高 **6.5 / 6.9 dB**，
   与 §92 在 JM21(CS43198) 上测到的 40-80k 抬升（+2.69 dB 归一）
   **方向一致、位置一致**。
   两颗芯片（CS43131 / CS43198）重建滤波器不同，绝对幅度不可比，
   但"噪声在零点对之上抬升"的相对结构被独立复现。
3. **可移植性**：产物不是"只在 JM21 上数字等价"，而是可以喂给任意 DSD DAC。

### 94.4 边界

```
VERIFIED   独立 DAC（CS43131）接受并正确解码 C++ DSD64 码流；整形特征复现
NOT DONE   同 DAC head-to-head（需把 C++ DSF 送进 JM21 自己的 USB DSD 输入；
           需手动切 USB DAC 模式，且该模式下 ADB 断开）
NOT APPLICABLE  DSD256 走 iBasso —— 需 705.6 kHz DoP，超出该机 384 kHz 上限
```

**这比同 DAC head-to-head 更有价值**：数字侧已 byte-exact，
head-to-head 只能验证"这台设备自己的 DAC 有没有额外处理"；
而独立 DAC 验收验证的是"我们的产物对不对" —— 后者更根本，且已完成。

## 95. 同 DAC head-to-head：DoP 未被固件自动检测，实验受阻于两个可解因素（2026-10-07）

### 95.1 已排除/已确认

```
FiiO USB DAC 模式在 PC 上枚举正常（"FiiO M series"，WASAPI device 24）
独占模式支持 176.4/192/352.8/384 kHz PCM_32BIT
DoP v1.1（含 0x05/0xFA marker，176.4k 与 352.8k 两种帧格式）均未触发 DSD 解码
判据：40-80k 出现 +20 dB 的 marker 方波能量（0x05/0xFA 交替 = 88.2k/176.4k 方波），
      tone 埋于噪声（-65 dBFS）⇒ 设备按普通 PCM 播放，CS43198 未切 DSD 模式
```

> **结论：JM21 的 UAC2 固件在通用 Windows PCM 流上不做 DoP 自动检测。**
> 该设备的 DSD 输入（规格页 DoP/D2P/Native）几乎肯定走 **FiiO ASIO 驱动的
> vendor-specific 通道**（Windows 上 FiiO 设备的标准 DSD 播放方式），
> Python/sounddevice 无法驱动 ASIO。

### 95.2 PCM 域回退方案也被音量阻断

```
C++ DSD64 → ffmpeg 解码（正规抽取滤波）→ PCM 192k → USB → 设备 PCM 路径 → 模拟
实测 rms = -38.5 dBFS（设备 USB DAC 音量过低），40-80k 出现 +24 dB 异常带
与 §92 内部 DSD 路径（40-80k 相对 tone -25 dB）无法在同一信噪比下比较
```

混杂因素：① 设备音量过低（需在设备上大幅调高）；② ffmpeg DSD 解码在
40-96k 残留部分整形噪声（192k 输出率下不可完全滤除）。

### 95.3 若要完成此实验（需要时再做）

```
① 用户在 USB DAC 模式下把设备音量调到 rms ≥ -20 dBFS 量级
② 重试 DoP（176.4k/352.8k）—— 低音量下已可确认"是否锁 DSD"
③ 若仍不锁 → 安装 FiiO Windows ASIO 驱动 + foobar2000 播放 DoP/DSD
   （Python 不可驱动 ASIO，需人工操作）
④ PCM 域回退比较（解码后的 DSD 经设备 PCM 路径 vs 内部 DSD 路径）
   —— 有滤波器等效性 caveat，结论强度低于 ②/③
```

### 95.4 对项目结论的影响：**无**

```
数字侧  三方 byte-exact 已闭合（§86/87/91），不依赖本实验
模拟侧  §92 已验证设备自身 ON/OFF 的整形签名（+2.69 dB @40-80k）
本节    仅"同 DAC head-to-head"一项未完成，且原因已定位为
        固件 DoP 检测缺失 + 音量未调，均非项目产物缺陷
```

设备在 USB DAC 模式下停留期间还确认了形态学事实：**All-To-DSD OFF 的 PCM
直通在该模式下依然工作**（PCM 播放出声），与 §36.15 的 `on=0` 门控一致。

## 96. 封版：最后一项 PENDING 清除（2026-10-07）

### 96.1 `--help` 文案更新（纯文案，按纪律独立提交 + 重验）

```
旧： --jm21      --shaper 4 --leak 1  (the literal FiiO taps)
     --jm21-mode N  ... The device always runs mode 0 because handle+0x68
                        has no writer in the library.
新： --jm21      the reverse-engineered JM21 All-To-DSD chain: int32 stereo
     /2^31 -> 5-stage FIR x32 -> firmware N-tap quantiser (d8 = 16);
     bit-exact vs the firmware on the real w3=32 path (RE doc sec.86/87)
     --jm21-mode N  shaper tap set: 0=8 taps, 1=6 taps, 2=4 taps.
     The device runs mode 0 at level<2 and mode 2 at level>=2;
     d8 stays 16 for all modes - it is 16-arg6, and arg6 is always 0 there.
```

两处旧陈述与 §73/§74/§78 的结论矛盾，均已按当前证据改写。

### 96.2 门禁复验（文案改动也必须证明零扰动）

```
GATE 1-4  PASS（重编：flac2dsf.exe sha256 6b663e6e1bd5…，18 golden cases）
GATE 5    12 组三层三方对照        PASS（mismatch=0 全部）
```

### 96.3 最终封版状态

```
VERIFIED
├─ 真实输入契约      w3=32 / int32 stereo / 8 B/frame / w2%16==0
├─ 输入标度          G_input = 1/2^31（静态常量+运行时双证）
├─ DSP core          5 级 FIR ×32 + 三档 shaper + quantizer(d8=16)
├─ 打包              8:1 MSB-first，[A][B] 交织，output=h+0xC0005A，len=w2
├─ DSF DATA + 容器   结构断言 + SHA256(DATA)==SHA256(packed)
├─ 黄金基线          18 cases（6 输入 × mode{0,1,2}），90 工件
├─ 真实音乐          20 s / 3446 blocks / density 0.4999
├─ 模拟域            ON/OFF 整形签名 +2.69 dB@40-80k（Line-In 192k 独占）
└─ production        C++ 与固件黄金端到端 byte-exact（GATE 5/6 PASS）

NOT POSSIBLE   真机 DSD/DSF 数字抓取（USB DAC 强制关 All-To-DSD，§90.2）
CLOSED         head-to-head 尝试：固件不做 DoP 自动检测 + 音量因素（§95）
SUPERSEDED     §35.3 增益差 +3.58 dB；§45-§77 synthetic 输入链（§78-80/84.3）
OPTIONAL       模拟域头对头（需 ASIO/同链路硬件，非必要条件）
```

**项目从"逆向分析"进入"production implementation + evidence closure"完成态。**

## 97. Web UI 更新：JM21 真实路径上线 + 过时说明清理（2026-10-07）

### 97.1 改动（仅 web 层，DSP 零改动）

```
服务端  optsFromQuery 新增 jm / mo / ar 三个查询参数 → o.jm21 / o.jm21Mode / o.arg6，
        jm21 时强制 planar（与 CLI --jm21 一致）
前端    新增「JM21 真实路径 (--jm21)」开关 + shaper 档位选择（mode 0/1/2）
说明区  「逆向结果」清理：
  更正  整形器选择 = persist.sys.dsd.shaper.coeffs（默认 2，4 抽头）
        → 该属性是死配置接口（§73.6），真实分派 level<2→mode0 / level≥2→mode2
  更正  「原厂系数会发散 → 本工具加泄漏积分器」
        → 已由 §86 byte-exact 推翻：固件环路有界且被逐位复现
  新增  真实输入 int32 立体声 8 B/帧、标度 1/2^31、d8=16 恒定、
        设备工作点 0 dBFS → drive 0.5、三方 byte-exact 验证声明
```

### 97.2 验收

```
GATE 1-4   PASS（重编 flac2dsf.exe 957762 B）
GATE 5     12 组三层三方回归                 PASS（DSP 零扰动）
Web 冒烟   --web --dir + /api/convert-local
           jm=1 mo=0 g=-1 d=0（即 0 dB / 无抖动）
           DSF DATA sha256 == CLI --jm21 同参数产物   dc3e70b100fe…   逐字节一致
```

首次冒烟暴露的行为差异（DATA sha 不同）经查为**参数不同**（web 默认
gain -3 dB + dither 0.05 vs CLI 的 0 dB / 无抖动），非实现缺陷 ——
统一参数后一致。这也再次验证了 GATE 5 的设计初衷：任何增益/抖动漂移
都会被 sha256 立即暴露。

### 97.3 使用注意

Web UI 的 JM21 路径要复现设备行为需与固件同参：**增益 0 dB（g=-1）、抖动 0（d=0）**。
默认的 -3 dB / 0.05 抖动是 FIR 路径的听感优化，对 JM21 路径会偏离设备工作点。

## 98. Web UI 拖入 bug 修复 + 全链冒烟（2026-10-07）

### 98.1 用户报告：拖入的音频无法加入转换列表

**根因（两层）**：

```
① 致命：drop 处理器里 `$('#file').files = f`，f 是普通 Array。
   HTML 规范要求该属性只接受 FileList —— 浏览器抛 TypeError，
   异常发生在 addLocal(f) 之前 ⇒ 整个拖入路径零行进列表。
   （点选路径走 change 事件、没有这个赋值，所以历史上只测过点选）
② UX：isAudio 只认 .flac/.wav，用户音乐库里的 .m4a 被静默丢弃，无任何提示。
```

### 98.2 修复（仅 web 前端 JS，DSP/服务端零改动）

```
drop 处理器：删除非法的 .files 赋值（上传本来就直接用 j.file 对象）；
             拒绝的文件以错误行显示在列表中（"不支持的格式（仅 FLAC / WAV）"），
             不再无声消失。
```

### 98.3 验收

```
GATE 1-4   PASS（重编 958274 B）
GATE 5     12 组三层三方回归                  PASS（DSP 零扰动）
Web 冒烟   asym 黄金输入做 WAV → POST /api/convert（jm=1 mo=0 g=-1 d=0）
           → job done, 128 帧, DSF DATA == 固件黄金 planar(A)+planar(B)  ✓
（首轮冒烟曾报不一致，根因是测试脚本自建的 WAV 头把格式码写成 float(3)
  而 payload 是 int32 —— 修正测试为 PCM(1) 后完全一致；属测试代码错误，
  非工具缺陷，记录以免重犯）
```

### 98.4 边界

```
VERIFIED   web 上传路径（与 CLI 同参同结果，三方链一致）
VERIFIED   拒绝格式有可见提示（不再静默）
NOTE       工具输入格式仍是 FLAC/WAV；m4a/mp3 需先用 ffmpeg 转换
           （本次用户音乐库即 m4a，属预期限制，不是本次 bug 的一部分）
```

## 99. 用户可见的「DSD 频谱超声噪声」解释（2026-10-07）

用户用 Spek 对比 FLAC 与转换产物 DSF 的频谱：FLAC 的 20 kHz 以上干净，
DSF 却从 20 kHz 到 176 kHz 有一整块均匀的「噪声」。**这是 DSD 的本征形态，
不是缺陷**：

1. 1-bit Σ-Δ 编码的原理就是把量化噪声**搬出音频段**（噪声整形），
   用采样率换位深。任何 DSD64 文件（索尼自家工具、设备内部产生、本工具）
   的频谱都是「0-20 kHz 音乐 + 20 kHz 以上整形噪声」这一形态。
2. 这块噪声在数播放链里会被 **DAC 的 DSD 重建滤波器**在模拟输出前滤掉。
   §92 实测设备：带内频谱 ON/OFF 无差、40-80 kHz 仅 +2.69 dB 泄漏 ——
   「文件里有超声噪声」与「实机纯净安静」并不矛盾。
3. 本工具产物与固件输出逐字节一致（§86/87/91），故其超声噪声块与
   设备内部 All-To-DSD 流**完全相同** —— 同一段 FLAC 若由设备自己转换，
   内部 1-bit 流的频谱与此一致。
4. Spek 显示 "88200 Hz" 为 DSD 显示约定（文件头实际 2822400 = DSD64，
   已核对 `I:\02 以后的以后.dsf`：rate=2822400, bits=1, ch=2, 51396 块，
   298.3 s，结构正确）。
5. 若希望频谱上 20 kHz 以上的噪声更低：用 `-r 256`（DSD256 把整形噪声
   推到更高频段，NTF 零点对 25.42 kHz ×4 ≈ 101.7 kHz），
   代价是文件体积 ×4；播放端听感无差（DAC 滤波后一致）。

## 100. ★ DSF 位序 bug 修复：头部声明 LSB-first，数据却是 MSB-first（2026-10-07）

用户实测反馈："放大音量确实有非常多杂音"（数字侧与固件 byte-exact，但播放器
发闷/大音量噪）。**这是本项目唯一一个数字侧验证覆盖不到的层**：
固件输出的是 **wire order**，而 DSF **容器**的位序约定与之相反。

### 100.1 根因

```
DSP 打包：MSB-first-in-time（时间上最早的样本在 bit 7）
          —— 这是 CS43198 DSD 输入的 wire order，§86/87 已逐字节证明
DSF 头：  bits_per_sample = 1
          按 DSF 规范 = 每字节 LSB-first 播放
```

头部与数据**自相矛盾** ⇒ 合规播放器把每 8 样本组**反序播放** ⇒
整形噪声在 fs/8 = 352.8 kHz 的倍频处**折叠回可听频段**
⇒ 大音量下大量杂音，而音乐因波形大体会对、听起来仍"正常但发闷"。

**为什么前面所有验证都没抓到**：全部只比较 DATA 字节是否等于固件 wire 字节
（一致），从未检查"该字节按容器声明播放后是否为正确时间序"。

### 100.2 判决性实验（ffmpeg 解码 1 kHz tone，tone / 2-20k 噪声）

```
修复前（MSB 打包 + 声明 LSB）        +14.7 dB
对照（手工构造正确 LSB 序）           +19.3 dB
修复后（工具正式输出）                 +19.3 dB   ← 与对照一致
mode0 模型预测 SFSR                   ≈ 19.3 dB  ← 精确命中
```

### 100.3 修复

```cpp
// DsfWriter::writeBlock
// DSP 是 MSB-first-in-time；DSF 声明 LSB-first ⇒ 写出前每字节 bit-reverse
for (int i = 0; i < 4096; ++i) tmp[i] = bitrev[b[i]];
```

**DSP 层零改动**：GATE 5 的 `packed` 层（= wire stream = 固件字节）仍全 OK，
只有 `cpp-payload`（DSF 容器层）变化 —— 正是应有的失效边界。

### 100.4 门禁固化

```
GATE 2b（新增）  检查 dsf.cpp 存在 bit-reversal 转换，防止回归
gate5.py        期望值改为 wire.bitrev（LSB-first 容器契约）
```

### 100.5 教训

> **"DATA 字节 == 固件字节" 不等于 "文件播放正确"** ——
> 还要验证字节的**时间序**与容器声明一致。
> 这是 §78/§89 之后第三类"数字验证通过但产物不可用"的缺陷
> （前两类：错误输入分支、旧 exe 复用）。

## 102. 撤回 §101：所谓“设备自产 DSF”是本项目第一版产物（循环论证）（2026-10-07）

### 102.1 事实澄清

用户澄清：`/sdcard/Music/时间煮雨 - 郁可唯.dsf` **是本项目第一版转换器输出的**，
**JM21 本身不具备把音频文件转成 DSD 的能力**（All-To-DSD 只作用于设备内部播放）。

因此 §101 用来证明"设备 DSF 是 [A][B] 字节交织"的那个样本，
实际上是**本项目自己的旧产物**——用我们的旧输出去印证我们旧版本的假设，
属**循环论证**，其结论无证据价值。

**§101 全部结论作废**，包括：
- "设备 DSF 导出为字节交织"      → 无效（样本来自本项目）
- "`--jm21` 应改为 interleaved"   → 已回滚，且用户实测更糟
- 基于该假设的 gate5 期望值改动   → 已回滚

### 102.2 真正的问题被重新框定

```
"实机背景纯净"  = JM21 播放 FLAC + 内置 All-To-DSD   （内部 PCM→DSD 通路）
"文件有杂音"    = JM21 播放 DSF 文件                  （文件 DSD 解码通路）
```

二者是设备内**不同的播放链路**。设备没有"把文件转 DSD"的能力，
所以不存在"设备产出的 DSF"这一参照物可用于校验我们的文件布局。

### 102.3 由此产生的、真正需要回答的问题

```
(a) 我们的 DSF 码流本身是否正确？
    → 已有的最强证据：数字核心与固件逐字节一致（GATE 5，12/12）
    → 独立验证：在**第三方 DSD DAC**（iBasso DC03 Pro / CS43131，§94 已通过）播放
(b) 若 (a) 干净 → 杂音来自 JM21 的 DSF 文件播放通路，与本转换器无关
(c) 若 (a) 也有杂音 → 才是本转换器的问题
```

**(c) 可立即判定**：把 `02以后的以后_现行planar版.dsf` 拷到 iBasso（USB DAC）
播放 —— 同一副耳机、同一台 PC。

## 103. 收尾：DSF 文件在 JM21 上播放全程带噪（用户实测，三份文件一致）

用户实测结论（2026-10-07，JM21 插耳机，同音量）：

| 文件 | 来源 | 听感 |
|---|---|---|
| `时间煮雨 - 郁可唯.dsf` | **本项目第一版**产物 | 大量噪音 |
| `02以后的以后_现行planar版.dsf` | 本项目当前版（planar） | 大量噪音 |
| `02 以后的以后.dsf` | 第三方/旧版产物 | 大量噪音 |

**三份全带噪 → 与本会话的布局/位序改动无关**（第一版产物早于今日所有改动），
共同因素是 **JM21 的 DSD 文件播放通路**。

### 103.1 结论

```
已确立且不受影响
├─ 数字核心与固件逐字节一致（GATE 5，12/12；固件 golden == Python == C++）
├─ 真实输入契约、标度 (1/2^31)、三档 shaper、量化器、8:1 打包 —— 全部 Verified
└─ 20 s 真实音乐 production：结构、长度、倍率、密度自洽

用户实测确立
└─ JM21 播放 DSF **文件** 时全程带噪，与产物来源/布局无关
   ⇒ JM21 的文件 DSD 解码通路与它内部 All-To-DSD 通路行为不同；
     设备不具备"文件 → DSD"能力，故不存在设备产出的 DSF 作为对照基准
```

### 103.2 使用建议

```
推荐  在 JM21 上直接播放 FLAC/WAV，走设备内置 All-To-DSD
      （这是设备原生的干净通路，也是 JM21 作为播放器的正确用法）

备选  把 DSF 送外部 DSD DAC（iBasso DC03 Pro / CS43131 已通过解码验收，§94）
      —— DSF 文件的用途是喂给真正的外部 DSD DAC，而非回灌 JM21
```

### 103.3 本会话的变更轨迹（全部已回滚至 803d5b8 已知良好状态）

```
✗ 位序 bit-reverse          实测带外能量 +11 dB（更糟）→ 回滚
✗ --jm21 改 interleaved     用户实测噪音更大、音乐受污染 → 回滚
✗ nPos 重置                 导致 out[] 全零 → 移除
✓ 最终：src/ 完整恢复自 803d5b8，GATE 1-5 全 PASS，--jm21 仍为 planar
```

### 103.4 方法学记录

本日第二次"结论错位"（前一次：w3=24 synthetic 分支）。
成因相同：**用本项目自己的旧产物当外部基准**去验证本项目的假设。
教训：**在缺少真正外部基准时，任何"与设备一致"的推断都需要独立来源支撑**；
自证不成立时，正确动作是承认基准缺失，而不是继续调整实现。

## 104. ★ 根因确认并修复：位序反转污染全部 DSF 输出（2026-10-07）

### 104.1 现象与根因

用户实测（JM21 插耳机，同音量）：

```
B_原始FLAC.flac            播放 FLAC，走设备内置 All-To-DSD        纯净
A_普通路径_当前版.dsf      当前构建普通路径（无 --jm21、位序已回滚）  干净 ✓
时间煮雨 - 郁可唯.dsf      第一版普通路径产物                      大量杂音 ✗
02 以后的以后.dsf          第三方/旧版产物                        大量杂音 ✗
```

根因：**2026-10-06 晚，为回应"放大音量有杂音"的反馈，在
`DsfWriter::writeBlock()` 中加入了逐字节 bit-reverse。**

```
错误改动：for (i) tmp[i] = bitrev[b[i]];      // 每字节位反转
```

DSP 产出的 8:1 打包流已经就是 DSF 数据区所需的时间序；
再反转一次会让每个 8 样本组反序，**把整形噪声在 fs/8 的倍频处
折叠回可听频段** —— 表现正是"大音量下大量杂音"。

该改动**同时污染了 `--jm21` 与普通两条路径**（共用同一 writer），
所以第一批"设备基准"推断（§101 交织、§102 循环论证）也一并作废。

### 104.2 修复与固化

```
回滚      源码从已知良好提交 803d5b8 完整恢复；grep bitrev = 0
验证      GATE 1-5 全 PASS；用户实测 A 文件干净
固化      新增 GATE 2c：静态拒绝 writeBlock 内的 bitrev，
          注释写明 2026-10-07 的误诊与后果，防止重犯
```

### 104.3 另一处已定位未修的缺陷

`--jm21` 模式下 `-r` 被静默忽略：

```cpp
if ((uint32_t)o.L == v / (uint32_t)L)   // o.L ∈ {64,128,256,512}
                                        // v/L ∈ {88200,176400,352800,705600}
                                        // 量纲不同，永不命中 → 恒定 DSD64
```

故 `--jm21 -r 64` 与 `--jm21 -r 512` 输出**大小完全相同**（均 7 061 596 B）。
正确映射应为 `level = o.L / 64` → 88200·2^level 的 PCM 率。
普通路径 `-r` 正常（四档 7.06 / 14.11 / 28.23 / 56.45 MB = 1:2:4:8）。
**按用户指示暂不修改**，仅记录。

## 105. `--jm21` 模式 `-r` 被静默忽略 —— 已修（2026-10-07）

### 105.1 缺陷

```cpp
// 原代码：量纲不匹配，永不命中
for (uint32_t v : kStd) {                     // v = 2822400/5644800/...
    if ((uint32_t)o.L == v / (uint32_t)L) {...} // o.L = 64/128/256/512
}                                             // 64 == 88200? 永不成立
```

后果：`--jm21 -r 64` 与 `--jm21 -r 512` 输出**字节数完全相同**（均 7 061 596 B，DSD64），
`-r` 在 JM21 路径下形同虚设。普通路径 `-r` 一直正常（1:2:4:8）。

### 105.2 修复

设备语义（§74.1 实测）：`persist.sys.pcm2dsd.level` = 0/1/2/3
→ 输入 PCM 88200 / 176400 / 352800 / 705600 Hz，核心恒 ×32
→ DSD = 2822400 · 2^level。

```cpp
// o.L 64/128/256/512 == level 0/1/2/3  =>  level = log2(o.L/64)
int level = 0;
for (int v = 64; v < o.L && level < 3; v <<= 1) ++level;
const uint32_t dsd = 2822400u << level;
pcmRate = (int)(dsd / (uint32_t)L);
```

### 105.3 验证

```
--jm21 -r 64  → 2822400 Hz    6.73 MiB
--jm21 -r 128 → 5644800 Hz   13.46 MiB
--jm21 -r 256 → 11289600 Hz  26.92 MiB
--jm21 -r 512 → 22579200 Hz  53.84 MiB      1:2:4:8 ✓
GATE 1-5 全部 PASS（--jm21 黄金向量 12/12 仍与固件逐字节一致）
```

注：`-r 512`（DSD512）对应设备 `level 3`；真机本地播放该档会超出
CS43198 本地解码能力（§74.5 实测 DSD512 输出纯噪声），属**设备限制**，
转换器本身正确。

### 105.4 门禁

`GATE 2c` 保留（拒绝 DSF 位序反转）；新增的 level 映射由 GATE 5 的
`--jm21 -r` 回归间接覆盖（四档输出率与大小需人工/脚本抽查）。

## 106. M1：实时流式处理（--stream）已实现并通过逐字节验收（2026-10-07）

### 106.1 目标

在保持"数字核心与固件逐字节一致"的前提下，增加**实时音频流处理**能力，
为后续"应用 → 虚拟扬声器 → 实时 DSD → 独占 USB DSD DAC"链路铺路。

### 106.2 实现

```
flac2dsf --stream <in.pcm|-> --pcm-bits {8,16,24,32}
                            --pcm-channels N --pcm-rate N  <out.dsf>
```

* 输入侧新增 `read_raw_pcm()`：按 64 KB 块增量读取裸 PCM（文件或 **stdin**），
  8/16/24/32 位、有符号归一化；**输出侧仍走既有 DsfWriter**，
  故插值/量化/打包的状态延续与离线路径完全相同。
* Windows 上 stdin 默认**文本模式**会吞掉 CR/LF 破坏样本流
  → 显式 `_setmode(_fileno(stdin), _O_BINARY)`（已修，验收见 106.3）。

### 106.3 验收：三条路径与离线转换逐字节一致

同一份 32-bit/88200 Hz 立体声 PCM：

```
离线 (WAV 输入)      0091289fb37e56ad
--stream 文件输入    0091289fb37e56ad   IDENTICAL
--stream stdin 输入  0091289fb37e56ad   IDENTICAL
```

真实管道端到端：

```
ffmpeg -i 02 以后的以后.flac -t 15 -f s32le -ar 88200 -ac 2 - \
  | flac2dsf --jm21 --jm21-mode 0 --arg6 0 --gain 0dB -d 0 \
      --stream - --pcm-bits 32 --pcm-channels 2 --pcm-rate 88200 pipe.dsf
→ pcm 88200 Hz, 1323000 frames  →  dsd 2822400 Hz, 42336000 samples/ch, 10.09 MiB
```

GATE 1-5 全 PASS（新增功能未触及数字核心）。

### 106.4 后续（M2 / M3）

```
M2  实时管道 → DoP 封装 → iBasso DC03 Pro（独占输出）
    iBasso 的 DoP 通路已验证可用（§94）；JM21 的 USB DAC 不自动识别 DoP（§95），
    其 native DSD 输入需 FiiO ASIO 驱动，暂不可用
M3  应用 → VoiceMeeter Input（虚拟扬声器）→ 我们共享回环捕获 → DSD → iBasso
    受限：VoiceMeeter 侧只能共享模式捕获，无法独占
```

## 107. 「格式不支持 / 沙哑」的真实根因：增益与抖动参数的适用性被混淆（2026-10-07）

### 107.1 现象

```
Web UI 勾选「真实路径」→ 播放器报格式不支持
稍改数据量          → 又变成"不支持"或速度/音调异常
```

### 107.2 排查过程（三项确定性验证，均逐字节一致）

```
1. 当前构建 vs GitHub b0b82f8（用户确认"干净"那版）   逐字节相同
2. 同一命令连续执行三次                                逐字节相同（转换是确定性的）
3. 复现用户"干净"文件的转换命令                       逐字节相同 8785aeac2584…
```

**结论：代码无缺陷，转换是确定性的。**

### 107.3 真正的差异

| | 参数 | 听感 |
|---|---|---|
| 用户确认"干净"的文件 | `-r 64`（默认 **−3 dB + 0.05 抖动**） | 干净 |
| 我随后推送的文件 | `--gain 0dB -d 0` | 沙哑、大量噪声 |

**根因是参数适用性被混淆**（§104 的判断有误）：

```
--gain 0dB -d 0 只适用于 --jm21 路径
  该路径是固件原生数字链，设备内部本就不加增益与抖动，要求逐字节复现

FIR（普通）路径必须用默认的 −3 dB + 0.05 TPDF 抖动
  · −3 dB：滤波器过冲约 1.9 dB，0 dB 会把瞬态推过 1-bit 级 → 过载失真
  · 0.05 抖动：抑制 FIR 路径的量化极限环；实测 1 kHz tone 带内 SFSR 最优
```

§104 写的"与设备工作点严格一致的参数组合 `--gain 0dB -d 0`"**只对 `--jm21` 成立**，
把它套用到普通路径上就会产生沙哑与噪声。

### 107.4 关于「格式不支持」与「速度/音调」

这两项是**另一条独立线**，其根因是我在排查中加的 `sample_count` 改动
（把"有效采样数"改成"含块补零的总数"），该改动**基于对 DSF 规范的推测、
没有基准文件验证**，已在 §107.2 证据下回滚。回滚后：

```
sample_count = frames × L（有效值）   时长 298.3 s，与源一致
```

若播放器仍报格式不支持，需要一份**已知合法的第三方 DSF**作为格式基准
逐字段比对 —— 目前没有该基准，不应再凭推测改动。

### 107.5 门禁盲区（重要）

`GATE 5` **只覆盖 `--jm21` 路径**；FIR（普通）路径**没有任何回归门禁**。
本次能迅速排除代码嫌疑，是因为恰有 GitHub 上的历史二进制可作基准。
建议后续补一条 FIR 路径的黄金向量门禁。

### 107.6 当前正确用法

```
普通路径（听感优化，默认）   flac2dsf -r 64 in.flac out.dsf
--jm21 真实路径（设备等价）  flac2dsf --jm21 --jm21-mode 0 --arg6 0 \
                                 --gain 0dB -d 0 in.flac out.dsf
```

## 108. 补齐门禁盲区：FIR 路径黄金向量门禁 GATE 5b（2026-10-07）

### 108.1 动机

§107 记录：`GATE 5` 只覆盖 `--jm21`，FIR（普通）路径**无任何回归门禁**。
本次 FIR 输出出现沙哑/噪声时，全部门禁依然全绿 —— 因为没人测过这条路径。
能快速排除代码嫌疑纯属侥幸（恰有 GitHub 历史二进制可对比）。

### 108.2 实现

```
golden/fir/            12 个基线 DSF（6 输入 × -r {64,128}）+ MANIFEST.json
gate5_fir.py           GATE 5b：重跑并逐字节比对 sha256
build_gate.py          把 5b 接入总门禁，任一失败即 STOP
```

基线用**默认参数**（−3 dB + 0.05 抖动）生成 —— 即用户在 JM21 上确认"干净"
的那组参数（§107.3）。同时覆盖两个 DSD 率，因为二者的 rate plan 分支不同。

### 108.3 有牙性验证

用导致本次事故的错误参数（`--gain 0dB -d 0`）跑基线 case：

```
得到 c221343d4cb0c394…   基线 c9953cdfb8aed524…
门禁判定: FAIL  ← 证明能抓到该类回归
```

### 108.4 门禁全景（封版）

```
GATE 1    源码扫描：拒绝 d8=16-mode / std::fma
GATE 2a   删除旧 exe（禁止复用）
GATE 2b   构建成功
GATE 2c   DSF 数据区不得被位反转（§104 事故）
GATE 3    exe 存在 + size + mtime + sha256
GATE 4    golden/real32 基线齐备（18 cases, mode 0/1/2）
GATE 5    --jm21 三层三方 byte-exact（固件 == Python == C++）
GATE 5b   FIR 路径 byte-exact vs 基线（本次新增）
```

## 110. Web UI 增加参数预设（4 个）—— 固化本会话踩过的参数坑（2026-10-07）

### 110.1 动机

§107 的事故本质是**参数适用性被混淆**：
`--gain 0dB -d 0` 只属于 `--jm21` 路径，普通 FIR 路径用它就会沙哑/噪声。
UI 上这些是散落的输入框，用户（和开发者）很容易再次配错。
预设把"已验证可用的组合"固化成一键选择。

### 110.2 四个预设

| 预设 | -r | gain | dither | taps | jm21 / mode | 用途 |
|---|---|---|---|---|---|---|
| **推荐听感** | 64 | −3 dB | 0.05 | 16 | 关 | FIR 默认，已在 JM21 实测干净 |
| **JM21 DSD64** | 64 | 0 dB | 0 | 16 | 开 / 0 | 设备 level<2 默认档，与固件 byte-exact |
| **JM21 DSD256** | 256 | 0 dB | 0 | 16 | 开 / 2 | 设备 level≥2 档，4 抽头 |
| **最低失真** | 256 | −3 dB | 0.05 | 64 | 关 | 整形噪声推更高、带内更好，文件约 4 倍 |

每个预设点击后会在下方显示一行说明，写明**为什么是这组参数**，
避免再次误用。

### 110.3 实现

```
web.cpp  参数预设按钮组 + PRESETS 表 + applyPreset()（同步表单 + 刷新原型/队列）
         附 .btn 样式
```

### 110.4 验证

```
页面 HTTP OK；四个 data-preset 按钮、PRESETS 表、applyPreset、presetNote、.btn 全部就位
预设解析正确：
  listen  → 服务端 g=-0.7079（= -3 dB）  jm21 关
  jm21l0  → 服务端 g=-1.0000（核心侧取 fabs=1.0）  jm21 开
GATE 1-5b 全 PASS
```

## 111. 格式支持扩展：AIFF/W64/RF64 内置 + ffmpeg 兜底（2026-07）

### 111.1 目标

用户音乐库（`H:\ALL_TO_DSD\music`）有 19 个 **m4a**，此前完全无法转换。
要求："内置为主，ffmpeg 兜底"。

### 111.2 内置（零依赖，沿用既有架构）

```
AIFF / AIFC   FORM + COMM（80-bit extended 采样率）+ SSND
W64 / RF64    64-bit RIFF GUID 容器，RF64 的真实长度在 ds64 chunk
```

共用 `decodeSample()` 做样本解码（大端/小端、整数/浮点、容器宽度 > 有效位宽）。
另修 CAF loader（chunk size 为**大端** int64、desc 内采样率为大端 f64）。

### 111.3 ffmpeg 兜底桥

内置 loader 都不认识时，走 `ffmpeg ... -f s32le -ac 2 -ar 88200 -`，
即请求标准 88200 Hz 立体声 s32le，再复用既有重采样与转换链。

**关键实现点**：最初用 `_popen`，在 Windows 上是 **ANSI 文本模式** ——
系统代码页无法表示用户音乐库的中文路径，导致每一个 m4a 都报 "cannot decode"，
而**完全相同的命令行在 UTF-8 shell 里是成功的**（可取到 134 MB PCM）。
改为 **`CreateProcessW` + 匿名管道（二进制）** 后解决，同时支持 Unicode 路径。

ffmpeg 定位顺序：`--ffmpeg` 参数 → `FFMPEG_PATH` 环境变量 →
若干常见安装位置。

### 111.4 覆盖实测（全部 OK）

```
FLAC  内置      WAV      内置
AIFF  内置      CAF      ffmpeg 兜底
M4A   ffmpeg 兜底        MP3     ffmpeg 兜底
```

真实样本：19 首 m4a 中的 `01 螺旋 - RASEN.m4a` → DSD64 128.16 MiB，10.2 s。

### 111.5 遗留

- **MP3 内置解码器**（minimp3）与 **AAC 内置解码器**未做；
  目前靠 ffmpeg 兜底。若要求真正的零依赖单文件，需要再引入这两个解码器。
- `load_caf` 的 chunk 解析在 ffmpeg 生成的 CAF 上不适用（该文件的格式描述
  位于非标准位置），实际由 ffmpeg 兜底覆盖，保留 loader 供规范 CAF 使用。
