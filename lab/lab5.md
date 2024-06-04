## 实现懒分配
### 修改sbrk函数 处理增长和减少进程大小的情况
```C
int64
sys_sbrk(void)
{
  int addr;
  int n;
  int sz;
  struct proc *p = myproc();

  if(argint(0, &n) < 0)
    return -1;
  
  addr = myproc()->sz;

  if(n < 0){  // 减小内存
    sz = p->sz;
    sz = uvmdealloc(p->pagetable, sz, sz + n);
    p->sz = sz;
  }
  else{
    p->sz += n;
  }
  return addr;
}
```

### 修改kernel/vm.c 中的uvmunmap函数，处理未分配的页面的映射不能报错
```C
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      continue;  // 修改这里
    if((*pte & PTE_V) == 0)
      continue;  // 修改这里
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

### 修改kernel/vm.c 中的uvmcopy函数
```C
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      continue;  // 修改这里
      //panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      continue;
      //panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

### kernel/vm.c 中的walkaddr函数
```C
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if((pte == 0) || (*pte & PTE_V) == 0){
      if(lazy_alloction(va) == 0){
        pte = walk(pagetable, va, 0);
      }else{
        return 0;
      }
  }
  .....
}
```

### 懒分配函数实现
```C
int lazy_alloction(uint64 va){
  struct proc *p = myproc();
  if(va < p->sz){  // 懒分配的错误
    char *mem;
    mem = kalloc();
    if(mem == 0) return -1;
    memset(mem, 0, PGSIZE);
    if(mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
      kfree(mem);
      return -1;
    }
    return 0;
  }else{
    return -1;
  }
}

```

### 修改usertrap函数
```C
void
usertrap(void)
{
  .....
  
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
  } else if(r_scause() == 13 || r_scause() == 15){
    uint64 va = r_stval();
    if(va < p->sz){  // 懒分配的错误
      char *mem;
      mem = kalloc();
      if(mem == 0) p->killed = 1;
      memset(mem, 0, PGSIZE);
      if(mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0){
        kfree(mem);
        p->killed = 1;
      }
    } else{
      p->killed = 1;
    }
  }else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }
  .....
}
```



