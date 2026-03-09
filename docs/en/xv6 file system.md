# xv6 File System Manual and Code Reading Notes

#1 – Read the manual

Divided into six levels,

![figure6-1] (https://th0ar.gitbooks.io/xv6-chinese/content/pic/f6-1.png)

The following layer reads and writes the IDE disk through the block buffer, synchronizes access to the disk, and guarantees that only one kernel process can modify the disk block at the same time. The second layer allows the higher-level interface to package the disk updates by session, and through the session the operation is atomic (either all applied or not applied). Layer 3 provides nameless files, each of which consists of an i node and a series of data blocks. Layer 4 implements the directory as a special i node, which contains a series of directory items, each with a node.The last layer abstracts many of UNIX resources (such as pipes, devices, files, etc.) as interfaces to the file system.



### Block buffer layer

Two tasks: (1) synchronize access to disk, so that for each block, only one copy is in memory at the same time and only one kernel thread uses this copy; (2) caches commonly used blocks to improve performance.

The main interfaces provided by the block buffer are `bread` and `bwrite`; the former takes a piece from the disk and puts it into the buffer, and the latter writes one piece of the buffer to the correct place on the disk. When the kernel processes a buffer block, it needs to be called `brelse` to release it.

Block buffers only allow up to one kernel thread to refer to it to synchronize access to the disk. If a kernel thread references a buffer block but has not released it, the other processes that call `bread` are blocked. The higher layers of the file system are the synchronous mechanism that relies on the block buffer layer to ensure its correctness.

A block buffer has a fixed number of buffers, which means that if the file system requests a block that is not in the buffer, it must swap out an already used buffer. The replacement policy uses LRU.



### Log Layer

Log design

xv6 solves the problem of crashing during file operation through a simple logging system. A system call does not directly lead to a write operation of the file system on the disk; instead, he wraps a description of the disk write operation into a log written into a disk. When the system call writes all the write operations to the log, it writes a special commit to the disk, representing a complete operation. From then on, the system call writes the data from the log to the data structure of the disk file system.

Common ways to use the log as follows

```c
begin_trans();
...
bp = bread (...)
bp->data[...] = ...
log_write(bp);
...
commit_trans();
```

`begin_trans` will wait until it takes possession of the log's right to use it.

`log_write` is like a proxy for `bwrite`; it records new contents of the block into the log and the block's sector number in memory. `log_write` still leaves the modified block in the buffer in memory, so that the reading of the block in the successive session returns the modified content. `log_write` can know that the same piece is read and written multiple times in a session, and overwrites the same log before.

`commit_trans` Writes the starting block of the log to the disk so that the system crash can be recovered after this point in time, simply rewriting the contents of the disk with the contents of the log. `commit_trans` calls `install_trans`(4221) to read from the log piece by piece and write it to the right place in the file system. Finally `commit_trans` changes the count in the log start block to 0, so that the system crash before the next session causes the recovery of the code to ignore the log.

`recover_from_log` is called in `initlog`, and `initlog` is called during the boot process before the first user process begins. It reads the starting block of the log, and if the starting block says that there is a submitted session in the log, it executes as `commit_trans`, thereby recovering from the error.

There is an example of using a log in `filewrite`:

```c
begin_trans();
ilock(f->ip);
r = writei(f->ip, ...);
iunlock(f->ip);
commit_trans();
```

This code is found in a loop that splits a large write operation into a session, and only part of the block is written in each session, because the size of the log is limitedly fixed. The call to the `writei` will write a lot of blocks in a session: the i-node of the file, one or more bit tiles, and some data blocks. After `begin_trans`, it's a way to avoid deadlock: because each session already has a lock protected, so it's necessary to ensure a certain lock order when holding two locks.



####i node

The term *i node can have two meanings. It can refer to the data structure of the file size on the disk, the data block sector number. It can also refer to an i-node in memory that contains a copy of the i-node on the disk, as well as additional information that the kernel needs.

All i-nodes on the disk are packaged in a continuous region called the i-node block. Each i-node is the same size, so for a given number n, it is easy to find the corresponding i-node on the disk. In fact, this given number is the number of the i-node in the operating system.

The i-node on the disk is defined by the struct `dinode`. The type field is used to distinguish the i-nodes of the file, directory, and special files. If `type` is 0, it means that it is an idle i-node. The `nlink` field is used to record the directory term that points to this i-node, which is used to determine whether an i-node should be released. The `size` field records the number of bytes of a file.

```c
On-disk inode structure
struct dinode {
  short type; // File type, files, directors, or special files (devices)
                        0 indicates dinode is free
  short major; // major device number (T_DEVICE only)
  short minor; // minor device number (T_DEVICE only)
  short nlink; // Number of links to inode in file system
                        Won't free an inode if its link count is greater than 0
                        // Statistically point to the inode directory entry (hard link number)
                        // Used to indicate when the inode on the disk and its corresponding data blocks should be released
  uint size; // size of file (bytes)
                        The number of bytes of content in the file
  uint addrs[NDIRECT+1]; // Data block addresses
                           records the block numbers of the disk blocks hold the file’s content.
  			   // NDIRECT = 12, 12 direct blocks, and the 13th is a first-level indirect block
                           // The default size is (12+256)*BISZE = 268KB
}
```

The kernel is a copy of the struct `dinode` in the disk. The kernel only saves the i-node when there is a C pointer to an i-node. The `ref` field is used to count how many C pointers point to it. If `ref` becomes 0, the kernel will lose the i-node. `iget` and `iput` two functions apply and release the i-node pointer, modifying the reference count.Some kernel code such as `exec` is generated.

```c
In-memory copy of an inode(struct dinode)
kernel stores an inode in memory only if there are Cpointers referring to that inode
struct inode {
  uint dev; // Device number
  uint inum; // Inode number
  int ref; // Reference count
                      // Number of pointers to the inode in memory, pay attention to the nlink that distinguishes between and dnode
                      // ref is greater than 0, the inode will continue to be saved in the icache, and the cache entry will not be replaced by another inode.
                      // when ref is 0, the kernel clears the copy of the inode in the icache
                      // nlink is a hard link, and will be stored on the disk after power outage
                      // and ref is the data structure in the memory, and it will disappear after power loss.
  struct sleeplock lock; // protects everything everything below here
                         // ensures exclusive access to the inode’s fields as well as to the inode’s file or directory content blocks.
  int valid; // inode has been read from disk?

  The following are members of the struct dinode.
  short type; // copy of disk inode
  short major;
  short minor;
  short nlink;
  the Uint size;
  uint addrs[NDIRECT+1];
(a);
```

The pointer holding the i node returned by `iget` will ensure that the i-node stays in the cache and is not deleted (in particular, it will not be used to cache another file). So the pointer returned by `iget` is quite a weak lock, although it does not require the holder to actually lock the i-node. Many parts of the file system rely on this feature, on the one hand, to hold references to the i node for a long time (such as the open file and the current directory), and on the one hand to avoid competition and deadlock in the program that manipulates multiple i nodes.

In order to ensure that it holds a valid copy of the i-node on a disk, the program must call `ilock`. It locks the i-node (so that other processes cannot use it) and reads the i-node's information from the disk (if it has not been read out). `iunlock` releases the lock on the i node.Live on an i-node.

The i-node cache only caches the i-nodes to which the C pointer points. Its main job is to synchronize the access of multiple processes to the i-node rather than the cache. If an i-node is frequently used, the block buffer may keep it in memory, even if the i-node cache does not cache it.



### File Descriptor Layer

xv6 gives each process a self-opening file table, each open file is represented by the struct `file`, which is an encapsulation of the i-node or pipeline and file offset. Each call to open `creates a new open file (a new `file` struct). If multiple processes open the same file independently of each other, different instances will have a different i/o offset. On the other hand, the same file can appear multiple times in a file table of a single file, and can also appear in multiple file tables.Process sharing causes this to happen. There is a reference count for each open file, and a file can be opened for reading, writing, or both. The readable domain and the `writable` domain record this.

All open files in the system are present in a global file table `ftable`. This file table has a function to assign files (`fileloc`), a function that repeats the reference to the file (`filedup`), and releases the function (`fileclose`) that references the file, and the function (`fileread` and `filewrite`) that read and write the file.

`Filealloc` scans the entire file table to find an unreferenced file (`file->ref == 0`) and returns a new reference; `filedup` increases the reference count; `fileclose` reduces the reference count. When a file's reference count becomes 0, the file close releases the current pipeline or i node (depending on the file type).

The function `filestat`,`fileread`,`filewrite` implements the `stat`, `read`,`write` operation of the file. `filestat` (5302) only acts on the i node, which is implemented by calling `stati`. `fileread` and `filewrite` check that the operation is allowed by the file's open property and then the implementation of the execution of the cedible to the i node or the implementation of the pipeline.The i-node function needs to be called to handle the lock. One of the convenient side effects of the i-node lock is that the read-write offset is automatically updated, so writing to a file at the same time does not cover the respective files, but the order of writing is not guaranteed, so the written result may be interwoven (insert another in the course of one write operation).



#2: Reading the Code

The file system part buf.h fcntl.h stat.h fs.h file.h ide.c bio.c log.c fs.c file.c sysfile.c exec.c

1.buf.h: Defines the data structure of disk blocks in xv6 with a block size of 512 bytes.

```c
// disk block data structure in xv6, block size 512 bytes
struct buf {
  int flags; // DIRTY, VALID
  the uint dev;
  uint sector; // corresponds to sector
  struct buf *prev; // LRU cache list
  struct buf *next; // chain structure for connection
  struct buf *qnext; // disk queue
  uchar data[512];
(a);
#define B_BUSY 0x1 // buffer is locked by some process
#define B_VALID 0x2 // buffer has been read from disk
#define B_DIRTY 0x4 // buffer needs to be written to disk
```

fcntl.h: macro defines the operation permission.

```c
#define O_RDONLY 0x000 // Read only
#define O_WRONLY 0x001 // only write
#define O_RDWR 0x002 // Read and write
#define O_CREATE 0x200 // Created
```

3.stat.h: Declares the data structure of the file or directory property.

```c
#define T_DIR 1 // Directory
#define T_FILE 2 // File
#define T_DEV 3 // Device

struct stat {
  short type; // type of file
  int dev; // File system's disk device
  uint ino; // Inode number
  short nlink; // Number of links to file
  uint size; // size of file in bytes
(a);
```

4.fs.h/fs.c: Declares superblocks, dinodes, file and directory data structures, and related macro definitions.

```c
#define ROOTINO 1 // root i-number
#define BSIZE 512 // block size

File system super block
Superstructblock {
  uint size; // size of file system image (blocks)
  uint nblocks; // Number of data blocks
  uint ninodes; // Number of inodes.
  uint nlog; // Number of log blocks
(a);

#define NDIRECT 12
#define NINDIRECT (BSIZE/sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT)

// inode node on disk embodies form
On-disk inode structure
struct dinode {
  short type; // file type
  short major; // major device number (T_DEV only)
  short minor; // minor device number (T_DEV only)
  short nlink; // Number of links to inode in file system
  uint size; // size of file (bytes)
  uint addrs[NDIRECT+1]; // Data block addresses
(a);

Inodes per block.
#define IPB (BSIZE / sizeof(struct dinode))

Block containing inode i
#define IBLOCK(i)((i) / IPB + 2)

Bitmap bits per block
#define BPB (BSIZE*8)

Block containing bit for block b
#define BBLOCK(b, ninodes) (b/BPB+ (ninodes)/IPB + 3)

Directory is a file containing a sequence of dirent structures.
#define DIRSIZ 14

// The file or directory data structure, the directory itself is stored on the disk in the form of a file, called the directory file.
struct dirent {
  ushort inum; // i-node
  char name[DIRSIZ]; // file or directory name
(a);
```

5.file.h: declares the inode, file data structure.

```c
struct file {
  // Divided into pipe files, equipment files, ordinary files
  enum { FD_NONE, FD_PIPE, FD_INODE } type; 
  int ref; // reference count
  char readable;
  char writable;
  struct pipe *pipe;
  struct inode *ip; // points to inode node
  Uint off;
(a);

// In-memory inside node representation form
in-memory copy of an inode
struct inode {
  uint dev; // Device number
  uint inum; // Inode number
  int ref; // Reference count
  int flags; // I_BUSY, I_VALID

	  The following programs are copies of the dnode.
	  Copy of disk inode
  the short type;         
  short major;
  short minor;
  short nlink;
  the Uint size;
  uint addrs[NDIRECT+1];
(a);
#define I_BUSY 0x1
#define I_VALID 0x2

Table major mapping device number to device functions
struct devsw {
  int (*read) (struct inode*, char*, int);
  int (*write) (struct inode*, char*, int);
(a);

extern struct devsw devsw[];

#define console 1
```

6.ide.c: The specific implementation of disk IO, xv6 maintains an process request for disk operation (idequeue). When the process calls **void iderw(struct buf \*b)** request to read and write the disk, the request is added to the waiting queue idequeue, and the process enters a sleep state. When the disk read-write operation is complete, an interrupt is triggered, and the interrupt handler ideintr() removes the request at the beginning of the queue, and the process corresponding to the request at the beginning of the queue request.

```c
// idequeue points to the buf now read/written to the disk.
// idequeue->qnext points to the next buf to be processed.
You must hold idelock while muctile queue.

static struct spinlock idelock; //protects idequeue
static struct buf *idequee; // request queue for disk read-write operations
...
Wait for the disk to enter the idle state
Wait for IDE disk to become ready.
static int idewait(int checkerr)
{
  ...
  // 
  while((r = inb(0x1f7)) & (IDE_BSY|IDE_DRDY)) != IDE_DRDY);
  ...
}

// Initialize IDE disk IO
void ideinit(void)
{
  ...
}

Start a disk read and write request
Start the request for b.  Caller must hold idelock.
static void idestart(struct buf *b)
{
  ...
}

// Functions that interrupt the handler when the disk request is complete
Interrupt handler.
void ideintr(void)
{
  // After processing a disk IO request, wake up the process waiting for the queue header
  wakeup(b);
  
  // If the queue is not empty, proceed to the next disk IO task
  Start disk on next buf in queue.
  if(idequee != 0)
    idestart (idequeue);
  ...
}

//PAGEBREAK! The disk IO interface called by the upper file system
Sync buf with disk. 
If B_DIRTY is set, write buf to disk, clear B_DIRTY, set B_VALID.
Else if B_VALID is not set, read buf from disk, set B_VALID.
void iderw(struct buf *b)
{
  The Competition Lock
  acquire(&idelock); //DOC:acquire-lock

  // Append b to idequeue.
  b->qnext = 0;
  for(pp=&idequee; *pp; pp=&(*pp)->qnext) //DOC:insert-queue
    ;
  *pp = b;
  
  Start disk if necessary. Start working with a disk IO task
  if(idequeue == b)
    idestart(b);
  
  // wait for request to finish.
  while(b->flags & (B_VALID|B_DIRTY)) != B_VALID){
    sleep(b, &idelock)
  }

  release(&idelock); // release the lock
}
```

7.bio.c: Specific implementation of Buffer Cache. Because the read/write disk is not efficient, according to the principle of time and space locality, the recently visited disk blocks are cached in memory. The main interfaces are struct buf* (uint dev, uint sector), void bwrite (struct buf *b), bread will first look for the presence of blocks from the cache, if there is a direct return, if not, request the disk read operation, read the cache and then return the result.
8.log.c: The module mainly maintains the consistency of the file system. After the introduction of the log module, all disk operations for the upper file system are divided into transactions, each transaction will first write the data and its corresponding disk number to the log area on the disk, and only after the log area is written to the real stored data block. Therefore, if the file system is not written to the log area after the restart, if the log area is written to the real area, the data can be recovered according to the log area.
9.sysfile.c: Mainly defines the system calls associated with the file. The main interface and meaning are as follows:

```c
// Allocate a file descriptor for the given file.
// Take over file reference from caller on success.
static int fdalloc(struct file *f)
{
  Apply for an unused document handle
}

int sys_dup(void)
{
  // call the filedup to count the reference to the file handle +1
  filed up(f);
  return fd;
}

int sys_read(void)
{
  Read the file data
  return fileread(f, p, n);
}

int sys_write(void)
{
  Write data to documents
  return filewrite(f, p, n);
}

int sys_close(void)
{
  // release the file handle resource
  fileclose(f);
  Return 0;
}

int sys_fstat(void)
{
  ... // Modify the statistical information of the document
  return filestat(f, st);
}

Create the path new as a link to the same inode as old.
int sys_link(void)
{
  Create a new name for an existing inode
}

Pagebreak!
int sys_unlink(void)
{
  // Unblock a name in the inode, and if the name is completely removed, the inode is released.
}

static struct inode* create(char *path, short type, 
	    Short major, short minor)
{
  ... // 
}

int sys_mkdir(void)
{
  ... // Create a directory
}

int sys_mknod(void)
{
  Creating a new document
}

int sys_chdir(void)
{
  ... // Change the directory
}

int sys_pipe(void)
{
  ... // Create a pipeline file
}
```

10.exec.c: There is only one exec interface, the essence is the executable file that passes into the elf format, is loaded into memory and allocates memory pages, and argv is a pointer array for carrying parameters.

```c
int exec(char *path, char **argv)
{
  Determine if the document exists.
  if(ip = namei(path)) == 0
    return -1;
  ilock (ip);
  pgdir = 0;

  Check the ELF header to check if the ELF header is legal
  if(readi(ip, (char*)&elf, 0, sizeof(elf)) < sizeof(elf))
    to go bad;
  ...
  
  Load program into memory.
  sz = 0;
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, (char*)&ph, off, sizeof(ph)) != sizeof(ph))
      to go bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      to go bad;
    if(sz = allocuvm(pgdir, sz, ph.vaddr + ph.mesz)) == 0)
      to go bad;
    if(loaduvm(pgdir, (char*)ph.vaddr, ip, ph.off, ph.filesz) < 0)
      to go bad;
  }
  iunlockput(ip);
  ip = 0;

  Allocate two pages at the next page boundary.
  Make the first inaccessible.  Use the second as the user stack.
  sz = PGROUNDUP(sz);
  if(sz = allocuvm(pgdir, sz, sz + 2*PGSIZE)) == 0)
    to go bad;
  clearpteu(pgdir, (char*)(sz - 2*PGSIZE));
  sp = sz;

  Push strings argument, prepare rest of stack in ustack.
  for(argc = 0; argv[argc]; argc++) {
    if(argc >= MAXARG)
      to go bad;
    sp = (sp - (strlen(argv[argc]) + 1)) & ~3;
    if(copyout(pgdir, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      to go bad;
    ustack[3+argc] = sp;
  }
  ...

 bad:
  if(pgdir)
    freevm(pgdir);
  if (ip)
    iunlockput(ip);
  return -1;
}
```
