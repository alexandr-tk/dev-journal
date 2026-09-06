---
project: Blender
tags: [c++, rna, python-api, open-source, nodes, compositor, geometry nodes]
status: WIP
---

# Porting the Integer Math Node from Geometry Nodes to the Compositor Prototype

**The Result:** [PR #155154](https://projects.blender.org/blender/blender/pulls/155154)

## 1. Workflow

- **Task:** Create a functional prototype of the Integer Math node in the Blender Compositor by porting the existing implementation from Geometry Nodes.
- **Reasoning:**
- Serve as a proof-of-concept for my GSoC 2026 application.
- Demonstrate to mentors that I am comfortable navigating both the C++ node definitions and the GLSL GPU shader architecture.
- Establish a porting pipeline to accelerate work during the actual GSoC period.

## 2. Context

This prototype was built to validate the core technical approach of my GSoC proposal. Geometry Nodes in Blender primarily rely on the CPU and process "fields" (a specific data paradigm suited for 3D geometry). The Compositor, however, is designed to process 2D images—essentially massive grids of pixels—and heavily utilizes the GPU for real-time performance.

Because of these architectural differences, nodes designed for Geometry Nodes cannot just be toggled on in the Compositor. While the UI and CPU logic can often be shared, the GPU processing requires a dedicated translation. The biggest hurdle: **Blender's GPU Compositor currently only works with floating-point numbers.** Therefore, introducing a true _Integer_ Math node requires either building native integer support into the GPU memory manager (my primary GSoC goal) or creating a temporary sandbox workaround to prove the logic works.

## 3. The Initial Plan vs. The Reality

Blender organizes shared node logic in special files called "Function Nodes". Instead of defining socket and RNA logic three times for the Shader, Geometry Node, and Compositor environments, it is defined once centrally.

**My Initial Plan:**

1. Implement the node UI using the existing function node code.
2. Write a custom preprocessing function for the CPU to iterate over the Compositor's image grid and apply the integer math.
3. Implement the logic for the GPU using typecasting (converting floats to ints, doing the math, and converting back).
4. Define fallback methods for mismatched data types.

**The Reality (Solving the Problem):**
After doing deeper research into the codebase, I realized Step 2 was entirely unnecessary. The Compositor's CPU execution engine was explicitly designed to wrap _any_ generic Function Node. When the CPU Compositor evaluates a node tree, it looks for the `build_multi_function` callback defined in the node's C++ file. It automatically wraps that callback in a `COM_MultiFunctionProcedureOperation`, which handles iterating over every pixel in the image buffer behind the scenes!

Therefore, simply exposing the node to the Compositor UI practically gave me the CPU implementation for free. My focus shifted entirely to the GPU execution backend.

## 4. Implementation Details

### 4.1 Enabling the Node in the Compositor

First, I needed to whitelist the Function Nodes to appear in the Compositor. I modified `fn_node_poll_default` in `node_function_util.cc` to ensure it returns `true` for `CompositorNodeTree` as well as `GeometryNodeTree`.

```cpp
/* Function nodes are supported in Geometry and Compositor node trees. */
if (!STREQ(ntree->idname, "GeometryNodeTree") && !STREQ(ntree->idname, "CompositorNodeTree")) {
  *r_disabled_hint = RPT_("Not a geometry or compositor node tree");
  return false;
}

```

Next, I added the node to the Python UI menu by registering `FunctionNodeIntegerMath` in `node_add_menu_compositor.py`.

### 4.2 The C++ GPU Dispatcher

To make the node work on the GPU, I needed to bridge the C++ logic to the GLSL shader code. I achieved this by modifying `node_fn_integer_math.cc` to include a custom GPU shader function.

```cpp
static int gpu_shader_integer_math(GPUMaterial *mat,
                                   bNode *node,
                                   bNodeExecData * /*execdata*/,
                                   GPUNodeStack *in,
                                   GPUNodeStack *out)
{
  const char *name = gpu_shader_get_name(node->custom1);
  if (name != nullptr) {
    int ret = GPU_stack_link(mat, node, name, in, out);
    return ret;
  }
  return 0;
}

```

This function uses `GPU_stack_link`, which acts as the master electrician. It takes the memory buffers from the Compositor (`in`, `out`) and maps them directly to the parameters of a GLSL string name. Finally, I registered this GPU function to the node's definition via `ntype.gpu_fn = gpu_shader_integer_math;`.

To dynamically get the correct GLSL string name (e.g., `"integer_math_add"` or `"integer_math_modulo"`), I defined an `IntegerMathOperationInfo` struct in `NOD_math_functions.hh` and built a switch statement registry in `math_functions.cc`.

### 4.3 The GLSL "Float Sandwich" Workaround

Because the GPU Compositor currently only passes float buffers, I had to trick the engine. I created a new shader file, `gpu_shader_material_integer_math.glsl`, and registered it in `CMakeLists.txt` so the compiler would recognize it.

Inside this file, I implemented explicit typecasting for every mathematical operation. I take the incoming floats, cast them to integers to force the GPU to truncate decimals and perform true integer math, and then cast the result back to a float for the Compositor output.

Here is an example of the safe division logic:

```glsl
[[node]]
void integer_math_divide(float a, float b, float c, float &result)
{
  int int_a = int(a);
  int int_b = int(b);
  if (int_b != 0) {
    result = float(int_a / int_b);
  }
  else {
    result = 0.0f;
  }
}

```

This guarantees that an operation like $5 / 2$ evaluates strictly to $2$ (integer division) rather than $2.5$ (floating-point division), safely working around the current engine limitations until native integer support is introduced during GSoC.

## 5. Visual Proof and Result

To verify real-time GPU execution, the node was tested in the Viewport Compositor. Applying the custom `integer_math_modulo` logic to a smooth float gradient produced stepped visual bands in the viewport test.

<img src="assets/integer-math-prototype.png" alt="Screenshot of core developer Hans Goudey's comment on the issue tracker" width="600"/>

This test demonstrates the modulo path in the prototype. The remaining function nodes and native type work are still part of the proposal.
