---
title: "Resolving Memory Fragmentation for Linkedlist Heap Allocator"
datePublished: Sun Feb 11 2024 21:29:06 GMT+0000 (Coordinated Universal Time)
cuid: clsi0pt9v00000ajt32pc8yh0
slug: resolving-memory-fragmentation-for-linkedlist-heap-allocator
tags: operating-system, rust, memory-management, linkedlists, heap-allocator

---

This blog is based on the chapter *Allocator Design* of tutorial Writing an OS in Rust ([https://os.phil-opp.com/](https://os.phil-opp.com/)).

### Heap Allocator

A heap is a memory region for a program to store dynamically-sized data at runtime. The data stored in heap can have varying lifecycles. A heap allocator is responsible for allocating a heap region of appropriate size to store new data, and deallocate the region when the data is released. In many languages, the deallocation of heap is handled by garbage collection system. However, in Rust, with the ownership and lifecycle system, the deallocation of heap is called directly by the compiler once the object ends its lifecycle.

In our OS kernel, we want to implement a heap allocator that can be used by the Rust compiler as a global allocator, which requires the implementation of **GlobalAlloc** trait:

```rust
pub unsafe trait GlobalAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);

    unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8 { ... }
    unsafe fn realloc(
        &self,
        ptr: *mut u8,
        layout: Layout,
        new_size: usize
    ) -> *mut u8 { ... }
}
```

We need to implement **alloc** and **dealloc** methods, and the other two methods have default implementations. For the alloc method, we need to find an available heap region with size and memory alignment described in layout parameter, or returns a null pointer if the heap is out of memory. For the dealloc method, we need to mark the region to be freed as unused so it can be allocated again.

### Linkedlist Heap Allocator Implementation

The heap allocator should be able to keep track of all the unused memory regions in the heap. Since the number of unused regions is dynamic at runtime, we need to store the list of regions (also known as free list) in a data structure with dynamic size. However, we cannot use collection types like Vectors or Hashmaps since their implementations are based on a heap, which leaves as the choice to use linkedlist.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1707682868179/94e42112-faa9-458b-bb1d-f3555979e9e3.png align="center")

A linkedlist allocator stores all the free regions in a linkedlist, whose each node stores the starting address and size of the region, and a pointer to the next node. When we need to allocate a region, we traverse the linkedlist to find a node that contains a region that is large enough, and remove and return it. During deallocation, we append the released region as a new node into the free list.

The implementation of a node:

```rust
struct ListNode {
    size: usize,
    next: Option<&'static mut ListNode>,
}

impl ListNode {
    const fn new(size: usize) -> Self {
        ListNode { size, next: None }
    }

    fn start_addr(&self) -> usize {
        self as *const Self as usize
    }

    fn end_addr(&self) -> usize {
        self.start_addr() + self.size
    }
}
```

The implementation of LinkedlistAllocator:

```rust
pub struct LinkedListAllocator {
    head: ListNode,
}

impl LinkedListAllocator {
    /// Creates an empty LinkedListAllocator.
    pub const fn new() -> Self {
        Self {
            head: ListNode::new(0),
        }
    }

    /// Initialize the allocator with the given heap bounds.
    ///
    /// This function is unsafe because the caller must guarantee that the given
    /// heap bounds are valid and that the heap is unused. This method must be
    /// called only once.
    pub unsafe fn init(&mut self, heap_start: usize, heap_size: usize) {
        self.add_free_region(heap_start, heap_size);
    }
}
```

Note that in this code we make the head of a list an empty ListNode, and head.next represent the first valid node in free list. The init method creates one node that records the whole heap region. The implementation of add\_free\_region is as below:

```rust
impl LinkedListAllocator {
    /// Adds the given memory region to the front of the list.
    unsafe fn add_free_region(&mut self, addr: usize, size: usize) {
        // ensure that the freed region is capable of holding ListNode
        assert_eq!(align_up(addr, mem::align_of::<ListNode>()), addr);
        assert!(size >= mem::size_of::<ListNode>());

        // create a new list node and append it at the start of the list
        let mut node = ListNode::new(size);
        node.next = self.head.next.take();
        let node_ptr = addr as *mut ListNode;
        node_ptr.write(node);
        self.head.next = Some(&mut *node_ptr)
    }
}
```

In the add\_free\_region method, we first check that whether the memory region is properly aligned and is greater than the size of a ListNode (to keep track of the region with a ListNode, we need to first ensure the region is large enough the store the node!) Then we append the node to the head of the list (self.head.next). Note that it is only during **node\_ptr.write(node)** the node is written to the memory and has a 'static lifecycle. Before that the node is a local variable stored on stack. We will come to this important distinction later.

This diagram shows the process of appending the node to the head of the list:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1707683778032/381ae07f-1354-4435-b0f1-35738cc18f32.png align="center")

Next, we need a method alloc\_from\_region that checks whether a list node is enough to allocate a memory region with a given size. We first need to check that the node's end address is greater than the end of allocation (the region recorded by the node is larger than the region). In addition, since we want to keep track of the remaining region of the node after allocation, we need to make sure the remaining region after allocation is large enough to store another ListNode.

```rust
impl LinkedListAllocator {
    /// Try to use the given region for an allocation with given size and
    /// alignment.
    ///
    /// Returns the allocation start address on success.
    fn alloc_from_region(region: &ListNode, size: usize, align: usize)
        -> Result<usize, ()>
    {
        let alloc_start = align_up(region.start_addr(), align);
        let alloc_end = alloc_start.checked_add(size).ok_or(())?;

        if alloc_end > region.end_addr() {
            // region too small
            return Err(());
        }

        let excess_size = region.end_addr() - alloc_end;
        if excess_size > 0 && excess_size < mem::size_of::<ListNode>() {
            // rest of region too small to hold a ListNode (required because the
            // allocation splits the region in a used and a free part)
            return Err(());
        }

        // region suitable for allocation
        Ok(alloc_start)
    }
}
```

The find\_region method is used to traverse the linkedlist to find a memory region that is large enough for the allocation. If such a node is found, it returns the node and removes it from the linkedlist. Otherwise, None is returned.

```rust
impl LinkedListAllocator {
    /// Looks for a free region with the given size and alignment and removes
    /// it from the list.
    ///
    /// Returns a tuple of the list node and the start address of the allocation.
    fn find_region(&mut self, size: usize, align: usize)
        -> Option<(&'static mut ListNode, usize)>
    {
        // reference to current list node, updated for each iteration
        let mut current = &mut self.head;
        // look for a large enough memory region in linked list
        while let Some(ref mut region) = current.next {
            if let Ok(alloc_start) = Self::alloc_from_region(&region, size, align) {
                // region suitable for allocation -> remove node from list
                let next = region.next.take();
                let ret = Some((current.next.take().unwrap(), alloc_start));
                current.next = next;
                return ret;
            } else {
                // region not suitable -> continue with next region
                current = current.next.as_mut().unwrap();
            }
        }

        // no suitable region found
        None
    }
}
```

With the methods we implemented above, we are able to implement the alloc and dealloc method for GlobalAlloc trait:

```rust
unsafe impl GlobalAlloc for Locked<LinkedListAllocator> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // perform layout adjustments
        let (size, align) = LinkedListAllocator::size_align(layout);
        let mut allocator = self.lock();

        if let Some((region, alloc_start)) = allocator.find_region(size, align) {
            let alloc_end = alloc_start.checked_add(size).expect("overflow");
            let excess_size = region.end_addr() - alloc_end;
            if excess_size > 0 {
                allocator.add_free_region(alloc_end, excess_size);
            }
            alloc_start as *mut u8
        } else {
            ptr::null_mut()
        }
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        // perform layout adjustments
        let (size, _) = LinkedListAllocator::size_align(layout);

        self.lock().add_free_region(ptr as usize, size)
    }
}
```

In the alloc method, we first find a memory region to allocate using find\_region method. Then, if there is remaining memory on the region after allocation, we store the excess region back to the linkedlist. In the dealloc method, we only need to append the freed region into our linkedlist.

### Memory Fragmentation Issue

The above implementation of linkedlist allocator works in general. However, it has a serious problem: after repeated allocations and deallocations, the memory will be fragmented into smaller and smaller list nodes and it will be harder to find regions to allocate large objects. The diagram below shows how this problem happens:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1707685631782/5514a039-c09d-4b19-bf2e-72f53c5da32a.png align="center")

Assume we have three regions being allocated. When these regions are freed, new list nodes are created to keep track of each freed regions during dealloc. However, even through these freed regions are consecutive, they are tracked by different nodes, and in out current implementation we do not have any ways to merge the nodes together. As a result, in repeated allocations, the memory will be fragmented into smaller and smaller nodes, which would compromises the efficiency of traversing the linkedlist and makes it unable to allocate large regions even if there is enough space.

### Solution to the Memory Fragmentation Issue

In order to resolve the issue above, we need to find a way to merge consecutive memory regions to a single node. To implement this, we first need to sort the nodes by their starting address in add\_free\_region method instead of appending new nodes directly at the head.

```rust
impl LinkedListAllocator {
    // apppend a free region with given starting address and size to the linkedlist
    unsafe fn add_free_region(&mut self, addr: usize, size: usize) {
        // ensure the memory is aligned 
        assert_eq!(align_up(addr, mem::align_of::<ListNode>()), addr);
        // ensure the memory is large enough to hold the linkedlist
        assert!(size >= mem::size_of::<ListNode>());

        // the new node
        let mut new_node = ListNode::new(size); // the node is now on stack
        // find the proper location of node in linkedlist
        
        // add node to the head of list
        let mut current = &mut self.head;

        if current.next.is_none() ||
            addr < current.next.as_ref().unwrap().start_addr() {    // the head is None or new node has smaller start address than first node
            new_node.next = current.next.take();
        } else {
            // traverse the linkedlist
            while let Some(ref mut next) = current.next {
                if next.start_addr() >= addr {  
                    break;
                }
                current = current.next.as_mut().unwrap()
            }
            // assign the node's next node
            // we can only assign its previous node when the node is written to memory (having a 'static lifecycle)
            new_node.next = current.next.take();
        }


        let node_ptr = addr as *mut ListNode;
        node_ptr.write(new_node);   // write the new node in memory
        current.next = Some(&mut *node_ptr);    // connect the node with the previous node

    }
}
```

In general, the code is similar to the method in find\_region that traverses the linkedlist. However, there are a few points to note about:

1. We need to modify self.head.next (the first node) if self.head.next is None or if the start address of new node is smaller than the start address of current self.head.next.
    
2. When we compare the address of nodes in the linkedlist with the newly appended node, **we have to use addr instead of new\_node.start\_address()**, this is because before new\_node is written to memory in node\_ptr.write(new\_node), new\_node is a local variable stored on stack. Comparing its start address on stack with addresses in the heap will very likely lead to confusing results.
    
3. The next attribute of ListNode need to ensure that the reference has a 'static lifecycle, meaning that the ListNode being refered to must be stored in heap. We can assign the next of new\_node in the while let loop (since it points to an existing node in heap), but we have to assign the next of the node before new\_node (current.next) only after new\_node is being written to heap. (We need to thank Rust's lifecycle system on this matter, otherwise it would be very easy to accidentally assign the reference to a local variable, which gets cleared after the method).
    

After maintaining the nodes sorted by start address, we are able to merge consecutive memory by checking whether the end address of one node is equal to the start address of its next node. If so, we creates a single node in the linkedlist that overrides the two nodes.

```rust
impl LinkedlistAllocator {
// merge consecutive free memory regions into larger regions
    fn merge_region(&mut self) {
        let mut current = &mut self.head;

        while let Some(ref mut node) = current.next {
            // get the start and end address of current node (prevent borrowing issue)
            let start_addr = node.start_addr();
            let end_addr = node.end_addr();
            // check whether it is adjacent to the next node
            if let Some(ref mut next) = node.next {
                if end_addr == next.start_addr() {
                    // combine two memory regions
                    node.size = next.end_addr() - start_addr;
                    node.next = next.next.take();
                }
            }

            current = current.next.as_mut().unwrap();
        }
    }
}
```

Finally, we call merge\_region method in dealloc to merge consecutive regions after every deallocation:

```rust
unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
    let (size, _) = LinkedListAllocator::size_align(layout);
    // add the freed region to free list
    self.lock().add_free_region(ptr as usize, size);;
    // merge unused regions
    self.lock().merge_region();
}
```

The merging of memory region:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1707686898845/b5ad3d73-a28c-402f-b2c5-b61e3eb9303d.png align="center")