# Chapter 2: PlompBufferItem - A Single Log Entry

In the [previous chapter](01_data_recording_api.md), we learned how to use Plomp's **Data Recording API** functions like `record_prompt()`, `complete()`, and `record_event()` to tell Plomp about things happening in our AI application. We saw these functions acting like a secretary, taking notes for us.

But what do these notes look like? When we record a prompt or an event, what information is actually stored? Where does the secretary write it down?

Each piece of information recorded by Plomp is stored as a **`PlompBufferItem`**. Think of your Plomp logbook ([PlompBuffer](04_plompbuffer.md)) as a collection of pages. A `PlompBufferItem` is *one single page* in that logbook. It's the fundamental unit of information storage in Plomp.

In this chapter, we'll look closely at the structure of a `PlompBufferItem` and understand what makes up a single entry in our Plomp log.

## Why Have a Standard Structure?

Imagine you're keeping a manual log. Sometimes you might jot down just the prompt, other times the prompt and response, and maybe sometimes just a note like "System crashed". If every entry is formatted differently, it becomes very hard to search through your log later or analyze patterns.

Plomp solves this by defining a standard structure for every single entry: the `PlompBufferItem`. Every item, whether it's recording a prompt, an event, or something else, shares some common information. This consistency makes it much easier to manage, search, and understand your logs.

## The Anatomy of a PlompBufferItem

Every `PlompBufferItem` (every page in our logbook) has the same basic layout, containing these key pieces of information:

1.  **`timestamp`**: When was this entry created? Plomp automatically records the date and time. This is crucial for knowing *when* things happened.
2.  **`tags`**: These are like labels or keywords you can attach to the entry. For example, you might tag a prompt with `{'user_id': 123, 'test_run': True}`. Tags help you categorize and find specific entries later. We'll learn all about tags in the next chapter on the [Tagging System](03_tagging_system.md).
3.  **`type_`**: What *kind* of information does this entry represent? Is it a prompt/completion cycle, a general event, or something else? Plomp uses this to know how to interpret the specific data stored in the entry.
4.  **`_data`**: This is the core content of the entry, and its exact structure *depends on the `type_`*. If the type is 'prompt', the data will contain the prompt text and maybe the response. If the type is 'event', the data will contain the custom dictionary you recorded.

Let's visualize this common structure:

```mermaid
graph TD
    A[PlompBufferItem] --> B(timestamp: datetime)
    A --> C(tags: dict)
    A --> D(type_: ItemType)
    A --> E(_data: Varies based on type_)

    subgraph ItemType
        F[PROMPT]
        G[EVENT]
        H[QUERY]
    end

    subgraph DataPayloads
        I[PlompCallTrace (for PROMPT)]
        J[PlompEvent (for EVENT)]
        K[PlompBufferQuery (for QUERY)]
    end

    D -- Determines --> E
    F -- Implies --> I
    G -- Implies --> J
    H -- Implies --> K
```

This diagram shows that every `PlompBufferItem` has a `timestamp`, `tags`, and a `type_`. The `type_` then determines what kind of data payload (`_data`) it holds.

## Types of PlompBufferItems

Plomp defines a few standard types for log entries:

1.  **`PlompBufferItemType.PROMPT`**:
    *   **Purpose**: Represents a prompt sent to an AI and (eventually) its completion.
    *   **`_data` Contents**: Holds a `PlompCallTrace` object. This object contains the `prompt` text. If the prompt has been completed (using `handle.complete()` as we saw in Chapter 1), it will also contain the `response` text and the timestamp of when the completion occurred.
    *   **Analogy**: A logbook page specifically for recording a question asked and the answer received.

2.  **`PlompBufferItemType.EVENT`**:
    *   **Purpose**: Represents any other piece of information you want to log. This could be system settings, errors, user feedback, or anything else.
    *   **`_data` Contents**: Holds a `PlompEvent` object. This object mainly contains the `payload` dictionary you provided when calling `plomp.record_event()`.
    *   **Analogy**: A general-purpose logbook page where you can write down any important note or observation, structured as a dictionary.

3.  **`PlompBufferItemType.QUERY`**:
    *   **Purpose**: Represents the act of searching or filtering the logbook itself. When you query your Plomp buffer (which we'll learn about in [PlompBufferQuery](05_plompbufferquery.md)), Plomp can optionally record the query itself as a new item.
    *   **`_data` Contents**: Holds a `PlompBufferQuery` object, which describes the query that was performed.
    *   **Analogy**: A logbook page that says, "On this date, I searched for all pages tagged 'important'". This is a more advanced feature.

So, when you call `plomp.record_prompt()`, Plomp creates a `PlompBufferItem` with `type_ = PlompBufferItemType.PROMPT`. When you call `plomp.record_event()`, it creates one with `type_ = PlompBufferItemType.EVENT`.

## Accessing Data within a PlompBufferItem

While we haven't learned how to *get* items *out* of the logbook yet (that comes later with the [PlompBuffer](04_plompbuffer.md) and [PlompBufferQuery](05_plompbufferquery.md)), let's imagine we have retrieved a single `PlompBufferItem` called `item`. How would we access its specific data?

Because the `_data` depends on the `type_`, Plomp provides helpful ways to access it safely:

*   If you know (or check) that `item.type_ == PlompBufferItemType.PROMPT`, you can access its prompt/completion data using `item.call_trace`.
*   If you know `item.type_ == PlompBufferItemType.EVENT`, you can access its event data using `item.event`.
*   If you know `item.type_ == PlompBufferItemType.QUERY`, you can access its query data using `item.query`.

```python
# Imagine 'item' is a PlompBufferItem we got from somewhere
# (We'll learn how to get items later)

# Always check the type first!
if item.type_ == PlompBufferItemType.PROMPT:
    # Access prompt-specific data
    print(f"Prompt Text: {item.call_trace.prompt}")
    if item.call_trace.completion:
        print(f"Response Text: {item.call_trace.completion.response}")
    else:
        print("Response: (Not completed yet)")

elif item.type_ == PlompBufferItemType.EVENT:
    # Access event-specific data
    print(f"Event Payload: {item.event.payload}")

elif item.type_ == PlompBufferItemType.QUERY:
    # Access query-specific data (details depend on PlompBufferQuery)
    print(f"Recorded Query Operation: {item.query.op_name}")

# Example Output (if item was a completed prompt):
# Prompt Text: What is the capital of France?
# Response Text: The capital of France is Paris.

# Example Output (if item was an event):
# Event Payload: {'model_name': 'super-ai-1000', 'temperature': 0.7}
```

This example shows how you'd inspect an item once you have it. You first check its `type_` and then use the corresponding property (`.call_trace`, `.event`, or `.query`) to get the specific data payload (`PlompCallTrace`, `PlompEvent`, or `PlompBufferQuery`). Trying to access `.call_trace` on an EVENT item, for instance, would result in an error, so checking the type is important.

## Under the Hood: The `PlompBufferItem` Class

Internally, `PlompBufferItem` is defined as a Python class, likely using `dataclass` for convenience. Let's look at a simplified version of its definition found in `plomp/_buffer_items.py`:

```python
# Simplified from plomp/_buffer_items.py

import datetime as dt
from enum import Enum
from dataclasses import dataclass
from typing import Union # Used to indicate _data can be one of several types

# Define the possible types
class PlompBufferItemType(Enum):
    PROMPT = "prompt"
    EVENT = "event"
    QUERY = "query"

# Define the data structures for each type
@dataclass
class PlompCallTrace: # Holds prompt/completion data
    prompt: str
    completion: ... = None # More complex type, holds response + timestamp

@dataclass
class PlompEvent: # Holds event data
    payload: dict

# (PlompBufferQuery class defined elsewhere, let's imagine it here)
@dataclass
class PlompBufferQuery: # Holds query data
    op_name: str
    matched_indices: list[int]
    # ... other query details ...

# The main PlompBufferItem class
@dataclass
class PlompBufferItem:
    timestamp: dt.datetime
    tags: dict # Actually TagsType, a more specific dict type
    type_: PlompBufferItemType
    # The actual data, which is one of the types above
    _data: Union[PlompCallTrace, PlompEvent, PlompBufferQuery]

    # Helper property to safely get Call Trace data
    @property
    def call_trace(self) -> PlompCallTrace:
        if self.type_ != PlompBufferItemType.PROMPT:
            raise ValueError("Item is not a prompt request")
        # Type assertion for static analysis tools
        assert isinstance(self._data, PlompCallTrace)
        return self._data

    # Helper property to safely get Event data
    @property
    def event(self) -> PlompEvent:
        # ... similar check and return for EVENT type ...

    # Helper property to safely get Query data
    @property
    def query(self) -> "PlompBufferQuery":
        # ... similar check and return for QUERY type ...

    # (Other methods like to_dict, render omitted for simplicity)
```

This simplified code shows:
1.  The `PlompBufferItemType` enum defines the allowed types (`PROMPT`, `EVENT`, `QUERY`).
2.  Separate classes (`PlompCallTrace`, `PlompEvent`, `PlompBufferQuery`) define the structure for the data specific to each type.
3.  The main `PlompBufferItem` class holds the common fields (`timestamp`, `tags`, `type_`) and the `_data` field, which can be one of the specific data classes.
4.  The `@property` methods (`.call_trace`, `.event`, `.query`) provide a safe way to access the specific data, including a check to make sure the item is of the correct type.

When functions like `plomp.record_prompt()` are called (as discussed in [Chapter 1](01_data_recording_api.md)), they construct one of these `PlompBufferItem` objects with the appropriate `type_` and `_data` structure, and add it to the main [PlompBuffer](04_plompbuffer.md).

## Conclusion

We've now explored the `PlompBufferItem` – the fundamental building block of logs in Plomp. We learned that every item recorded has a standard structure: a `timestamp`, descriptive `tags`, a `type_` (like `PROMPT` or `EVENT`), and the specific `_data` payload corresponding to that type. This consistent structure is key to making Plomp logs organized and useful.

We saw that the `type_` determines whether the item holds details about a prompt/completion (`PlompCallTrace`), a custom event (`PlompEvent`), or a recorded query (`PlompBufferQuery`).

One crucial part of the `PlompBufferItem` structure is the `tags`. They allow us to add custom labels and context to our log entries, making them much easier to find and analyze later. How do we effectively use these tags? That's exactly what we'll cover in the next chapter!

Next Up: [Chapter 3: Tagging System](03_tagging_system.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)