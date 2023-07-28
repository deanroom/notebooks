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

## Debugging Techniques

### Debugging Support in the Kernel

One of the stron- gest reasons for running your own kernel is that the kernel developers have built sev- eral debugging features into the kernel itself. These features can create extra output and slow performance, so they tend not to be enabled in production kernels from distributors.

```c
CONFIG_DEBUG_KERNEL
CONFIG_DEBUG_SLAB
CONFIG_DEBUG_PAGEALLOC
CONFIG_DEBUG_SPINLOCK
CONFIG_DEBUG_SPINLOCK_SLEEP
CONFIG_INIT_DEBUG
CONFIG_DEBUG_INFO
CONFIG_MAGIC_SYSRQ
CONFIG_DEBUG_STACKOVERFLOW
CONFIG_DEBUG_STACK_USAGE
CONFIG_KALLSYMS
CONFIG_IKCONFIG
CONFIG_IKCONFIG_PROC
CONFIG_ACPI_DEBUG
CONFIG_DEBUG_DRIVER
CONFIG_SCSI_CONSTANTS
CONFIG_INPUT_EVBUG
CONFIG_PROFILING
```

### Debugging by Printing

#### printk

One of the differences is that printk lets you classify messages according to their severity by associating different loglevels, or priorities, with the messages.

KERN_EMERG

Used for emergency messages, usually those that precede a crash.

KERN_ALERT

A situation requiring immediate action.

KERN_CRIT

Critical conditions, often related to serious hardware or software failures.

KERN_ERR

Used to report error conditions; device drivers often use KERN_ERR to report hard- ware difficulties.

KERN_WARNING

Warnings about problematic situations that do not, in themselves, create seri- ous problems with the system.

KERN_NOTICE

Situations that are normal, but still worthy of note. A number of security-related conditions are reported at this level.

KERN_INFO

Informational messages. Many drivers print information about the hardware they find at startup time at this level.

KERN_DEBUG

Used for debugging messages.

If both klogd and sys- logd are running on the system, kernel messages are appended to /var/log/messages

When using klogd, you should remember that it doesn’t save consecutive identical lines; it only saves the first such line and, at a later time, the number of repetitions it received.

#### Redirecting Console Messages

> On Unix-like systems such as Linux and macOS, virtual consoles are accessed by pressing a key combination such as Ctrl+Alt+F1 through Ctrl+Alt+F6. Each virtual console is associated with a different "virtual terminal" device file, such as /dev/tty1 through /dev/tty6. Users can switch between virtual consoles by pressing the appropriate key combination.

Linux allows for some flexibility in console logging policies by letting you send messages to a specific virtual console (if your console lives on the text screen).

#### How Messages Get Logged

The printk function writes messages into a circular buffer that is \_\_LOG_BUF_LEN bytes long: a value from 4 KB to 1 MB chosen while configuring the kernel. The function then wakes any process that is waiting for messages, that is, any process that is sleep- ing in the syslog system call or that is reading /proc/kmsg. These two interfaces to the logging engine are almost equivalent, but note that reading from /proc/kmsg con- sumes the data from the log buffer, whereas the syslog system call can optionally return log data while leaving it for other processes as well. In general, reading the /proc file is easier and is the default behavior for klogd. The dmesg command can be used to look at the content of the buffer without flushing it; actually, the command returns to stdout the whole content of the buffer, whether or not it has already been read.

If the circular buffer fills up, printk wraps around and starts adding new data to the beginning of the buffer, overwriting the oldest data. Therefore, the logging process loses the oldest data. This problem is negligible compared with the advantages of using such a circular buffer.

#### Turning the Messages On and Off

Here we show one way to code printk calls so you can turn them on and off individu- ally or globally; the technique depends on defining a macro that resolves to a printk (or printf) call when you want it to:

- Each print statement can be enabled or disabled by removing or adding a single letter to the macro’s name.
- All the messages can be disabled at once by changing the value of the CFLAGS variable before compiling.
- The same print statement can be used in kernel code and user-level code,so that the driver and test programs can be managed in the same way with regard to extra messages.

The following code fragment implements these features and comes directly from the header scull.h:

```makefile
#undef PDEBUG /* undef it, just in case */ #ifdef SCULL_DEBUG
# ifdef __KERNEL__
    /* This one if debugging is on, and kernel space */
#    define PDEBUG(fmt, args...) printk( KERN_DEBUG "scull: " fmt, ## args)
#  else
    /* This one for user space */
#    define PDEBUG(fmt, args...) fprintf(stderr, fmt, ## args)
#  endif
#else
#  define PDEBUG(fmt, args...) /* not debugging: nothing */
#endif
#undef PDEBUGG
#define PDEBUGG(fmt, args...) /* nothing: it's a placeholder */
```

The symbol PDEBUG is defined or undefined, depending on whether SCULL_DEBUG is defined, and displays information in whatever manner is appropriate to the environ- ment where the code is running: it uses the kernel call printk when it’s in the kernel and the libc call fprintf to the standard error when run in user space. The PDEBUGG symbol, on the other hand, does nothing; it can be used to easily “comment” print statements without removing them entirely.
To simplify the process further, add the following lines to your makefile:

```makefile
     # Comment/uncomment the following line to disable/enable debugging
     DEBUG = y
     # Add your debugging flag (or not) to CFLAGS
     ifeq ($(DEBUG),y)
       DEBFLAGS = -O -g -DSCULL_DEBUG # "-O" is needed to expand inlines
     else
       DEBFLAGS = -O2
     endif
     CFLAGS += $(DEBFLAGS)
```

> The art of good pro- gramming is in choosing the best trade-off between flexibility and efficiency, and we can’t tell what is the best for you.

#### Rate Limiting

If you are not careful, you can find yourself generating thousands of messages with printk, overwhelming the console and, possibly, overflowing the system log file. When using a slow console device (e.g., a serial port), an excessive message rate can also slow down the system or just make it unresponsive. It can be very hard to get a handle on what is wrong with a system when the console is spewing out data non- stop. Therefore, you should be very careful about what you print, especially in pro-duction versions of drivers and especially once initialization is complete. In general, production code should never print anything during normal operation; printed output should be an indication of an exceptional situation requiring attention.

In many cases, the best behavior is to set a flag saying, “I have already complained about this,” and not print any further messages once the flag gets set. In others, though, there are reasons to emit an occasional “the device is still broken” notice. The kernel has provided a function that can be helpful in such cases:

```c
int printk_ratelimit(void);

```

This function should be called before you consider printing a message that could be repeated often. If the function returns a nonzero value, go ahead and print your mes- sage, otherwise skip it. Thus, typical calls look like this:

```c
if (printk_ratelimit())
printk(KERN_NOTICE "The printer is still on fire\n");
```

#### Printing Device Numbers

Occasionally, when printing a message from a driver, you will want to print the device number associated with the hardware of interest. It is not particularly hard to print the major and minor numbers, but, in the interest of consistency, the kernel provides a couple of utility macros (defined in <linux/kdev_t.h>) for this purpose:

```c
int print_dev_t(char *buffer, dev_t dev);
char *format_dev_t(char *buffer, dev_t dev);
```

The buffer should be large enough to hold a device number; given that 64-bit device numbers are a distinct possibility in future kernel releases, the buffer should proba- bly be at least 20 bytes long.

### Debugging by Querying

A massive use of printk can slow down the system noticeably, even if you lower console_loglevel to avoid loading the console device, because syslogd keeps syncing its output files; thus, every line that is printed causes a disk operation. This is the right implementation from syslogd’s perspective. It tries to write everything to disk in case the system crashes right after printing the message; however, you don’t want to slow down your system just for the sake of debugging messages. This problem can be solved by prefixing the name of your log file as it appears in /etc/syslogd.conf with a

#### Using the /proc Filesystem

The /proc filesystem is a special, software-created filesystem that is used by the ker- nel to export information to the world. Each file under /proc is tied to a kernel func- tion that generates the file’s “contents” on the fly when the file is read. We have already seen some of these files in action; /proc/modules, for example, always returns a list of the currently loaded modules.

> The /proc filesystem is often referred to as a "virtual filesystem" because it doesn't represent actual files on disk. Instead, the files in /proc represent system and process information served by the kernel in real-time.

> The Virtual Filesystem (sometimes called the Virtual File Switch or more commonly sim- ply the VFS) is the subsystem of the kernel that implements the file and filesystem-related interfaces provided to user-space programs

> Block devices are hardware devices distinguished by the random (that is, not necessarily sequential) access of fixed-size chunks of data.The fixed-size chunks of data are called blocks.The most common block device is a hard disk, but many other block devices exist, such as floppy drives, Blu-ray readers, and flash memory. Notice how these are all devices on which you mount a filesystem—**filesystems are the lingua franca of block devices**.
> The other basic type of device is a character device. Character devices, or char devices, are accessed as a stream of sequential data, one byte after another. Example character devices are serial ports and keyboards. **If the hardware device is accessed as a stream of data, it is implemented as a character device. On the other hand, if the device is accessed randomly (nonsequentially), it is a block device.**

we should mention that adding files under /proc is dis- couraged. The /proc filesystem is seen by the kernel developers as a bit of an uncon- trolled mess that has gone far beyond its original purpose (which was to provide information about the processes running in the system). The recommended way of making information available in new code is via sysfs.

#### Implementing files in /proc

> The kernel treats physical pages as the basic unit of memory management.Although the processor’s smallest addressable unit is a byte or a word, the memory management unit (MMU, the hardware that manages memory and performs virtual to physical address translations) typically deals in pages.Therefore, the MMU maintains the system’s page tables with page-sized granularity (hence their name). In terms of virtual memory, pages are the smallest unit that matters.
> As you can see in Chapter 19,“Portability,”each architecture defines its own page size. Many architectures even support multiple page sizes. Most **32-bit architectures have 4KB pages**, whereas **most 64-bit architectures have 8KB pages**.This implies that on a machine with **4KB pages and 1GB of memory,** physical memory is divided into 262,144 distinct pages.

When a process reads from your /proc file, the kernel allocates a page of memory (i.e., PAGE_SIZE bytes) where the driver can write data to be returned to user space. That buffer is passed to your function, which is a method called read_proc:

```c
int (*read_proc)(char *page, char **start, off_t offset, int count,
                int *eof, void *data);
```

The page pointer is the buffer where you’ll write your data; start is used by the func- tion to say where the interesting data has been written in page (more on this later); offset and count have the same meaning as for the read method. The eof argument points to an integer that must be set by the driver to signal that it has no more data to return, while data is a driver-specific data pointer you can use for internal bookkeeping.

#### Creating your /proc file

Once you have a read_proc function defined, you need to connect it to an entry in the /proc hierarchy. This is done with a call to create_proc_read_entry:

```c
struct proc_dir_entry *create_proc_read_entry(const char *name,
                             mode_t mode, struct proc_dir_entry *base,
                             read_proc_t *read_proc, void *data)
```

Here, name is the name of the file to create, mode is the protection mask for the file (it can be passed as 0 for a system-wide default), base indicates the directory in which the file should be created (if base is NULL, the file is created in the /proc root), read_proc is the read_proc function that implements the file, and data is ignored by the kernel (but passed to read_proc). Here is the call used by scull to make its /proc function available as /proc/scullmem:

```c
create_proc_read_entry("scullmem", 0 /* default mode */,
                      NULL /* parent dir */, scull_read_procmem,
                      NULL /* client data */);
```

remove_proc_entry is the function that undoes what create_proc_read_entry already did:

```c
remove_proc_entry("scullmem", NULL /_ parent dir _/);
```

#### The seq_file interface

As we noted above, the implementation of large files under /proc is a little awkward. Over time, /proc methods have become notorious for buggy implementations when the amount of output grows large. As a way of cleaning up the /proc code and mak- ing life easier for kernel programmers, the seq_file interface was added. This inter- face provides a simple set of functions for the implementation of large kernel virtual files.

#### The ioctl Method

ioctl, which we show you how to use in Chapter 6, is a system call that acts on a file descriptor; it receives a number that identifies a command to be performed and (optionally) another argument, usually a pointer. As an alternative to using the /proc filesystem, you can implement a few ioctl commands tailored for debugging. These commands can copy relevant data structures from the driver to user space where you can examine them.

### Debugging by Watching

The strace command is a powerful tool that shows all the system calls issued by a user-space program. Not only does it show the calls, but it can also show the argu- ments to the calls and their return values in symbolic form.

The trace information is often used to support bug reports sent to application devel- opers, but it’s also invaluable to kernel programmers. We’ve seen how driver code executes by reacting to system calls; strace allows us to check the consistency of input and output data of each call.

### Debugging System Faults

Even if you’ve used all the monitoring and debugging techniques, sometimes bugs remain in the driver, and the system faults when the driver is executed. When this happens, **it’s important to be able to collect as much information as possible to solve the problem.**

Note that “fault” doesn’t mean “panic.” The Linux code is robust enough to respond gracefully to most errors: a fault usually results in the destruction of the current pro- cess while the system goes on working.The system can panic, and it may if a fault happens outside of a process’s context or if some vital part of the system is compro- mised. But when the problem is due to a driver error, it usually results only in the sudden death of the process unlucky enough to be using the driver. The only unre- coverable damage when a process is destroyed is that some memory allocated to the process’s context is lost; for instance, dynamic lists allocated by the driver through kmalloc might be lost. However, since the kernel calls the close operation for any open device when a process dies, your driver can release what was allocated by the open method.

#### Oops Messages

Most bugs show themselves in NULL pointer dereferences or by the use of other incor- rect pointer values. The usual outcome of such bugs is an oops message.

Almost any address used by the processor is a virtual address and is mapped to phys- ical addresses through a complex structure of page tables (the exceptions are physi- cal addresses used with the memory management subsystem itself). When an invalid pointer is dereferenced, the paging mechanism fails to map the pointer to a physical address, and the processor signals a page fault to the operating system. If the address is not valid, the kernel is not able to “page in” the missing address; it (usually) gener- ates an oops if this happens while the processor is in supervisor mode.

An oops displays the processor status at the time of the fault, including the contents of the CPU registers and other seemingly incomprehensible information. The mes- sage is generated by printk statements in the fault handler (arch/\*/kernel/traps.c) and is dispatched as described earlier in the section “printk".

> "Slab poisoning"是一种调试技术，主要用于发现和调试内存错误。这种技术在操作系统级别应用，特别是在 Linux 内核中广泛使用。
>
> 在内存管理的上下文中，"slab"是一种数据结构，用于存储预先分配的对象的缓存。这种数据结构有助于提高内存分配的效率，尤其是针对小对象的频繁分配和释放。
>
> "Poisoning"在这里是指填充内存块，通常是用一个明显不常见的字节值（比如 0xA5，0x5A 或者其他），以便在未初始化的内存被错误地使用时能够被轻易地识别出来。
>
> 例如，如果一个程序员忘记初始化一个从 slab 分配的内存块，然后试图读取它，这个特殊的“poison”值将会出现，从而揭示了这个错误。这种技术也可以帮助检测过早释放的内存的再次使用（称为“悬挂指针”）以及内存越界错误。
>
> 需要注意的是，"slab poisoning"通常仅在调试期间使用，因为它可能会导致正常操作的性能下降，并且在生产环境中通常不需要这种级别的错误检查。

#### System Hangs

Although most bugs in kernel code end up as oops messages, sometimes they can completely hang the system. If the system hangs, no message is printed. For example, if the code enters an endless loop, the kernel stops scheduling,\* and the system doesn’t respond to any action, including the magic Ctrl-Alt-Del combination. You have two choices for dealing with system hangs—either prevent them beforehand or be able to debug them after the fact.

You can prevent an endless loop by inserting schedule invocations at strategic points. The schedule call (as you might guess) invokes the scheduler and, therefore, allows other processes to steal CPU time from the current process. If a process is looping in kernel space due to a bug in your driver, the schedule calls enable you to kill the pro- cess after tracing what is happening.

An indispensable tool for many lockups is the “magic SysRq key,” which is available on most architectures. Magic SysRq is invoked with the combination of the Alt and SysRq keys on the PC keyboard, or with other special keys on other platforms (see Documentation/sysrq.txt for details), and is available on the serial console as well. A third key, pressed along with these two, performs one of a number of useful actions:

- r Turns off keyboard raw mode; useful in situations where a crashed application (such as the X server) may have left your keyboard in a strange state.
- k Invokes the “secure attention key” (SAK) function. SAK kills all processes run- ning on the current console, leaving you with a clean terminal.
- s Performs an emergency synchronization of all disks.
- u Umount. Attempts to remount all disks in a read-only mode. This operation, usually invoked immediately after s, can save a lot of filesystem checking time in cases where the system is in serious trouble.
- b Boot. Immediately reboots the system. Be sure to synchronize and remount the disks first.
- p Prints processor registers information. t Prints the current task list.
- m Prints memory information.

### Debuggers and Related Tools

The last resort in debugging modules is using a debugger to step through the code, watching the value of variables and machine registers. **This approach is time-consuming and should be avoided whenever possible**. Nonetheless, the fine-grained per-spective on the code that is achieved through a debugger is sometimes invaluable.
Using an interactive debugger on the kernel is a challenge. The kernel runs in its own address space on behalf of all the processes on the system. As a result, a number of common capabilities provided by user-space debuggers, such as breakpoints and sin- gle-stepping, are harder to come by in the kernel. In this section we look at several ways of debugging the kernel; each of them has advantages and disadvantages.

The debugger must be invoked as though the kernel were an application. In addition to specifying the filename for the ELF kernel image, you need to provide the name of a core file on the command line. For a running kernel, that core file is the kernel core image, /proc/kcore. A typical invocation of gdb looks like the following:

```shell
gdb /usr/src/linux/vmlinux /proc/kcore
```

Linux loadable modules are ELF-format executable images; as such, they have been divided up into numerous sections. A typical module can contain a dozen or more sections, but there are typically three that are relevant in a debugging session:

- .text
  This section contains the executable code for the module. The debugger must know where this section is to be able to give tracebacks or set breakpoints. (Neither of these operations is relevant when running the debugger on /proc/kcore, but they can useful when working with kgdb, described below).
- .bss
- .data
  These two sections hold the module’s variables. Any variable that is not initial- ized at compile time ends up in .bss, while those that are initialized go into .data.

The first argument is the name of the uncompressed ELF kernel executable, not the zImage or bzImage or anything built specifically for the boot environment.
The second argument on the gdb command line is the name of the core file. Like any file in /proc, /proc/kcore is generated when it is read. When the read system call exe- cutes in the /proc filesystem, it maps to a data-generation function rather than a data- retrieval one; we’ve already exploited this feature in the section “Using the /proc Filesystem” earlier in this chapter. kcore is used to represent the kernel “executable” in the format of a core file; it is a huge file, because it represents the whole kernel address space, which corresponds to all physical memory. From within gdb, you can look at kernel variables by issuing the standard gdb commands. For example, p jiffies prints the number of clock ticks from system boot to the current time.

#### The kdb Kernel Debugger

Many readers may be wondering why the kernel does not have any more advanced debugging features built into it. The answer, quite simply, is that **Linus does not believe in interactive debuggers**. He fears that they lead to poor fixes, those which patch up symptoms rather than addressing the real cause of problems. Thus, no built-in debuggers.

Once you are running a kdb-enabled kernel, there are a couple of ways to enter the debugger. Pressing the Pause (or Break) key on the console starts up the debugger. kdb also starts up when a kernel oops happens or when a breakpoint is hit. In any case, you see a message that looks something like this:

```shell
Entering kdb (0xc0347b80) on processor 0 due to Keyboard Entry
[0]kdb>
```

Note that just about everything the kernel does stops when kdb is running. Nothing else should be running on a system where you invoke kdb; in particular, you should not have networking turned on—unless, of course, you are debugging a network driver. It is generally a good idea to boot the system in single-user mode if you will be using kdb.

#### The kgdb Patches

The two interactive debugging approaches we have looked at so far (using gdb on /proc/kcore and kdb) both fall short of the sort of environment that user-space appli- cation developers have become used to. Wouldn’t it be nice if there were a true debugger for the kernel that supported features like changing variables, breakpoints, etc.?

As it turns out, such a solution does exist. There are, as of this writing, two separate patches in circulation that allow gdb, with full capabilities, to be run against the ker- nel. Confusingly, both of these patches are called kgdb. They work by separating the system running the test kernel from the system running the debugger; the two are typically connected via a serial cable. Therefore, the developer can run gdb on his or her stable desktop system, while operating on a kernel running on a sacrificial test box. Setting up gdb in this mode takes a little time at the outset, but that investment can pay off quickly when a difficult bug shows up.

#### The User-Mode Linux Port

User-Mode Linux (UML) is an interesting concept. It is structured as a separate port of the Linux kernel with its own arch/um subdirectory. It does not run on a new type of hardware, however; instead, it runs on a virtual machine implemented on the Linux system call interface. Thus, UML allows the Linux kernel to run as a separate, user-mode process on a Linux system.

#### The Linux Trace Toolkit

The Linux Trace Toolkit (LTT) is a kernel patch and a set of related utilities that allow the tracing of events in the kernel. The trace includes timing information and can create a reasonably complete picture of what happened over a given period of time. Thus, it can be used not only for debugging but also for tracking down perfor- mance problems.

#### Dynamic Probes

Dynamic Probes (or DProbes) is a debugging tool released (under the GPL) by IBM for Linux on the IA-32 architecture. It allows the placement of a “probe” at almost any place in the system, in both user and kernel space. The probe consists of some code (written in a specialized, stack-oriented language) that is executed when con- trol hits the given point. This code can report information back to user space, change registers, or do a number of other things. The useful feature of DProbes is that once the capability has been built into the kernel, probes can be inserted anywhere within a running system without kernel builds or reboots. DProbes can also work with the LTT to insert new tracing events at arbitrary locations.

## Concurrency and Race Conditions

The purpose of this chapter is to begin the process of creating that understanding. To that end, we introduce facilities that are immediately applied to the scull driver from Chapter 3. Other facilities presented here are not put to use for some time yet. But first, we take a look at what could go wrong with our simple scull driver and how to avoid these potential problems.

### Pitfalls in scull

Race conditions are a result of uncontrolled access to shared data. When the wrong access pattern hap- pens, something unexpected results. For the race condition discussed here, the result is a memory leak. That is bad enough, but race conditions can often lead to system crashes, corrupted data, or security problems as well. Programmers can be tempted to disregard race conditions as extremely low probability events. But, in the comput- ing world, one-in-a-million events can happen every few seconds, and the conse- quences can be grave.

### Concurrency and Its Management

In a modern Linux system, there are numerous sources of concurrency and, there- fore, possible race conditions. Multiple user-space processes are running, and they can access your code in surprising combinations of ways. SMP systems can be executing your code simultaneously on different processors. Kernel code is preemptible; your driver’s code can lose the processor at any time, and the process that replaces it could also be running in your driver. Device interrupts are asynchronous events that can cause concurrent execution of your code. The kernel also provides various mech- anisms for delayed code execution, such as workqueues, tasklets, and timers, which can cause your code to run at any time in ways unrelated to what the current pro- cess is doing. In the modern, hot-pluggable world, your device could simply disap- pear while you are in the middle of working with it.

Race conditions come about as a result of shared access to resources. When two threads of execution\* have a reason to work with the same data structures (or hard- ware resources), the potential for mixups always exists. **So the first rule of thumb to keep in mind as you design your driver is to avoid shared resources whenever possible.** If there is no concurrent access, there can be no race conditions. So carefully-written kernel code should have a minimum of sharing. **The most obvious application of this idea is to avoid the use of global variables**. If you put a resource in a place where more than one thread of execution can find it, there should be a strong reason for doing so.

Here is the hard rule of resource sharing: **any time that a hardware or software resource is shared beyond a single thread of execution, and the possibility exists that one thread could encounter an inconsistent view of that resource, you must explicitly manage access to that resource.** In the scull example above, process B’s view of the situation is inconsistent; unaware that process A has already allocated memory for the (shared) device, it performs its own allocation and overwrites A’s work. In this case, we must control access to the scull data structure. We need to arrange things so that the code either sees memory that has been allocated or knows that no memory has been or will be allocated by anybody else. The usual technique for access management is called **locking or mutual exclusion**--making sure that only one thread of execution can manipulate a shared resource at any time. Much of the rest of this chapter will be devoted to locking.

First, however, we must briefly consider one other important rule. **When kernel code creates an object that will be shared with any other part of the kernel, that object must continue to exist (and function properly) until it is known that no outside refer- ences to it exist.** The instant that scull makes its devices available, it must be pre- pared to handle requests on those devices. And scull must continue to be able to handle requests on its devices until it knows that no reference (such as open user- space files) to those devices exists. Two requirements come out of this rule: no object can be made available to the kernel until it is in a state where it can function prop- erly, and references to such objects must be tracked. In most cases, you’ll find that the kernel handles reference counting for you, but there are always exceptions.

### Semaphores and Mutexes

Our goal is to make our operations on the scull data structure atomic, meaning that the entire operation happens at once as far as other threads of execution are concerned. For our memory leak example, we need to ensure that if one thread finds that a particular chunk of memory must be allocated, it has the opportunity to perform that allocation before any other thread can make that test. To this end, we must set up critical sections: code that can be exe- cuted by only one thread at any given time.

Semaphores are a well-understood concept in computer science. At its core, a sema- phore is a single integer value combined with a pair of functions that are typically called P and V. A process wishing to enter a critical section will call P on the relevant semaphore; if the semaphore’s value is greater than zero, that value is decremented by one and the process continues. If, instead, the semaphore’s value is 0 (or less), the process must wait until somebody else releases the semaphore. Unlocking a sema- phore is accomplished by calling V; this function increments the value of the sema- phore and, if necessary, wakes up processes that are waiting.

When semaphores are used for mutual exclusion—keeping multiple processes from running within a critical section simultaneously—their value will be initially set to 1. Such a semaphore can be held only by a single process or thread at any given time. A semaphore used in this mode is sometimes called a mutex, which is, of course, an abbreviation for “mutual exclusion.” Almost all semaphores found in the Linux ker- nel are used for mutual exclusion.

> Mutexes and semaphores are similar. Having both in the kernel is confusing.Thankfully, the formula dictating which to use is quite simple: Unless one of mutex’s additional con- straints prevent you from using them, prefer the new mutex type to semaphores.When writing new code, only specific, often low-level, uses need a semaphore. Start with a mu- tex and move to a semaphore only if you run into one of their constraints and have no other alternative.
> Knowing when to use a spin lock versus a mutex (or semaphore) is important to writing optimal code. In many cases, however, there is little choice. Only a spin lock can be used in interrupt context, whereas only a mutex can be held while a task sleeps.

| Requirement                             | Recommended Lock        |
| --------------------------------------- | ----------------------- |
| RLow overhead locking                   | Spin lock is preferred. |
| RShort lock hold time                   | Spin lock is preferred. |
| RLong lock hold time                    | Mutex is preferred.     |
| RNeed to lock from interrupt context    | Spin lock is required.  |
| RNeed to sleep while holding lock Mutex | is required.            |

#### The Linux Semaphore Implementation

The Linux kernel provides an implementation of semaphores that conforms to the above semantics, although the terminology is a little different. To use semaphores, kernel code must include <asm/semaphore.h>. The relevant type is struct semaphore; actual semaphores can be declared and initialized in a few ways. One is to create a semaphore directly, then set it up with sema_init:

```c
     void sema_init(struct semaphore *sem, int val);
```

where val is the initial value to assign to a semaphore.

Usually, however, semaphores are used in a mutex mode. To make this common case a little easier, the kernel has provided a set of helper functions and macros. Thus, a mutex can be declared and initialized with one of the following:

```c
     DECLARE_MUTEX(name);
     DECLARE_MUTEX_LOCKED(name);
```

Here, the result is a semaphore variable (called name) that is initialized to 1 (with DECLARE_MUTEX) or 0 (with DECLARE_MUTEX_LOCKED). In the latter case, the mutex starts out in a locked state; it will have to be explicitly unlocked before any thread will be allowed access.

If the mutex must be initialized at runtime (which is the case if it is allocated dynami- cally, for example), use one of the following:

```c
     void init_MUTEX(struct semaphore *sem);
     void init_MUTEX_LOCKED(struct semaphore *sem);
```

In the Linux world, the P function is called down—or some variation of that name. Here, “down” refers to the fact that the function decrements the value of the sema- phore and, perhaps after putting the caller to sleep for a while to wait for the sema- phore to become available, grants access to the protected resources. There are three versions of down:

```c
     void down(struct semaphore *sem);
     int down_interruptible(struct semaphore *sem);
     int down_trylock(struct semaphore *sem);
```

When the operations requiring mutual exclusion are complete, the semaphore must be returned. The Linux equivalent to V is up:

```c
    void up(struct semaphore *sem);
```

Once up has been called, the caller no longer holds the semaphore.

#### Using Semaphores in scull

The keys to proper use of locking primitives are to specify exactly which resources are to be protected and to make sure that every access to those resources uses the proper locking. In our example driver, everything of interest is contained within the scull_dev structure, so that is the logical scope for our locking regime.

We have chosen to use a separate semaphore for each virtual scull device. It would have been equally correct to use a single, global semaphore. The var-ious scull devices share no resources in common, however, and there is no reason to make one process wait while another process is working with a different scull device. Using a separate semaphore for each device allows operations on different devices to proceed in parallel and, therefore, improves performance.

#### Reader/Writer Semaphores

It is often possible to allow multiple concurrent readers, as long as nobody is trying to make any changes. Doing so can optimize performance significantly; read-only tasks can get their work done in parallel without having to wait for other readers to exit the critical section.

The Linux kernel provides a special type of semaphore called a rwsem (or “reader/writer semaphore”) for this situation

An rwsem allows either one writer or an unlimited number of readers to hold the semaphore. Writers get priority; as soon as a writer tries to enter the critical section, no readers will be allowed in until all writers have completed their work. This imple- mentation can lead to reader starvation—where readers are denied access for a long time—if you have a large number of writers contending for the semaphore. For this reason, rwsems are best used when write access is required only rarely, and writer access is held for short periods of time.

### Completions

In normal use, **code attempting to lock a semaphore finds that semaphore available almost all the time**; if there is significant contention for the semaphore, performance suffers and the locking scheme needs to be reviewed. So semaphores have been heavily optimized for the “available” case. When used to communicate task completion in the way shown above, however, the thread calling down will almost always have to wait; performance will suffer accordingly. Semaphores can also be subject to a (difficult) race condition when used in this way if they are declared as automatic variables. In some cases, the semaphore could vanish before the process calling up is finished with it.

A typical use of the completion mechanism is with kernel thread termination at module exit time. In the prototypical case, some of the driver internal workings is performed by a kernel thread in a while (1) loop. When the module is ready to be cleaned up, the exit function tells the thread to exit and then waits for completion. To this aim, the kernel includes a specific function to be used by the thread:

```c
void complete_and_exit(struct completion *c, long retval);
```

### Spinlocks

Spinlocks are simple in concept. A spinlock is a mutual exclusion device that can have only two values: “locked” and “unlocked.” It is usually implemented as a single bit in an integer value. Code wishing to take out a particular lock tests the relevant bit. If the lock is available, the “locked” bit is set and the code continues into the crit- ical section. If, instead, the lock has been taken by somebody else, the code goes into a tight loop where it repeatedly checks the lock until it becomes available. This loop is the “spin” part of a spinlock.

#### Introduction to the Spinlock API

The required include file for the spinlock primitives is <linux/spinlock.h>. An actual lock has the type spinlock_t. Like any other data structure, a spinlock must be ini- tialized. This initialization may be done at compile time as follows:

```c
spinlock_t my_lock = SPIN_LOCK_UNLOCKED;
```

or at runtime with:

```c
void spin_lock_init(spinlock_t *lock);
```

Before entering a critical section, your code must obtain the requisite lock with:

```c
void spin_lock(spinlock_t *lock);
```

**Note that all spinlock waits are, by their nature, uninterruptible. Once you call spin_lock, you will spin until the lock becomes available.**
To release a lock that you have obtained, pass it to:

```c
 void spin_unlock(spinlock_t *lock);
```

#### Spinlocks and Atomic Context

Imagine for a moment that your driver acquires a spinlock and goes about its busi- ness within its critical section. Somewhere in the middle, your driver loses the pro- cessor. Perhaps it has called a function (copy_from_user, say) that puts the process to sleep. Or, perhaps, kernel preemption kicks in, and a higher-priority process pushes your code aside. Your code is now holding a lock that it will not release any time in the foreseeable future. If some other thread tries to obtain the same lock, it will, in the best case, wait (spinning in the processor) for a very long time. In the worst case, the system could deadlock entirely.

Most readers would agree that this scenario is best avoided. **Therefore, the core rule that applies to spinlocks is that any code must, while holding a spinlock, be atomic. It cannot sleep; in fact, it cannot relinquish the processor for any reason except to service interrupts (and sometimes not even then).**

Avoiding sleep while holding a lock can be more difficult; many kernel functions can sleep, and this behavior is not always well documented. Copying data to or from user space is an obvious example: the required user-space page may need to be swapped in from the disk before the copy can proceed, and that operation clearly requires a sleep. Just about any operation that must allocate memory can sleep; kmalloc can decide to give up the processor, and wait for more memory to become available unless it is explicitly told not to. Sleeps can happen in surprising places; writing code that will execute under a spinlock requires paying attention to every function that you call.

Here’s another scenario: your driver is executing and has just taken out a lock that controls access to its device. While the lock is held, the device issues an interrupt, which causes your interrupt handler to run. The interrupt handler, before accessing the device, must also obtain the lock. Taking out a spinlock in an interrupt handler is a legitimate thing to do; that is one of the reasons that spinlock operations do not sleep. But what happens if the interrupt routine executes in the same processor as the code that took out the lock originally? While the interrupt handler is spinning, the noninterrupt code will not be able to run to release the lock. That processor will spin forever.

Avoiding this trap requires disabling interrupts (on the local CPU only) while the spinlock is held. There are variants of the spinlock functions that will disable interrupts for you.

The last important rule for spinlock usage is that spinlocks must always be held for the minimum time possible. The longer you hold a lock, the longer another proces- sor may have to spin waiting for you to release it, and the chance of it having to spin at all is greater. Long lock hold times also keep the current processor from schedul- ing, meaning that a higher priority process—which really should be able to get the CPU—may have to wait.

#### The Spinlock Functions

There are actually four functions\* that can lock a spinlock:

```c
void spin_lock(spinlock_t *lock);
void spin_lock_irqsave(spinlock_t *lock, unsigned long flags);
void spin_lock_irq(spinlock_t *lock);
void spin_lock_bh(spinlock_t *lock)
```

spin_lock_irqsave disables interrupts (on the local processor only) before taking the spinlock; the previous interrupt state is stored in flags. If you are absolutely sure nothing else might have already disabled interrupts on your processor (or, in other words, you are sure that you should enable interrupts when you release your spinlock), you can use spin_lock_irq instead and not have to keep track of the flags. Finally, spin_lock_bh disables software interrupts before taking the lock, but leaves hardware interrupts enabled.

**If you have a spinlock that can be taken by code that runs in (hardware or software) interrupt context, you must use one of the forms of spin_lock that disables interrupts**. Doing otherwise can deadlock the system, sooner or later. If you do not access your lock in a hardware interrupt handler, but you do via software interrupts (in code that runs out of a tasklet, for example, a topic covered in Chapter 7), you can use spin*lock* bh to safely avoid deadlocks while still allowing hardware interrupts to be serviced.

There are also four ways to release a spinlock; the one you use must correspond to the function you used to take the lock:

```c
void spin_unlock(spinlock_t *lock);
void spin_unlock_irqrestore(spinlock_t *lock, unsigned long flags);
void spin_unlock_irq(spinlock_t *lock);
void spin_unlock_bh(spinlock_t *lock);
```

There is also a set of nonblocking spinlock operations:

```c
int spin_trylock(spinlock_t *lock);
int spin_trylock_bh(spinlock_t *lock);
```

These functions return nonzero on success (the lock was obtained), 0 otherwise. There is no “try” version that disables interrupts.

#### Reader/Writer Spinlocks

The kernel provides a reader/writer form of spinlocks that is directly analogous to the reader/writer semaphores we saw earlier in this chapter. These locks allow any number of readers into a critical section simultaneously, but writers must have exclu- sive access. Reader/writer locks have a type of rwlock_t, defined in <linux/spinlock.h>. They can be declared and initialized in two ways:

```c
rwlock_t my_rwlock = RW_LOCK_UNLOCKED; /_ Static way _/
rwlock_t my_rwlock;
rwlock_init(&my_rwlock); /_ Dynamic way _/
```

The list of functions available should look reasonably familiar by now. For readers, the following functions are available:

```c
void read_lock(rwlock_t *lock);
void read_lock_irqsave(rwlock_t *lock, unsigned long flags);
void read_lock_irq(rwlock_t *lock);
void read_lock_bh(rwlock_t *lock);

void read_unlock(rwlock_t *lock);
void read_unlock_irqrestore(rwlock_t *lock, unsigned long flags);
void read_unlock_irq(rwlock_t *lock);
void read_unlock_bh(rwlock_t *lock);
```

Interestingly, there is no read_trylock. The functions for write access are similar:

```c
void write_lock(rwlock_t *lock);
void write_lock_irqsave(rwlock_t *lock, unsigned long flags);
void write_lock_irq(rwlock_t *lock);
void write_lock_bh(rwlock_t *lock);
int write_trylock(rwlock_t *lock);
void write_unlock(rwlock_t *lock);
void write_unlock_irqrestore(rwlock_t *lock, unsigned long flags);
void write_unlock_irq(rwlock_t *lock);
void write_unlock_bh(rwlock_t \*lock);
```

Reader/writer locks can starve readers just as rwsems can. This behavior is rarely a problem; however, if there is enough lock contention to bring about starvation, per- formance is poor anyway.

### Locking Traps

Many years of experience with locks—experience that predates Linux—have shown that locking can be very hard to get right. Managing concurrency is an inherently tricky undertaking, and there are many ways of making mistakes. In this section, we take a quick look at things that can go wrong.

#### Ambiguous Rules

When you create a resource that can be accessed concurrently, you should define which lock will control that access. **Locking should really be laid out at the beginning; it can be a hard thing to retrofit in afterward**. Time taken at the outset usually is paid back generously at debugging time.

As you write your code, you will doubtless encounter several functions that all require access to structures protected by a specific lock. At this point, you must be careful: if one function acquires a lock and then calls another function that also attempts to acquire the lock, your code deadlocks. Neither semaphores nor spin- locks allow a lock holder to acquire the lock a second time; should you attempt to do so, things simply hang.

To make your locking work properly, you have to write some functions with the assumption that their caller has already acquired the relevant lock(s). Usually, only your internal, static functions can be written in this way; functions called from outside must handle locking explicitly. **When you write internal functions that make assumptions about locking, do yourself (and anybody else who works with your code) a favor and document those assumptions explicitly**. It can be very hard to come back months later and figure out whether you need to hold a lock to call a particular function or not.

#### Lock Ordering Rules

In systems with a large number of locks (and the kernel is becoming such a system), it is not unusual for code to need to hold more than one lock at once. **If some sort of computation must be performed using two different resources, each of which has its own lock, there is often no alternative to acquiring both locks.**

Taking multiple locks can be dangerous, however. **If you have two locks, called Lock1 and Lock2, and code needs to acquire both at the same time, you have a potential deadlock**. Just imagine one thread locking Lock1 while another simulta- neously takes Lock2. Then each thread tries to get the one it doesn’t have. Both threads will deadlock.

The solution to this problem is usually simple: **when multiple locks must be acquired, they should always be acquired in the same order**. As long as this convention is followed, simple deadlocks like the one described above can be avoided. However, following lock ordering rules can be easier said than done. It is very rare that such rules are actually written down anywhere. Often the best you can do is to see what other code does.

A couple of rules of thumb can help. **If you must obtain a lock that is local to your code (a device lock, say) along with a lock belonging to a more central part of the kernel, take your lock first. If you have a combination of semaphores and spinlocks, you must, of course, obtain the semaphore(s) first;** calling down (which can sleep) while holding a spinlock is a serious error. But most of all, **try to avoid situations where you need more than one lock.**

### Alternatives to Locking

The Linux kernel provides a number of powerful locking primitives that can be used to keep the kernel from tripping over its own feet. But, as we have seen, the design and implementation of a locking scheme is not without its pitfalls. Often there is no alternative to semaphores and spinlocks; they may be the only way to get the job done properly. There are situations, however, where atomic access can be set up without the need for full locking. This section looks at other ways of doing things.

#### Lock-Free Algorithms

A data structure that can often be useful for lockless producer/consumer tasks is the circular buffer. This algorithm involves a producer placing data into one end of an array, while the consumer removes data from the other. When the end of the array is reached, the producer wraps back around to the beginning. So a circular buffer requires an array and two index values to track where the next new value goes and which value should be removed from the buffer next.

When carefully implemented, a circular buffer requires no locking in the absence of multiple producers or consumers. The producer is the only thread that is allowed to modify the write index and the array location it points to. As long as the writer stores a new value into the buffer before updating the write index, the reader will always see a consistent view. The reader, in turn, is the only thread that can access the read index and the value it points to. With a bit of care to ensure that the two pointers do not overrun each other, the producer and the consumer can access the buffer concur- rently with no race conditions.

Circular buffers show up reasonably often in device drivers. Networking adaptors, in particular, often use circular buffers to exchange data (packets) with the processor. Note that, as of 2.6.10, there is a generic circular buffer implementation available in the kernel; see <linux/kfifo.h> for information on how to use it.

#### Atomic Variables

An atomic_t holds an int value on all supported architectures. Because of the way this type works on some processors, however, the full integer range may not be available; thus, **you should not count on an atomic_t holding more than 24 bits**. The following operations are defined for the type and are guaranteed to be atomic with respect to all processors of an SMP computer. The operations are very fast, because they compile to a single machine instruction whenever possible.

```c
void atomic_set(atomic_t *v, int i);
atomic_t v = ATOMIC_INIT(0);
```

Set the atomic variable v to the integer value i. You can also initialize atomic val- ues at compile time with the ATOMIC_INIT macro.

```c
int atomic_read(atomic_t *v); Return the current value of v.
void atomic_add(int i, atomic_t *v);
```

Add i to the atomic variable pointed to by v. The return value is void, because there is an extra cost to returning the new value, and most of the time there’s no need to know it.

```c
void atomic_sub(int i, atomic_t *v); Subtract i from *v.
void atomic_inc(atomic_t *v);
void atomic_dec(atomic_t *v);
```

Increment or decrement an atomic variable.

```c
int atomic_inc_and_test(atomic_t *v);
int atomic_dec_and_test(atomic_t *v);
int atomic_sub_and_test(int i, atomic_t *v);
```

Perform the specified operation and test the result; if, after the operation, the atomic value is 0, then the return value is true; otherwise, it is false. Note that there is no atomic_add_and_test.

```c
int atomic_add_negative(int i, atomic_t *v);
```

Add the integer variable i to v. The return value is true if the result is negative, false otherwise.

```c
int atomic_add_return(int i, atomic_t *v);
int atomic_sub_return(int i, atomic_t *v);
int atomic_inc_return(atomic_t *v);
int atomic_dec_return(atomic_t *v);
```

Behave just like atomic_add and friends, with the exception that they return the new value of the atomic variable to the caller.

#### Bit Operations

The atomic_t type is good for performing integer arithmetic. It doesn’t work as well, however, when you need to manipulate individual bits in an atomic manner. For that purpose, instead, the kernel offers a set of functions that modify or test single bits atomically. Because the whole operation happens in a single step, no interrupt (or other processor) can interfere.

Atomic bit operations are very fast, since they perform the operation using a single machine instruction without disabling interrupts whenever the underlying platform can do that. The functions are architecture dependent and are declared in <asm/ bitops.h>. They are guaranteed to be atomic even on SMP computers and are useful to keep coherence across processors.

Unfortunately, data typing in these functions is architecture dependent as well. The nr argument (describing which bit to manipulate) is usually defined as int but is unsigned long for a few architectures. The address to be modified is usually a pointer to unsigned long, but a few architectures use void \* instead.

The available bit operations are:

```c
void set_bit(nr, void *addr);
```

Sets bit number nr in the data item pointed to by addr.

```c
void clear_bit(nr, void *addr);
```

Clears the specified bit in the unsigned long datum that lives at addr. Its seman- tics are otherwise the same as set_bit.

```c
void change_bit(nr, void *addr);
```

Toggles the bit.

```c
test_bit(nr, void *addr);
```

This function is the only bit operation that doesn’t need to be atomic; it simply returns the current value of the bit.

```c
int test_and_set_bit(nr, void *addr);
int test_and_clear_bit(nr, void *addr);
int test_and_change_bit(nr, void *addr);
```

Behave atomically like those listed previously, except that they also return the previous value of the bit.
When these functions are used to access and modify a shared flag, you don’t have to do anything except call them; they perform their operations in an atomic manner.

#### seqlocks

Seqlocks work in situations where the resource to be protected is small, simple, and frequently accessed, and where write access is rare but must be fast. Essentially, they work by allowing readers free access to the resource but requiring those readers to check for collisions with writers and, when such a collision happens, retry their access. Seqlocks generally cannot be used to protect data structures involving pointers, because the reader may be following a pointer that is invalid while the writer is changing the data structure.

#### Read-Copy-Update

Read-copy-update (RCU) is an advanced mutual exclusion scheme that can yield high performance in the right conditions.

RCU places a number of constraints on the sort of data structure that it can protect. It is optimized for situations where reads are common and writes are rare. The resources being protected should be accessed via pointers, and all references to those resources must be held only by atomic code. When the data structure needs to be changed, the writing thread makes a copy, changes the copy, then aims the relevant pointer at the new version—thus, the name of the algorithm. When the kernel is sure that no references to the old version remain, it can be freed.

Code using an RCU-protected data structure should bracket its ref- erences with calls to rcu_read_lock and rcu_read_unlock. As a result, RCU code tends to look like:

```c
struct my_stuff *stuff;
rcu_read_lock( );
stuff = find_the_stuff(args...);
do_something_with(stuff);
rcu_read_unlock( );
```

Code that needs to change the protected structure has to carry out a few steps. The first part is easy; it allocates a new structure, copies data from the old one if need be, then replaces the pointer that is seen by the read code. At this point, for the pur- poses of the read side, the change is complete; any code entering the critical section sees the new version of the data.

> [RCU1](http://www.rdrop.com/users/paulmck/rclock/intro/rclock_intro.html) > [RCU2](https://www.linuxjournal.com/article/6993) > [RCU3](https://docs.google.com/document/d/1X0lThx8OK0ZgLMqVoXiR4ZrGURHrXK6NyLRbeXe3Xac/edit)

## Advanced Char Driver Operations

This chapter examines a few concepts that you need to understand to write fully fea- tured char device drivers. We start with implementing the ioctl system call, which is a common interface used for device control. Then we proceed to various ways of syn- chronizing with user space; by the end of this chapter you have a good idea of how to put processes to sleep (and wake them up), implement nonblocking I/O, and inform user space when your devices are available for reading or writing. We finish with a look at how to implement a few different device access policies within drivers.

### ioctl

Most drivers need—in addition to the ability to read and write the device—the abil- ity to perform various types of hardware control via the device driver. Most devices can perform operations beyond simple data transfers; user space must often be able to request, for example, that the device lock its door, eject its media, report error information, change a baud rate, or self destruct. These operations are usually sup- ported via the ioctl method, which implements the system call by the same name.
In user space, the ioctl system call has the following prototype:

```c
int ioctl(int fd, unsigned long cmd, ...);
```

The ioctl driver method has a prototype that differs somewhat from the user-space version:

```c
     int (*ioctl) (struct inode *inode, struct file *filp,
                   unsigned int cmd, unsigned long arg);
```

#### Choosing the ioctl Commands

To help programmers create unique ioctl command codes, these codes have been split up into several bitfields. The first versions of Linux used 16-bit numbers: the top eight were the “magic” numbers associated with the device, and the bottom eight were a sequential number, unique within the device. This happened because Linus was “clueless” (his own word); a better division of bitfields was conceived only later. Unfortunately, quite a few drivers still use the old convention. They have to: chang- ing the command codes would break no end of binary programs, and that is not something the kernel developers are willing to do.


