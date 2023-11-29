## 问答作业
### 思考题1.1

 `log` 库的代码具体在` ~/.cargo` 下的哪个文件夹？

按下转到定义按键后，`VSCode`自动跳转到`log`库代码定义文件，同时显示出文件路径：

![image-20231128133305674](https://jgox-image-1316409677.cos.ap-guangzhou.myqcloud.com/blog/image-20231128133305674.png)

代码具体在`~/.cargo`下的`registry/src/index.crates.io-6f17d22bba15001f/log-0.4.20/`。

### 思考题1.2

如果忘记 `make clean` 会发生什么？

我们去`makefile`文件看看`make clean`到底执行了什么操作：

```
clean:
	@cargo clean
```

`cargo clean`会删除整个`target`目录即之前构建的文件。如果修改了`crate`的源代码后忘记`make clean`直接`make run`，那么修改不会起作用，因为在上一次`make run`的时候编译器已经完成了对`crate`的编译，不`clean`再次`make run`的时候不会再去编译我们修改后的代码所以看不到修改后的效果。

### 思考题2

在[部分往届内核及运行指引](https://scpointer.github.io/rcore2oscomp/docs/lab3/before.html) 一节提到的内核中挑选一个，描述它在默认情况下启动后会执行哪些测例（抑或是直接启动终端）。你不一定要真的运行那个内核，读文档或搜索即可。

我选择的内核是[ByteOS](https://github.com/yfblock/ByteOS)。先查看项目文档了解项目结构，但没有具体的文档提到了第一个进程，于是直接跑去查看内核代码结构。

```
├── arch //该目录为ByteOS的架构支持，目前支持了RISCV64 
├── Cargo.lock
├── Cargo.toml
├── crates	//该目录为ByteOS的辅助功能crates，包括错误回溯，时间戳和虚拟文件系统以及为华山派开发板编写的SD卡驱动
├── cv1811h-burn.sh
├── docs	//该目录为项目的文档，需要查询文档可在该目录下查看
├── drivers	//该目录为ByteOS的设备驱动，包括K210的SD卡驱动，虚拟网卡驱动等
├── kernel	//该目录为ByteOS内核源代码
├── Makefile
├── modules	//该目录为ByteOS的功能模块，如日志，内存分配，设备管理，文件系统等
├── netperf.log
├── README.md
├── run.png
├── rust-toolchain.toml
├── target
└── tools
```

``` 
├── Cargo.toml
└── src
├── banner.txt
├── epoll	//epoll模块实现代码
│   └── mod.rs
├── main.rs	//主函数，系统入口
├── modules.rs	//内核模块定义
├── panic.rs	//错误处理
├── socket.rs	//套接字操作
├── syscall	//系统调用实现
│   ├── consts.rs	//系统调用相关常量定义
│   ├── fd.rs	//文件描述符的相关实现
│   ├── func.rs	//系统调用相关函数实现
│   ├── mm.rs	//内存管理相关调用的实现
│   ├── mod.rs	//系统调用公共接口和函数
│   ├── shm.rs	//共享内存的系统调用实现
│   ├── signal.rs	//信号处理相关系统调用的实现
│   ├── socket.rs	//套接字相关操作的实现
│   ├── sys.rs	//其他系统调用的实现
│   ├── task.rs	//任务管理实现
│   └── time.rs //时间和计时器的实现
├── task_cache.rs	//任务缓存的实现
└── tasks
├── async_ops.rs	//异步操作的实现
├── elf.rs	//ELF文件处理
├── initproc.rs	//初始化进程
├── kernel.rs	
├── mod.rs	//任务模块实现
└── user
├── entry.rs //用户空间代码入口点的实现
├── mod.rs	//用户空间任务实现
└── signal.rs	//用户任务信号处理
```

有一个`task`模块，那我们直接去查看`task`模块，找到了`initproc.rs`文件，`init process`，这就是我们要找的了🙏，翻一下函数找到了这个函数。

```
pub async fn initproc() {
```

![image-20231128202718963](https://jgox-image-1316409677.cos.ap-guangzhou.myqcloud.com/blog/image-20231128202718963.png)

那么很明显，内核执行的第一个进程是`busybox`！

### 思考题3.1

为什么要在开头结尾各输出一句，会不会太过重复？（提示：考虑执行出错的情况，或者 `sys_exit`）

如果进程陷入内核态执行`sys_call`的时候运行出错导致进程结束或者执行的系统调用就是`sys_exit`导致进程结束，但我们又只在开头加了输出语句，我们就不能够发现到这种情况，只在开头加也不能反应返回值。

或者我们只在结尾加了输出语句，那么对于那些`sys_call`执行过程中结束的进程我们根本不能发现。

### 思考题3.2

为什么要结尾还要输出一遍 `syscall` 的完整参数，只输出返回值行不行？（提示：考虑像 `sys_yield` 这样的 `syscall`）

像`sys_yield`这样的`syscall`，执行`syscall`的时候，处理机转而去执行其它进程，当进程再次被调度，可以验证上下文被正确恢复。

## 编程作业

###  实验3.1

修改 `ulib/axstarry/syscall_entry/src/test.rs` 的`SDCARD_TESTCASES` 常量为：

```rust
#[allow(dead_code)]
pub const SDCARD_TESTCASES: &[&str] = &[
    "busybox touch abc",
    "busybox mv abc bin/",
    "busybox ls bin/abc",
];
```

其中不能正常工作的命令是 `busybox mv abc bin/`。`busybox touch` 用于创建文件 `abc`，`busybox ls bin/abc` 用于检查文件 `bin/abc` 是否存在。你的任务是修改代码，使得 `busybox ls bin/abc` 正常输出 `bin/abc`。

第一步，首先在本机调用相同的命令，并且使用strace查看发生了哪些系统调用用于对拍，知道正常的mv到底发生了什么，在合适的终端输入下列指令，结果如下图。

```
mkdir bin
touch abc
strace mv abc bin/
```

![image-20231128161123470](https://jgox-image-1316409677.cos.ap-guangzhou.myqcloud.com/blog/image-20231128161123470.png)

不难理解图中蓝色方框的位置应当就是`mv`具体发生的过程。

- 先判断`new_path`的文件属性
- 再去判断移动后的文件是否已经存在
- 最后完成对文件的`rename`操作。

接下来，我们在`Starry`内核中运行同样的指令结果如下：

![image-20231128161050433](https://jgox-image-1316409677.cos.ap-guangzhou.myqcloud.com/blog/image-20231128161050433.png)

系统调用的名称不太一样没关系，但是很明显，这里在申请完内存后有一个叫作FSTATAT的系统调用返回了`-64`，和本机运行的结果不一致，查一下发现64意味着`Machine is not on the network`，这么离谱？？！这是一个根据给定的路径获取对应的`stat`的函数，调试发现函数的输入为`/bin/abc`，看来是对文件路径的判断有问题，找一下发现与判断逻辑有问题将`&&`改成`||`即可。

```rust
// wrong: if !real_path.ends_with("/") && !real_path.ends_with("include") {
if !real_path.ends_with("/") || !real_path.ends_with("include") {
        // 是文件
        return if let Ok(file) = new_fd(real_path.to_string(), 0.into()) {
            match file.get_stat() {
                Ok(stat) => Ok(stat),
                Err(e) => {
                    debug!("get stat error: {:?}", e);
                    Err(SyscallError::EINVAL)
                }
            }
        } else {
            Err(SyscallError::ENONET)
        };
    } else {
        // 是目录
        return if let Ok(dir) = new_dir(real_path.to_string(), OpenFlags::DIR) {
            match dir.get_stat() {
                Ok(stat) => Ok(stat),
                Err(e) => {
                    debug!("get stat error: {:?}", e);
                    Err(SyscallError::EINVAL)
                }
            }
        } else {
            Err(SyscallError::ENONET)
        };
    }
```

接下来再次`make run`，发现是缺少了一个系统调用，那我们就去完善这个系统调用就好了。

![image-20231128193452019](https://jgox-image-1316409677.cos.ap-guangzhou.myqcloud.com/blog/image-20231128193452019.png)

```rust
pub fn syscall_renameat2(
    old_dirfd: usize,
    old_path: *const u8,
    new_dirfd: usize,
    new_path: *const u8,
    flags: usize,
) -> SyscallResult {
    let old_path = deal_with_path(old_dirfd, Some(old_path), false).unwrap();
    let new_path = deal_with_path(new_dirfd, Some(new_path), false).unwrap();
    let proc_path = FilePath::new("/proc").unwrap();
    if old_path.start_with(&proc_path) || new_path.start_with(&proc_path) {
        return Err(SyscallError::EPERM);
    }
    error!(
        "old path:{} and new path:{} and proc path:{}",
        old_path.path(),
        new_path.path(),
        proc_path.path()
    );
    // LAB3 从此处往上的代码不需要修改
    //
    // HINT 1
    // path_exists, metadata, remove_dir, remove_file, rename 函数的输入参数都是 &str，
    // 你可以通过 old_path.path() new_path.path() 获取它俩的 &str 形式。
    //
    // HINT 2
    // RenameFlags 是一个 bitflags，它常用的函数有 .contains(...) 和 ::from_bits(...)
    // 可以找找内核中其他地方是怎么使用 bitflags 的，比如这个文件中的 syscall_fcntl64
    //
    // HINT 3
    // metadata(old_path.path()) 会返回一个 Result<Metadata>
    // 如果它是 Err()，说明文件打开失败，你可以用它检查文件是否存在；
    // 如果它是 Ok()，则可以获取到一个 Metadata 类。
    // 这个类里 .is_dir() 和 .is_file() 可以帮助你判断它是路径还是文件

   
    axlog::warn!("renameat2 not implemented");
    Ok(0)
}
```

我们先把关键的输入参数打印出来：

```
old_path:/abc new_path:/bin/ proc_path:/proc
```

我们的目标是把`/abc`这个文件移动到`/bin/`目录下。

文件系统提供了`use axfs::api::{metadata, path_exists, remove_dir, remove_file, rename};`这么些函数。其中`rename`这个函数是关键函数，不过需要注意的是，`rename`是对一个文件进行重命名，参数分别是这个文件`old_name`，`new_name`，我们需要先处理一下已知的`old_path`和`new_path`，再调用`rename`即可。

```
old_name:/abc new_name:/bin/abc
```

```rust
pub fn syscall_renameat2(
    old_dirfd: usize,
    old_path: *const u8,
    new_dirfd: usize,
    new_path: *const u8,
    flags: usize,
) -> SyscallResult {
    let old_path = deal_with_path(old_dirfd, Some(old_path), false).unwrap();
    let new_path = deal_with_path(new_dirfd, Some(new_path), false).unwrap();
    let proc_path = FilePath::new("/proc").unwrap();
    if old_path.start_with(&proc_path) || new_path.start_with(&proc_path) {
        return Err(SyscallError::EPERM);
    }
    error!(
        "old path:{} and new path:{} and proc path:{}",
        old_path.path(),
        new_path.path(),
        proc_path.path()
    );

    let mut new_path_str = new_path.path().to_string();

    new_path_str.pop();
    let new_name = new_path_str + old_path.path();
    match rename(old_path.path(), &new_name) {
        Ok(()) => return Ok(0),
        _ => return Err(SyscallError::EINVAL),
    }
}
```

修改后再次运行，`ls`正确输出！😊

![image-20231128194608516](https://jgox-image-1316409677.cos.ap-guangzhou.myqcloud.com/blog/image-20231128194608516.png)

![image-20231128194655585](https://jgox-image-1316409677.cos.ap-guangzhou.myqcloud.com/blog/image-20231128194655585.png)

### 实验3.2

修改 `ulib/axstarry/syscall_entry/src/test.rs` 的`SDCARD_TESTCASES` 常量为：

```rust
#[allow(dead_code)]
pub const SDCARD_TESTCASES: &[&str] = &[
    "busybox touch def",
    "busybox mv def bin",
    "busybox ls bin/def",
];
```

其中不能正常工作的命令是 `busybox mv def bin`。它和实验3.1的区别在于，`bin` 目录后没有斜线 `/`。你的任务是修改代码，使得 `busybox mv def bin` 正常输出 `bin/def`。

先直接运行试试：

![image-20231128195803411](https://jgox-image-1316409677.cos.ap-guangzhou.myqcloud.com/blog/image-20231128195803411.png)

那么问题也很显然，我们之前对`rename`的参数的处理太简单了，没有考虑到所有情况，我们在代码中改变对`path`的处理逻辑即可。代码中还加了一些判断，我们通过文件路径获取文件的`metadata`来判断文件类型去区分 `bin` 到底是个文件还是个目录。还有很多逻辑判断如复制一个目录到文件，这里就不去判断了。。。

```rust
pub fn syscall_renameat2(
    old_dirfd: usize,
    old_path: *const u8,
    new_dirfd: usize,
    new_path: *const u8,
    flags: usize,
) -> SyscallResult {
    let old_path = deal_with_path(old_dirfd, Some(old_path), false).unwrap();
    let new_path = deal_with_path(new_dirfd, Some(new_path), false).unwrap();
    let proc_path = FilePath::new("/proc").unwrap();
    if old_path.start_with(&proc_path) || new_path.start_with(&proc_path) {
        return Err(SyscallError::EPERM);
    }
    error!(
        "old path:{} and new path:{} and proc path:{}",
        old_path.path(),
        new_path.path(),
        proc_path.path()
    );

    let new_metadata = match metadata(new_path.path()) {
        Ok(metadata) => metadata,
        Err(_) => return Err(SyscallError::ENOENT), // File doesn't exist
    };

    let mut new_path_str = new_path.path().to_string();

    if new_metadata.is_dir() {
        if !new_path_str.ends_with('/') {
            new_path_str.push('/');
        }
        new_path_str.pop();
        let new_name = new_path_str + old_path.path();
        match rename(old_path.path(), &new_name) {
            Ok(()) => return Ok(0),
            _ => return Err(SyscallError::EINVAL),
        }
    }
    Ok(0)
}
```

再次运行，成功显示结果。😊

![image-20231128200135506](https://jgox-image-1316409677.cos.ap-guangzhou.myqcloud.com/blog/image-20231128200135506.png)

