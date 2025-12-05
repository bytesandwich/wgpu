# Naga WGSL Bundling and Module Composition Capabilities

This document summarizes the current state of naga's support for shader modularity, bundling, and code organization features.

## Executive Summary

**Naga itself does not currently provide built-in import/include directives, module linking, or shader bundling capabilities.** The WGSL specification that naga implements does not include a module system or preprocessor. However, there are external solutions and potential extension points that enable these features.

## Current Naga Architecture

### Core Module Structure

Naga's IR is organized around the `Module` struct (`naga/src/ir/mod.rs`), which contains:
- `types`: Arena of type definitions
- `constants`: Arena of constant values
- `overrides`: Arena of pipeline-overridable constants
- `global_variables`: Arena of global variables
- `global_expressions`: Arena of constant/override expressions
- `functions`: Arena of function definitions
- `entry_points`: Vector of entry point definitions

Each module is a self-contained compilation unit with no built-in mechanism for referencing or importing from other modules.

### Extension Mechanisms

Naga does provide extension points via WGSL directives:

1. **Enable Extensions** (`enable <extension>;`)
   - `f16` - Half-precision floating point support
   - `dual_source_blending` - Dual source blending
   - `clip_distances` - Clip distance arrays
   - `wgpu_mesh_shader` - Mesh shader support (wgpu-specific)
   - `wgpu_ray_query` - Ray query support (wgpu-specific)

2. **Language Extensions** (`requires <extension>;`)
   - `readonly_and_readwrite_storage_textures`
   - `packed_4x8_integer_dot_product`
   - `pointer_composite_access`

These extensions do **not** provide import/include functionality.

### GLSL Preprocessor Support

For GLSL input (not WGSL), naga uses `pp_rs` for preprocessing, which supports:
- `#define` macros
- `#include` directives
- Conditional compilation (`#ifdef`, `#ifndef`, etc.)

This is only available for GLSL frontend, not WGSL.

## External Solutions for Shader Modularity

### 1. naga_oil (Recommended for Bevy/Rust Projects)

[naga_oil](https://github.com/bevyengine/naga_oil) is the primary external solution for WGSL shader composition. It provides:

**Features:**
- Module imports with path syntax: `#import my_module::function`
- Module definitions: `#define_import_path my_module`
- Preprocessor-style directives for conditional compilation
- Function and type sharing across modules
- Virtual/override functions

**Example syntax:**
```wgsl
#define_import_path my_module

fn my_func() -> f32 { return 1.0; }
```

```wgsl
#import my_module;

fn main() -> f32 {
    return my_module::my_func();
}
```

**Limitations:**
- Uses regex/simple lexer for preprocessing (not naga's lexer)
- Separate tool from naga proper
- Import resolution happens before naga parsing

### 2. WESL (WebGPU Extended Shader Language)

[WESL](https://wesl-lang.dev) is an emerging standard for WGSL extensions:

**Features:**
- Standardized import syntax: `import my::module::{ item1, item2 };`
- Package management concepts
- Language server support

**Status:** Under active development, being considered for Bevy's future shader system.

### 3. Manual Concatenation

The simplest approach is to concatenate WGSL source files before parsing:

```rust
let common = std::fs::read_to_string("common.wgsl")?;
let main = std::fs::read_to_string("main.wgsl")?;
let combined = format!("{}\n{}", common, main);
let module = naga::front::wgsl::parse_str(&combined)?;
```

**Limitations:**
- No namespace isolation
- Name collision risks
- Manual dependency ordering

## Programmatic Module Composition

While naga doesn't have built-in linking, its Rust API allows programmatic module manipulation:

### Using naga's IR directly

```rust
use naga::{Module, Function, Handle};

// Parse multiple modules
let module_a = naga::front::wgsl::parse_str(source_a)?;
let module_b = naga::front::wgsl::parse_str(source_b)?;

// Manually merge (requires careful handle remapping)
let mut combined = Module::default();
// ... copy types, functions, etc. with handle translation
```

**Challenges:**
- Handles are arena-specific and must be remapped
- Type deduplication is complex
- No built-in merge functionality

### Module Compaction

Naga provides compaction (`naga::compact::compact()`) which:
- Removes unused declarations
- Can filter to a specific entry point
- Useful for dead code elimination after manual merging

## Relevant GitHub Issues

1. **[#6250](https://github.com/gfx-rs/wgpu/issues/6250)** - Request to expose lexer/parser for tools like naga_oil
2. **[#5713](https://github.com/gfx-rs/wgpu/issues/5713)** - Request for compile-with-context for module linking (closed)
3. **[#4429](https://github.com/gfx-rs/wgpu/issues/4429)** - Discussion on error API for preprocessor tooling

## Recommendations

### For Production Use

1. **Use naga_oil** if you need import/module functionality today, especially with Bevy
2. **Watch WESL development** for standardized future solutions
3. **Simple concatenation** works for basic use cases with careful naming

### For Test Harness Separation

To separate test code from production shaders:

1. **Preprocessor approach** (requires external tool):
   ```wgsl
   #ifdef TEST
   fn test_helper() { ... }
   #endif
   ```

2. **Multiple entry points** (built-in):
   ```wgsl
   @compute @workgroup_size(1)
   fn production_main() { ... }
   
   @compute @workgroup_size(1)  
   fn test_main() { ... }
   ```
   Use naga's `--entry-point` and `--compact` flags to extract specific entry points.

3. **Module splitting** (with naga_oil):
   ```wgsl
   // production.wgsl
   #define_import_path production
   #import common;
   
   // test.wgsl  
   #define_import_path test
   #import common;
   #import production;  // import production functions to test
   ```

## CLI Usage for Module Processing

```bash
# Validate a shader
naga shader.wgsl

# Extract specific entry point with dead code elimination
naga shader.wgsl output.wgsl --entry-point main --compact

# Convert to different formats
naga shader.wgsl output.spv  # SPIR-V
naga shader.wgsl output.metal  # Metal
naga shader.wgsl output.hlsl  # HLSL
```

## Conclusion

Naga is designed as a shader translation library, not a build system or module bundler. For module composition needs, external tools like naga_oil provide the necessary functionality. The WGSL specification itself does not include imports, and naga faithfully implements this specification while providing extension points for wgpu-specific features.

Future developments in WESL may provide standardized import syntax, but for now, projects requiring shader modularity should use naga_oil or implement custom preprocessing solutions.
