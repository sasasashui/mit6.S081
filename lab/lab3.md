## Print a page table 打印页表
### 在vm.c中增加两个函数vmprint(pagatable_t pagetable,int u)和vvmprint(pagatable_t pagetable,int u)
```C
void vmprint(pagetable_t pagetable){
  printf("pagetable %p\n",pagetable);
  vvmprint(pagetable,0);
}

// 遍历三级页表打印有效的pte和指向的物理地址
void vvmprint(pagetable_t pagetable,int u){
  if(u == 3) return;
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V)){
      for(int i = 0;i <= u;i ++)
      {
        if(i == 0) printf("..");
        else printf(" ..");
      }
      printf("%d:",i);
      printf(" pte %p pa %p\n",pte,PTE2PA(pte));
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        vvmprint((pagetable_t)child, u + 1);
      }
    } else if(pte & PTE_V){
      continue;
    }
  }
}
```

## 为每个用户进程设置一个内核页表
### 在kernel/proc.h中增加一个内核页表字段
```C
struct proc {
  ...
  pagetable_t k_pagetable;     //指向内核页表
};
```
### 仿照kvmmap进行页表的映射,在kernel/vm.c中增加
```C
void
ukvmmap(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(pagetable, va, sz, pa, perm) != 0) // 和kvmmap不同的是可以是用户页表
    panic("uvmmap");
}
```
### 进行内核页表映射同时创建内核页表
```C
pagetable_t
proc_k_pagetable()      
{
  pagetable_t k_pagetable = (pagetable_t) kalloc();
  memset(k_pagetable, 0, PGSIZE);
  
  // uart registers                
  ukvmmap(k_pagetable, UART0,UART0, PGSIZE, PTE_R | PTE_W);         

  // virtio mmio disk interface
  ukvmmap(k_pagetable,VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  ukvmmap(k_pagetable,CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  ukvmmap(k_pagetable,PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  ukvmmap(k_pagetable,KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  ukvmmap(k_pagetable,(uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  ukvmmap(k_pagetable,TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
  
  
  return  k_pagetable;

}
```

### 在allocproc中为每个进程内核页表进行初始化
```C
static struct proc*
allocproc(void){
  ...
  p->k_pagetable = uvmcreate();
  if(p->k_pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
  char *pa = kalloc();
  if(pa == 0)
    panic("kalloc");
  uint64 va = KSTACK((int) (p - proc));
  ukvmmap(p->k_pagetable, va,(uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack = va;
  ...
}
```

### 增加解除映射的函数
```C
// 解除映射但是不能释放叶子节点的内存,只是把叶子节点的值置为0
void
ukvmunmap(pagetable_t pagetable,uint64 va, uint64 npages)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("ukvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)  // 叶子节点应该是PTE_V加R/W/X的标志位
      panic("ukvmunmap: not a leaf");
    *pte = 0;  // 叶子节点解除映射
  }
}
```

### 释放进程的内核页表所占的内存函数
```C
// 这个递归函数终止于最后一层，因为调用了ukvmunmap，最后一层的pte内容全为0
void
ukfreewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){  // 这是一个PTE
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      ukfreewalk((pagetable_t)child);
      pagetable[i] = 0;
    }
  }
  kfree((void*)pagetable);
}
```

### 释放进程的内核页表
```C
void
proc_k_freepagetable(pagetable_t kpagetable)
{
  
  ukvmunmap(kpagetable, TRAMPOLINE, PGSIZE/PGSIZE);

  ukvmunmap(kpagetable, (uint64)etext, (PHYSTOP-(uint64)etext)/PGSIZE);

  ukvmunmap(kpagetable, KERNBASE, ((uint64)etext-KERNBASE)/PGSIZE);

  ukvmunmap(kpagetable, PLIC, 0x400000/PGSIZE);

  ukvmunmap(kpagetable, CLINT, 0x10000/PGSIZE);

  ukvmunmap(kpagetable, VIRTIO0, PGSIZE/PGSIZE);

  ukvmunmap(kpagetable, UART0, PGSIZE/PGSIZE);

  ukfreewalk(kpagetable);
}
```
### 在freeproc中调用
```C
static void
freeproc(struct proc *p)
{
  ```
  ukvmunmap(p->k_pagetable, p->kstack, PGSIZE/PGSIZE);
  if(p->k_pagetable)
    proc_k_freepagetable(p->k_pagetable);
  p->pagetable = 0;
  p->sz = 0;
  ````
}
```

## 简化copyin/copyinstr
###




