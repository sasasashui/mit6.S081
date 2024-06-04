## 实现copy_on_write
### 在riscv.h中增加标志位
```C
#define PTE_C (1L << 5) // cow
```

### 在kalloc.c中增加结构体
```C
struct{
  struct spinlock lock;
  int pg_ref_cnt[PHYSTOP / PGSIZE];  // 每一页都有一个引用计数
}ref_cnt;
```

### 修改freeange和kinit函数
```C
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&ref_cnt.lock,"ref_cnt");
  freerange(end, (void*)PHYSTOP);
}

void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
  {
    acquire(&ref_cnt.lock);
    ref_cnt.pg_ref_cnt[(uint64)p / PGSIZE] = 1;
    release(&ref_cnt.lock);
    kfree(p);
  }
}
```

### 修改kfree
```C
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  acquire(&ref_cnt.lock);
  if(-- ref_cnt.pg_ref_cnt[(uint64)pa / PGSIZE] > 0){
    return;
  }else{  // 只有引用数为0的时候才会释放空间
    r = (struct run*)pa;

    acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
    release(&ref_cnt.lock);
  }
```

### 增加三个函数用于操作每个页的引用计数
```C
int add_ref_cnt(uint64 pa){
  acquire(&ref_cnt.lock);
  ref_cnt.pg_ref_cnt[pa / PGSIZE] ++;
  release(&ref_cnt.lock);
  return ref_cnt.pg_ref_cnt[pa / PGSIZE];
}

int sub_ref_cnt(uint64 pa){
  acquire(&ref_cnt.lock);
  ref_cnt.pg_ref_cnt[pa / PGSIZE] --;
  release(&ref_cnt.lock);
  return ref_cnt.pg_ref_cnt[pa / PGSIZE];
}

int get_ref_cnt(uint64 pa){
  return ref_cnt.pg_ref_cnt[pa / PGSIZE];
}
```

### 修改uvmcopy
```C
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  .....
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);

    flags &= ~PTE_W;  // 消除写标志位
    flags |= PTE_C;
    *pte = PA2PTE(pa) & flags;

    // if((mem = kalloc()) == 0)
    //   goto err;
    //memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, pa, flags) != 0){
      //kfree(mem);
      goto err;
    }
   .....
}
```

### 增加函数判断是不是copy_on_write的页面错误
```C
int is_cow(pagetable_t pagetable,uint64 va){
  pte_t * pte = walk(pagetable,va,0);
  if(*pte | PTE_C)
    return 0;
  else return -1;
}
```
### 增加函数用于解决copy_on_write的页面错误
```C
int cow_allocation(pagetable_t pagetable,uint64 va){
  if(va % PGSIZE != 0)
    return -1;
  pte_t *pte;
  uint flags;
  uint64 pa;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    return -1;
  if((*pte & PTE_V) == 0)
    return -1;
  
  pa = PTE2PA(*pte);
  flags = PTE_FLAGS(*pte);

  if(get_ref_cnt(pa) == 1){  // 引用计数为1 直接修改标志返回
    flags |= PTE_W;
    flags &= ~PTE_C;
    *pte = PA2PTE(pa) | flags;
    return 0;
  }else{  // 分配内存
    sub_ref_cnt(pa);
    char* mem = kalloc();
    if(mem == 0)
      return -1;
    memmove(mem, (char*)pa, PGSIZE);

    *pte &= ~PTE_V;  // 防止remap

    // 为新页面添加映射
    if(mappages(pagetable, va, PGSIZE, (uint64)mem, (PTE_FLAGS(*pte) | PTE_W) & ~PTE_C) != 0) {
      kfree(mem);
      *pte |= PTE_V;
      return -1;
    }
  }
  return 0;
}
```

### 修改trap.c
```C
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if(r_scause() == 13 || r_scause() == 15) {  // 在这里处理copy_on_write错误
    uint64 va = r_stval();  // 发送错误的虚拟地址
    if(is_cow(va,p->pagetable) != 0 && cow_allocation(p->pagetable,va) != 0){
      p->killed = -1;
    }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```
