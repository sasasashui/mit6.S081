## 回溯(Backtrace)，通过栈指针打印函数的调用链
```C
void
backtrace()
{
  printf("backtrace:\n");
  uint64 now_fp = r_fp();  // 指向上一个函数的栈帧
  while(PGROUNDUP(now_fp) - PGROUNDDOWN(now_fp) == PGSIZE){
    printf("%p\n",*(uint64*)(now_fp - 8));  // 返回地址
    now_fp = *(uint64*)(now_fp - 16);  // 回到上一个函数
  }

}
```

## 每隔几个时钟中断进入一次alarm函数
### 在kernel/proc.h中加入字段
```C
struct proc {
  ....
  int alarm_interval;          // 报警间隔
  uint64 handler;           // 报警函数
  int tick;                    // 该进程的滴答声

  int is_alarm;                // 是否正在报警中
  struct trapframe *alarm_trapframe;  // 执行完报警函数后用于恢复原进程
};
```

### 增加系统调用 接受报警间隔和报警函数
```C
uint64
sys_sigalarm(void){
  int n;
  uint64 addr;

  if(argint(0, &n) < 0){
    return -1;
  }
  if(argaddr(1,&addr) < 0){
    return -1;
  }
  struct proc* p = myproc();
  p->alarm_interval = n;
  p->handler = addr;
  return 0;
}
```

### 增加系统调用，当函数通过系统调用从中断返回时返回到原中断函数
```C
uint64
sys_sigreturn(void){
  struct proc *p = myproc();
  memmove(myproc()->trapframe, myproc()->alarm_trapframe, sizeof(struct trapframe));
  p->is_alarm = 0;
  p->tick = 0;
  // p->handler = (uint64)0;
  // p->alarm_interval = 0;
  return 0;
}
```

### 修改kernel/usertrap.c,当出现定时器中断时加入报警函数
```C
uint64
sys_sigreturn(void){
  struct proc *p = myproc();
  memmove(myproc()->trapframe, myproc()->alarm_trapframe, sizeof(struct trapframe));
  p->is_alarm = 0;
  p->tick = 0;
  // p->handler = (uint64)0;
  // p->alarm_interval = 0;
  return 0;
}
```
