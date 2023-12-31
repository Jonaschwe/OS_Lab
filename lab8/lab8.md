#### 练习1: 完成读文件操作的实现（需要编码）首先了解打开文件的处理流程，然后参考本实验后续的文件读写操作的过程分析，填写在 kern/fs/sfs/sfs_inode.c中 的sfs_io_nolock()函数，实现读文件中数据的代码。
![lab81.png](https://s2.loli.net/2023/12/31/U8tdTuzHyZowAmx.png)

首先计算偏移量在一个数据块中的偏移(blkoff)。如果blkoff不等于0，说明起始位置不在一个数据块的开头，需要分两步读取数据： a. 计算需要读取的数据块大小(size)，如果nblks不等于0，则为一个数据块减去blkoff，否则为剩余数据长度(endpos - offset)。 b. 调用sfs_bmap_load_nolock函数加载对应的数据块，并得到对应的inode号(ino)。 c. 调用sfs_buf_op函数将数据读取到缓冲区(buf)中，参数包括文件系统(sfs)、缓冲区(buf)、数据块大小(size)、inode号(ino)以及数据块内的偏移量(blkoff)。 d. 更新已读取的总字节数(alen)和缓冲区指针(buf)。 e. 如果nblks等于0，说明已经读取完所需的数据块，直接跳转到结束(out)。 f. 更新下一个数据块的编号(blkno)和剩余数据块数量(nblks)。如果nblks大于0，说明还需要继续读取数据块： a. 调用sfs_bmap_load_nolock函数加载下一个数据块，并得到对应的inode号(ino)。 b. 调用sfs_block_op函数将数据读取到缓冲区(buf)中，参数包括文件系统(sfs)、缓冲区(buf)、inode号(ino)以及需要读取的数据块数量(nblks)。 c. 更新已读取的总字节数(alen)和缓冲区指针(buf)。 d. 更新下一个数据块的编号(blkno)和剩余数据块数量(nblks)。

![lab82.png](https://s2.loli.net/2023/12/31/MhFa6As8RUWliz5.png)

首先计算结束位置在一个数据块中的偏移(size)。如果size不等于0，说明结束位置不在数据块末尾，需要读取部分数据： a. 调用sfs_bmap_load_nolock函数加载对应的数据块，并得到对应的inode号(ino)。 b. 调用sfs_buf_op函数将数据读取到缓冲区(buf)中，参数包括文件系统(sfs)、缓冲区(buf)、数据块大小(size)、inode号(ino)以及数据块内的偏移量(0)。 c. 更新已读取的总字节数(alen)。

#### 练习2: 完成基于文件系统的执行程序机制的实现（需要编码）

改写proc.c中的load_icode函数和其他相关函数，实现基于文件系统的执行程序机制。执行：make qemu。如果能看看到sh用户程序的执行界面，则基本成功了。如果在sh用户界面上可以执行”ls”,”hello”等其他放置在sfs文件系统中的其他执行程序，则可以认为本实验基本成功。


该实验核心是对已有的进程初始化时分配对应的文件系统资源。同时将文件的加载改为真正的从“磁盘”加载。

修改alloc_proc，添加初始化资源：

```cpp
proc->filesp = NULL; // lab8
```

修改加载文件的操作：

```cpp
    //进行了修改，因为之前load_icode传的参数是binary，而这次传的参数是fd，所以需要用一个elfhdr来存储读进来的内容
    //使用fd索引fd_array，得到对应的file，根据file->inode->sfs_inode->sfs_disk_inode->size得到ELF文件大小。
    struct elfhdr __elf, *elf = &__elf;
    //这里调用了load_icode_read函数，它调用的read和seek函数，seek是找到要读的这个文件，read就是具体的读操作
    if ((ret = load_icode_read(fd, elf, sizeof(struct elfhdr), 0)) != 0) {
        goto bad_elf_cleanup_pgdir;
    }

    if (elf->e_magic != ELF_MAGIC) {
        ret = -E_INVAL_ELF;
        goto bad_elf_cleanup_pgdir;
    }
```

对应的lode_icode_read真正执行读取文件的操作，通过练习一提供的接口找到偏移位置并读出数据：

```cpp
// load_icode_read 在 LAB8 中被 load_icode 函数调用
static int
load_icode_read(int fd, void *buf, size_t len, off_t offset) {
    int ret;

    // 将文件指针设置到指定的偏移位置
    if ((ret = sysfile_seek(fd, offset, LSEEK_SET)) != 0) {
        return ret;
    }

    // 从文件中读取 len 字节的数据到 buf 中
    if ((ret = sysfile_read(fd, buf, len)) != len) {
        return (ret < 0) ? ret : -1;
    }

    return 0;  // 成功加载数据
}
```

同时，修改用户栈空间，计算栈顶和栈底后填入argc与argv：

```cpp
    // 为用户栈设置 vma
    vm_flags = VM_READ | VM_WRITE | VM_STACK;
    if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0) {
        goto bad_cleanup_mmap;
    }

    // 分配用户栈页面
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP - PGSIZE, PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP - 2 * PGSIZE, PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP - 3 * PGSIZE, PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP - 4 * PGSIZE, PTE_USER) != NULL);

    mm_count_inc(mm);
    current->mm = mm;
    current->cr3 = PADDR(mm->pgdir);
    lcr3(PADDR(mm->pgdir));

    // 设置 argc, argv
    uint32_t argv_size = 0, i;
    for (i = 0; i < argc; i++) {
        argv_size += strnlen(kargv[i], EXEC_MAX_ARG_LEN + 1) + 1;
    }

    // 计算用户栈的起始地址
    uintptr_t stacktop = USTACKTOP - (argv_size / sizeof(long) + 1) * sizeof(long);
    char **uargv = (char **)(stacktop - argc * sizeof(char *));

    // 复制参数到用户栈
    argv_size = 0;
    for (i = 0; i < argc; i++) {
        uargv[i] = strcpy((char *)(stacktop + argv_size), kargv[i]);
        argv_size += strnlen(kargv[i], EXEC_MAX_ARG_LEN + 1) + 1;
    }

    // 将 argc 放入用户栈
    stacktop = (uintptr_t)uargv - sizeof(int);
    *(int *)stacktop = argc;
```
