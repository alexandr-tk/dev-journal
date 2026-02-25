---
project: Blender
tags: [c++, rna, python-api, open-source]
status: PR Submitted
---

# Optimizing is_input_used and is_input_visible Functions

**The Result:** [PR #154834](https://projects.blender.org/blender/blender/pulls/154834)

## 1. Workflow

- **Task:** Optimize functions created for PR #154686 to access the socket from the runtime in O(1) instead of O(n)
- **Original PR Link:** [PR #154686](https://projects.blender.org/blender/blender/pulls/154686)

- **Reasoning:**
    - Implement the suggesstion made by one of the reviewers
    - Achieve better function perfomance.

## 2. Context

In one of the reviews under my PR for exposing input visibility function to Python API I got a suggestion to access the sockets in O(1) instead of O(n) how it is currently done in my implementation. This is possible as all the input sockets are already stored in the bNodeTreeInterfaceRuntime as a VectorSet which is a custom data structure which allows lookups in constant time. However, a new function should be introduced for that in the bNodeTreeInterfaceRuntime creating the need for a separate patch.

## 3. Plan for Solving

In order to be able to optimize the function, I must intoduce the function lookup_by_identifier in the bNodeTreeInterfaceRuntime. As I am already using this class(?) in my orginal functions I will just need to replace the iteration through the identifiers to this newly created lookup function therefore creating this significant perfomance improvement for the function. Likely similar function already exists in the bNodeRuntime which I can use as a reference. Therefore my plan is:

1. Research how the input_by_identifier function works in bNodeRuntime.
2. Create a similar function in the bNodeTreeInterfaceRuntime.
3. Replace looping in the original functions with the new lookup function.
4. Run the same tests to make sure is_input_visible and is_input_used work correctly.

## 4. Solving the Issue

### Reference function

As I mentioned I will start by looking at the lookup implementation in the bNodeRuntime.

```cpp
inline const bNodeSocket *bNode::input_by_identifier(StringRef identifier) const
{
  BLI_assert(bke::node_tree_runtime::topology_cache_is_available(*this));
  return this->runtime->inputs_by_identifier.lookup_default_as(identifier, nullptr);
}
```

To breakdown in details what it does:

```cpp
inline const bNodeSocket *bNode::input_by_identifier(StringRef identifier) const
```

Inline function tells the compiler instead of jumping to the place where the function is stored, just copy the code in the file where it was called. It is very useful for tiny frequently called functions. const makes sure that the return is read-only. The functions returns the pointer to the bNodeSocket, and belongs to the bNode class, not the runtime where it is defined. The last const specifies that the function itself will not change the node.

```cpp
BLI_assert(bke::node_tree_runtime::topology_cache_is_available(*this));
```

This line is used for development only. If it returns false in the debug mode Blender will crash pointing to this node. In real usage compiler will just delete this line. In this case it checks if the cache is available for the function.

```cpp
return this->runtime->inputs_by_identifier.lookup_default_as(identifier, nullptr);
```

This is the most interesting line. It access the runtime of the node which called the function and then uses a hashmap variable (at which we will look in just a second), and calles a lookup_default_as function which is Blender's built-in method working in O(1). It will return the pointer to the identifier in case of the success, and nullptr otherwise.

### Lookup Map Declaration

Now we need to look at the lookup Map which was used in the function above and which we'll need to recreate. In the bNodeRuntime it is declaerd as:

```cpp
Map<StringRefNull, bNodeSocket *> inputs_by_identifier;
```

We can immediatly recreate similar variable in the bNodeTreeInterface. The only change that it will store pointers to the bNodeTreeInterfaceSockets:

```cpp
/** Only valid when the item cache is built. */
Map<StringRefNull, bNodeTreeInterfaceSocket *> inputs_by_identifier;
```

So now we already have runtime input pointers, we have a Map structure which will give quick lookup, now we need to fill out the items in this structure.

### Fill the Map

The map for the bNodeRuntime is filled in the different file - node_runtime.cc and looks like that:

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

        //The line I will use to fill my map
        node->runtime->inputs_by_identifier.add_new(socket->identifier, socket);
      }
      for (bNodeSocket *socket : node->runtime->outputs) {
        node->runtime->outputs_by_identifier.add_new(socket->identifier, socket);
      }
    }
  });
}
```

This function is located inside complicated ensure topology cache of the bNode. Fortunately cache ensure for the bNodeTreeInterface is less filled:

```cpp
void bNodeTreeInterface::ensure_items_cache() const
{
  bke::bNodeTreeInterfaceRuntime &runtime = *this->runtime;

  runtime.items_cache_mutex_.ensure([&]() {
    /* Rebuild draw-order list of interface items for linear access. */
    runtime.items_.clear();
    runtime.inputs_.clear();
    runtime.outputs_.clear();

    /* Items in the cache are mutable pointers, but node tree update considers ID data to be
     * immutable when caching. DNA ListBaseT pointers can be mutable even if their container is
     * const, but the items returned by #foreach_item inherit qualifiers from the container. */
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

We also do not need multicore processing that is used for the nodes as scale of the sockets is much smaller and CPU can easily handle them without multicore processing, therefore I can just fill my map inside this function:

```cpp
    runtime.items_.clear();
    runtime.inputs_.clear();
    runtime.outputs_.clear();

    //Clearing current state of the maps before updating it.
    runtime.inputs_by_identifier.clear();
    runtime.outputs_by_identifier.clear();
```

```cpp
    if (socket->flag & NODE_INTERFACE_SOCKET_INPUT) {
        runtime.inputs_.add_new(socket);
        runtime.inputs_by_identifier.add_new(socket->identifier, socket); //Filling inputs
    }
    if (socket->flag & NODE_INTERFACE_SOCKET_OUTPUT) {
        runtime.outputs_.add_new(socket);
        runtime.outputs_by_identifier.add_new(socket->identifier, socket); //Filling outputs
    }
```

To be able to call the functions I am going to create in files, I need to define them in the DNA_node_tree_interface_types.hh first:

```cpp
  /** Get an input socket by its identifier string. */
  const bNodeTreeInterfaceSocket *input_by_identifier(StringRef identifier) const;

  /** Get an output socket by its identifier string. */
  const bNodeTreeInterfaceSocket *output_by_identifier(StringRef identifier) const;
```

It is needed so blender saving and loading process works properly.

Now I can define my new functions in the BKE_node_tree_interface.hh:

```cpp

inline const bNodeTreeInterfaceSocket *bNodeTreeInterface::input_by_identifier(
    StringRef identifier) const
{
  ensure_items_cache(); //Make sure our map is filled before calling
  return this->runtime->inputs_by_identifier.lookup_default_as(identifier, nullptr);
}

//Same for the output lookup...
```

Now I will also add the wrapper functions to the BKE_node_runtime.hh so I can access the function without complex nesting, straight from ntree:

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

I also need to define these wrappers in the DNA of the node_types:

```cpp
  const bNodeTreeInterfaceSocket *interface_input_by_identifier(StringRef identifier) const;
  const bNodeTreeInterfaceSocket *interface_output_by_identifier(StringRef identifier) const;
```

Now I can easily update the original loop in the rna_modifier.cc:

```cpp
  for (bNodeTreeInterfaceSocket *socket : ntree->interface_inputs()) {
    if (STREQ(socket->identifier, identifier)) {
      return input_usages[ntree->interface_input_index(*socket)].is_used;
    }
  }
```

with our new function:

```cpp
  //Without wrappers I would have to call it like ntree->tree_interface.input_by_identifier(...)
  const bNodeTreeInterfaceSocket *socket = ntree->interface_input_by_identifier(identifier);
  if (socket != nullptr) {
    return input_usages[ntree->interface_input_index(*socket)].is_visible;
  }
```

Inline functions for the BKE_node_tree_interface.hh were not the best decision so instead I defined similar functions in the node_tree_interface.cc:

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

For the testing I successfully compiled the blender and ran the python script to get the is_input_used and is_input_visible. Code sucessfully compiled so I submitted a PR.

## 6. Review

Feedback from Hans Goudey was to use original VectorSet variables can be changed to a structure CustomIDVectorSet which turns out already include hash table and therefore there is no need to create and fill new data structure. So I am removing the changes I made in the node_tree_interface.cc and will use already defined variables.

Now I should change the variables to a new datatype:

```cpp
CustomIDVectorSet<bNodeTreeInterfaceSocket *, SocketIdentifierGetter> inputs_;
CustomIDVectorSet<bNodeTreeInterfaceSocket *, SocketIdentifierGetter> outputs_;
```

The first value we are giving to it remains the same as in the VectorSet. The second value is a struct that tells blender which part of the socket we want to use as a key in our hash table. Now I need to define this custom struct:

```cpp
  struct SocketIdentifierGetter {
    StringRef operator()(const bNodeTreeInterfaceSocket *socket) const
    {
      return socket->identifier;
    }
  };
```

Struct functions almost in the same way as a class, but unlike class keeps everything inside public by default. It is usually used to group things together, like x,y,and z for struct Vector 3D. When we give a struct a specific function that allows it to act like a machine, it is called a "Functor" (a function-object).

```cpp
StringRef operator()(const bNodeTreeInterfaceSocket *socket) const
```

With this line we are telling compiler when the struct is run it will return the value of StringRef. By using operator() function we can pass the values into the stuct directly and run it without calling the function name. And in the brackets we specify what we are giving this struct as an input. We need to use our identifier as a key therefore we return exactly it:

```cpp
return socket->identifier;
```

After this changes were made we can get the index of our socket by callling index_of_as(identifier). This index will give us an access to our socket because VectorSets are half hash tables half arrays. We get the index for the second.

Using this built in blender function we can change our getter and wrapper funtions. Notice that now we return the int type as we access the index.

```cpp
//Getters
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
//Wrappers
inline const int bNodeTree::interface_input_index_by_identifier(StringRef identifier) const
{
  return this->tree_interface.input_index_by_identifier(identifier);
}

inline const int bNodeTree::interface_output_index_by_identifier(StringRef identifier) const
{
  return this->tree_interface.output_index_by_identifier(identifier);
}
```

I also updated the descrirptions in the DNA to correspond the new definitions.

Now I will replace the functions in the rna_modifier.cc to the newly made ones:

```cpp
  int index = ntree->interface_input_index_by_identifier(identifier);
  if (index != -1) {
    return input_usages[index].is_visible;
  }
```

I also need to make existing functions which used the vectorSet to use our new version now.

Replacing this:

```cpp
inline int bNodeTree::interface_output_index(const bNodeTreeInterfaceSocket &io_socket) const
{
  BLI_assert(this->tree_interface.items_cache_is_available());
  return this->tree_interface.runtime->inputs_.index_of_as(io_socket.identifier);
}
```

With this:

```cpp
inline int bNodeTree::interface_output_index(const bNodeTreeInterfaceSocket &io_socket) const
{
  BLI_assert(this->tree_interface.items_cache_is_available());
  return this->tree_interface.runtime->outputs_.index_of_as(io_socket.identifier);
}
```
