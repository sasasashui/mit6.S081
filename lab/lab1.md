### sleep 学习调用系统调用的接口
```C
#include "kernel/types.h"
#include "user.h"

int
main(int argc, char *argv[])
{
    if(argc < 2){
        fprintf(2,"the number of arguments is wrong\n");
        exit(1);
    }

    int time = atoi(argv[1]);
    sleep(time);
    exit(0);

}
```
### pingpong 父进程和子进程通过管道传输数据
```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user.h"

int
main(int argc, char *argv[]){
    int p1[2],p2[2];  // p1是父进程 p2是子进程
    char s1[10] = "father";
    char s2[10] = "child";

    pipe(p1),pipe(p2);

    int pid = fork();
    if(pid == 0){  // 子进程
        close(p2[1]);  // 只读父进程的数据
        close(p1[0]);  // 只写数据

        printf("<%d> received ping\n",getpid());
        write(p1[1],s2,sizeof(s2));

        close(p1[1]);
        close(p2[0]);
    }else{  // 父进程
        close(p2[0]);  // 往管道2写数据
        close(p1[1]);  // 从管道1读数据

        printf("<%d> received pong\n",getpid());
        write(p2[1],s1,sizeof(s1));

        close(p2[1]);
        close(p1[0]);
    }
    exit(0);
}
```

### primes使用管道编写prime sieve(筛选素数)的并发版本
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user.h"

void primes(int *p1){
    int p2[2];
    int n;  // 读取父进程来的素数
    int tmp;  // 后续要判断的数据

    pipe(p2);
    close(p1[1]);

    if(read(p1[0],&n,sizeof(int)) == sizeof(int)){
        int pid = fork();
        if(pid == 0){
            primes(p2);
        }else{
            close(p2[0]);
            printf("primes %d",n);
            while(read(p1[0],&tmp,sizeof(int)) == sizeof(int)){
                if(tmp % n != 0){  // 如果余数不为0 传到下一个管道
                    write(p2[1],&tmp,sizeof(int));
                }
            }
            close(p1[0]);
            close(p2[1]);
            close(p2[2]);
            exit(0);
        }

    }else{  // 没有字符那么长度为0
        close(p1[0]);
        close(p2[0]);
        close(p2[1]);
        exit(0);  
    }

}

int
main(int argc, char *argv[]){
    int p[2];
    int i;
    pipe(p);

    int pid = fork();
    if(pid == 0){  // 子进程
        primes(p);
    }else{  // 父进程
        close(p[0]);  // 往内部写数据
        for(i = 2;i <= 35;i ++){
            write(p[1],&i,sizeof(int));
        }
        close(p[1]);
        wait(0);
    }
    exit(0);
}
```

### 实现find 在指定目录下递归查找文件
```C

#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
 
void find(char *path, char *target){
	char buf[256], *p;
	int fd;
	struct dirent de;
	struct stat st;
	if((fd = open(path, 0)) < 0){
		fprintf(2, "find: cannot open %s\n", path);
		return;
	}
	if(fstat(fd, &st) < 0){
		fprintf(2, "find: cannot stat %s\n", path);
		return;
	}
	
	switch(st.type){
	case T_FILE:
		if(strcmp(path + strlen(path) - strlen(target), target) == 0){
			printf("%s\n", path);
		}
		break;
	case T_DIR:
		if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
			printf("find: path too long\n");
			break;
		}
		strcpy(buf, path);
		p = buf + strlen(buf);
		*p++ = '/';
		while(read(fd, &de, sizeof(de)) == sizeof(de)){
			if(de.inum == 0 || strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
				continue;
			memmove(p, de.name, strlen(de.name));
			p[strlen(de.name)] = 0;
			find(buf, target);
		}
		break;
	}
	close(fd);
}
 
int main(int argc, char *argv[]){
	if(argc < 3){
		exit(0);
	}

	find(argv[1], argv[2]);  // .  第二个参数在传递的时候是不变的
	exit(0);
}
```
