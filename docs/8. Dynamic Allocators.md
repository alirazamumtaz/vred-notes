### Heap

In C and C++, the heap is a region of memory that can be dynamically allocated during the execution of a program. To request memory from the heap, programmers use functions such as malloc. The allocated memory, known as an "allocation," can be used, modified, or referenced by the programmer until it is no longer needed and is returned to the heap manager using the free function.

Example

```c
#include<stdio.h>
#include<stdlib.h>

int main(void)
{
	char *hello = (char * )malloc(8);
	free(hello);
	return 0;
}
```

Read further at : 
- https://azeria-labs.com/heap-exploitation-part-1-understanding-the-glibc-heap-implementation/
- https://azeria-labs.com/heap-exploitation-part-2-glibc-heap-free-bins/

These two links have  well defined details about heap and there are no useless information in them which makes it difficult to cut them and make short version out of them these blogs missed the detail implementation of the arenas so if i got chance to understand it then i'll add the Arenas sections here.

Same thing but with different details and rdare2 =>  [Heap analysis with radare2 - Hackliza](https://hackliza.gal/en/posts/r2heap/)
This article contains how the meta data is changed during the chunks being free and allocated
	https://sourceware.org/glibc/wiki/MallocInternals

### Arenas

- [ ] Add the Details of Arenas

### Danger of Heaps

There are several ways that a programmer can misuse the heap, which is a region of memory used for dynamic memory allocation.

1.  Forgetting to free memory: When a program dynamically allocates memory on the heap, it is the programmer's responsibility to ensure that the memory is eventually freed, typically using the `free()` function. If a program does not free memory that it has allocated, it can lead to resource exhaustion, where the program continues to consume memory until it exhausts all available memory on the system.
   
2.  Forgetting that we have freed memory: Once a programmer has freed memory, it is important to remember that the memory is no longer in use by the program. Attempting to use memory that has been freed can lead to undefined behavior, such as accessing memory that has been overwritten by another part of the program.
   
3.  Freeing free memory: If a program calls `free()` on a pointer that has already been freed, it can corrupt the metadata used by the allocator to keep track of heap state. This can lead to undefined behavior and potential security vulnerabilities.

4.  Corrupting metadata used by the allocator: The heap memory manager uses metadata to keep track of the state of the heap and the memory allocated within it. If a program corrupts this metadata, it can lead to undefined behavior and potential security vulnerabilities. This is similar to corrupting internal function state on the stack, which can cause a program to crash or execute unintended code.

### Tcache

Thread Local Caching is a feature in the ptmalloc memory allocator that aims to speed up repeated (small) allocations in a single thread. It is implemented as a singly-linked list, with each thread having a list header for different-sized allocations.

The data structure used to implement thread local caching is a struct called `tcache_perthread_struct`. This struct contains two arrays:

-   `counts`: This array keeps track of the number of free chunks of memory available in the cache for each size class.
  
-   `entries`: This array is an array of pointers to `tcache_entry` structures. Each entry in the array corresponds to a different size class, and the pointer points to the head of the singly-linked list of chunks of memory available for that size class.

The `tcache_entry` struct represents a chunk of memory that is available in the cache. It contains a pointer to the next chunk in the same size class and a pointer to the actual memory.

When a thread wants to allocate memory, it first checks if there is a suitable chunk of memory available in its cache. If so, it removes the chunk from the cache and returns it to the caller. If the cache is empty, or if the requested size is not available in the cache, the thread falls back to the global memory pool.

When a thread frees memory, it checks if the size of the memory chunk can fit in the cache. If so, it adds the chunk to the head of the singly-linked list for the corresponding size class. If the cache is full, the memory is returned to the global memory pool.

Thread Local Caching can improve the performance of a program by reducing the number of calls to the global memory pool, which can be a bottleneck. However, it also increases the memory overhead and can lead to fragmentation if not used properly.

```c

typedef struct tcache_perthread_struct

{

  char counts[TCACHE_MAX_BINS];

  tcache_entry *entries[TCACHE_MAX_BINS];

} tcache_perthread_struct;



typedef struct tcache_entry

{

  struct tcache_entry *next;

  struct tcache_perthread_struct *key;

} tcache_entry;
```

### Freeing in Tcache

When a memory allocation is freed in a program using the ptmalloc memory allocator with thread local caching enabled, the following steps occur:

1.  Select the right "bin" based on the size: The size of the freed allocation is used to determine which size class the allocation belongs to. This is done by dividing the size of the allocation by 16 and using the result as an index into the `entries` array in the `tcache_perthread_struct` struct.

2.  Check to make sure the entry hasn't already been freed (double-free): Before the freed allocation can be added to the cache, ptmalloc checks if the allocation has already been freed. This is done by checking the second long word of the allocation, which ptmalloc stores the address of the thread's `tcache_perthread_struct` struct when the memory was freed the first time. If the value stored there is the address of the thread's `tcache_perthread_struct`, it means that the memory has already been freed and it's a double-free.
   
3.  Push the freed allocation to the front of the list: The freed allocation is added to the front of the singly-linked list for the corresponding size class. This is done by storing the current head of the list in the first long word of the freed allocation and then updating the head of the list to point to the freed allocation.
   
4.  Record the `tcache_perthread_struct` associated with the freed allocation: The address of the thread's `tcache_perthread_struct` struct is stored in the second long word of the freed allocation. This is used in step 2 to check for double-frees.
   
5.  Increase the count: The number of chunks of memory available in the cache for the corresponding size class is incremented by one.   

### Tcache Allocation

When a program using the ptmalloc memory allocator with thread local caching enabled requests memory allocation, the following steps occur:

1.  Select the bin number based on the requested size: The size of the requested allocation is used to determine which size class the allocation belongs to. This is done by dividing the size of the allocation by 16 and using the result as an index into the `counts` and `entries` arrays in the `tcache_perthread_struct` struct.
   
2.  Check the appropriate cache for available entries: The allocator checks the number of available entries in the cache for the corresponding size class. If there are any available entries, it proceeds to the next step.
   
3.  Reuse the allocation in the front of the list if available: If there are available entries in the cache, the allocator takes the entry from the front of the list, updates the head of the list to the next entry, and decrements the count of available entries. Then, it returns the reused allocation to the caller.


It's worth noting that there are some things that the allocator doesn't do:

1.  Clearing all sensitive pointers: The allocator doesn't clear all sensitive pointers in the memory. It only clears the key, which is used to check for double-frees, for some reason.

2.  Checking if the next (return[0]) address makes sense: The allocator doesn't validate the address of the next entry in the list, it assumes it's a valid address. This can lead to undefined behavior if the memory has been corrupted or if the entry has been freed.  

These two points are important to keep in mind, as these two points may lead to bugs, security vulnerabilities or undefined behavior if not handled carefully.

### Heap Meta-Data & it's Corruption

As we saw with tcache, the ptmalloc uses a bunch of metadata to track its operation. 
1. global metadata (i.e., the tcache structure)
2. per-chunk metadata

#### Allocated Chunk Meta-Data

When a program calls `malloc(x)`, it requests a block of memory with a size of `x` bytes from the memory allocator. The standard `malloc()` function, as well as the ptmalloc memory allocator, return a pointer to the start of the allocated memory block, which is referred to as the "mem_addr". However, internally, the ptmalloc memory allocator keeps track of a different address, which is referred to as the "chunk_addr".

A "chunk" in ptmalloc is a block of memory that is used to store both user data and metadata. The "mem_addr" is the address of the start of the user data within the chunk, while the "chunk_addr" is the address of the start of the entire chunk, including the metadata.

When the program calls `malloc(x)`, ptmalloc finds a suitable chunk in the memory pool, reserves it, and returns the mem_addr to the user. However, internally it keeps track of the chunk_addr, which is the address of the start of the entire chunk, including the metadata.

The metadata stored in the chunk includes information such as the size of the chunk, whether the previous chunk is currently in use, and information used for bookkeeping and memory management. The ptmalloc memory allocator uses this metadata to manage the memory pool and keep track of the state of the heap.

By keeping track of the "chunk_addr" internally, ptmalloc can efficiently manage and reuse memory chunks without needing to traverse the entire heap to find a suitable block of memory.

![[Pasted image 20230124014310.png]]

To conserve memory, the ptmalloc memory allocator uses the 'prev_size' field of a chunk for the previous chunk when the 'PREV_INUSE' flag of the chunk is set, which indicates that the previous chunk is not free.

![[Pasted image 20230124014520.png]]

#### Free Chunk Meta-Data

This information is constantly changing (see: tcache) and PTMALLOC IS VERY COMPLEX. This is an approximation. Currently, the ptmalloc caching design is (in order of use):
1. 64 singly-linked tcache bins for allocations of size 16 to 1032 (functionally "covers" fastbins and smallbins)
2. 10 singly-linked "fast" bins for allocations of size up to 160 bytes
3. 1 doubly-linked "unsorted" bin to quickly stash free()d chunks that don't fit into tcache or fastbins
4. 64 doubly-linked "small" bins for allocations up to 512 bytes
5. doubly-linked "large" bins (anything over 512 bytes) that contain different-sized chunks
   
###### Tcache Free Chunk

In the ptmalloc memory allocator with thread local caching enabled, when a chunk of memory is freed, it is placed into a singly-linked list associated with the thread that freed it. This list is used to cache freed chunks of memory so that they can be quickly reused by the same thread, without having to go to the global memory pool.

Each freed chunk in the thread-local cache has two pointers associated with it:

1.  A pointer to the allocated space of the next chunk: Each freed chunk in the thread-local cache is stored in a singly-linked list, with each chunk pointing to the next chunk in the list. The pointer to the allocated space of the next chunk allows the allocator to traverse the list and find the next available chunk of memory.

2.  A pointer to the per-thread struct: Each thread has its own `tcache_perthread_struct` struct, which keeps track of the thread-local cache. Each freed chunk in the thread-local cache has a pointer to the per-thread struct that created it. This pointer is used to check if a chunk has already been freed in order to detect double-free bugs, and also for other bookkeeping tasks.
   
![[Pasted image 20230124015148.png]]

###### Largebin Free Chunk

Each large bin has a list of free chunks of memory that are of similar size. Each free chunk in the large bin has the following metadata associated with it:

1.  Chunk size: The size of the chunk, including the metadata.
   
2.  Forward pointer: A pointer to the next chunk in the large bin's singly-linked list.
   
3.  Backward pointer: A pointer to the previous chunk in the large bin's singly-linked list.
   
4.  Forward pointer in the global free list: A pointer to the next chunk in the global free list.

5.  Backward pointer in the global free list: A pointer to the previous chunk in the global free list.   

These pointers allow the ptmalloc memory allocator to efficiently manage the large bins and quickly find suitable chunks of memory when they are needed.

It's worth noting that large bin chunks are not used for thread local caching unlike small chunks, as they are usually larger and caching them is not necessary. Also, Large bin chunks are also not used for small allocations, as the size of the chunks in the large bin is larger than the requested size.


![[Pasted image 20230124015349.png]]

### The Unlink Attack

When a chunk of memory that is larger than the maximum size of the thread-local cache (tcache) is allocated, it is removed from the doubly-linked list of available chunks in the global memory pool. This is done by updating the forward and backward pointers of the adjacent chunks in the list to bypass the allocated chunk. Specifically, the forward pointer of the previous chunk is updated to point to the next chunk, and the backward pointer of the next chunk is updated to point to the previous chunk. If an attacker can control the forward and backward pointers of a chunk, they can manipulate the doubly-linked list to overwrite an arbitrary location in memory with an arbitrary (but valid) pointer, potentially leading to a type of vulnerability called a "Use-After-Free" vulnerability.

### Poison Null Bytes

![[Pasted image 20230124020337.png]]

### House of Spirit

House of Spirit is a technique for exploiting a vulnerability in the memory allocator, specifically the `free()` function. The basic idea behind the technique is to forge a block of memory that looks like a valid memory chunk, and then pass it to the `free()` function.

By doing this, the memory allocator will treat the forged block of memory as a valid chunk of memory that is available for reuse. The next time the program calls the `malloc()` function, the memory allocator will return the forged block of memory to the program, which can be controlled by the attacker.

This technique can be used to exploit a wide range of memory-corruption vulnerabilities, such as buffer overflows, use-after-free bugs, and other types of memory-corruption vulnerabilities.

The statement also mentions that this technique can be used to overwrite a pointer and later malloc() a stack pointer, which means that the attacker can change the value of a pointer, such as a pointer to the stack, and then use the memory allocator to allocate memory at that location. This can be used to execute arbitrary code or gain control of the program.

The technique can be done with or without tcache, which means that it can work in the case of thread-local caching and also in the case where thread-local caching is disabled.

It's important to note that this technique requires the attacker to have a way to control the data passed to the free() function, and also that this technique is not a vulnerability itself, but a technique to exploit vulnerabilities, such as memory corruption vulnerabilities.

Further Reading:
[how2heap](https://github.com/shellphish/how2heap) => it is a repo maintained by Yan and his defcon team