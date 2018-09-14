
# Multi-Threaded Malloc Library (using C)

Writing this library was a part of the course 'CS5600 Computer Systems' at Northeastern 
University, Boston; and hence I have its complete implementation in a private repository
in order to comply with the policies of my university. If you are interested in having a 
look at it, please send me an email at: shah.sauri@husky.neu.edu mentioning the purpose 
for the same.

The library features the following APIs:
* malloc()
* calloc()
* realloc()
* malloc_stats()
* mallinfo()
* free()

-----------------------------------------------------------------------------------------

## Design Overview:
	
This malloc library has been implemented based on the "Buddy allocation" technique for 
managing memory as it is reasonably fast and efficient. 

Separate linked lists whose nodes hold the metadata that include size of the memory block and the address of the next memory block of the same size are maintained. This consists of maintaing an array in the data segment (global variable)
which maintains the heads of the each linked list. The nodes are themselves a part of the
memory blocks; as a header, and are a part of the heap memory.

My design supports memory allocation in block sizes that are powers of 2, from 32 bytes
to 2048 bytes; both inclusive, (including the metadata/header size). Any request to
malloc beyond 2048 (2048 - 16 = 2032 bytes effectively) bytes is handled by a call to
mmap() and that memory is free using munmap().

Initially and when there is no free memory of appropriate size available, a system call to sbrk() is made in multiples of page size (= 4096 bytes). Once we get 4096 bytes, it's broken into two blocks of 2048 bytes each and are added to the appropriate linked list of free blocks; whose headers are maintained by the global array (freeMemLists).

When the user requests memory, the requested amount of memory is added to the precalculated size of our header after which we try to find the smallest memory block that is a power of 2 which can fit this requested memory. If the required size block is not available in the free memory lists, the algorithm tries to find a larger block which is then broken into two parts recursively until we get the desired size. If neither the required size, nor a larger size is available, sbrk() system call is made. 

While freeing an allocated block of memory, the size of the block is retrieved from its header. This memory block is added to the linked list that has free nodes of that specific size. While inserting the blocks in the linked lists, we ensure that they are inserted in the sorted order by the address of the memory blocks. This sorted order helps us to merge two free blocks that are buddies to form a larger sized block (Merging of buddies is yet to be implemented).


-----------------------------------------------------------------------------------------

## More about the Enhanced Arena Per core feature:

This library uses an arena per core. So depending on your system specifications, the number of cores are identified and an arena is created for each CPU core. (For example, a dual-core machine can support 2 arenas, while an octa-core machine can support 8 arenas. If you're using a Virtualization product, you might have limited cores alloted to your system running on the VM. Accordingly, this Malloc library will only use the cores that are actually available to the VM; even though the physical processor may have more cores.)

When the first ever request to malloc() is made, all the per-core arenas are initialized. On subsequent calls, new threads are assigned to the available arena. If no arena is available, the thread waits for one to be available. Once an arena is allocated to a thread, the thread uses the same arena in its lifetime to perform all allocation and free requests. This is done with the help of a thread specific variable that stores the pointer to its allocated arena.

The struct MallocArena stores the free memory blocks available in that arena and a lock specific to that arena. It also stores some other information like the total available number of free blocks, total used blocks, total free requests and total allocation requests.

Each arena has its own lock. This ensures that at any point of time, only one thread can do any sort of operation on the arena which is allocated to it. Multiple threads can be allocated to the same arena, but only one can work on it at any given point of time. Hence this enhanced implementation provides a faster service of memory allocation and de-allocation depending on the number of CPU cores available which can work parallelly.

The malloc_stats() API prints on the console information like used blocks, free blocks, total free requests and total allocation requests for each arena. The number of allocation and free requests are considering both, the ones handled by my library entirely and the ones handled by mmap() and munmap(). The number of used blocks and free blocks are given considering just the ones present or used from arenas created by my library and not mmap/munmap.

mallinfo() has also been implemented. But it is not a full version like glibc. My implementation will provide information like the number of free blocks and used blocks taken across all the arena together. This API provides total information of all arenas, which is unlike malloc_stats() that provides information per arena.


## Running the tests and other Makefile recipes

Please check the 'Makefile' for detailed recipes. Important ones are as follows:

1) make check: Runs the test suite for t-test1.c
2) make check1: Runs the test suite for test1.c (my single-threaded test)
3) make check2: Runs the test suite for test2_withThreads.c (my multi-threaded test)
4) make val: Runs Valgrind tool for t-test1.c.
5) make val1: Runs Valgrind tool for test1.c.
6) make clean: Removes generated binaries.
7) make deb : Debug using gdb t-test1.c
8) make deb1 : Debug using gdb test1.c (my single-threaded test)
9) make deb2 : Debug using gdb test2_withThreads.c (my multi-threaded test)




## Valgrind tool's results for t-test1.c

HEAP SUMMARY:
==29744==     in use at exit: 0 bytes in 0 blocks
==29744==   total heap usage: 0 allocs, 0 frees, 0 bytes allocated
==29744== 
==29744== All heap blocks were freed -- no leaks are possible
==29744== 
==29744== For counts of detected and suppressed errors, rerun with: -v
==29744== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
==29742== 
==29742== HEAP SUMMARY:
==29742==     in use at exit: 2,653 bytes in 79 blocks
==29742==   total heap usage: 82 allocs, 3 frees, 2,765 bytes allocated
==29742== 
==29742== LEAK SUMMARY:
==29742==    definitely lost: 0 bytes in 0 blocks
==29742==    indirectly lost: 0 bytes in 0 blocks
==29742==      possibly lost: 0 bytes in 0 blocks
==29742==    still reachable: 2,653 bytes in 79 blocks
==29742==         suppressed: 0 bytes in 0 blocks
==29742== Reachable blocks (those to which a pointer was found) are not shown.
==29742== To see them, rerun with: --leak-check=full --show-leak-kinds=all
==29742== 
==29742== For counts of detected and suppressed errors, rerun with: -v
==29742== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)


## NOTE  

This library has been developed and tested on a 64-bit Linux (Ubuntu) system with a page size of 
4096 bytes. Using the library on a system with a different configuration might give rise to some 
minor issues as it has not been tested for the same.


## References

1) https://en.wikipedia.org/wiki/Buddy_memory_allocation
2) www.cs.uml.edu/~jsmith/OSReport/frames.html
3) http://www.cs.cmu.edu/afs/cs/academic/class/15213-s15/www/recitations/rec11.pdf
4) http://www.cs.cmu.edu/afs/cs/academic/class/15213-s11/www/lectures/20-allocation-advanced.pdf
5) https://stackoverflow.com/questions/1919183/how-to-allocate-and-free-aligned-memory-in-c

