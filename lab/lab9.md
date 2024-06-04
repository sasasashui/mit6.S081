## 为xv6实现二级索引块 以增加文件存储大小
### 修改直接索引块的大小 inode的数据块地址数组的表现形式
```C
#define NDIRECT 11
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+2];   // Data block addresses
};

struct inode {
  ...
  uint addrs[NDIRECT+2];
};
```
### 修改bmap，实现二级索引块的分配和查找
```C
static uint
bmap(struct inode *ip, uint bn)  // 根据inode和bn读取数据块号
{
  uint addr, *a;
  struct buf *bp,*bp2;

  if(bn < NDIRECT){  // 直接块
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  bn -= NINDIRECT;
  uint level1 = bn / NINDIRECT;
  uint level2 = bn % NINDIRECT;

  if(bn < NINDIRECT * NINDIRECT){
    if((addr = ip->addrs[NDIRECT + 1]) == 0)
      ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
    bp = bread(ip->dev,addr);
    a = (uint*)bp->data;
    if((addr = a[level1]) == 0){
      a[level1] = addr = balloc(ip->dev);
      log_write(bp);
    }
    bp2 = bread(ip->dev,addr);
    a = (uint*)bp->data;
    if((addr = a[level2]) == 0){
      a[level2] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp2);
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

### 修改itrunck函数，使得索引块都合理释放
```C
void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp,*bp2;
  uint *a,*a2;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  if(ip->addrs[NDIRECT + 1]){
    bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    a = (uint*)bp->data;
    for(i = 0;i < NINDIRECT;i ++){
      bp2 = bread(ip->dev,a[j]);
      a2 = (uint*)bp2->data;
      for(int j = 0;j < NINDIRECT;j ++){
        if(a2[j])
          bfree(ip->dev, a[j]);
      }
      brelse(bp2);
      bfree(ip->dev,a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```

## 设置软链接
### 新增文件类型
```C
#define T_SYMLINK 4   // 软链接
```

### 实现sys_symlink软链接函数
```C
uint64
sys_symlink(void)
{
  char target[MAXPATH],path[MAXPATH];
  struct inode *ip;
  if(argstr(0, target, MAXPATH) < 0 || argstr(0, target, MAXPATH) < 0){
    return -1;
  }
  begin_op();
  ip = create(path,T_SYMLINK,0,0);
  if(ip == 0){
    end_op();
    return -1;
  }
  
  if(writei(ip,0,(uint64)target,0,sizeof(target)) < sizeof(target)){
    iunlockput(ip);
    end_op();
    return -1;
  }

  iunlockput(ip);
  end_op();
  return 0;
}
```

### 修改sys_open系统调用函数，使其能够打开软件接的函数
```C
uint64
sys_open(void)
{
  char path[MAXPATH],target[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  if((n = argstr(0, path, MAXPATH)) < 0 || argint(1, &omode) < 0)
    return -1;

  begin_op();

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
      end_op();
      return -1;
    }
    ilock(ip);
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
  }

  if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
    iunlockput(ip);
    end_op();
    return -1;
  }

  if(ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)){
    int depth = 0;
    int tmp;
    while(ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)){
      if((tmp = readi(ip,0,(uint64)target,0,sizeof(target))) < -1){
        iunlockput(ip);
        end_op();
        return -1;
      }
      depth ++;
      if(depth == 10)
      {
        iunlockput(ip);
        end_op();
        return -1;
      }
      iunlockput(ip);
      if((ip = namei(target)) == 0)
      {
        end_op();
        return -1;
      }
      ilock(ip);
      }
  }

  if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
    if(f)
      fileclose(f);
    iunlockput(ip);
    end_op();
    return -1;
  }

  if(ip->type == T_DEVICE){
    f->type = FD_DEVICE;
    f->major = ip->major;
  } else {
    f->type = FD_INODE;
    f->off = 0;
  }
  f->ip = ip;
  f->readable = !(omode & O_WRONLY);
  f->writable = (omode & O_WRONLY) || (omode & O_RDWR);

  if((omode & O_TRUNC) && ip->type == T_FILE){
    itrunc(ip);
  }

  iunlock(ip);
  end_op();

  return fd;
}
```

