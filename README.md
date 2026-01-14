# Custom heap allocator project documentation

Author: Giuseppe Didonna

## Project description

The following program is a hybrid heap memory allocator written in the C programming language that simulates the dynamic memory management of a process in a modern operating system.

This allocator provides an interface similar to the standard C memory allocation functions and manages a private heap region in user space. Memory allocation and deallocation are handled explicitly by the allocator, while additional memory is requested from the operating system only when necessary.

The allocator supports the following operations:

- `my_malloc(size_t size)`: allocate a block of memory of the requested size
- `my_free(void *ptr)`: deallocate a previously allocated block

And manages an internal heap composed of memory blocks, each containing metadata describing its size and allocation status. Free blocks are reused to satisfy future allocation requests whenever possible.

Initially, the allocator manages a fixed-size memory region that represents the process heap. This region simulates the behavior of a traditional heap and is entirely controlled in user space.

When the available heap memory is exhausted, the allocator dynamically extends the heap by requesting additional memory from the operating system using the `sbrk` system call. The newly acquired memory is integrated into the existing heap and managed using the same allocation policies.

When an allocation requests exceed a predefined size threshold, the allocator bypasses the internal heap and instead obtains memory directly from the operating system using `mmap`.

Blocks allocated via `mmap` are managed separately from the heap and are released back to the operating system immediately using `munmap` when freed.

The allocator implements *Segregated Free Lists*, where free blocks are grouped into multiple lists based on their size ranges. Each list uses a **first-fit** allocation strategy to satisfy allocation requests.

When memory is freed, the corresponding block is marked as available and inserted back into the appropriate free list. Adjacent free blocks are merged using **coalescing** to reduce external fragmentation.

## Project structure

The project is composed of 5 header files and one C script file which is the entry point of the program with all the tests.

The header files are the following ones:

- **Algorithms.h**:
    Contains all the algorithms used by `my_malloc` and `my_free` in order to work properly. These algorithms are:
    - Align
    - Coalesce
    - First Fit
    - Split Block
    - Sbrk Allocation

- **Data_structure.h**:
    Includes all the data structures used by the main functions and algorithms in order to work properly. These data structures are:
    - Memory Block struct
    - The heap array with all the necessary pointers needed to use it
    - Segregated lists double linked list type
    - Footer and machine size word
- **Mmap_allocator.h**:
    Manages all the mmap related functions (i.e., allocation and deallocation with `mmap` and `munmap`).
- **Utils.h**:
    Includes all the functions and wrappers used around the program to make the code cleaner and more maintainable.
- **Heap_allocator.h**:
    Implements the body of `my_malloc` and `my_free`. It's the public interface that the user has to import in order to use the dynamic allocator.
- **Allocator.c**:
    Entry-point of the program. Tests the functions with a set of defined tests.

### Dependency diagram

![Project structure](images/project_structure.png)

## Description of the main data structures

### `typedef struct Block`

It's the base of our implementation; we use it to allocate data in the heap. Each data item is stored in a block which resides in the heap. In this implementation, a block has 5 attributes:
size, is_used flag, pointers to the previous and next block in the doubly linked list (for the segregated list), and footer. As we can see, the actual object, as defined, has only the header and the pointers for the doubly linked list. The reason is that the size and is_used flag are compressed in 
the header through a bitmask in which the last three bits are used as flags to indicate whether the block is in use and whether the block was allocated with mmap. Furthermore, the block uses a special C data structure called union:
the union allows storing two different data types in the same memory space, overlapping them. When a block is free (i.e., it doesn't store any data,  and therefore has to be placed in the segregated list), the memory zone will store the two pointers needed by the segregated list (which, remember, is a doubly linked list). When the block contains some data, it will just store the payload in that area (a pointer to the memory area where the data are stored). The size of this section is 16 bytes (8 + 8) because the compiler reserves space based on the largest data type it could store. Finally, we have the footer, which as we can see is not stored in the struct because we cannot know in advance where it will be stored in memory (given that it's after the payload). Its purpose is to make it easy to find the previous adjacent block's header in the heap. We need this information to implement coalescing in $O(1)$.

![Block example](images/block_example.png "Block example")

#### Struct definition:

``` C
typedef struct Block {
    size_t header; 

    union {
        struct {
            struct Block *next_free;
            struct Block *prev_free;
        };
        unsigned char payload[0]; 
    };
} Block;
```

### `unsigned char heap[HEAP_TOTAL_SIZE]`

Is an array of bytes that represents the address space of our heap. To operate on the heap, we need to know at which address it starts (heap_start), the current top (heap_top) since we need to know where the unallocated memory starts, and at which address it ends (heap_end) since we need to know when to extend the heap space through sbrk.

#### Heap support pointers

``` C
// Points to the top of the allocated portion of the heap
static unsigned char *heap_top = heap; 
//Pointer to the end of the heap array
static unsigned char *heap_end = heap + HEAP_TOTAL_SIZE;

// Pointer to the first block, which starts at the beginning of the heap
static Block* heap_start = (Block *)heap;
```

### `Block *segregatedLists[NUM_LISTS]`

It's an array of doubly linked lists that keeps track of the current free blocks. The purpose of this method is to make searching through the free blocks more efficient, to find one of the right size for allocation.

## Description of the main algorithms

### `size_t align(size_t n)`

Aligns the block of memory based on the hardware architecture. In short, it adds padding to the size of the block so that it will be a multiple of the machine word (on a x64 machine, they will be multiples of 8). By doing so, not only is aligned memory access more efficient, but we are also sure that the last 3 least significant bits are always free (since the number will be a multiple of 8).

### `Block* coalesce(Block* block)`

Is a technique that reduces memory fragmentation. Basically, the algorithm works by merging two adjacent blocks when we are freeing one of them.

- **Step 1)** Check if the adjacent blocks are valid and not in use. Besides checking if the blocks are not in use, we have to verify that the next block we are calculating is valid (i.e., it doesn't extend beyond the current heap or before it). Since the heap array is stored in the BSS section, it can happen that after the heap is extended and the program break has moved forward, some data from other BSS-allocated variables can be present between the old heap and the new space. In this case, the physical calculation of the blocks through the footer will fail if we don't properly validate the memory zone we calculated.

- **Step 2)** Generate the new merged block

### `Block* first_fit(size_t size)`

Is the policy chosen to find an already-created block in the heap when new data is allocated. Among the search policies, first-fit is the simplest one: it just returns the first valid block in the lists.

> Note: because of splitting and coalescing, blocks can "migrate" across different lists. For this reason, we should iterate through all the lists after the target one.

### `void split_block(Block *block, size_t needed_size)`

Splits the block into two different blocks, one of the exact size that the allocation needs, and the other of the remaining size. This simple technique helps avoid internal fragmentation caused by first-fit when it chooses a block much larger than what the allocation needs.

### `void* sbrk_allocation(size_t total_size)`

It's the algorithm that allows extension of our heap memory. The problem with allocating this way is that most of the time the memory reserved by sbrk will not be contiguous with our already-allocated heap. Therefore, the algorithm fills the gap by creating a new block between the last allocated block in the heap and the new address. Note that since the heap is allocated in the BSS section, the new address returned by sbrk will always be higher than the end of the heap. This happens because the sbrk system call manages a kernel-level pointer called "program break" which indicates the end of the data zone managed by the operating system. When we call the `sbrk()` syscall, the program break will always grow toward higher addresses.

![Sbrk allocation](images/sbrk_allocation.png "Sbrk allocation")

## Allocation & deallocation tasks

### Allocation through `void* my_malloc(size_t size)`

`my_malloc` offers 3 ways to allocate data:
1. Standard allocation in a static heap: When the program starts, the heap offered to it has a size of 4KB. Allocation in a static heap is more efficient than using sbrk. The blocks are created in the memory space of the static heap if there aren't free and valid blocks that can be used for that allocation request; otherwise, deallocated blocks are reused and chosen through the first-fit policy.
2. Sbrk allocation: if the space in the heap runs out, the allocator uses the `sbrk` syscall to map more space in the process memory. The heap memory is then extended and can be enlarged further through another sbrk allocation.
3. Mmap allocation: if the data to allocate exceeds a certain threshold, the allocator uses the mmap syscall to handle the large block independently.

![Malloc flow diagram](images/flow_diagram.png "Malloc flow diagram")

### Deallocation through `void my_free(void* ptr)`

The functioning of `my_free` is straightforward:
1. The pointer to the payload is passed by the user to the function. The block associated with that payload is obtained by the `get_block_from_payload(void* ptr)` utility function.
2. If the block is marked as mmap allocated, `mmap_free` is called.
3. The block is set to free (unused) and the footer is updated.
4. The `coalesce` function is performed to try to merge the block with its neighbors.
5. The block is inserted into the segregated lists.

## Test report

In `allocator.c` script, tests are performed to check the correctness and expected behavior of the dynamic allocator. Five tests are performed in the script:

1. **Test Basic Allocation**: tests a simple allocate-write-deallocate flow
2. **Test Reuse**: tests if a block that is allocated, and then freed, would be reused another time when a data of the same size is requested
3. **Test Coalescing**: tests if the key feature of the coalescing works properly. 3 blocks of the same type are allocated and then freed one after the other. The expected behave is to have then just single bigger block in memory which is reused by the last bigger allocation
4. **Test Large Allocation**: tests if a large allocation, which would trigger the mmap allocator, works properly in the allocate-write-free flow
5. **Test Many Allocations**: tests the correct functioning of sbrk by allocating a huge number of blocks which total size would exceed the default size of the heap

**Bonus test (real use case test)**: The allocator is tested also in a real script which implements hash table data structure. The script has been taken from a real repository on github and all the instances of `malloc` and `free` were changed with the ones of the project.
The test showed that the data structure works properly also with the custom allocator.

### Results of the tests

```
=== Dynamic Allocator Test ===

=== Test 1: Base allocation ===
p1 allocated (32 bytes): 0x57d995f360a8
p2 allocated (64 bytes): 0x57d995f360d8
p3 allocated (128 bytes): 0x57d995f36128
Writing in blocks test passed 

Deallocation test passed

=== Test 2: Reuse of free blocks ===
First allocation (64 bytes): 0x57d995f360a8
Free p1
Second allocation (64 bytes): 0x57d995f360a8
SUCCESS: a block of the same reused correctly!

=== Test 3: Coalescing ===
Allocating 3 contiguous blocks
p1: 0x57d995f360a8, p2: 0x57d995f360c8, p3: 0x57d995f360e8
Freed all of 3 blocks (coalescing should merge them)
Allocating a more bigger block (sizeof(int)*3): 0x57d995f360a8

p4[0] = 10
p4[1] = 20
p4[2] = 30
=== Test 4: Use of mmap on bigger allocations ===
Allocating (256KB) with mmap: 0x70310a1a5008
Writing on the bigger block test passed
Deallocating a bigger block test passed

=== Test 5: Test many allocations ===
Allocating 70 blocks
Freed every even blocks
Freed every odd blocks

=== All tests passed successfully ===
```
