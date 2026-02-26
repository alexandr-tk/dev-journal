---
project: Blender
tags: [c++, rna, python-api, open-source]
status: PR Merged
---

# Optimizing is_input_used and is_input_visible Functions

**The Result:** [PR #154834](https://projects.blender.org/blender/blender/pulls/154834)

## 1. Workflow

- **Task:** Optimize the functions created in PR #154686 to access sockets from the runtime in O(1) time instead of O(n).
- **Original PR Link:** [PR #154686](https://projects.blender.org/blender/blender/pulls/154686)
- **Reasoning:**
    - Implement a suggestion made by one of the core reviewers.
    - Achieve significantly better function performance.

## 2. Context

In one of the reviews on my PR (which exposed the input visibility function to the Python API), a reviewer suggested accessing the sockets in O(1) time rather than O(n), as it was currently implemented. This is possible because all the input sockets are already stored in the `bNodeTreeInterfaceRuntime` as a `VectorSet`, a custom data structure that allows lookups in constant time. However, a new function needed to be introduced in `bNodeTreeInterfaceRuntime` to handle this, creating the need for a separate patch.

## 3. Plan for Solving

To optimize the function, I need to introduce a `lookup_by_identifier` function in the `bNodeTreeInterfaceRuntime`. Since I am already using this class in my original functions, I will just need to replace the O(n) iteration through the identifiers with this newly created lookup function, instantly getting a significant performance improvement.

A similar function likely already exists in `bNodeRuntime` that I can use as a reference. Therefore, my plan is:

1. Research how the `input_by_identifier` function works in `bNodeRuntime`.
2. Create a similar function in `bNodeTreeInterfaceRuntime`.
3. Replace the loops in my original functions with the new lookup function.
4. Run the same tests to ensure `is_input_visible` and `is_input_used` still work correctly.

## 4. Solving the Issue

### Reference Function

As mentioned, I will start by looking at the lookup implementation in `bNodeRuntime`.

```cpp
inline const bNodeSocket *bNode::input_by_identifier(StringRef identifier) const
{
  BLI_assert(bke::node_tree_runtime::topology_cache_is_available(*this));
  return this->runtime->inputs_by_identifier.lookup_default_as(identifier, nullptr);
}

```

To break down what this does in detail:

```cpp
inline const bNodeSocket *bNode::input_by_identifier(StringRef identifier) const
```

The `inline` keyword tells the compiler that instead of jumping to the place where the function is stored in memory, it should just copy the code directly into the file where it is called. This is very useful for tiny, frequently called functions.
The first `const` ensures the return is read-only. The function returns a pointer to a `bNodeSocket` and belongs to the `bNode` class (not the runtime where it is defined). The final `const` specifies that the function itself will not modify the node.

```cpp
BLI_assert(bke::node_tree_runtime::topology_cache_is_available(*this));
```

This line is used for development purposes. If it evaluates to false in debug mode, Blender will crash and point directly to this node. In a real release build, the compiler will just strip this line out. Here, it checks if the cache is available before the function runs.

```cpp
return this->runtime->inputs_by_identifier.lookup_default_as(identifier, nullptr);
```

This is the most interesting line. It accesses the runtime of the node that called the function, looks at a hash map variable, and calls `lookup_default_as()`—one of Blender's built-in methods that operates in O(1) time. It will return the pointer to the identifier upon success, and a `nullptr` otherwise.

### Lookup Map Declaration

Next, we need to look at the lookup Map used in the function above so we can recreate it. In `bNodeRuntime`, it is declared as:

```cpp
Map<StringRefNull, bNodeSocket *> inputs_by_identifier;
```

We can immediately recreate a similar variable in `bNodeTreeInterface`. The only difference is that ours will store pointers to `bNodeTreeInterfaceSocket`:

```cpp
/** Only valid when the item cache is built. */
Map<StringRefNull, bNodeTreeInterfaceSocket *> inputs_by_identifier;
```

So, we now have runtime input pointers and a Map structure for quick lookups. Next, we need to populate this structure.

### Filling the Map

The map for `bNodeRuntime` is populated in a different file (`node_runtime.cc`) and looks like this:

```cpp
static void update_sockets_by_identifier(const bNodeTree &ntree)
{
  bNodeTreeRuntime &tree_runtime = *ntree.runtime;
  Span<bNode *> nodes = tree_runtime.nodes_by_id;
  threading::parallel_for(nodes.index_range(), 128, [&](const IndexRange range) {
    for (bNode *node : nodes.slice(range)) {
      node->runtime->inputs_by_identifier.clear();
      node->runtime->outputs_by_identifier.clear();
      for (bNodeSocket *socket : node->runtime->inputs) {
        // The line I will use to fill my map
        node->runtime->inputs_by_identifier.add_new(socket->identifier, socket);
      }
      for (bNodeSocket *socket : node->runtime->outputs) {
        node->runtime->outputs_by_identifier.add_new(socket->identifier, socket);
      }
    }
  });
}
```

This function is located inside a rather complicated "ensure topology cache" block for `bNode`. Fortunately, ensuring the cache for `bNodeTreeInterface` is much simpler:

```cpp
void bNodeTreeInterface::ensure_items_cache() const
{
  bke::bNodeTreeInterfaceRuntime &runtime = *this->runtime;

  runtime.items_cache_mutex_.ensure([&]() {
    /* Rebuild draw-order list of interface items for linear access. */
    runtime.items_.clear();
    runtime.inputs_.clear();
    runtime.outputs_.clear();

    /* Items in the cache are mutable pointers... */
    bNodeTreeInterface &mutable_self = const_cast<bNodeTreeInterface &>(*this);

    mutable_self.foreach_item([&](bNodeTreeInterfaceItem &item) {
      runtime.items_.add_new(&item);
      if (bNodeTreeInterfaceSocket *socket = get_item_as<bNodeTreeInterfaceSocket>(&item)) {
        if (socket->flag & NODE_INTERFACE_SOCKET_INPUT) {
          runtime.inputs_.add_new(socket);
        }
        if (socket->flag & NODE_INTERFACE_SOCKET_OUTPUT) {
          runtime.outputs_.add_new(socket);
        }
      }
      return true;
    });
  });
}
```

We also don't need the multi-threading used for `bNode` because the scale of interface sockets is much smaller, and the CPU can handle them sequentially. Therefore, I can just clear and fill my map directly inside this function:

```cpp
    runtime.items_.clear();
    runtime.inputs_.clear();
    runtime.outputs_.clear();

    // Clearing the current state of the maps before updating
    runtime.inputs_by_identifier.clear();
    runtime.outputs_by_identifier.clear();
```

```cpp
    if (socket->flag & NODE_INTERFACE_SOCKET_INPUT) {
        runtime.inputs_.add_new(socket);
        runtime.inputs_by_identifier.add_new(socket->identifier, socket); // Filling inputs
    }
    if (socket->flag & NODE_INTERFACE_SOCKET_OUTPUT) {
        runtime.outputs_.add_new(socket);
        runtime.outputs_by_identifier.add_new(socket->identifier, socket); // Filling outputs
    }
```

To call the functions across files, I first need to define them in `DNA_node_tree_interface_types.hh`. This ensures Blender's saving and loading processes work properly:

```cpp
  /** Get an input socket by its identifier string. */
  const bNodeTreeInterfaceSocket *input_by_identifier(StringRef identifier) const;

  /** Get an output socket by its identifier string. */
  const bNodeTreeInterfaceSocket *output_by_identifier(StringRef identifier) const;
```

Now I can define my new functions in `BKE_node_tree_interface.hh`:

```cpp
inline const bNodeTreeInterfaceSocket *bNodeTreeInterface::input_by_identifier(
    StringRef identifier) const
{
  ensure_items_cache(); // Make sure our map is filled before calling
  return this->runtime->inputs_by_identifier.lookup_default_as(identifier, nullptr);
}
// Same for the output lookup...
```

I will also add wrapper functions to `BKE_node_runtime.hh` so I can access the method without complex nesting, straight from `ntree`:

```cpp
inline const bNodeTreeInterfaceSocket *bNodeTree::interface_input_by_identifier(
    StringRef identifier) const
{
  return this->tree_interface.input_by_identifier(identifier);
}

inline const bNodeTreeInterfaceSocket *bNodeTree::interface_output_by_identifier(
    StringRef identifier) const
{
  return this->tree_interface.output_by_identifier(identifier);
}
```

And define these wrappers in the DNA of the node types:

```cpp
  const bNodeTreeInterfaceSocket *interface_input_by_identifier(StringRef identifier) const;
  const bNodeTreeInterfaceSocket *interface_output_by_identifier(StringRef identifier) const;
```

With everything wired up, I can easily update the original O(n) loop in `rna_modifier.cc`:

```cpp
  // Old code
  for (bNodeTreeInterfaceSocket *socket : ntree->interface_inputs()) {
    if (STREQ(socket->identifier, identifier)) {
      return input_usages[ntree->interface_input_index(*socket)].is_used;
    }
  }
```

And replace it with our new O(1) function:

```cpp
  // New code (The wrapper saves us from calling ntree->tree_interface.input_by_identifier...)
  const bNodeTreeInterfaceSocket *socket = ntree->interface_input_by_identifier(identifier);
  if (socket != nullptr) {
    return input_usages[ntree->interface_input_index(*socket)].is_visible;
  }
```

_Note: Inline functions for `BKE_node_tree_interface.hh` were ultimately not the best approach, so I instead defined them in `node_tree_interface.cc`:_

```cpp
const bNodeTreeInterfaceSocket *bNodeTreeInterface::input_by_identifier(StringRef identifier) const
{
  this->ensure_items_cache();
  return this->runtime->inputs_by_identifier.lookup_default_as(identifier, nullptr);
}

const bNodeTreeInterfaceSocket *bNodeTreeInterface::output_by_identifier(
    StringRef identifier) const
{
  this->ensure_items_cache();
  return this->runtime->outputs_by_identifier.lookup_default_as(identifier, nullptr);
}
```

## 5. Testing

For testing, I successfully compiled Blender and ran a Python script to retrieve `is_input_used` and `is_input_visible`. The code executed correctly, so I submitted the PR.

## 6. Review & Refactoring

Feedback from Hans Goudey suggested that instead of adding new Maps, the original `VectorSet` variables could be upgraded to a `CustomIDVectorSet`. This structure already includes a hash table, eliminating the need to create and populate a redundant data structure.

I discarded the Map changes in `node_tree_interface.cc` and updated the existing variables to the new data type:

```cpp
CustomIDVectorSet<bNodeTreeInterfaceSocket *, SocketIdentifierGetter> inputs_;
CustomIDVectorSet<bNodeTreeInterfaceSocket *, SocketIdentifierGetter> outputs_;
```

The first argument remains the same as in a standard `VectorSet`. The second argument is a struct that tells Blender which part of the socket we want to use as the hash table key. I defined this custom struct as follows:

```cpp
  struct SocketIdentifierGetter {
    StringRef operator()(const bNodeTreeInterfaceSocket *socket) const
    {
      return socket->identifier;
    }
  };
```

A `struct` functions similarly to a class but keeps everything `public` by default. When we give a struct an `operator()` function, it allows the struct to act like a function itself (this is called a "Functor"). Because we need the identifier to act as our key, the functor simply returns it.

With these changes, we can retrieve the index of our socket by calling `index_of_as(identifier)`. Since `VectorSet`s are essentially half hash-table and half array, getting the index solves our lookup problem.

Using this built-in method, I changed the getters and wrappers to return an `int` instead of a pointer:

```cpp
// Getters
int bNodeTreeInterface::input_index_by_identifier(StringRef identifier) const
{
  this->ensure_items_cache();
  return this->runtime->inputs_.index_of_as(identifier);
}

int bNodeTreeInterface::output_index_by_identifier(StringRef identifier) const
{
  this->ensure_items_cache();
  return this->runtime->outputs_.index_of_as(identifier);
}
```

```cpp
// Wrappers
inline int bNodeTree::interface_input_index_by_identifier(StringRef identifier) const
{
  return this->tree_interface.input_index_by_identifier(identifier);
}

inline int bNodeTree::interface_output_index_by_identifier(StringRef identifier) const
{
  return this->tree_interface.output_index_by_identifier(identifier);
}
```

I also updated the descriptions in the DNA files to reflect these new integer returns, and updated `rna_modifier.cc` to use the new logic:

```cpp
  int index = ntree->interface_input_index_by_identifier(identifier);
  if (index != -1) {
    return input_usages[index].is_visible;
  }
```

During review, I added a safety check to ensure missing identifiers properly return `-1`, and later, was asked to optimize this to avoid double lookups by using `index_of_try_as`:

```cpp
int bNodeTreeInterface::input_index_by_identifier(const StringRef identifier) const
{
  BLI_assert(this->items_cache_is_available());

  std::optional<int> index_opt = this->runtime->inputs_.index_of_try_as(identifier);

  if (index_opt.has_value()) {
    return index_opt.value();
  }
  return -1;
}
```

Additionally, Jacques Lucke requested that we pass `identifier` as a `const StringRef` to ensure the string cannot be modified. I initially tried updating the DNA functions as well, but `StringRef` is a trivially copyable type, so enforcing `const` there was unnecessary.

Finally, we realized that the entire `index_opt` block:

```cpp
  if (index_opt.has_value()) {
    return index_opt.value();
  }
  return -1;
```

Could be further simplified by exluding intermediate variable. This condenses the final implementation down to a single, elegant return (if no socket is found it will automatically return `-1`):

```cpp
int bNodeTreeInterface::output_index_by_identifier(const StringRef identifier) const
{
  BLI_assert(this->items_cache_is_available());
  return this->runtime->outputs_.index_of_try_as(identifier);
}
```

After implementing these changes, I updated the PR description and it was approved and merged.
