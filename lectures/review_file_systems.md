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

It's easy to create a new file because we just need to adjust the directory table fo the parent directory and set the busy bit for the starting block.

Adding to the file is as easy as adjusting the next field of the FAT.

Space efficiency is pretty good too, all that matters is if a file is free. It is good for sequential access but poor for random access because we have to follow the links to get to the middle of the file. 

### Inode Structure