+++
title = 'Shader Compilation: From Source to Silicon'
date = '2026-06-02T00:00:00-08:00'
draft = false
+++

# Shader Compilation: From Source to Silicon

When you write a shader, the code you type is about as far from GPU execution as Java source is from x86 machine code. Between your `float4 main()` and actual transistors switching, there's a multi-stage pipeline involving language front-ends, portable intermediate representations, driver-level optimizing compilers, and kernel-mode memory allocation.

This post covers the general pipeline first, then follows a concrete example all the way from GLSL source to AMD GCN ISA — using real tool output at each step.

---

## Part 1: The General Pipeline

### Stage 1 — Source Parsing and Preprocessing

The compiler starts by reading shader source. A preprocessor handles `#include` directives, `#define` macros, and conditional compilation (`#ifdef USE_NORMAL_MAP`). The result is a single expanded translation unit with all includes resolved and macros substituted.

This is where **shader permutations** originate. A single `.hlsl` file with 10 boolean `#define` flags can produce up to 1024 distinct translation units before a single line of actual compilation happens. MJP has [written extensively](https://therealmjp.github.io/posts/shader-permutations-part1/) about why this combinatorial explosion is one of the harder unsolved problems in real-time rendering tooling.

### Stage 2 — Lexing, Parsing, and AST Construction

The preprocessed source is tokenized into a stream of tokens — keywords, identifiers, operators, literals. The parser consumes this stream and constructs an **Abstract Syntax Tree (AST)**: a hierarchical data structure representing the program's syntactic structure.

At this stage:

- **Type checking** — does this operation make sense for these types?
- **Semantic validation** — are all variables declared? Are return types correct?
- **Constant folding** — `3.0 * 4.0` becomes `12.0` in the AST

Tools: DXC (DirectX Shader Compiler) for HLSL, glslang for GLSL, clang for MSL.

### Stage 3 — IR Generation

The AST is lowered into an **Intermediate Representation (IR)**. This is the central artifact of the compilation pipeline — a structured, analyzable representation that is portable across hardware.

Key IRs in the graphics world:

| API | IR Format | Notes |
|---|---|---|
| Vulkan | **SPIR-V** | Binary, 32-bit word stream, SSA form, defined by Khronos |
| DX12 | **DXIL** | LLVM 3.7 bitcode + DX metadata, validated by `DXIL.dll` |
| Metal | **AIR** | Apple's LLVM IR bitcode, stored in `.metallib` containers |

Worth being precise about terminology here: these are not bytecode in the Java or Python sense. Bytecode implies a virtual machine that interprets a linear instruction stream. SPIR-V, DXIL, and AIR are **serialized compiler IRs** — data structures encoding types, SSA values, and operations that a backend compiler consumes. No driver ever "interprets" SPIR-V; it always compiles it.

#### SSA Form

Static Single Assignment means every variable is assigned exactly once. If a value changes, a new variable is created. This makes data flow analysis straightforward: you always know where a value came from without alias analysis.

```
; Non-SSA
x = 1
x = x + 2

; SSA equivalent
x1 = 1
x2 = x1 + 2
```

SPIR-V's SSA encoding is one reason driver compilers can optimize shader code aggressively — the data flow graph is explicit in the binary.

### Stage 4 — Mid-Level Optimizations

Before hardware-specific lowering, optimization passes run on the IR:

- **Dead code elimination** — unreachable branches, outputs never consumed by the next stage
- **Constant propagation** — if `x = 4.0` is known at compile time, every use of `x` becomes `4.0`
- **Loop unrolling** — small loops over fixed iteration counts get unrolled; relevant for texture fetch loops
- **Function inlining** — virtually all shader function calls are inlined. GPUs have no hardware call stack; implementing one via simulated stack in registers is expensive enough that inlining is less an optimization and more just how it works
- **Algebraic simplification** — `x * 1.0`, `x + 0.0`, `pow(x, 1.0)` → `x`

### Stage 5 — Driver-Level JIT Compilation

This is where portability ends. The driver's compiler receives the portable IR and compiles it to the actual GPU's **Instruction Set Architecture (ISA)** — the binary machine code the hardware executes.

This happens almost entirely in **user space** (inside the driver's usermode `.dll` or `.so`). The kernel mode driver only receives the final ISA binary, validates it lightly, allocates GPU-accessible memory, and copies it in. All the heavy lifting — register allocation, instruction scheduling, ISA emission — is done in usermode.

```
User space:   SPIR-V → driver IR → optimizations → register alloc → ISA bytes
                                                                          |
                                                                    ioctl() call
                                                                          |
Kernel space: validate → alloc GPU memory → copy ISA → record address
```

The kernel separation is intentional: letting the kernel run an arbitrary optimizing compiler would be a large attack surface, and a driver update shipping a new `.so` without touching the kernel module is much cleaner.

This is also why **PSO (Pipeline State Object) compilation** has measurable latency. Register allocation is NP-hard in the general case. A complex shader encountered for the first time may take 50–200ms to compile. In a title with thousands of shader permutations this is the source of first-run stutter.

Mitigation mechanisms:
- `VkPipelineCache` (Vulkan) — serialize compiled ISA to disk, reuse on subsequent runs
- `ID3D12PipelineLibrary` (DX12) — same
- `MTLBinaryArchive` (Metal) — same
- In-game "shader compilation" screens — burning through PSO compilation ahead of time to populate caches

### Stage 6 — PSO Linking and Fixup

For rasterization pipelines, compiled vertex and fragment shaders must be **linked** — the output signature of the VS must match the input signature of the FS. Input/output locations, interpolation qualifiers, and built-ins are reconciled. The driver may patch the ISA at this stage (e.g. adjusting export targets).

---

## Part 2: Following One Shader From Source to ISA

All output below was produced by running real tools. The toolchain:

- **glslang 15.1.0** — GLSL → SPIR-V
- **spirv-tools 2025.1** — SPIR-V validation and optimization (`spirv-opt`, `spirv-dis`)
- **LLVM 18 / llc** — SPIR-V-equivalent LLVM IR → AMDGCN ISA (`-mcpu=gfx1030`, RDNA2)

The example shader:

```glsl
#version 450

layout(location = 0) in vec2 inUV;
layout(set = 0, binding = 0) uniform sampler2D albedoTex;
layout(location = 0) out vec4 outColor;

void main() {
    vec4 color = texture(albedoTex, inUV);
    outColor = color * vec4(1.0, 0.8, 0.6, 1.0);
}
```

Simple on purpose: one texture sample, one vec4 multiply. Enough to see each stage clearly without noise.

---

### Step 1 — GLSL → SPIR-V

```bash
glslangValidator -V shader.frag -o shader.spv
```

732 bytes of binary. Disassembled with `spirv-dis`:

```spirv
; SPIR-V
; Version: 1.0
; Generator: Khronos Glslang Reference Front End; 11
; Bound: 28
; Schema: 0
               OpCapability Shader
          %1 = OpExtInstImport "GLSL.std.450"
               OpMemoryModel Logical GLSL450
               OpEntryPoint Fragment %main "main" %inUV %outColor
               OpExecutionMode %main OriginUpperLeft
               OpSource GLSL 450
               OpName %main "main"
               OpName %color "color"
               OpName %albedoTex "albedoTex"
               OpName %inUV "inUV"
               OpName %outColor "outColor"
               OpDecorate %albedoTex Binding 0
               OpDecorate %albedoTex DescriptorSet 0
               OpDecorate %inUV Location 0
               OpDecorate %outColor Location 0
       %void = OpTypeVoid
          %3 = OpTypeFunction %void
      %float = OpTypeFloat 32
    %v4float = OpTypeVector %float 4
%_ptr_Function_v4float = OpTypePointer Function %v4float
         %10 = OpTypeImage %float 2D 0 0 0 1 Unknown
         %11 = OpTypeSampledImage %10
%_ptr_UniformConstant_11 = OpTypePointer UniformConstant %11
  %albedoTex = OpVariable %_ptr_UniformConstant_11 UniformConstant
    %v2float = OpTypeVector %float 2
%_ptr_Input_v2float = OpTypePointer Input %v2float
       %inUV = OpVariable %_ptr_Input_v2float Input
%_ptr_Output_v4float = OpTypePointer Output %v4float
   %outColor = OpVariable %_ptr_Output_v4float Output
    %float_1 = OpConstant %float 1
%float_0_800000012 = OpConstant %float 0.800000012
%float_0_600000024 = OpConstant %float 0.600000024
         %26 = OpConstantComposite %v4float %float_1 %float_0_800000012 %float_0_600000024 %float_1
       %main = OpFunction %void None %3
          %5 = OpLabel
      %color = OpVariable %_ptr_Function_v4float Function
         %14 = OpLoad %11 %albedoTex
         %18 = OpLoad %v2float %inUV
         %19 = OpImageSampleImplicitLod %v4float %14 %18
               OpStore %color %19
         %22 = OpLoad %v4float %color
         %27 = OpFMul %v4float %22 %26
               OpStore %outColor %27
               OpReturn
               OpFunctionEnd
```

A few things worth noting:

**The binary encoding.** Each instruction is one or more 32-bit words. The first word packs `[word_count:16 | opcode:16]`. The file opens with magic number `0x07230203`. The `Bound: 28` header field is the highest SSA ID used — the entire shader fits in 28 IDs.

**Debug names are optional metadata.** `OpName %color "color"` and friends give human-readable names to SSA IDs. These survive into the debug binary but get stripped by `spirv-opt --strip-debug`.

**Decorations carry binding metadata.** `OpDecorate %albedoTex Binding 0` and `DescriptorSet 0` are annotations that sit outside the instruction stream. The driver reads these to wire up descriptor sets — they don't affect the execution logic.

**The `%color` variable is a `Function`-storage-class pointer** — a local variable in function scope. Notice the pattern: `OpStore %color %19` writes the sample result, then `OpLoad %v4float %color` reads it back immediately before the multiply. The next step eliminates this.

*Validation:*
```bash
spirv-val shader.spv
# → (no output, exit 0)
```

---

### Step 2 — SPIR-V Optimization

```bash
spirv-opt --strip-debug --eliminate-dead-code-aggressive \
          --eliminate-local-single-block \
          --eliminate-local-single-store \
          --simplify-instructions \
          -O shader.spv -o shader_opt.spv
```

572 bytes — 160 bytes smaller. Disassembly:

```spirv
; SPIR-V
; Version: 1.0
; Generator: Khronos Glslang Reference Front End; 11
; Bound: 28
               OpCapability Shader
          %1 = OpExtInstImport "GLSL.std.450"
               OpMemoryModel Logical GLSL450
               OpEntryPoint Fragment %4 "main" %17 %21
               OpExecutionMode %4 OriginUpperLeft
               OpDecorate %13 Binding 0
               OpDecorate %13 DescriptorSet 0
               OpDecorate %17 Location 0
               OpDecorate %21 Location 0
       %void = OpTypeVoid
          %3 = OpTypeFunction %void
      %float = OpTypeFloat 32
    %v4float = OpTypeVector %float 4
         %10 = OpTypeImage %float 2D 0 0 0 1 Unknown
         %11 = OpTypeSampledImage %10
%_ptr_UniformConstant_11 = OpTypePointer UniformConstant %11
         %13 = OpVariable %_ptr_UniformConstant_11 UniformConstant
    %v2float = OpTypeVector %float 2
%_ptr_Input_v2float = OpTypePointer Input %v2float
         %17 = OpVariable %_ptr_Input_v2float Input
%_ptr_Output_v4float = OpTypePointer Output %v4float
         %21 = OpVariable %_ptr_Output_v4float Output
    %float_1 = OpConstant %float 1
%float_0_800000012 = OpConstant %float 0.800000012
%float_0_600000024 = OpConstant %float 0.600000024
         %26 = OpConstantComposite %v4float %float_1 %float_0_800000012 %float_0_600000024 %float_1
          %4 = OpFunction %void None %3
          %5 = OpLabel
         %14 = OpLoad %11 %13
         %18 = OpLoad %v2float %17
         %19 = OpImageSampleImplicitLod %v4float %14 %18
         %27 = OpFMul %v4float %19 %26
               OpStore %21 %27
               OpReturn
               OpFunctionEnd
```

What changed: the `%color` local variable is gone. The `OpStore %color %19` / `OpLoad %v4float %color` round-trip was eliminated by `--eliminate-local-single-store` — `%19` (the sample result) feeds directly into `OpFMul`. The debug names (`OpName`) are gone via `--strip-debug`. The function body went from 8 instructions to 6.

This is the SPIR-V that a Vulkan driver would receive at `vkCreateGraphicsPipelines`.

---

### Step 3 — AMDGCN ISA via LLVM

The SPIR-V → LLVM IR translation for Vulkan GLSL SPIR-V requires a full driver stack (Mesa's `spirv_to_nir` + ACO). To show the ISA step in isolation we compile the equivalent arithmetic directly as LLVM IR and feed it to `llc`:

```bash
llc-18 -march=amdgcn -mcpu=gfx1030 -filetype=asm shader.ll -o shader_gfx1030.s
```

Target: `gfx1030` = RDNA2 (RX 6000 series). Output (instruction section only):

```asm
shader_arithmetic:
    s_clause 0x1
    s_load_dwordx4 s[0:3], s[4:5], 0x8       ; load r,g,b,a from kernel args
    s_load_dwordx2 s[4:5], s[4:5], 0x0        ; load output pointer
    v_mov_b32_e32 v4, 0                        ; offset = 0 for store
    s_waitcnt lgkmcnt(0)                       ; wait for SGPR loads
    v_mul_f32_e64 v1, 0x3f4ccccd, s1          ; g * 0.8
    v_mul_f32_e64 v2, 0x3f19999a, s2          ; b * 0.6
    v_mov_b32_e32 v0, s0                       ; r (move to VGPR, * 1.0 eliminated)
    v_mov_b32_e32 v3, s3                       ; a (move to VGPR, * 1.0 eliminated)
    global_store_dwordx4 v4, v[0:3], s[4:5]   ; store all 4 channels at once
    s_endpgm
```

Compiler metadata at the bottom of the `.s` file:
```
; NumSgprs: 6
; NumVgprs: 5
; ScratchSize: 0
; Occupancy: 16
```

A few things worth unpacking:

**The `* 1.0` multiplications were eliminated.** We started with four `fmul` instructions. Only two `v_mul_f32_e64` appear — G (`0x3f4ccccd` = 0.8) and B (`0x3f19999a` = 0.6). R and A became plain `v_mov_b32_e32` — a register move with no arithmetic. The LLVM backend recognized `x * 1.0` as a no-op under `fast` math flags and lowered it to a mov.

**Verify the constants.** The hex values in the ISA are IEEE 754 single-precision:
```
0x3f4ccccd = 0.800000012  (closest f32 to 0.8)
0x3f19999a = 0.600000024  (closest f32 to 0.6)
```
These match the `OpConstant` values in the SPIR-V exactly.

**`s_clause` + `s_load` + `s_waitcnt`.** The two scalar loads (`s_load_dwordx4`, `s_load_dwordx2`) are grouped by `s_clause 0x1` (telling the hardware they're a pair) and followed by `s_waitcnt lgkmcnt(0)` — stall until both LDS/SGPR memory operations complete. This is the latency hiding pattern: issue scalar loads early, do other work, then wait only when you actually need the result. Here there's not much independent work to fill the gap, but in a real PBR shader there would be.

**`global_store_dwordx4`** writes all four channels in a single 128-bit store — the backend merged the four individual stores into one vector operation.

**5 VGPRs, 6 SGPRs, occupancy 16.** This is about as low as register pressure gets. At 5 VGPRs on RDNA2, the hardware can run the maximum 16 waves per compute unit simultaneously.

*Reference: [AMD RDNA2 ISA Reference Manual — GPUOpen](https://gpuopen.com/amd-rdna2-shader-instruction-set-architecture/)*  
*Reference: [AMD RDNA3 ISA Reference Manual — GPUOpen](https://gpuopen.com/amd-rdna3-shader-instruction-set-architecture/)*

---

### Caveat on the ISA Step

The LLVM path above uses the `amdgpu_kernel` (compute) calling convention to stay within what `llc` can compile standalone. A real fragment shader uses `amdgpu_ps`, goes through Mesa's `spirv_to_nir` + ACO pipeline, and looks different: inputs arrive as interpolated varyings via `v_interp_p1_f32` / `v_interp_p2_f32`, and the output uses `exp` (export) instructions rather than a global store. The arithmetic core — the `v_mul_f32` instructions — is the same.

To capture real fragment shader ISA from the full RADV pipeline, set `RADV_DEBUG=shaders` before launching a Vulkan application. Mesa will dump the ISA for every compiled shader to stderr.

---

## The Full Journey

```
shader.frag (GLSL source, 7 lines)
    ↓  glslangValidator -V
shader.spv (SPIR-V binary, 732 bytes, 29 instructions)
    ↓  spirv-opt -O --strip-debug ...
shader_opt.spv (SPIR-V optimized, 572 bytes, 6 instructions in main)
    ↓  [driver: spirv_to_nir → NIR passes → ACO]
    ↓  [shown here via: llc -march=amdgcn -mcpu=gfx1030]
shader_gfx1030.s (AMDGCN ISA, 10 instructions, 5 VGPRs, occupancy 16)
```

---

## A Few Things Worth Taking Away

**SPIR-V/DXIL/AIR are not bytecode.** They're serialized compiler IRs. No driver interprets them; they're always compiled to hardware ISA. The industry uses "shader bytecode" loosely and it's technically wrong.

**All shader functions are inlined.** GPUs have no hardware call stack. Inlining is the execution model, not an optimization over it.

**The optimizer sees through `* 1.0`.** Even with four explicit `fmul` instructions in the LLVM IR, the backend emitted only two. Algebraic simplification is not just a mid-level pass — it happens during instruction selection too.

**Driver JIT compilation is expensive.** Register allocation is NP-hard in the general case. PSO compilation latency is real and measurable. Pipeline caches exist to amortize this cost across runs.

**Register pressure determines occupancy.** Fewer VGPRs per wave means more waves in flight, which is how GPUs hide memory latency. This shader used 5 — as low as it gets.

**Driver compilation is entirely user space.** The kernel only places the final ISA binary into GPU memory.

---

## References

### Specifications
- [SPIR-V Specification — Khronos Registry](https://registry.khronos.org/SPIR-V/specs/unified1/SPIRV.html)
- [DXIL Specification — Microsoft / DirectXShaderCompiler](https://github.com/microsoft/DirectXShaderCompiler/blob/main/docs/DXIL.rst)
- [AMD RDNA2 ISA Reference Manual — GPUOpen](https://gpuopen.com/amd-rdna2-shader-instruction-set-architecture/)
- [AMD RDNA3 ISA Reference Manual — GPUOpen](https://gpuopen.com/amd-rdna3-shader-instruction-set-architecture/)

### Mesa / RADV / ACO
- [Mesa NIR Documentation](https://docs.mesa3d.org/nir/index.html)
- [Mesa source: spirv_to_nir.c](https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/compiler/spirv/spirv_to_nir.c)
- [Mesa source: ACO backend](https://gitlab.freedesktop.org/mesa/mesa/-/tree/main/src/amd/compiler)
- [ACO: A New Compiler Backend for AMD GPUs — Marek Olšák, XDC 2019](https://xdc2019.x.org/event/5/contributions/334/)

### Related Reading
- [The Shader Permutation Problem — MJP](https://therealmjp.github.io/posts/shader-permutations-part1/)
- [A Quick Note on Shader Compilers — MJP](https://therealmjp.github.io/posts/a-quick-note-on-shader-compilers/)
- [AMD RDNA2 Performance Guide — GPUOpen](https://gpuopen.com/performance/)
- [Vulkan Pipeline Cache — Khronos Blog](https://www.khronos.org/blog/vulkan-pipeline-caching)
- [Fossilize: Vulkan PSO cache serialization — Valve](https://github.com/ValveSoftware/Fossilize)
- [A Trip Through the Graphics Pipeline 2011 — Fabian Giesen](https://fgiesen.wordpress.com/2011/07/09/a-trip-through-the-graphics-pipeline-2011-index/)

### Tooling
- [DirectXShaderCompiler (DXC)](https://github.com/microsoft/DirectXShaderCompiler)
- [glslang](https://github.com/KhronosGroup/glslang)
- [SPIRV-Tools (spirv-dis, spirv-opt, spirv-val)](https://github.com/KhronosGroup/SPIRV-Tools)
- [spirv-cross](https://github.com/KhronosGroup/SPIRV-Cross)
- [LLVM AMDGPU backend](https://llvm.org/docs/AMDGPUUsage.html)

### Books
- *Engineering a Compiler* — Cooper & Torczon (2nd ed.)
- *Computer Organization and Design* — Patterson & Hennessy
- *Real-Time Rendering* (4th ed.) — Akenine-Möller et al.

### Kernel
- [Linux amdgpu driver source](https://github.com/torvalds/linux/tree/master/drivers/gpu/drm/amd/amdgpu)
