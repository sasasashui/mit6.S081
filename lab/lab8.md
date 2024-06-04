## 设置内存分配器 减少锁的争用
### 为每个cpu设置一个内存链表
```C
struct {
  struct spinlock lock;
  struct run *freelist;
} cpu_kmem[NCPU];  // 一个cpu一个内存空闲表
```

### 修改kinit函数，初始化每个cpu的锁
```C
void
kinit()
{
  char lockname[8];
  for(int i = 0;i < NCPU;i ++){
    snprintf(lockname,sizeof lockname,"kmem%d",i);
    initlock(&cpu_kmem[i].lock, lockname);
  }
  //initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}
```

### 修改kfree释放锁
```C
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  push_off();  // 关中断
  uint64 cpu_id = r_tp();
  acquire(&cpu_kmem[cpu_id].lock);
  r->next = cpu_kmem[cpu_id].freelist;
  cpu_kmem[cpu_id].freelist = r;
  release(&cpu_kmem[cpu_id].lock);
  pop_off();  // 开中断

  // acquire(&kmem.lock);
  // r->next = kmem.freelist;
  // kmem.freelist = r;
  // release(&kmem.lock);
}
```

### 修改kalloc函数，使得当当前cpu没有内存的时候需要去其他cpu上获取
```C
void *
kalloc(void)
{
  struct run *r;

  push_off();
  uint64 cpu_id = r_tp();
  acquire(&cpu_kmem[cpu_id].lock);
  r = cpu_kmem[cpu_id].freelist;
  if(r){
    cpu_kmem[cpu_id].freelist = r->next;
    release(&cpu_kmem[cpu_id].lock);
    pop_off();
  }else{  // 当前cpu没有内存 去窃取其他的
    for(int i = 0;i < NCPU;i ++){
      if(i != cpu_id){
        acquire(&cpu_kmem[i].lock);
        struct run *tmp = cpu_kmem[i].freelist;
        if(tmp){
          r = tmp;
          cpu_kmem[i].freelist = cpu_kmem[i].freelist->next;
          release(&cpu_kmem[i].lock);
          break;
        }else{
          release(&cpu_kmem[i].lock);
        }
      }else{
        continue;
      }
    }
    release(&cpu_kmem[cpu_id].lock);
    pop_off();
  }

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

## 对于bcache锁使用的优化
### 设置哈希表的大小为13，每个桶的大小为固定大小5
```C
#define NBUCKET 13
#define NPERBUCKET 5
```

### 设置bacache的哈希形式
```C
struct {
  struct spinlock lock;
  struct buf buf[NPERBUCKET];
} bcache[NBUCKET];
```

### 初始化每个哈希桶的锁和每个buf的锁
```C
void
binit(void)
{
  // struct buf *b;

  // initlock(&bcache.lock, "bcache");

  // // Create linked list of buffers
  // bcache.head.prev = &bcache.head;
  // bcache.head.next = &bcache.head;
  // for(b = bcache.buf; b < bcache.buf+NBUF; b++){
  //   b->next = bcache.head.next;
  //   b->prev = &bcache.head;
  //   initsleeplock(&b->lock, "buffer");
  //   bcache.head.next->prev = b;
  //   bcache.head.next = b;
  // }

  char lockname[10];
  for(int i = 0;i < NBUCKET;i ++){
      snprintf(lockname,sizeof lockname,"bcache_%d",i);
      initlock(&bcache[i].lock,lockname);
      for(int j = 0;j < NPERBUCKET;j ++){
        initsleeplock(&bcache[i].buf[j].lock,"sleeplock");
      }
  }
}
```

### 更新buf结构体 增加tick表示时间戳
```C
struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE];
  uint ticks;  // 用于记录时间戳
};
```

### 修改bget函数
```C
static struct buf*
bget(uint dev, uint blockno)
{
  int hashid = blockno % NBUCKET;
  struct buf *b;

  acquire(&bcache[hashid].lock);

  for(int i = 0;i < NPERBUCKET;i ++){
    b = &bcache[hashid].buf[i];
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      b->ticks = ticks;
      release(&bcache[hashid].lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  int min_tick = 0x3f3f3f3f;
  int j;
  int new_id = -1;
  for(j = 0;j < NPERBUCKET;j ++){
    if(b->refcnt == 0 && b->ticks < min_tick) {
      new_id = j;
      min_tick = b->ticks;
    }
  }
  if(new_id == -1){
    panic("bget: no buffers");
  }else{
    b = &bcache[hashid].buf[new_id];
    b->dev = dev;
    b->blockno = blockno;
    b->valid = 0;
    b->refcnt = 1;
    release(&bcache[hashid].lock);
    acquiresleep(&b->lock);
    return b;
  }
  
}
```
### 修改brelese函数
```C
void
brelse(struct buf *b)
{
  int hashid = b->blockno % NBUCKET;
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  acquire(&bcache[hashid].lock);
  b->refcnt --;
  release(&bcache[hashid].lock);
  // acquire(&bcache.lock);
  // b->refcnt--;
  // if (b->refcnt == 0) {
  //   // no one is waiting for it.
  //   b->next->prev = b->prev;
  //   b->prev->next = b->next;
  //   b->next = bcache.head.next;
  //   b->prev = &bcache.head;
  //   bcache.head.next->prev = b;
  //   bcache.head.next = b;
  // }
  
  // release(&bcache.lock);
}
```

### 修改bpin和bunpin函数
```C
void
bpin(struct buf *b) {
  int hash_id = b->blockno % NBUCKET;
  acquire(&bcache[hash_id].lock);
  b->refcnt++;
  release(&bcache[hash_id].lock);
}

void
bunpin(struct buf *b) {
  int hash_id = b->blockno % NBUCKET;
  acquire(&bcache[hash_id].lock);
  b->refcnt --;
  release(&bcache[hash_id].lock);
}
```
