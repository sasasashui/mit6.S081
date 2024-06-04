## 添加一个系统调用跟踪功能，您将创建一个新的trace系统调用来控制跟踪，传入的参数是系统调用的掩码
### 修改kernel/proc文件 为每个进程增加一个掩码
```C
// Per-process state
struct proc {
  ....
  int mask;                    // 用于跟踪sys
};
```
### 增加一个系统调用函数sys_trace
```C
uint64
sys_trace(void)
{
  int n;  // 系统调用号

  if(argint(0, &n) < 0)
    return -1;

  myproc()->mask = n;
}
```

### 修改kernel/syscall.c 增加一个系统调用名称数组
```C
static char *syscall_names[] = {
  "", "fork", "exit", "wait", "pipe", 
  "read", "kill", "exec", "fstat", "chdir", 
  "dup", "getpid", "sbrk", "sleep", "uptime", 
  "open", "write", "mknod", "unlink", "link", 
  "mkdir", "close", "trace"};
```

### 修改kernel/syscall.c的syscall函数，在每次调用系统调用后打印信息
```C
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;  // 系统调用号存储在a7寄存器
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();  // 返回值存储在a0寄存器
    if(1 << num & p->mask){
      printf("%d syscall %s -> %d\n",p->pid,syscall_names[num],p->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

### 修改fork函数 使得在子进程复制父进程信息时，将mask一起复制
```C
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  ...
  np->mask = p->mask;
  ...
}
```

## 您将添加一个系统调用sysinfo，它收集有关正在运行的系统的信息。系统调用采用一个参数：一个指向struct sysinfo的指针（参见kernel/sysinfo.h）。内核应该填写这个结构的字段：freemem字段应该设置为空闲内存的字节数

### 在kernel/kalloc.c中增加一个函数获取空闲内存字节数
```C
uint64 get_freemem()
{
  int n;
  struct run* r = kmem.freelist;  // 空闲内存的头

  acquire(&kmem.lock);
  while(r){
    n ++;
    r = r->next;
  }
  release(&kmem.lock);
  return n * PGSIZE;
}
```
### 在kernel/proc.c中增加一个函数获取空闲进程数
```C
uint64 getproc()
{
  struct proc *p;
  int n;
  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      n ++;
    }
    release(&p->lock);
  }
  return 0;
}
```

### 增加系统调用 sys_info 将结构体复制到用户内存中
```C
uint64
sys_sysinfo(void)
{
  struct sysinfo si;
  uint64 addr;  // 用户地址
  struct proc* p =myproc();
  if (argaddr(0, &addr) < 0)
	  return -1;

  si.freemem = getfreemem();
  si.nproc = getproc();
  if(copyout(p->pagetable,addr,&si,sizeof(struct sysinfo)) != 0)// 将内核数据复制到用户的地址中
  {
    return -1;
  }  
  return 0;
}
```
