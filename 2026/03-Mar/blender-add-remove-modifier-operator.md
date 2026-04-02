---
project: Blender
tags: [c++, animation, operators, open-source]
status: PR Merged
---

# Batch Operator for Deleting Modifiers from F-Curves/Channels

**The Result:** [PR \#154347](https://projects.blender.org/blender/blender/pulls/154347)

## 1\. Workflow

- **Task:** Implement a batch operator (`ANIM_OT_modifiers_delete`) to remove modifiers from all selected F-Curves and Channels.
- **Original Proposal/Issue:** \#143705
- **Reasoning:**
    - Blender already supported batch-adding modifiers to channels, but deleting them required tedious, manual work on a per-channel basis.
    - Adding this parity feature significantly speeds up the animation workflow in the Graph Editor and Dope Sheet.

## 2\. Context

The goal was to create a new C++ operator that iterates over selected animation data, filters for valid curves, and frees their associated modifiers. Once the core logic was written, the operator needed to be registered and exposed to the UI (specifically the Channel menus and the Right-Mouse-Button context menu). While my initial implementation functionally worked, the code structure was heavily nested, and the UX lacked feedback, leaving users unsure if the operator affected all selected curves or just the active one.

## 3\. Plan for Solving

To implement the batch deletion, the plan was:

1.  Retrieve the animation context and filter the visible/selected data for F-Curves.
2.  Iterate through the filtered `anim_data` list.
3.  Identify `FCurve` and `NLACurve` data types.
4.  Check if the curve contains any modifiers. If so, free them and tag the dependency graph for updates.
5.  Send a notifier to update the UI.
6.  Register the operator and bind it to the relevant menus.

## 4\. Solving the Issue (Initial Approach)

In my initial approach, I used a massive `switch` statement to iterate through the animation list elements (`bAnimListElem`). I explicitly listed out almost every single `ANIMTYPE_` enum just to break out of them, and only executed logic on the curve types:

```cpp
  for (bAnimListElem &ale : anim_data) {
    switch (ale.type) {
      case ANIMTYPE_FCURVE:
      case ANIMTYPE_NLACURVE: {
        FCurve *fcu = (FCurve *)ale.data;
        if (BLI_listbase_is_empty(&fcu->modifiers) == false) {
          free_fmodifiers(&fcu->modifiers);
          ale.update |= ANIM_UPDATE_DEPS;
        }
        break;
      }
      case ANIMTYPE_GPLAYER:
      case ANIMTYPE_GREASE_PENCIL_LAYER:
      // ... [dozens of other cases] ...
      case ANIMTYPE_NUM_TYPES:
        break;
    }
  }
```

While this technically worked, it was incredibly verbose. It also contained nested `if` statements and utilized a C-style cast `(FCurve *)ale.data`. Furthermore, I initially left a `break;` statement at the end of the block, which (after removing the `switch`) would have accidentally broken the entire `for` loop after processing just one curve.

## 5\. Review & Refactoring

Feedback from core reviewer Sybren A. Stüvel highlighted two major areas for improvement: **code flattening** and **UI feedback**.

### Code Flattening (Guard Clauses)

Sybren suggested replacing the massive `switch` statement with early exits (guard clauses) using Blender's `ELEM` C-macro. This checks if the type is _not_ an F-Curve or NLA-Curve and immediately continues to the next loop iteration.

He also recommended flipping the modifier list check so that empty lists trigger a `continue`, completely eliminating the need for nested brackets. I paired this with a modern C++ `static_cast` for safety:

```cpp
    if (!ELEM(ale.type, ANIMTYPE_FCURVE, ANIMTYPE_NLACURVE)) {
      continue;
    }

    FCurve *fcu = static_cast<FCurve *>(ale.data);
    if (BLI_listbase_is_empty(&fcu->modifiers)) {
      continue;
    }

    // Core logic is now completely un-nested
    free_fmodifiers(&fcu->modifiers);
    ale.update |= ANIM_UPDATE_DEPS;
```

### Adding UI Feedback

Sybren also pointed out a UX flaw: _"Without such a notification, I feel that it's too ambiguous whether it removes the modifiers from all selected F-Curves or only the active F-Curve."_ To fix this, I needed to track exactly what the loop was doing and report it back to the user via `BKE_reportf`. I introduced two counters adhering to Blender's `_count` naming convention for accumulated values: `modifier_count` and `fcurve_count`.

Inside the loop, I utilized `BLI_listbase_count` to tally the modifiers before freeing them.

```cpp
  int modifier_count = 0;
  int fcurve_count = 0;

  for (bAnimListElem &ale : anim_data) {
    // ... [Guard clauses] ...

    fcurve_count++;
    modifier_count += BLI_listbase_count(&fcu->modifiers);

    free_fmodifiers(&fcu->modifiers);
    ale.update |= ANIM_UPDATE_DEPS;
  }
```

Finally, to ensure the UI remained clean, I wrapped the `BKE_reportf` call in a conditional so it would only fire if curves were actually modified.

```cpp
  if (fcurve_count > 0) {
    BKE_reportf(op->reports, RPT_INFO, "Removed %d modifiers from %d FCurve(s)", modifier_count, fcurve_count);
  }
```

After implementing these structural improvements, applying strict C++ style guide formatting (like fixing pointer spacing `wmOperator *op`), and removing redundant code comments, I updated the PR description. The refactored logic provided a much cleaner codebase and a vastly improved user experience.
