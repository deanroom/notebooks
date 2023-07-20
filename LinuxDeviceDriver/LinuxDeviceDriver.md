# Linux Device Driver

## Char Device

Throughout the chapter, we present code fragments extracted from a real device driver: scull (Simple Character Utility for Loading Localities). scull is a char driver that acts on a memory area as though it were a device. In this chapter, because of that peculiarity of scull, we use the word device interchangeably with “the memory area used by scull.”
The advantage of scull is that it isn’t hardware dependent. scull just acts on some memory, allocated from the kernel. Anyone can compile and run scull, and scull is portable across the computer architectures on which Linux runs. On the other hand, the device doesn’t do anything “useful” other than demonstrate the interface between the kernel and char drivers and allow the user to run some tests.

### The Design of scull
The first step of driver writing is defining the capabilities (the mechanism) the driver will offer to user programs. Since our “device” is part of the computer’s memory, we’re free to do what we want with it. It can be a sequential or random-access device, one device or many, and so on.
To make scull useful as a template for writing real drivers for real devices, we’ll show you how to implement several device abstractions on top of the computer memory, each with a different personality.
The scull source implements the following devices. Each kind of device implemented by the module is referred to as a type.

- scull0 to scull3
  
  Four devices, each consisting of a memory area that is both global and persis- tent. Global means that if the device is opened multiple times, the data con- tained within the device is shared by all the file descriptors that opened it. Persistent means that if the device is closed and reopened, data isn’t lost. This device can be fun to work with, because it can be accessed and tested using con- ventional commands, such as cp, cat, and shell I/O redirection.
- scullpipe0 to scullpipe3

  Four FIFO (first-in-first-out) devices, which act like pipes. One process reads what another process writes. If multiple processes read the same device, they contend for data. The internals of scullpipe will show how blocking and non- blocking read and write can be implemented without having to resort to inter- rupts. Although real drivers synchronize with their devices using hardware interrupts, the topic of blocking and nonblocking operations is an important one and is separate from interrupt handling (covered in Chapter 10).

- scullsingle scullpriv sculluid scullwuid
  
  These devices are similar to scull0 but with some limitations on when an open is permitted. The first (scullsingle) allows only one process at a time to use the driver, whereas scullpriv is private to each virtual console (or X terminal ses- sion), because processes on each console/terminal get different memory areas. sculluid and scullwuid can be opened multiple times, but only by one user at a time; the former returns an error of “Device Busy” if another user is locking the device, whereas the latter implements blocking open. These variations of scull would appear to be confusing policy and mechanism, but they are worth look- ing at, because some real-life devices require this sort of management.
  Each of the scull devices demonstrates different features of a driver and presents dif- ferent difficulties. 



### Major and Minor Numbers

So, as a driver writer, you have a choice: you can simply pick a number that appears to be unused, or you can allocate major numbers in a dynamic manner. Picking a num- ber may work as long as the only user of your driver is you; once your driver is more widely deployed, a randomly picked major number will lead to conflicts and trouble.
Thus, for new drivers, we strongly suggest that you use dynamic allocation to obtain your major device number, rather than choosing a number randomly from the ones that are currently free. In other words, your drivers should almost certainly be using alloc_chrdev_region rather than register_chrdev_region.

If you are loading and unloading only a single driver, you can just use rmmod and insmod after the first time you create the special files with your script: dynamic numbers are not randomized,† and you can count on the same number being chosen each time if you don’t load any other (dynamic) modules. Avoiding lengthy scripts is useful during development. But this trick, clearly, doesn’t scale to more than one driver at a time.

The best way to assign major numbers, in our opinion, is by defaulting to dynamic allocation while leaving yourself the option of specifying the major number at load time, or even at compile time. The scull implementation works in this way; it uses a global variable, scull_major, to hold the chosen number (there is also a scull_minor for the minor number). The variable is initialized to SCULL_MAJOR, defined in scull.h. The default value of SCULL_MAJOR in the distributed source is 0, which means “use dynamic assignment.” The user can accept the default or choose a particular major number, either by modifying the macro before compiling or by specifying a value for scull_major on the insmod command line. Finally, by using the scull_load script, the user can pass arguments to insmod on scull_load’s command line.

### Some Important Data Structure

Most of the fundamental driver operations involve three important kernel data structures, called file_operations, file, and inode.

> The file_operations structure is defined in <linux/fs.h> and is used to tell the kernel which functions to call when a process does something to the device.

#### File Operations

We can consider the file to be an “object” and the functions operating on it to be its “methods,” using object-oriented programming terminology to denote actions declared by an object to act on itself.

> There are object-oriented programming in Linux kernel.

unsigned int (_poll) (struct file _, struct poll_table_struct \*);

The poll method is the back end of three system calls: poll, epoll, and select, all of which are used to query whether a read or write to one or more file descriptors would block. The poll method should return a bit mask indicating whether non- blocking reads or writes are possible, and, possibly, provide the kernel with information that can be used to put the calling process to sleep until I/O becomes possible. If a driver leaves its poll method NULL, the device is assumed to be both readable and writable without blocking.

int (_mmap) (struct file _, struct vm_area_struct \*);

mmap is used to request a mapping of device memory to a process’s address space. If this method is NULL, the mmap system call returns -ENODEV.

```c
struct file_operations scull_fops = {
    .owner = THIS_MODULE,
    .llseek = scull_llseek,
    .read = scull_read,
    .write = scull_write,
    .ioctl = scull_ioctl,
    .open = scull_open,
    .release =  scull_release,
    };
```

This declaration uses the standard C tagged structure initialization syntax. This syn- tax is preferred because it makes drivers more portable across changes in the defini- tions of the structures and, arguably, makes the code more compact and readable. Tagged initialization allows the reordering of structure members; in some cases, sub- stantial performance improvements have been realized by placing pointers to fre- quently accessed members in the same hardware cache line.

#### The file Structure

Note that a file has nothing to do with the FILE pointers of user-space programs. A FILE is defined in the C library and never appears in kernel code. A struct file, on the other hand, is a kernel structure that never appears in user programs.

The file structure represents an open file. (It is not specific to device drivers; every open file in the system has an associated struct file in kernel space.) It is created by the kernel on open and is passed to any function that operates on the file, until the last close. After all instances of the file are closed, the kernel releases the data structure.

In the kernel sources, a pointer to struct file is usually called either file or filp (“file pointer”). We’ll consistently call the pointer filp to prevent ambiguities with the structure itself. Thus, file refers to the structure and filp to a pointer to the structure.

```c
struct file_operations *f_op;
```

The operations associated with the file. The kernel assigns the pointer as part of its implementation of open and then reads it when it needs to dispatch any oper- ations. The value in filp->f_op is never saved by the kernel for later reference; this means that you can change the file operations associated with your file, and the new methods will be effective after you return to the caller. For example, the code for open associated with major number 1 (/dev/null, /dev/zero, and so on) substitutes the operations in filp->f_op depending on the minor number being opened. This practice allows the implementation of several behaviors under the same major number without introducing overhead at each system call. The abil- ity to replace the file operations is the kernel equivalent of “method overriding” in object-oriented programming.

#### The inode Structure

The inode structure is used by the kernel internally to represent files. Therefore, it is different from the file structure that represents an open file descriptor. There can be numerous file structures representing multiple open descriptors on a single file, but they all point to a single inode structure.

### Char Device Registration

As we mentioned, the kernel uses structures of type struct cdev to represent char devices internally. Before the kernel invokes your device’s operations, you must allo- cate and register one or more of these structures.

```c

struct cdev *my_cdev = cdev_alloc(); 
my_cdev->ops = &my_fops;
void cdev_init(struct cdev *cdev, struct file_operations *fops);
int cdev_add(struct cdev *dev, dev_t num, unsigned int count);
void cdev_del(struct cdev *dev);

```

#### Device Registration in scull

```c
struct scull_dev {
    struct scull_qset *data;  /* Pointer to first quantum set */
    int quantum;
    int qset;
    unsigned long size;
    unsigned int access_key;  /* used by sculluid and scullpriv */
    struct semaphore sem;     /* mutual exclusion semaphore     */
    struct cdev cdev;    /* Char device structure      */
};

static void scull_setup_cdev(struct scull_dev *dev, int index)
{
    int err, devno = MKDEV(scull_major, scull_minor + index);
    cdev_init(&dev->cdev, &scull_fops);
    dev->cdev.owner = THIS_MODULE;
    dev->cdev.ops = &scull_fops;
    err = cdev_add (&dev->cdev, devno, 1);
    /* Fail gracefully if need be */
    if (err)
    printk(KERN_NOTICE "Error %d adding scull%d", err, index);
}

```

### open and release

#### The open Method

The open method is provided for a driver to do any initialization in preparation for later operations. In most drivers, open should perform the following tasks:
• Check for device-specific errors (such as device-not-ready or similar hardware problems)
• Initialize the device if it is being opened for the first time
• Update the f_op pointer, if necessary
• Allocate and fill any data structure to be put in filp->private_data

```c
   container_of(pointer, container_type, container_field);
```

This macro takes a pointer to a field named container_field, within a structure of type container_type, and returns a pointer to the containing structure. In scull_open, this macro is used to find the appropriate device structure:

```c
     struct scull_dev *dev; /* device information */
     dev = container_of(inode->i_cdev, struct scull_dev, cdev);
     filp->private_data = dev; /* for other methods */

     int scull_open(struct inode *inode, struct file *filp)
     {
        struct scull_dev *dev; /* device information */
        dev = container_of(inode->i_cdev, struct scull_dev, cdev);
        filp->private_data = dev; /* for other methods */
        /* now trim to 0 the length of the device if open was write-only */
        if ( (filp->f_flags & O_ACCMODE) == O_WRONLY) {
             scull_trim(dev); /* ignore errors */
         }
         return 0;          /* success */
     }

```

### scull’s Memory Usage

Philosophically, it’s always a bad idea to put arbitrary limits on data items being managed. Practically, scull can be used to temporarily eat up your system’s memory in order to run tests under low-memory conditions.

```c
     struct scull_qset {
         void **data;
         struct scull_qset *next;
     };

    int scull_trim(struct scull_dev *dev)
    {
        struct scull_qset *next, *dptr;
        int qset = dev->qset;   /* "dev" is not-null */
        int i;
        for (dptr = dev->data; dptr; dptr = next) { /* all the list items */
            if (dptr->data) {
                for (i = 0; i < qset; i++)
                    kfree(dptr->data[i]);
                kfree(dptr->data);
                dptr->data = NULL;
            }
            next = dptr->next;
            kfree(dptr);
        }
        dev->size = 0;
        dev->quantum = scull_quantum;
        dev->qset = scull_qset;
        dev->data = NULL;
        return 0;
    }
```

### read and write

The read and write methods both perform a similar task, that is, copying data from and to application code. Therefore, their prototypes are pretty similar, and it’s worth introducing them at the same time:

```c
ssize_t read(struct file *filp, char __user *buff, size_t count, loff_t *offp);
ssize_t write(struct file *filp, const char __user *buff, size_t count, loff_t *offp);
```

For both methods, filp is the file pointer and count is the size of the requested data transfer. The buff argument points to the user buffer holding the data to be written or the empty buffer where the newly read data should be placed. Finally, offp is a pointer to a “long offset type” object that indicates the file position the user is accessing. The return value is a “signed size type”;

```c
    unsigned long copy_to_user(void __user *to, const void *from,
                                unsigned long count);
    unsigned long copy_from_user(void *to,
const void __user *from, unsigned long count);
```

Although these functions behave like normal memcpy functions, a little extra care must be used when accessing user space from kernel code. The user pages being addressed might not be currently present in memory, and the virtual memory sub- system can put the process to sleep while the page is being transferred into place. This happens, for example, when the page must be retrieved from swap space. The net result for the driver writer is that any function that accesses user space must be reentrant, must be able to execute concurrently with other driver functions, and, in particular, must be in a position where it can legally sleep. We return to this subject in Chapter 5.

#### The read Method

The return value for read is interpreted by the calling application program:

- If the value equals the count argument passed to the read system call, the
  requested number of bytes has been transferred. This is the optimal case.
- If the value is positive, but smaller than count, only part of the data has been transferred. This may happen for a number of reasons, depending on the device. Most often, the application program retries the read. For instance, if you read using the fread function, the library function reissues the system call until com- pletion of the requested data transfer.
- If the value is 0, end-of-file was reached (and no data was read).
- A negative value means there was an error. The value specifies what the error was, according to <linux/errno.h>. Typical values returned on error include -EINTR (interrupted system call) or -EFAULT (bad address).

#### The write Method

write, like read, can transfer less data than was requested, according to the following rules for the return value:

- If the value equals count, the requested number of bytes has been transferred.
- If the value is positive, but smaller than count, only part of the data has been
  transferred. The program will most likely retry writing the rest of the data.
- If the value is 0, nothing was written. This result is not an error, and there is no reason to return an error code. Once again, the standard library retries the call to write. We’ll examine the exact meaning of this case in Chapter 6, where block- ing write is introduced.
- A negative value means an error occurred; as for read, valid error values are those defined in <linux/errno.h>.

#### readv and writev

If your driver does not supply methods to handle the vector operations, readv and writev are implemented with multiple calls to your read and write methods. In many situations, however, greater efficiency is acheived by implementing readv and writev directly.

### Playing with the New Devices

## Quick Reference

This chapter introduced the following symbols and header files. The list of the fields in struct file_operations and struct file is not repeated here.
```c
#include <linux/types.h>
dev_t
```

dev_t is the type used to represent device numbers within the kernel.
```c
int MAJOR(dev_t dev);
int MINOR(dev_t dev);
```
Macros that extract the major and minor numbers from a device number.
```c
dev_t MKDEV(unsigned int major, unsigned int minor);
```
Macro that builds a dev_t data item from the major and minor numbers.
```c
#include <linux/fs.h>
```
The “filesystem” header is the header required for writing device drivers. Many important functions and data structures are declared in here.

```c
int register_chrdev_region(dev_t first, unsigned int count, char *name);
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, unsigned int count, char *name);
void unregister_chrdev_region(dev_t first, unsigned int count);
```
Functions that allow a driver to allocate and free ranges of device numbers. register_chrdev_region should be used when the desired major number is known in advance; for dynamic allocation, use alloc_chrdev_region instead.
`int register_chrdev(unsigned int major, const char *name, struct file_operations *fops);`
The old (pre-2.6) char device registration routine. It is emulated in the 2.6 ker- nel but should not be used for new code. If the major number is not 0, it is used unchanged; otherwise a dynamic number is assigned for this device.
`int unregister_chrdev(unsigned int major, const char *name);`
Function that undoes a registration made with register_chrdev. Both major and the name string must contain the same values that were used to register the driver.

```c
struct file_operations;
struct file;
struct inode;
```

Three important data structures used by most device drivers. The file_operations structure holds a char driver’s methods; struct file represents an open file, and struct inode represents a file on disk.

```c
#include <linux/cdev.h>
struct cdev *cdev_alloc(void);
void cdev_init(struct cdev *dev, struct file_operations *fops);
int cdev_add(struct cdev *dev, dev_t num, unsigned int count);
void cdev_del(struct cdev *dev);
```

Functions for the management of cdev structures, which represent char devices within the kernel.

```c
#include <linux/kernel.h>
container_of(pointer, type, field);
```

A convenience macro that may be used to obtain a pointer to a structure from a pointer to some other structure contained within it.

```c
#include <asm/uaccess.h>
```

This include file declares functions used by kernel code to move data to and from user space.

```c
unsigned long copy_from_user (void *to, const void *from, unsigned long
  count);
unsigned long copy_to_user (void *to, const void *from, unsigned long count);
```

Copy data between user space and kernel space.
