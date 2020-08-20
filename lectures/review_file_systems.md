# Review - File Systems

## Table of Contents

## Introduction

A file system is an intuitive interface that allows computer users to not have to think about where data is physically stored.

There are three key abstractions at work:
1. **File** - in reality data might not be placed next to each other in order.
2. **Filename** - we don't have to remember the physical information about the file, we just have a name.
3. **Directories** - Containers for files that help us organize them.

The directory structure looks like an upside-down tree, the top-most directory is called the `root` or `/`. 

## Access Rights

As soon as we start letting multiple users interact with a filesystem, it becomes clear that we will need some form of control over the files.

Conceptually, we can think of access rights as a giant matrix with users and the files as the columns/rows respectively.

Unix, like OS's achieve a fairly efficient system by assigning an `owner` and `group` for each file.

<img src="file_systems_resources/access_rights.png">

There are separate read, write, and execute bits for the user, group and then another set for everybody else. This gives us 9 bits for representing the possibilities.

Directories are a little bit more complicated. Reading effects what you can see, writing effects your ability to create, delete and rename files. Execution permissions correspond to whether you can pass through a directory and do other more abstruse actions.

In some instances access-control lists are used. 

## Developer Interface

There are two ways that developers interact with the filesystem, the first is a *positional interface* where the developer uses a cursor to read and write to the file. The other way treats the file as if it is *just a block of memory*.

### Positional Interface

Let's look at the positional interface first:

<img src="file_systems_resources/positional.png">

Search through a simple text file and replace the value if it matches a key:

```c
int main(int argc, char **argv){
int fd;
ssize_t len;
char *filename;
int key, srch_key, new_value;
if(argc < 4){
  fprintf(stderr, "usage: sabotage filename key value\n");
  exit(EXIT_FAILURE);
}
filename = argv[1];
srch_key = strtol(argv[2], NULL, 10);
new_value = strtol(argv[3], NULL, 10);
fd = open(filename, O_RDWR);
while( sizeof(int) == read(fd, &key, sizeof(int)) ){
  if (key != srch_key)
    lseek(fd, sizeof(int), SEEK_CUR);
  else{
    write(fd, &new_value, sizeof(int));
    close(fd);
    return EXIT_SUCCESS;
  }
}
fprintf(stderr, "Key not found!");
return EXIT_FAILURE;
}
```

### Memory Interface

It's also possible to interact with a file in a way that let's us treat it just like memory. As before, we open a file using `open`, instead of using `read` and `write` however we use **mmap**, which returns a pointer to a region of memory that we can manipulate in our code. Eventually, whatever changes we make to this memory gets reflected in the file. When we are done we call **munmap**, which syncs the contents of memory with the file and then frees up the memory.

<img src="file_systems_resources/mmap.png">

Use mmap to shuffle a binary file:

```c
int main(int argc, char **argv){
  int fd, card_size;
  ssize_t len;
  void *buf;
  char *filename;

  if(argc < 3){
    fprintf(stderr, "usage: shuffle filename card_size\n");
    exit(EXIT_FAILURE);
  }

  filename = argv[1];
  card_size = strtol(argv[2], NULL, 10);

  fd = open(filename, O_RDWR);

  len = lseek(fd, 0, SEEK_END); 
  lseek(fd, 0, SEEK_SET); //bring cursor back to beginning

  buf = mmap(NULL, len, PROT_READ | PROT_WRITE | MAP_FILE | MAP_SHARED, fd, 0, 0);
  if ( buf == (void*) -1) {
    fprintf(stderr, "mmap failed \n");
    exit(EXIT_FAILURE);
  }

  memshuffle(buf, len, card_size);
  
  munmap(buf, len);

  close(fd);

  return EXIT_SUCCESS;
```

## Allocation Strategies

Just like caches work in units of cache lines, and virtual memory works in term of pages, **file systems work in terms of blocks**, or sometimes clusters.

<img src="file_system_resources/allocation.png">

There are several strategies for **keeping track of which blocks are free** and which are used. One easy way is to keep a **list** of free blocks, another way is to keep a bit vector which indicates if the block is free or not. 

We want our file creation to be fast, we want them flexibly sized, have an efficient use of space, and have fast sequential and random access.

<img src="file_system_resources/allocation2.png">

### File Allocation Table (FAT) format

FAT format was originally used on floppy disks, but became the standard for DOS and windows in the 90s, it is still commonly used on some flash and solid-state drives.

In FAT, a file is represented as a linked-list of blocks. The **file allocation table** is a key-value data structure that tells us where each block points to. 

The two key ideas of FAT are:
1. Each **file** is represented as a **linked-list of blocks**. The links are indexed in the file allocation table, which is indexed by number. Given the starting number of a file, we can use a constant-time look-up to retrieve the next block. A special value, (in the image below, a -1, indicates the EOF).
2. The **directory table** **associates filenames with starting blocks** and captures the heirarchy of the files in the filesystem.

<img src="file_system_resources/FAT.png">

### Strengths and weaknesses of FAT

**File Creation**: It's easy to create a new file because we just need to adjust the directory table fo the parent directory and set the busy bit for the starting block.

**File Appending:** Adding to the file is as easy as adjusting the next field of the FAT.

**Space Efficiency**: Space efficiency is pretty good too, all that matters is if a file is free. 

**Access:** It is good for sequential access but poor for random access because we have to follow the links to get to the middle of the file. 

### Inode Structure

Another popular file system format is the **extended(ext) format**. **Each file** on the disk has a data structure called an **inode** associated with it. 

Each inode is of fixed length. It contains the metadata for the file, and the inode serves as the glue that links the data blocks together. 

The first twelve pointers are direct pointers and they point to the first 12 blocks of the file. This makes the strategy efficient for small files. 

The thirteenth block is an indirect pointer to a table of addresses for the next blocks in the file. The indirection increases the amount of blocks an inode can reference, the downside is longer seek time to find the right block.

The fourteenth block is a double indirect pointer. This is used for much larger files. 

The fifteenth block is a triple indirect pointer. 

Just like in FAT, directories map file names to their inodes.

<img src="file_system_resources/inode.png">

### Strengths and Weaknesses of Inode system

**File Creation**: Grab an inode and update the parent directory.

**File Appending**: Grab a new block and update the inode.

**Space Efficiency**: Still efficient with space, but there is a little waste with potentially unused fields.

**Access:** Inodes add an extra level of indirection as we go through the directory tree. A directory points to the inode of the file instead of the first data block of the file. What's more important is that the inode group the data blocks in a tree-like structure, which makes random access much faster!

## File System Optimizations

### Buffer Cache

Most operating systems use free portions of main memory as a cache for the much slower mass storage device. Main memory is as much as 100,000x faster than disk for random access.

The part of main memory that caches disk data is called the **unified buffer cache**. When data is read from disk, it is stored in this cache so that subsequent reads can access it in memory and not have to bother the disk again.

<img src="file_system_resources/buffer_cache.png">

Because disk access is often sequential, it is common for the disk to read ahead and save it to the unified buffer cache so that it is there when the application needs it.

Writing to disk is often done using the **write-back strategy**. Data is written only to the cache and the page is marked as dirty. The slower operation to the disk is postponed until an opportune time. 

The advantage of this strategy is that the program gets to resume faster and get on with its work, the downside is that if the system crashes before memory has been written to disk, the data will be lost. 

If you really want to be sure that your changes are reflected, use `fsync` or `msync`.

### Journaling

As we discussed earlier, a system crash or a power failure could cause data loss if a write-back strategy is being used. 

It would be nice if we could periodically copy all the changes in memory to disk. The trouble with this strategy though is that we would have to spend an inordinate amount of time seeking the disk head to the right spots to update the dirty blocks. It is **the seek time of the disk head that makes random access to disks so slow in general**. 

Sequential access can be up to a 1000x faster than random access, so journaling file systems take advantage of this efficiency. They **reserve a contiguous portion of the disk** for the purpose of copying dirty blocks in a contiguous sequence from main memory. In a more opportune time the disk applies the journaled changes to the disparate blocks scattered throughout the disk.

The only problem with this strategy is that it complicates reading. When we want to read, we have to check the journal to see if there is newer information that hasn't been reflected in the block that is being searched for. In total, journaling is actually slower, however it allows us to free up memory when we have too many dirty blocks. It also helps with crashes :). 