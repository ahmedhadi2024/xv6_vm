# Assessment

**Register No:** 24BCE2743
**Name:** Ahmed Hadi
**Course name:** Operating Systems
**Course code:** BCSE303L

**Question:**

1.  Inspect vm.c, mmu.h and proc.h. Identify walkpgdir(), mappages() and copyuvm().
2.  Write a user program that declares a global variable, a local variable, and dynamically allocated memory using malloc().
3.  Add a kernel diagnostic or test-only system call to inspect the page-table mapping for valid user virtual addresses.
4.  Display the virtual address, physical page address, and page permission flags for each address.
5.  Use fork() to demonstrate that parent and child initially have the same variable values but can modify their private copies independently.

**Solution:**

The work was carried out on the x86 version of xv6 (xv6-public), since walkpgdir(), copyuvm() and mmu.h are identifiers specific to that tree; the RISC-V version names the equivalents walk(), uvmcopy() and riscv.h.

**1) The three functions**

**walkpgdir()** performs the translation itself. Given a page directory and a virtual address, it returns a pointer to the page table entry governing that address. The x86 architecture uses a two-level table: the top ten bits of the virtual address index the page directory to locate a page table, and the next ten bits index into that table to reach the entry. The lowest twelve bits are the offset within the page and are never translated, passing through to the physical address unchanged. The alloc argument determines the behaviour when the second-level table does not yet exist: create one, or return zero. The function is declared static, so it is visible only within vm.c.

```c
static pte_t *
walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
  pde_t *pde;
  pte_t *pgtab;

  pde = &pgdir[PDX(va)];                       // top 10 bits
  if(*pde & PTE_P){
    pgtab = (pte_t*)P2V(PTE_ADDR(*pde));
  } else {
    if(!alloc || (pgtab = (pte_t*)kalloc()) == 0)
      return 0;
    memset(pgtab, 0, PGSIZE);
    *pde = V2P(pgtab) | PTE_P | PTE_W | PTE_U;
  }
  return &pgtab[PTX(va)];                      // next 10 bits
}
```

**mappages()** is the writer. For each page in a range it walks to the corresponding entry, this time with alloc set, and installs a physical address together with a set of permission flags. Every mapping created in the system passes through this function.

```c
if((pte = walkpgdir(pgdir, a, 1)) == 0)
  return -1;
*pte = pa | perm | PTE_P;
```

**copyuvm()** is the function responsible for isolation across fork(). It constructs an entirely new page directory for the child. For each page present in the parent it locates the existing mapping with walkpgdir(), allocates a fresh physical page with kalloc(), copies the contents across with memmove(), and installs the new page in the child at the same virtual address using mappages().

```c
for(i = 0; i < sz; i += PGSIZE){
  pte   = walkpgdir(pgdir, (void*)i, 0);      // parent's entry
  pa    = PTE_ADDR(*pte);
  flags = PTE_FLAGS(*pte);
  mem   = kalloc();                           // new physical page
  memmove(mem, (char*)P2V(pa), PGSIZE);       // copy contents
  mappages(d, (void*)i, PGSIZE, V2P(mem), flags);
}
```

This is an eager copy. xv6 implements no copy-on-write, so as soon as fork() returns, the child's virtual addresses are identical to the parent's while the physical pages behind them are entirely separate. This is what the demonstration in part 5 makes visible.

**The supporting definitions.** mmu.h describes the layout of a page table entry. The upper twenty bits hold the physical page frame and the lower twelve hold flags, with PTE_ADDR and PTE_FLAGS provided to separate them.

```c
#define PTE_P    0x001   // Present
#define PTE_W    0x002   // Writeable
#define PTE_U    0x004   // User

#define PTE_ADDR(pte)   ((uint)(pte) & ~0xFFF)    // top 20 bits: frame
#define PTE_FLAGS(pte)  ((uint)(pte) &  0xFFF)    // bottom 12: flags
```

PTE_U is the access control bit. A page without it cannot be accessed from user mode at all, which is the mechanism by which kernel memory remains unreachable from every process.

In proc.h, each process carries its own pgdir pointer, which is the root of its private address space.

```c
struct proc {
  ...
  pde_t*  pgdir;      // Page table
  ...
};
```

**3) and 4) The diagnostic system call**

The diagnostic is implemented in vm.c, because walkpgdir() is static to that file and is not visible from anywhere else in the kernel. Rather than removing that qualifier, the lookup is placed alongside it and exported through a single declaration in defs.h.

**vm.c**

```c
int
vminfo(uint va)
{
  struct proc *p = myproc();
  pte_t *pte;
  uint frame, pa, flags;

  pte = walkpgdir(p->pgdir, (char*)va, 0);
  if(pte == 0){
    cprintf("  [kernel] va 0x%x -> no page table at this address\n", va);
    return -1;
  }
  if(!(*pte & PTE_P)){
    cprintf("  [kernel] va 0x%x -> pte present bit clear\n", va);
    return -1;
  }

  frame = PTE_ADDR(*pte);          // physical page frame
  pa    = frame | (va & 0xFFF);    // frame + offset within the page
  flags = PTE_FLAGS(*pte);

  cprintf("  [kernel] va 0x%x  pa 0x%x  frame 0x%x  flags 0x%x [%s%s%s]\n",
          va, pa, frame, flags,
          (flags & PTE_P) ? "P" : "-",
          (flags & PTE_W) ? "W" : "-",
          (flags & PTE_U) ? "U" : "-");
  return 0;
}
```

The lookup passes zero for alloc, so it inspects the existing mapping and never creates one. The reported physical address is the frame with the page offset restored, since those low bits are not part of the translation.

**defs.h**

```c
int             vminfo(uint);
```

**syscall.h**

```c
#define SYS_vminfo 22
```

**sysproc.c**

```c
int
sys_vminfo(void)
{
  int va;

  if(argint(0, &va) < 0)
    return -1;
  return vminfo((uint)va);
}
```

**syscall.c**

```c
extern int sys_vminfo(void);

[SYS_vminfo]  sys_vminfo,
```

**user.h and usys.S**

```c
int vminfo(uint);

SYSCALL(vminfo)
```

**2) and 5) The test program**

The program declares one variable of each storage class: a global in the data segment, a local on the stack, and an integer obtained from malloc() on the heap. These occupy three distinct regions of the process address space. It reports the mapping of each, forks, and repeats the same report in the child before modifying all three values.

**vmtest.c**

```c
#include "types.h"
#include "stat.h"
#include "user.h"

int global_var = 42;        // .data  — initialised global

int
main(void)
{
  int local_var = 7;        // stack
  int *heap_var;            // heap, via malloc -> sbrk
  int pid;

  heap_var = malloc(sizeof(int));
  *heap_var = 99;

  printf(1, "=== parent, pid %d ===\n", getpid());
  printf(1, "  global_var = %d   at va 0x%x\n", global_var, (uint)&global_var);
  printf(1, "  local_var  = %d   at va 0x%x\n", local_var,  (uint)&local_var);
  printf(1, "  heap_var   = %d   at va 0x%x\n", *heap_var,  (uint)heap_var);

  printf(1, "parent page-table mappings:\n");
  vminfo((uint)&global_var);
  vminfo((uint)&local_var);
  vminfo((uint)heap_var);

  pid = fork();

  if(pid == 0){
    printf(1, "\n=== child, pid %d — values inherited ===\n", getpid());
    printf(1, "  global %d   local %d   heap %d\n",
           global_var, local_var, *heap_var);

    printf(1, "child page-table mappings (same VAs, new frames):\n");
    vminfo((uint)&global_var);
    vminfo((uint)&local_var);
    vminfo((uint)heap_var);

    global_var = 100;
    local_var  = 200;
    *heap_var  = 300;

    printf(1, "child after writing: global %d  local %d  heap %d\n",
           global_var, local_var, *heap_var);
    exit();
  }

  wait();

  printf(1, "\n=== parent after child exited ===\n");
  printf(1, "  global %d   local %d   heap %d   <- unchanged\n",
         global_var, local_var, *heap_var);

  exit();
}
```

**Output:**

```c
$ make clean
$ make qemu-nox
```

```
$ vmtest
=== parent, pid 3 ===
  global_var = 42   at va 0xCE0
  local_var  = 7   at va 0x2FBC
  heap_var   = 99   at va 0xAFF8
parent page-table mappings:
  [kernel] va 0xce0  pa 0xdee2ce0  frame 0xdee2000  flags 0x67 [PWU]
  [kernel] va 0x2fbc  pa 0xdedffbc  frame 0xdedf000  flags 0x67 [PWU]
  [kernel] va 0xaff8  pa 0xdfc4ff8  frame 0xdfc4000  flags 0x67 [PWU]

=== child, pid 4 - values inherited ===
  global 42   local 7   heap 99
child page-table mappings (same VAs, new frames):
  [kernel] va 0xce0  pa 0xdf2cce0  frame 0xdf2c000  flags 0x67 [PWU]
  [kernel] va 0x2fbc  pa 0xdf74fbc  frame 0xdf74000  flags 0x67 [PWU]
  [kernel] va 0xaff8  pa 0xded7ff8  frame 0xded7000  flags 0x67 [PWU]
child after writing: global 100  local 200  heap 300

=== parent after child exited ===
  global 42   local 7   heap 99   <- unchanged
```

**Observations**

**The three variables occupy three distinct regions.** The global sits at virtual address 0xCE0, inside the very first page, which is the data segment immediately above the program text. The local sits at 0x2FBC, which is the top of the page beginning at 0x2000 and is the user stack. The heap allocation is at 0xAFF8, higher than both.

| Variable   | Storage class      | Virtual address | Region       |
|------------|--------------------|-----------------|--------------|
| global_var | initialised global | 0x0CE0          | data segment |
| local_var  | automatic          | 0x2FBC          | stack        |
| heap_var   | malloc()           | 0xAFF8          | heap         |

This matches the layout exec() builds. After loading the text and data it rounds the size up to a page boundary and allocates two further pages: the lower is left without PTE_U as a guard page, and the upper becomes the stack, which is why the local variable appears at the top of the page at 0x2000. Growth through sbrk() then continues above the stack, so the heap allocation lies at a higher address still. malloc() requests a large block from sbrk() on its first call and carves the returned pointer from near the end of it, which accounts for 0xAFF8 rather than an address immediately after the stack.

**The permission flags read 0x67, not 0x07.** The three bits named in mmu.h account for 0x07: PTE_P, PTE_W and PTE_U. The remaining two, 0x20 and 0x40, are the x86 Accessed and Dirty bits. mmu.h does not define names for them, yet they are set, because the processor itself writes them into the entry when a page is first read and first written. The page table is therefore not only read by the hardware during translation but modified by it, and those two bits are what a paging system would consult to decide which pages are candidates for eviction.

**The child's virtual addresses are identical and its physical frames are not.**

| Variable   | Virtual address | Parent frame | Child frame |
|------------|-----------------|--------------|-------------|
| global_var | 0x0CE0          | 0xDEE2000    | 0xDF2C000   |
| local_var  | 0x2FBC          | 0xDEDF000    | 0xDF74000   |
| heap_var   | 0xAFF8          | 0xDFC4000    | 0xDED7000   |

Every frame differs, and the page offsets within each frame are preserved exactly, since the low twelve bits of the virtual address are never translated. The child reported the inherited values 42, 7 and 99 before writing, then set them to 100, 200 and 300; the parent, on resuming after wait(), still held 42, 7 and 99.

**Explanation**

The three variables occupy different regions of the address space because of how exec() lays a process out. The initialised global lies in the data segment immediately above the program text. exec() then rounds the size up and allocates two further pages, the lower of which is left without PTE_U as a guard page and the upper of which becomes the user stack, so the local variable resides there. Subsequent growth through sbrk(), which is what malloc() uses when its free list is empty, extends the address space above the stack, so the heap allocation appears at a higher address still.

All three mappings carry the flags P, W and U: present, writable, and accessible from user mode. The user bit is the significant one. The kernel's own mappings are present and writable but lack PTE_U, and the processor itself refuses any user-mode access to such a page. Isolation here is not a convention enforced by kernel code; it is the memory management unit declining the access before any kernel instruction executes.

Across fork(), the child's virtual addresses are identical to the parent's while the physical frames differ, because copyuvm() allocated new pages and copied the contents rather than sharing them. The two processes therefore interpret the same numeric address through different page tables and reach different physical memory, which is why the child's writes of 100, 200 and 300 leave the parent's values of 42, 7 and 99 untouched.
