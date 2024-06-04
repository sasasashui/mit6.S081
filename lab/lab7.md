## 完成用户级线程的切换
### 修改线程的结构体，增加用于线程切换的寄存器
```C
struct thread {

  uint64 ra;
  uint64 sp;
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;

  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
};
```

### 修改初始化线程函数，设置线程的返回地址和栈指针
```C
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->ra = (uint64)func;
  t->sp = (uint64)t->stack + STACK_SIZE - 1;

}
```

### 编写线程切换的汇编函数，用于切换不同线程的寄存器
```C
thread_switch:
	/* YOUR CODE HERE */

	sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)

	ret    /* return to ra */
```

## unix下使用锁来实现哈希查找和删除

### 初始化锁，每一个哈希桶一个锁
```C
pthread_mutex_t lock[NBUCKET];

void init()
{
  for(int i = 0;i < NBUCKET;i ++)
  {
    pthread_mutex_init(&lock[i], NULL); // initialize the lock
  }
}
```

### 在put函数中，插入数的时候需要加锁,防止两个进程找到同一个位置导致插入的值相互覆盖
```C
static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  pthread_mutex_lock(&lock[i]);       // acquire lock
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&lock[i]);     // release lock
}
```

### 实现一个屏障，使得所有线程在这个屏障处等待
```C
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);       // acquire lock
  bstate.nthread ++;
  if(bstate.nthread == nthread){
    bstate.nthread = 0;
    bstate.round ++;
    pthread_cond_broadcast(&bstate.barrier_cond);
  }else{
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);     // release lock
}
```



