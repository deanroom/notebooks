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

In the Linux world, the P function is called down—or some variation of that name. Here, “down” refers to the fact that the function decrements the value of the semaphore and, perhaps after putting the caller to sleep for a while to wait for the semaphore to become available, grants access to the protected resources. There are three versions of down:

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

Most drivers need—in addition to the ability to read and write the device—the ability to perform various types of hardware control via the device driver. Most devices can **perform operations beyond simple data transfers**; user space must often be able to request, for example, that the device lock its door, eject its media, report error information, change a baud rate, or self destruct. These operations are usually sup- ported via the ioctl method, which implements the system call by the same name.
In user space, the ioctl system call has the following prototype:

```c
int ioctl(int fd, unsigned long cmd, ...);
```

The ioctl driver method has a prototype that differs somewhat from the user-space version:

```c
int (*ioctl) (struct inode *inode, struct file *filp,
             unsigned int cmd, unsigned long arg);
```

Each ioctl command is, essentially, a separate, usually undocu- mented system call, and there is no way to audit these calls in any sort of comprehensive manner

#### Choosing the ioctl Commands

To help programmers create unique ioctl command codes, these codes have been split up into several bitfields. The first versions of Linux used 16-bit numbers: the top eight were the “magic” numbers associated with the device, and the bottom eight were a sequential number, unique within the device. This happened because Linus was “clueless” (his own word); a better division of bitfields was conceived only later. Unfortunately, quite a few drivers still use the old convention. They have to: chang- ing the command codes would break no end of binary programs, and that is not something the kernel developers are willing to do.

To choose ioctl numbers for your driver according to the Linux kernel convention, you should first check include/asm/ioctl.h and Documentation/ioctl-number.txt. The header defines the bitfields you will be using: type (magic number), ordinal number, direction of transfer, and size of argument. The ioctl-number.txt file lists the magic numbers used throughout the kernel,\* so you’ll be able to choose your own magic number and avoid overlaps. The text file also lists the reasons why the convention should be used.

We chose to implement both ways of passing integer arguments: by pointer and by explicit value (although, by an established convention, ioctl should exchange values by pointer). Similarly, both ways are used to return an integer number: by pointer or by setting the return value. This works as long as the return value is a positive inte- ger; as you know by now, on return from any system call, a positive value is pre- served (as we saw for read and write), while a negative value is considered an error and is used to set errno in user space.

#### The Return Value\

The implementation of ioctl is usually a switch statement based on the command number. But what should the default selection be when the command number doesn’t match a valid operation? The question is controversial. Several kernel func- tions return -EINVAL (“Invalid argument”), which makes sense because the com- mand argument is indeed not a valid one. The POSIX standard, however, states that if an inappropriate ioctl command has been issued, then -ENOTTY should be returned. This error code is interpreted by the C library as “inappropriate ioctl for device,” which is usually exactly what the programmer needs to hear. It’s still pretty com- mon, though, to return -EINVAL in response to an invalid ioctl command.

#### The Predefined Commands

Although the ioctl system call is most often used to act on devices, a few commands are recognized by the kernel. Note that these commands, when applied to your device, are decoded before your own file operations are called. Thus, if you choose the same number for one of your ioctl commands, you won’t ever see any request for that command, and the application gets something unexpected because of the con- flict between the ioctl numbers.

The predefined commands are divided into three groups:

- Those that can be issued on any file (regular, device, FIFO, or socket)
- Those that are issued only on regular files
- Those specific to the filesystem type

Commands in the last group are executed by the implementation of the hosting file- system (this is how the chattr command works). Device driver writers are interested only in the first group of commands, whose magic number is “T.” Looking at the workings of the other groups is left to the reader as an exercise; ext2_ioctl is a most interesting function (and easier to understand than one might expect), because it implements the append-only flag and the immutable flag.

#### Using the ioctl Argument

When a pointer is used to refer to user space, we must ensure that the user address is valid. An attempt to access an unverified user-supplied pointer can lead to incorrect behavior, a kernel oops, system corruption, or security problems. It is the driver’s responsibility to make proper checks on every user-space address it uses and to return an error if it is invalid.

To start, address verification (without transferring data) is implemented by the function access_ok, which is declared in <asm/uaccess.h>:

```c
int access_ok(int type, const void *addr, unsigned long size);
```

The first argument should be either VERIFY_READ or VERIFY_WRITE, depending on whether the action to be performed is reading the user-space memory area or writing it. The addr argument holds a user-space address, and size is a byte count.

In addition to the copy_from_user and copy_to_user functions, the programmer can exploit a set of functions that are optimized for the most used data sizes (one, two, four, and eight bytes). These functions are described in the following list and are defined in <asm/ uaccess.h>:

```c
put_user(datum, ptr)
__put_user(datum, ptr)
get_user(local, ptr)
__get_user(local, ptr)

```

If an attempt is made to use one of the listed functions to transfer a value that does not fit one of the specific sizes, the result is usually a strange message from the com- piler, such as “conversion to non-scalar type requested.” In such cases, copy_to_user or copy_from_user must be used.

#### Capabilities and Restricted Operations

The Linux kernel pro- vides a more flexible system called capabilities. A capability-based system leaves the all-or-nothing mode behind and breaks down privileged operations into separate subgroups. In this way, a particular user (or program) can be empowered to perform a specific privileged operation without giving away the ability to perform other, unre- lated operations. The kernel uses capabilities exclusively for permissions manage- ment and exports two system calls capget and capset, to allow them to be managed from user space.

The full set of capabilities can be found in <linux/capability.h>. These are the only capabilities known to the system; it is not possible for driver authors or system admin- istrators to define new ones without modifying the kernel source. A subset of those capabilities that might be of interest to device driver writers includes the following:

CAP_DAC_OVERRIDE
The ability to override access restrictions (data access control, or DAC) on files and directories.
CAP_NET_ADMIN
The ability to perform network administration tasks, including those that affect network interfaces.
CAP_SYS_MODULE
The ability to load or remove kernel modules.
CAP_SYS_RAWIO
The ability to perform “raw” I/O operations. Examples include accessing device ports or communicating directly with USB devices.
CAP_SYS_ADMIN
A catch-all capability that provides access to many system administration opera- tions.
CAP_SYS_TTY_CONFIG
The ability to perform tty configuration tasks.

Before performing a privileged operation, a device driver should check that the call- ing process has the appropriate capability; failure to do so could result user pro- cesses performing unauthorized operations with bad results on system stability or security. Capability checks are performed with the capable function (defined in <linux/sched.h>):

```c
int capable(int capability);
```

In the scull sample driver, any user is allowed to query the quantum and quantum set sizes. Only privileged users, however, may change those values, since inappropriate values could badly affect system performance. When needed, the scull implementa- tion of ioctl checks a user’s privilege level as follows:

```c
if (! capable (CAP_SYS_ADMIN))
      return -EPERM;
```

In the absence of a more specific capability for this task, CAP_SYS_ADMIN was chosen for this test.

#### The Implementation of the ioctl Commands

#### Device Control Without ioctl

Sometimes controlling the device is better accomplished by writing control sequences to the device itself. For example, this technique is used in the console driver, where so-called escape sequences are used to move the cursor, change the default color, or perform other configuration tasks. The benefit of implementing device control this way is that the user can control the device just by writing data, without needing to use (or sometimes write) programs built just for configuring the device. When devices can be controlled in this manner, the program issuing commands often need not even be running on the same system as the device it is controlling.

The drawback of controlling by printing is that it adds policy constraints to the device; for example, it is viable only if you are sure that the control sequence can’t appear in the data being written to the device during normal operation. This is only partly true for ttys. Although a text display is meant to display only ASCII charac- ters, sometimes control characters can slip through in the data being written and can, therefore, affect the console setup. This can happen, for example, when you cat a binary file to the screen; the resulting mess can contain anything, and you often end up with the wrong font on your console.

Controlling by write is definitely the way to go for those devices that don’t transfer data but just respond to commands, such as robotic devices.

When writing command-oriented drivers, there’s no reason to implement the ioctl method. An additional command in the interpreter is easier to implement and use.

### Blocking I/O

A call to read may come when no data is available, but more is expected in the future. Or a process could attempt to write, but your device is not ready to accept the data, because your output buffer is full. The calling process usually does not care about such issues; the programmer simply expects to call read or write and have the call return after the necessary work has been done. So, in such cases, your driver should (by default) block the process, putting it to sleep until the request can proceed.

This section shows how to put a process to sleep and wake it up again later on. As usual, however, we have to explain a few concepts first.

#### Introduction to Sleeping

A couple of rules that you must keep in mind to be able to code sleeps in a safe manner.

- The first of these rules is: **never sleep when you are running in an atomic context**. What that means, with regard to sleeping, is that your driver cannot sleep while holding a spinlock, seqlock, or RCU lock. You also cannot sleep if you have disabled interrupts. It is legal to sleep while holding a semaphore, but you should look very carefully at any code that does so. If code sleeps while holding a sema- phore, any other thread waiting for that semaphore also sleeps. So any sleeps that happen while holding semaphores should be short, and you should convince your- self that, by holding the semaphore, you are not blocking the process that will even- tually wake you up.

- Another thing to remember with sleeping is that, **when you wake up, you never know how long your process may have been out of the CPU or what may have changed in the mean time**. You also do not usually know if another process may have been sleeping for the same event; that process may wake before you and grab whatever resource you were waiting for. The end result is that you can make no assumptions about the state of the system after you wake up, and **you must check to ensure that the condition you were waiting for is, indeed, true.**

- One other relevant point, of course, is that **your process cannot sleep unless it is assured that somebody else, somewhere, will wake it up**. The code doing the awakening must also be able to find your process to be able to do its job. Making sure that a wakeup happens is a matter of thinking through your code and knowing, for each sleep, exactly what series of events will bring that sleep to an end. Making it possible for your sleeping process to be found is, instead, accomplished through a data structure called a wait queue. A wait queue is just what it sounds like: a list of processes, all waiting for a specific event.

In Linux, a wait queue is managed by means of a “wait queue head,” a structure of type wait_queue_head_t, which is defined in <linux/wait.h>. A wait queue head can be defined and initialized statically with:

```c
     DECLARE_WAIT_QUEUE_HEAD(name);
```

or dynamically as follows:

```c
     wait_queue_head_t my_queue;
     init_waitqueue_head(&my_queue);
```

#### Simple Sleeping

When a process sleeps, it does so in expectation that some condition will become true in the future. As we noted before, any process that sleeps must check to be sure that the condition it was waiting for is really true when it wakes up again. The sim- plest way of sleeping in the Linux kernel is a macro called wait_event (with a few variants); it combines handling the details of sleeping with a check on the condition a process is waiting for. The forms of wait_event are:

```c
     wait_event(queue, condition)
     wait_event_interruptible(queue, condition)
     wait_event_timeout(queue, condition, timeout)
     wait_event_interruptible_timeout(queue, condition, timeout)
```

In all of the above forms, queue is the wait queue head to use. Notice that it is passed “by value.” The condition is an arbitrary boolean expression that is evaluated by the macro before and after sleeping; until condition evaluates to a true value, the pro- cess continues to sleep. Note that condition may be evaluated an arbitrary number of times, so it should not have any side effects.

The other half of the picture, of course, is waking up. Some other thread of execu- tion (a different process, or an interrupt handler, perhaps) has to perform the wakeup for you, since your process is, of course, asleep. The basic function that wakes up sleeping processes is called wake_up. It comes in several forms (but we look at only two of them now):

```c
     void wake_up(wait_queue_head_t *queue);
     void wake_up_interruptible(wait_queue_head_t *queue);
```

> [Windows Synchonsization](https://learn.microsoft.com/en-us/windows/win32/sync/about-synchronization)
>
> To synchronize access to a resource, use one of the synchronization objects in one of the wait functions. The state of a synchronization object is either signaled or nonsignaled. The wait functions allow a thread to block its own execution until a specified nonsignaled object is set to the signaled state. For more information, see Interprocess Synchronization.
>
> **Synchronization Objects**
>
> - Event
>   Notifies one or more waiting threads that an event has occurred. For more information, see [Event Objects](https://learn.microsoft.com/en-us/windows/win32/sync/event-objects).
> - Mutex
>   Can be owned by only one thread at a time, enabling threads to coordinate mutually exclusive access to a shared resource. For more information, see [Mutex Objects](https://learn.microsoft.com/en-us/windows/win32/sync/mutex-objects).
> - Semaphore
>   Maintains a count between zero and some maximum value, limiting the number of threads that are simultaneously accessing a shared resource. For more information, see [Semaphore Objects](https://learn.microsoft.com/en-us/windows/win32/sync/semaphore-objects).
> - Waitable timer
>   Notifies one or more waiting threads that a specified time has arrived. For more information, see [Waitable Timer Objects](https://learn.microsoft.com/en-us/windows/win32/sync/waitable-timer-objects).
>
> [Condition Variables](https://learn.microsoft.com/en-us/windows/win32/sync/condition-variables)
> Condition variables are synchronization primitives that enable threads to wait until a particular condition occurs. Condition variables are **user-mode objects** that cannot be shared across processes.
> Condition variables enable threads to atomically release a lock and enter the sleeping state. They can be used with critical sections or slim reader/writer (SRW) locks. Condition variables support operations that "wake one" or "wake all" waiting threads. After a thread is woken, it re-acquires the lock it released when the thread entered the sleeping state.
>
> [Critical Section Objects](https://learn.microsoft.com/en-us/windows/win32/sync/critical-section-objects)
> A critical section object provides synchronization similar to that provided by a mutex object, except that **a critical section can be used only by the threads of a single process**. Critical section objects cannot be shared across processes.

#### Blocking and Nonblocking Operations

In the case of a blocking operation, which is the default, the following behavior should be implemented in order to adhere to the standard semantics:

- If a process calls read but no data is (yet) available, the process must block. The process is awakened as soon as some data arrives, and that data is returned to the caller, even if there is less than the amount requested in the count argument to the method.
- If a process calls write and there is no space in the buffer, the process must block, and it must be on a different wait queue from the one used for reading. When some data has been written to the hardware device, and space becomes free in the output buffer, the process is awakened and the write call succeeds, although the data may be only partially written if there isn’t room in the buffer for the count bytes that were requested.

Both these statements assume that there are both input and output buffers; in practice, almost every device driver has them. The input buffer is required to avoid los- ing data that arrives when nobody is reading. In contrast, data can’t be lost on write, because if the system call doesn’t accept data bytes, they remain in the user-space buffer. Even so, the output buffer is almost always useful for squeezing more perfor- mance out of the hardware.

The behavior of read and write is different if O_NONBLOCK is specified. In this case, the calls simply return -EAGAIN (“try it again”) if a process calls read when no data is available or if it calls write when there’s no space in the buffer.

As you might expect, nonblocking operations return immediately, allowing the application to poll for data. Applications must be careful when using the stdio func- tions while dealing with nonblocking files, because they can easily mistake a non- blocking return for EOF. They always have to check errno.

Only the read, write, and open file operations are affected by the nonblocking flag.

#### A Blocking I/O Example

```c
struct scull_pipe {
  wait_queue_head_t inq, outq;
  char *buffer, *end;
  int buffersize;
  char *rp, *wp;
  int nreaders, nwriters;
  struct fasync_struct *async_queue; /* asynchronous readers */
  struct semaphore sem;              /* mutual exclusion semaphore */
  struct cdev cdev;                  /* Char device structure */
};

static ssize_t scull_p_read (struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
{
    struct scull_pipe *dev = filp->private_data;

    if (down_interruptible(&dev->sem))
        return -ERESTARTSYS;

    while (dev->rp == dev->wp) { /* nothing to read */ up(&dev->sem); /* release the lock */
        up(&dev->sem); /* release the lock */
        if (filp->f_flags & O_NONBLOCK)
                return -EAGAIN;
        PDEBUG("\"%s\" reading: going to sleep\n", current->comm);
        if (wait_event_interruptible(dev->inq, (dev->rp != dev->wp)))
            return -ERESTARTSYS; /* signal: tell the fs layer to handle it */
        /* otherwise loop, but first reacquire the lock */
        if (down_interruptible(&dev->sem))
    }

    /* ok, data is there, return something */
    if (dev->wp > dev->rp)
        count = min(count, (size_t)(dev->wp - dev->rp));
    else /* the write pointer has wrapped, return data up to dev->end */
        count = min(count, (size_t)(dev->end - dev->rp));
    if (copy_to_user(buf, dev->rp, count)) {
        up (&dev->sem);
        return -EFAULT;
    }
    dev->rp += count;
    if (dev->rp == dev->end)
        dev->rp = dev->buffer; /* wrapped */
    up (&dev->sem);
    /* finally, awake any writers and return */
    wake_up_interruptible(&dev->outq);
    PDEBUG("\"%s\" did read %li bytes\n",current->comm, (long)count);
    return count;
}

```

#### Advanced Sleeping

##### How a process sleeps

##### Manual sleeps

##### Exclusive waits

##### The details of waking up

##### Ancient history: sleep_on

##### Testing the Scullpipe Driver

```c
int main(int argc, char **argv)
    {
        int delay = 1, n, m = 0;
        if (argc > 1)
            delay=atoi(argv[1]);
        fcntl(0, F_SETFL, fcntl(0,F_GETFL) | O_NONBLOCK); /* stdin */
        fcntl(1, F_SETFL, fcntl(1,F_GETFL) | O_NONBLOCK); /* stdout */
        while (1) {
            n = read(0, buffer, 4096);
            if (n >= 0)
                m = write(1, buffer, n);
            if ((n < 0 || m < 0) && (errno != EAGAIN))
                break;
            sleep(delay);
        }
        perror(n < 0 ? "stdin" : "stdout");
        exit(1);
}

```

### poll and select

Applications that use nonblocking I/O often use the poll, select, and epoll system calls as well. poll, select, and epoll have essentially the same functionality: **each allows a process to determine whether it can read from or write to one or more open files without blocking**. These calls can also block a process until any of a given set of file descriptors becomes available for reading or writing. Therefore, they are often used in applications that must use multiple input or output streams without getting stuck on any one of them.

Support for any of these calls requires support from the device driver. This support (for all three calls) is provided through the driver’s poll method. This method has the following prototype:

```c
     unsigned int (*poll) (struct file *filp, poll_table *wait);
```

The driver method is called whenever the user-space program performs a poll, select, or epoll system call involving a file descriptor associated with the driver. The device method is in charge of these two steps:

1. Call poll_wait on one or more wait queues that could indicate a change in the poll status. If no file descriptors are currently available for I/O, the kernel causes the process to wait on the wait queues for all file descriptors passed to the sys- tem call.
2. Return a bit mask describing the operations (if any) that could be immediately performed without blocking.

#### Interaction with read and write

##### Reading data from the device

- If there is data in the input buffer, the read call should return immediately, with no noticeable delay, even if less data is available than the application requested, and the driver is sure the remaining data will arrive soon. You can always return less data than you’re asked for if this is convenient for any reason (we did it in scull), provided you return at least one byte. In this case, poll should return POLLIN|POLLRDNORM.
- If there is no data in the input buffer, by default read must block until at least one byte is there. If O_NONBLOCK is set, on the other hand, read returns immedi- ately with a return value of -EAGAIN (although some old versions of System V return 0 in this case). In these cases, poll must report that the device is unread- able until at least one byte arrives. As soon as there is some data in the buffer, we fall back to the previous case.
- If we are at end-of-file, read should return immediately with a return value of 0, independent of O_NONBLOCK. poll should report POLLHUP in this case.

##### Writing to the device

- If there is space in the output buffer, write should return without delay. It can accept less data than the call requested, but it must accept at least one byte. In this case, poll reports that the device is writable by returning POLLOUT|POLLWRNORM.
- If the output buffer is full, by default write blocks until some space is freed. If O_NONBLOCK is set, write returns immediately with a return value of -EAGAIN (older System V Unices returned 0). In these cases, poll should report that the file is not writable. If, on the other hand, the device is not able to accept any more data, write returns -ENOSPC (“No space left on device”), independently of the setting of O_NONBLOCK.
- Never make a write call wait for data transmission before returning, even if O_NONBLOCK is clear. This is because many applications use select to find out whether a write will block. If the device is reported as writable, the call must not block. If the program using the device wants to ensure that the data it enqueues in the output buffer is actually transmitted, the driver must provide an fsync method. For instance, a removable device should have an fsync entry point.??? (I don't understand this--Jerry)

#### Flushing pending output

We’ve seen how the write method by itself doesn’t account for all data output needs. The fsync function, invoked by the system call of the same name, fills the gap. This method’s prototype is

```c
      int (*fsync) (struct file *file, struct dentry *dentry, int datasync);
```

If some application ever needs to be assured that data has been sent to the device, the fsync method must be implemented regardless of whether O_NONBLOCK is set. A call to fsync should return only when the device has been completely flushed (i.e., the out- put buffer is empty), even if that takes some time. The datasync argument is used to distinguish between the fsync and fdatasync system calls; as such, it is only of inter- est to filesystem code and can be ignored by drivers.

#### The Underlying Data Structure

Whenever a user application calls poll, select, or epoll_ctl, the kernel invokes the poll method of all files referenced by the system call, passing the same poll_table to each of them. The poll_table structure is just a wrapper around a func- tion that builds the actual data structure. That structure, for poll and select, is a linked list of memory pages containing poll_table_entry structures. Each poll_table_entry holds the struct file and wait_queue_head_t pointers passed to poll_wait, along with an associated wait queue entry. The call to poll_wait sometimes also adds the process to the given wait queue. The whole structure must be maintained by the kernel so that the process can be removed from all of those queues before poll or select returns.

#### Asynchronous Notification

Let’s imagine a process that executes a long computational loop at low priority but needs to process incoming data as soon as possible. If this process is responding to new observations available from some sort of data acquisition peripheral, it would like to know immediately when new data is available. This application could be writ- ten to call poll regularly to check for data, but, for many situations, there is a better way. By enabling asynchronous notification, this application can receive a signal whenever data becomes available and need not concern itself with polling.

User programs have to execute two steps to enable asynchronous notification from an input file. First, they specify a process as the “owner” of the file. When a process invokes the F_SETOWN command using the fcntl system call, the process ID of the owner process is saved in filp->f_owner for later use. This step is necessary for the kernel to know just whom to notify. In order to actually enable asynchronous notifi- cation, the user programs must set the FASYNC flag in the device by means of the F_SETFL fcntl command.

After these two calls have been executed, the input file can request delivery of a SIGIO signal whenever new data arrives. The signal is sent to the process (or process group, if the value is negative) stored in filp->f_owner.

##### The Driver’s Point of View

A more relevant topic for us is how the device driver can implement asynchronous signaling. The following list details the sequence of operations from the kernel’s point of view:

1. When F_SETOWN is invoked, nothing happens, except that a value is assigned to filp->f_owner.
2. When F_SETFL is executed to turn on FASYNC, the driver’s fasync method is called. This method is called whenever the value of FASYNC is changed in filp->f_flags to notify the driver of the change, so it can respond properly. The flag is cleared by default when the file is opened. We’ll look at the standard implementation of the driver method later in this section.
3. When data arrives, all the processes registered for asynchronous notification must be sent a SIGIO signal.

### Seeking a Device

#### The llseek Implementation

### Access Control on a Device File

Offering access control is sometimes vital for the reliability of a device node. Not only should unauthorized users not be permitted to use the device (a restriction is enforced by the filesystem permission bits), but sometimes only one authorized user should be allowed to open the device at a time.

#### Single-Open Devices

Allowing only a single process to open a device has undesirable properties, but it is also the easiest access control to implement for a device driver.

#### Restricting Access to a Single User at a Time

The next step beyond a single-open device is to let a single user open a device in mul- tiple processes but allow only one user to have the device open at a time.

#### Blocking open as an Alternative to EBUSY

The alternative to EBUSY, as you may have guessed, is to implement blocking open.

The problem with a blocking-open implementation is that it is really unpleasant for the interactive user, who has to keep guessing what is going wrong.

This kind of problem (a need for different, incompatible policies for the same device) is often best solved by implementing one device node for each access policy. An example of this practice can be found in the Linux tape driver, which provides multi- ple device files for the same device. Different device files will, for example, cause the drive to record with or without compression, or to automatically rewind the tape when the device is closed.

#### Cloning the Device on open

Another technique to manage access control is to create different private copies of the device, depending on the process opening it.

When copies of the device are created by the software driver, we call them virtual devices—just as virtual consoles use a single physical tty device.

#### Quick Reference

This chapter introduced the following symbols and header files:

```c
#include <linux/ioctl.h>
#include <asm/uaccess.h>
#include <linux/capability.h>
#include <linux/wait.h>
#include <linux/sched.h>
```

Declares all the macros used to define ioctl commands. It is currently included by <linux/fs.h>.

## Time, Delays, and Deferred Work

### Measuring Time Lapses

Timer interrupts are generated by the system’s timing hardware at regular intervals; this interval is programmed at boot time by the kernel according to the value of HZ, which is an architecture-dependent value defined in <linux/param.h> or a subplat- form file included by it. Default values in the distributed kernel source range from 50 to 1200 ticks per second on real hardware, down to 24 for software simulators. Most platforms run at 100 or 1000 interrupts per second; the popular x86 PC defaults to 1000, although it used to be 100 in previous versions (up to and including 2.4). As a general rule, even if you know the value of HZ, you should never count on that spe- cific value when programming.

#### Using the jiffies Counter

Timer interrupts are generated by the system’s timing hardware at regular intervals; this interval is programmed at boot time by the kernel according to the value of HZ, which is an architecture-dependent value defined in <linux/param.h> or a subplat- form file included by it. Default values in the distributed kernel source range from 50 to 1200 ticks per second on real hardware, down to 24 for software simulators. Most platforms run at 100 or 1000 interrupts per second; the popular x86 PC defaults to 1000, although it used to be 100 in previous versions (up to and including 2.4). As a general rule, even if you know the value of HZ, you should never count on that spe- cific value when programming.

The counter and the utility functions to read it live in <linux/jiffies.h>, although you’ll usually just include <linux/sched.h>, that automatically pulls jiffies.h in. Need- less to say, both jiffies and jiffies_64 must be considered read-only.

To compare your cached value (like stamp_1 above) and the current value, you should use one of the following macros:

```c

     #include <linux/jiffies.h>
     int time_after(unsigned long a, unsigned long b);
     int time_before(unsigned long a, unsigned long b);
     int time_after_eq(unsigned long a, unsigned long b);
     int time_before_eq(unsigned long a, unsigned long b);
```

#### Processor-Specific Registers

In modern processors, the pressing demand for empirical performance figures is thwarted by the intrinsic unpredictability of instruction timing in most CPU designs due to cache memories, instruction scheduling, and branch prediction. As a response, CPU manufacturers introduced a way to count clock cycles as an easy and reliable way to measure time lapses. Therefore, most modern processors include a counter register that is steadily incremented once at each clock cycle. Nowadays, this clock counter is the only reliable way to carry out high-resolution timekeeping tasks.

### Knowing the Current Time

It’s quite unlikely that a driver will ever need to know the wall-clock time, expressed in months, days, and hours; the information is usually needed only by user pro- grams such as cron and syslogd. Dealing with real-world time is usually best left to user space, where the C library offers better support; besides, such code is often too policy-related to belong in the kernel.

There is a kernel function that turns a wall- clock time into a jiffies value, however:

```c
     #include <linux/time.h>
     unsigned long mktime (unsigned int year, unsigned int mon,
                           unsigned int day, unsigned int hour,
                           unsigned int min, unsigned int sec);
```

To repeat: dealing directly with wall-clock time in a driver is often a sign that policy is being implemented and should therefore be questioned.

### Delaying Execution

#### Long Delays

Occasionally a driver needs to delay execution for relatively long periods—more than one clock tick. There are a few ways of accomplishing this sort of delay; we start with the simplest technique, then proceed to the more advanced techniques.

##### Busy waiting

If you want to delay execution by a multiple of the clock tick, allowing some slack in the value, the easiest (though not recommended) implementation is a loop that mon- itors the jiffy counter. The busy-waiting implementation usually looks like the follow- ing code, where j1 is the value of jiffies at the expiration of the delay:

```c
while (time_before(jiffies, j1)) cpu_relax( );
```

In any case, this approach should definitely be avoided whenever possible. We show it here because on occasion you might want to run this code to better understand the internals of other code.

##### Yielding the processor

As we have seen, busy waiting imposes a heavy load on the system as a whole; we would like to find a better technique. The first change that comes to mind is to xplicitly release the CPU when we’re not interested in it. This is accomplished by calling the schedule function, declared in <linux/sched.h>:

```c
while (time_before(jiffies, j1)) { schedule( );
}

```

However, is still isn’t optimal. The current process does nothing but release the CPU, but it remains in the run queue. If it is the only runnable process, it actually runs (it calls the scheduler, which selects the same process, which calls the scheduler, which...). In other words, the load of the machine (the average number of running processes) is at least one, and the idle task (process number 0, also called swapper for historical reasons) never runs.

##### Timeouts

But the best way to implement a delay, as you may imagine, is usually to ask the kernel to do it for you. There are two ways of setting up jiffy- based timeouts, depending on whether your driver is waiting for other events or not.

If your driver uses a wait queue to wait for some other event, but you also want to be sure that it runs within a certain period of time, it can use wait_event_timeout or wait_event_interruptible_timeout:

```c

     #include <linux/wait.h>
     long wait_event_timeout(wait_queue_head_t q, condition, long timeout);
     long wait_event_interruptible_timeout(wait_queue_head_t q,
                           condition, long timeout);
```

#### Short Delays

When a device driver needs to deal with latencies in its hardware, the delays involved are usually a few dozen microseconds at most. In this case, relying on the clock tick is definitely not the way to go.
The kernel functions ndelay, udelay, and mdelay serve well for short delays, delaying execution for the specified number of nanoseconds, microseconds, or milliseconds respectively.\* Their prototypes are:

```c

     #include <linux/delay.h>
     void ndelay(unsigned long nsecs);
     void udelay(unsigned long usecs);
     void mdelay(unsigned long msecs);
```

It’s important to remember that the three delay functions are busy-waiting; other tasks can’t be run during the time lapse. Thus, they replicate, though on a different scale, the behavior of jitbusy. Thus, these functions should only be used when there is no practical alternative.

There is another way of achieving millisecond (and longer) delays that does not involve busy waiting. The file <linux/delay.h> declares these functions:

```c
     void msleep(unsigned int millisecs);
     unsigned long msleep_interruptible(unsigned int millisecs);
     void ssleep(unsigned int seconds)
```

In general, if you can tolerate longer delays than requested, you should use schedule_timeout, msleep, or ssleep.

### Kernel Timers

Whenever you need to schedule an action to happen later, without blocking the current process until that time arrives, kernel timers are the tool for you. These timers are used to schedule execution of a function at a particular time in the future, based on the clock tick, and can be used for a variety of tasks; for example, polling a device by checking its state at regular intervals when the hardware can’t fire interrupts. Other typical uses of kernel timers are turning off the floppy motor or finishing another lengthy shut down operation. In such cases, delaying the return from close would impose an unnecessary (and surprising) cost on the application program. Finally, the kernel itself uses the timers in several situations, including the implementation of schedule_timeout.

This asynchronous execution resembles what happens when a hardware interrupt happens (which is discussed in detail in Chapter 10). In fact, kernel timers are run as the result of a “software interrupt.” When running in this sort of atomic context, your code is subject to a number of constraints. Timer functions must be atomic in all the ways we discussed in the section “Spinlocks and Atomic Context” in Chapter 5, but there are some additional issues brought about by the lack of a pro- cess context. We will introduce these constraints now; they will be seen again in sev- eral places in later chapters. Repetition is called for because the rules for atomic contexts must be followed assiduously, or the system will find itself in deep trouble.

A number of actions require the context of a process in order to be executed. When you are outside of process context (i.e., in interrupt context), you must observe the following rules:

- No access to user space is allowed. Because there is no process context, there is no path to the user space associated with any particular process.
- The current pointer is not meaningful in atomic mode and cannot be used since the relevant code has no connection with the process that has been interrupted.
- No sleeping or scheduling may be performed. Atomic code may not call schedule or a form of wait_event, nor may it call any other function that could sleep. For example, calling kmalloc(..., GFP_KERNEL) is against the rules. Sema- phores also must not be used since they can sleep.

#### The Timer API

The kernel provides drivers with a number of functions to declare, register, and remove kernel timers. The following excerpt shows the basic building blocks:

```c
 #include <linux/timer.h>
     struct timer_list {
             /* ... */
             unsigned long expires;
             void (*function)(unsigned long);
             unsigned long data;
};
     void init_timer(struct timer_list *timer);
     struct timer_list TIMER_INITIALIZER(_function, _expires, _data);
     void add_timer(struct timer_list * timer);
     int del_timer(struct timer_list * timer);

```

#### The Implementation of Kernel Timers

The implementation of the timers has been designed to meet the following requirements and assumptions:

- Timer management must be as lightweight as possible.
- The design should scale well as the number of active timers increases.
- Most timers expire within a few seconds or minutes at most, while timers with long delays are pretty rare.
- A timer should run on the same CPU that registered it.

### Tasklets

Another kernel facility related to timing issues is the tasklet mechanism. It is mostly used in interrupt management.

Tasklets resemble kernel timers in some ways. They are always run at interrupt time, they always run on the same CPU that schedules them, and they receive an unsigned long argument. Unlike kernel timers, however, **you can’t ask to execute the function at a specific time. By scheduling a tasklet, you simply ask for it to be executed at a later time chosen by the kernel**. This behavior is especially useful with interrupt han- dlers, where the hardware interrupt must be managed as quickly as possible, but most of the data management can be safely delayed to a later time. Actually, a tasklet, just like a kernel timer, is executed (in atomic mode) in the context of a “soft interrupt,” a kernel mechanism that executes asynchronous tasks with hardware interrupts enabled.

### Workqueues

Workqueues are, superficially, similar to tasklets; they allow kernel code to request that a function be called at some future time. There are, however, some significant differences between the two, including:

- Tasklets run in software interrupt context with the result that all tasklet code must be atomic. Instead, workqueue functions run in the context of a special kernel process; as a result, they have more flexibility. In particular, workqueue functions can sleep.
- Tasklets always run on the processor from which they were originally submitted. Workqueues work in the same way, by default.
- Kernel code can request that the execution of workqueue functions be delayed for an explicit interval.

The key difference between the two is that tasklets execute quickly, for a short period of time, and in atomic mode, while workqueue functions may have higher latency but need not be atomic. Each mechanism has situations where it is appropriate.

#### The Shared Queue

A device driver, in many cases, does not need its own workqueue. If you only submit tasks to the queue occasionally, it may be more efficient to simply use the shared, default workqueue that is provided by the kernel. If you use this queue, however, you must be aware that you will be sharing it with others. Among other things, that means that you should not monopolize the queue for long periods of time (no long sleeps), and it may take longer for your tasks to get their turn in the processor.

## Allocating Memory

Thus far, we have used **kmalloc and kfree for the allocation and freeing of memory**. The Linux kernel offers a richer set of memory allocation primitives, however. In this chapter, we look at other ways of using memory in device drivers and how to optimize your system’s memory resources.

### The Real Story of kmalloc

The kmalloc allocation engine is a powerful tool and easily learned because of its similarity to malloc. The function is fast (unless it blocks) and doesn’t clear the mem- ory it obtains; the allocated region still holds its previous content.\* The allocated region is also contiguous in physical memory. In the next few sections, we talk in detail about kmalloc, so you can compare it with the memory allocation techniques that we discuss later.

#### The Flags Argument

Remember that the prototype for kmalloc is:

```c
#include <linux/slab.h>
     void *kmalloc(size_t size, int flags);
```

##### Memory zones

The Linux kernel knows about a minimum of three memory zones: DMA-capable memory, normal memory, and high memory. While allocation normally happens in the normal zone, setting either of the bits just mentioned requires memory to be allo- cated from a different zone. The idea is that every computer platform that must know about special memory ranges (instead of considering all RAM equivalents) will fall into this abstraction.

#### The Size Argument

The kernel manages the system’s physical memory, which is available only in page- sized chunks. As a result, kmalloc looks rather different from a typical user-space malloc implementation. A simple, heap-oriented allocation technique would quickly run into trouble; it would have a hard time working around the page boundaries. Thus, the kernel uses a special page-oriented allocation technique to get the best use from the system’s RAM.

Linux handles memory allocation by creating a set of pools of memory objects of fixed sizes. Allocation requests are handled by going to a pool that holds sufficiently large objects and handing an entire memory chunk back to the requester. The mem- ory management scheme is quite complex, and the details of it are not normally all that interesting to device driver writers.

### Lookaside Caches

A device driver often ends up allocating many objects of the same size, over and over. Given that the kernel already maintains a set of memory pools of objects that are all the same size, why not add some special pools for these high-volume objects? In fact, the kernel does implement a facility to create this sort of pool, which is often called a lookaside cache. Device drivers normally do not exhibit the sort of memory behavior that justifies using a lookaside cache, but there can be exceptions; the USB and SCSI drivers in Linux 2.6 use caches.

The cache manager in the Linux kernel is sometimes called the “slab allocator.” For that reason, its functions and types are declared in <linux/slab.h>. The slab allocator implements caches that have a type of kmem_cache_t; they are created with a call to kmem_cache_create:

```c
kmem_cache_t *kmem_cache_create(const char *name, size_t size,
                               size_t offset,
                               unsigned long flags,
                               void (*constructor)(void *, kmem_cache_t *,
                                                   unsigned long flags),
                               void (*destructor)(void *, kmem_cache_t *,
                                                  unsigned long flags));
```

#### A scull Based on the Slab Caches: scullc

Time for an example. scullc is a cut-down version of the scull module that imple- ments only the bare device—the persistent memory region. Unlike scull, which uses kmalloc, scullc uses memory caches. The size of the quantum can be modified at compile time and at load time, but not at runtime—that would require creating a new memory cache, and we didn’t want to deal with these unneeded details.

#### Memory Pools

There are places in the kernel where memory allocations cannot be allowed to fail. As a way of guaranteeing allocations in those situations, the kernel developers created an abstraction known as a memory pool (or “mempool”). A memory pool is really just a form of a lookaside cache that tries to always keep a list of free memory around for use in emergencies.

### get_free_page and Friends

If a module needs to **allocate big chunks of memory**, it is usually better to use a page- oriented technique.

To allocate pages, the following functions are available:
get_zeroed_page(unsigned int flags);
Returns a pointer to a new page and fills the page with zeros.
**get_free_page(unsigned int flags);
Similar to get_zeroed_page, but doesn’t clear the page.
**get_free_pages(unsigned int flags, unsigned int order);
Allocates and returns a pointer to the first byte of a memory area that is poten- tially several (physically contiguous) pages long but doesn’t zero the area.

#### A scull Using Whole Pages: scullp

In order to test page allocation for real, we have released the scullp module together with other sample code. It is a reduced scull, just like scullc introduced earlier.
Memory quanta allocated by scullp are whole pages or page sets: the scullp_order variable defaults to 0 but can be changed at either compile or load time.
The following lines show how it allocates memory:

```c
/* Here's the allocation of a single quantum */
if (!dptr->data[s_pos]) {
    dptr->data[s_pos] =
    (void *)__get_free_pages(GFP_KERNEL, dptr->order);
            if (!dptr->data[s_pos])
                goto nomem;
            memset(dptr->data[s_pos], 0, PAGE_SIZE << dptr->order);
        }
```

The code to deallocate memory in scullp looks like this:

```c
/* This code frees a whole quantum-set */
for (i = 0; i < qset; i++)
   if (dptr->data[i])
       free_pages((unsigned long)(dptr->data[i]),
               dptr->order);
```

The main advantage of page-level allocation isn’t actually speed, but rather more efficient memory usage. Allocating by pages wastes no memory, whereas using kmalloc wastes an unpredictable amount of memory because of allocation granularity.

But the biggest advantage of the \_\_get_free_page functions is that the pages obtained are completely yours, and you could, in theory, assemble the pages into a linear area by appropriate tweaking of the page tables. For example, you can allow a user pro- cess to mmap memory areas obtained as single unrelated pages.

#### The alloc_pages Interface

For now, suffice it to say that struct page is an internal kernel structure that describes a page of memory. As we will see, there are many places in the kernel where it is necessary to work with page structures; they are especially useful in any situation where you might be deal- ing with high memory, which does not have a constant address in kernel space.

The real core of the Linux page allocator is a function called
alloc_pages_node:

```c
 struct page *alloc_pages_node(int nid, unsigned int flags,
                                   unsigned int order);
```

### vmalloc and Friends

The next memory allocation function that we show you is vmalloc, which allocates a contiguous memory region in the virtual address space. Although the pages are not con- secutive in physical memory (each page is retrieved with a separate call to alloc_page), the kernel sees them as a contiguous range of addresses. vmalloc returns 0 (the NULL address) if an error occurs, otherwise, it returns a pointer to a linear memory area of size at least size.

### Per-CPU Variables

When you create a per-CPU variable, each processor on the system gets its own copy of that variable. This may seem like a strange thing to want to do, but it has its advantages. Access to per-CPU variables requires (almost) no locking, because each processor works with its own copy. Per-CPU variables can also remain in their respective processors’ caches, which leads to significantly better performance for frequently updated quantities.

A good example of per-CPU variable use can be found in the networking subsystem. The kernel maintains no end of counters tracking how many of each type of packet was received; these counters can be updated thousands of times per second. Rather than deal with the caching and locking issues, the networking developers put the sta- tistics counters into per-CPU variables. Updates are now lockless and fast. On the rare occasion that user space requests to see the values of the counters, it is a simple matter to add up each processor’s version and return the total.

### Obtaining Large Buffers

#### Acquiring a Dedicated Buffer at Boot Time

If you really need a huge buffer of physically contiguous memory, the best approach is often to allocate it by requesting memory at boot time. Allocation at boot time is the only way to retrieve consecutive memory pages while bypassing the limits imposed by \_\_get_free_pages on the buffer size, both in terms of maximum allowed size and limited choice of sizes. Allocating memory at boot time is a “dirty” tech- nique, because it bypasses all memory management policies by reserving a private memory pool. This technique is inelegant and inflexible, but it is also the least prone to failure. Needless to say, a module can’t allocate memory at boot time; only driv- ers directly linked to the kernel can do that.

## Communicating with Hardware

### I/O Ports and I/O Memory

> In Linux and many other operating systems, I/O ports and I/O memory (often referred to as memory-mapped I/O or MMIO) are mechanisms that the CPU and OS use to communicate with peripherals and other hardware devices. Both methods provide interfaces to access hardware registers, but they use different address spaces and instructions.
>
>I/O Ports:
>
>This is a method for CPUs and devices to communicate using dedicated I/O address space, distinct from the main memory address space.
I/O ports are accessed using special CPU instructions like IN and OUT (on x86 architectures).
In Linux, /dev/port represents the I/O ports, but direct access is typically restricted for safety reasons. You'd use functions like inb(), outb(), inw(), outw(), etc., in kernel space to access I/O ports.
>
>I/O Memory (MMIO):
>
>In MMIO, the device registers are mapped into the same address space as the main memory. Accessing a memory address in this region communicates with a device instead of reading or writing to RAM.
MMIO is accessed using regular load/store CPU instructions, but the addresses point to device registers instead of actual memory.
In Linux kernel development, you'd use functions like ioread32(), iowrite32(), and so on to read from or write to MMIO regions. Also, before accessing MMIO, you'd use functions like ioremap() to map device memory into kernel space.

Every peripheral device is controlled by writing and reading its registers. Most of the time a device has several registers, and they are accessed at consecutive addresses, either in the memory address space or in the I/O address space.

While some CPU manufacturers implement a single address space in their chips, oth- ers decided that peripheral devices are different from memory and, therefore, deserve a separate address space.

Linux implements the concept of I/O ports on all computer platforms it runs on, even on platforms where the CPU implements a single address space. The implementation of port access sometimes depends on the specific make and model of the host computer (because different models use different chipsets to map bus transactions into memory address space).

#### I/O Registers and Conventional Memory

The main difference between I/O registers and RAM is that I/O operations have **side effects**, while memory operations have none: the only effect of a memory write is storing a value to a location, and a memory read returns the last value written there. Because memory access speed is so critical to CPU performance, the no-side-effects case has been optimized in several ways: values are cached and read/write instruc- tions are reordered.

### Using I/O Ports

I/O ports are the means by which drivers communicate with many devices, at least part of the time. This section covers the various functions available for making use of I/O ports; we also touch on some portability issues.

#### I/O Port Allocation

The kernel provides a registration interface that allows your driver to claim the ports it needs. The core function in that interface is request_region:

```c
#include <linux/ioport.h>
struct resource *request_region(unsigned long first, unsigned long n,
                               const char *name);
void release_region(unsigned long start, unsigned long n);
int check_region(unsigned long first, unsigned long n);
```

#### Manipulating I/O ports

After a driver has requested the range of I/O ports it needs to use in its activities, it must read and/or write to those ports. To this end, most hardware differentiates between 8-bit, 16-bit, and 32-bit ports. Usually you can’t mix them like you nor- mally do with system memory access.\*
A C program, therefore, must call different functions to access different size ports. As suggested in the previous section, computer architectures that support only memory- mapped I/O registers fake port I/O by remapping port addresses to memory addresses, and the kernel hides the details from the driver in order to ease portabil- ity. The Linux kernel headers (specifically, the architecture-dependent header <asm/ io.h>) define the following inline functions to access I/O ports:

```c
unsigned inb(unsigned port);
void outb(unsigned char byte, unsigned port);
```

Read or write byte ports (eight bits wide). The port argument is defined as unsigned long for some platforms and unsigned short for others. The return type of inb is also different across architectures.

```c
unsigned inw(unsigned port);
void outw(unsigned short word, unsigned port);
```

These functions access 16-bit ports (one word wide); they are not available when compiling for the S390 platform, which supports only byte I/O.

```c
unsigned inl(unsigned port);
void outl(unsigned longword, unsigned port);
```

These functions access 32-bit ports. longword is declared as either unsigned long or unsigned int, according to the platform. Like word I/O, “long” I/O is not available on S390.

#### I/O Port Access from User Space

The functions just described are primarily meant to be used by device drivers, but they can also be used from user space, at least on PC-class computers. The GNU C library defines them in <sys/io.h>. The following conditions should apply in order for inb and friends to be used in user-space code:

- The program must be compiled with the -O option to force expansion of inline functions.
- The ioperm or iopl system calls must be used to get permission to perform I/O operations on ports. ioperm gets permission for individual ports, while iopl gets permission for the entire I/O space. Both of these functions are x86-specific.
- The program must run as root to invoke ioperm or iopl.\* Alternatively, one of its ancestors must have gained port access running as root.

#### String Operations

In addition to the single-shot in and out operations, some processors implement spe- cial instructions to transfer a sequence of bytes, words, or longs to and from a single I/O port of the same size. These are the so-called string instructions, and they per- form the task more quickly than a C-language loop can do. The following macros implement the concept of string I/O either by using a single machine instruction or by executing a tight loop if the target processor has no instruction that performs string I/O.

The prototypes for string functions are:

```c
void insb(unsigned port, void *addr, unsigned long count);
void outsb(unsigned port, void *addr, unsigned long count);
```

Read or write count bytes starting at the memory address addr.
Data is read from or written to the single port port.

```c
void insw(unsigned port, void *addr, unsigned long count);
void outsw(unsigned port, void *addr, unsigned long count);
```

Read or write 16-bit values to a single 16-bit port.

```c
void insl(unsigned port, void *addr, unsigned long count);
void outsl(unsigned port, void *addr, unsigned long count);
```

Read or write 32-bit values to a single 32-bit port.

There is one thing to keep in mind when using the string functions: they move a straight byte stream to or from the port. When the port and the host system have dif- ferent byte ordering rules, the results can be surprising. Reading a port with inw swaps the bytes, if need be, to make the value read match the host ordering. The string functions, instead, do not perform this swapping.

#### Pausing I/O

If your device misses some data, or if you fear it might miss some, you can use paus- ing functions in place of the normal ones.

#### Platform Dependencies

I/O instructions are, by their nature, highly processor dependent. Because they work with the details of how the processor handles moving data in and out, it is very hard to hide the differences between systems. As a consequence, much of the source code related to port I/O is platform-dependent.

### An I/O Port Example

A digital I/O port, in its most common incarnation, is a byte-wide I/O location, either memory-mapped or port-mapped. When you write a value to an output loca- tion, the electrical signal seen on output pins is changed according to the individual bits being written. When you read a value from the input location, the current logic level seen on input pins is returned as individual bit values.

#### An Overview of the Parallel Port

The parallel interface, in its minimal configuration (we overlook the ECP and EPP modes) is made up of **three 8-bit ports**. The PC standard starts the I/O ports for the first parallel interface at 0x378 and for the second at 0x278. The first port is a **bidirectional data register**; it connects directly to pins 2–9 on the physical connector. The second port is a **read-only status register**; when the parallel port is being used for a printer, this register reports several aspects of printer status, such as being online, out of paper, or busy. The third port is an **output-only control register**, which, among other things, controls whether interrupts are enabled.

#### A Sample Driver

The driver we introduce is called short (Simple Hardware Operations and Raw Tests). All it does is read and write a few 8-bit ports, starting from the one you select at load time. By default, it uses the port range assigned to the parallel interface of the PC. Each device node (with a unique minor number) accesses a different port. The short driver doesn’t do anything useful; it just isolates for external use as a single instruction acting on a port. If you are not used to port I/O, you can use short to get familiar with it; you can measure the time it takes to transfer data through a port or play other games.

### Using I/O Memory

Despite the popularity of I/O ports in the x86 world, the main mechanism used to communicate with devices is through memory-mapped registers and device mem- ory. Both are called I/O memory because the difference between registers and mem- ory is transparent to software.

#### I/O Memory Allocation and Mapping

I/O memory regions must be allocated prior to use. The interface for allocation of memory regions (defined in <linux/ioport.h>) is:

```c
struct resource *request_mem_region(unsigned long start, unsigned long len,
                                   char *name);
```

#### Accessing I/O Memory

On some platforms, you may get away with using the return value from ioremap as a pointer. Such use is not portable, and, increasingly, the kernel developers have been working to eliminate any such use. The proper way of getting at I/O memory is via a set of functions (defined via <asm/io.h>) provided for that purpose.
To read from I/O memory, use one of the following:

```c
     unsigned int ioread8(void *addr);
     unsigned int ioread16(void *addr);
     unsigned int ioread32(void *addr);
```

Here, addr should be an address obtained from ioremap (perhaps with an integer off- set); the return value is what was read from the given I/O memory.

There is a similar set of functions for writing to I/O memory:

```c
     void iowrite8(u8 value, void *addr);
     void iowrite16(u16 value, void *addr);
     void iowrite32(u32 value, void *addr);
```

If you must read or write a series of values to a given I/O memory address, you can use the repeating versions of the functions:

```c
     void ioread8_rep(void *addr, void *buf, unsigned long count);
     void ioread16_rep(void *addr, void *buf, unsigned long count);
     void ioread32_rep(void *addr, void *buf, unsigned long count);
     void iowrite8_rep(void *addr, const void *buf, unsigned long count);
     void iowrite16_rep(void *addr, const void *buf, unsigned long count);
     void iowrite32_rep(void *addr, const void *buf, unsigned long count);
```

The ioread functions read count values from the given addr to the given buf, while the iowrite functions write count values from the given buf to the given addr. Note that count is expressed in the size of the data being written; ioread32_rep reads count 32- bit values starting at buf.
The functions described above all perform I/O to the given addr. If, instead, you need to operate on a block of I/O memory, you can use one of the following:

```c
     void memset_io(void *addr, u8 value, unsigned int count);
     void memcpy_fromio(void *dest, void *source, unsigned int count);
     void memcpy_toio(void *dest, void *source, unsigned int count);
```

These functions behave like their C library analogs.
If you read through the kernel source, you see many calls to an older set of functions when I/O memory is being used. These functions still work, but their use in new code is discouraged. Among other things, they are less safe because they do not per- form the same sort of type checking. Nonetheless, we describe them here:

```c
unsigned readb(address);
unsigned readw(address);
unsigned readl(address);
```

These macros are used to retrieve 8-bit, 16-bit, and 32-bit data values from I/O memory.

```c
void writeb(unsigned value, address);
void writew(unsigned value, address);
void writel(unsigned value, address);
```

Like the previous functions, these functions (macros) are used to write 8-bit, 16- bit, and 32-bit data items.
Some 64-bit platforms also offer readq and writeq, for quad-word (8-byte) memory operations on the PCI bus. The quad-word nomenclature is a historical leftover from the times when all real processors had 16-bit words. Actually, the L naming used for 32-bit values has become incorrect too, but renaming everything would confuse things even more.

#### Ports as I/O Memory

Some hardware has an interesting feature: some versions use I/O ports, while others use I/O memory. The registers exported to the processor are the same in either case, but the access method is different. As a way of making life easier for drivers dealing with this kind of hardware, and as a way of minimizing the apparent differences between I/O port and memory accesses, the 2.6 kernel provides a function called ioport_map:
```c
void *ioport_map(unsigned long port, unsigned int count);
```

## Interrupt Handling

An interrupt is simply a signal that the hardware can send when it wants the processor’s attention. Linux handles interrupts in much the same way that it handles signals in user space. For the most part, a driver need only register a handler for its device’s interrupts, and handle them properly when they arrive. Of course, underneath that simple picture there is some complexity; in particular, interrupt handlers are somewhat limited in the actions they can perform as a result of how they are run.

### Installing an Interrupt Handler

Interrupt lines are a precious and often limited resource, particularly when there are only 15 or 16 of them. The kernel keeps a registry of interrupt lines, similar to the registry of I/O ports. A module is expected to request an interrupt channel (or IRQ, for interrupt request) before using it and to release it when finished. In many situa- tions, modules are also expected to be able to share interrupt lines with other driv- ers, as we will see. The following functions, declared in <linux/interrupt.h>, implement the interrupt registration interface:

```c

int request_irq(unsigned int irq,
                irqreturn_t (*handler)(int, void *, struct pt_regs *),
                unsigned long flags,
                const char *dev_name,
                void *dev_id);
void free_irq(unsigned int irq, void *dev_id);
```

The correct place to call request_irq is when the device is first opened, before the hardware is instructed to generate interrupts. The place to call free_irq is the last time the device is closed, after the hardware is told not to interrupt the processor any more. The disadvantage of this technique is that you need to keep a per-device open count so that you know when interrupts can be disabled.

### The /proc Interface

Whenever a hardware interrupt reaches the processor, an internal counter is incre- mented, providing a way to check whether the device is working as expected. Reported interrupts are shown in /proc/interrupts.

### Autodetecting the IRQ Number

One of the most challenging problems for a driver at initialization time can be how to determine which IRQ line is going to be used by the device. The driver needs the information in order to correctly install the handler.

#### Kernel-assisted probing

The Linux kernel offers a low-level facility for probing the interrupt number. It works for only nonshared interrupts, but most hardware that is capable of working in a shared interrupt mode provides better ways of finding the configured interrupt num- ber anyway. The facility consists of two functions, declared in <linux/interrupt.h> (which also describes the probing machinery):
unsigned long probe_irq_on(void);
This function returns a bit mask of unassigned interrupts. The driver must pre- serve the returned bit mask, and pass it to probe_irq_off later. After this call, the driver should arrange for its device to generate at least one interrupt.
int probe_irq_off(unsigned long);
After the device has requested an interrupt, the driver calls this function, passing as its argument the bit mask previously returned by probe_irq_on. probe_irq_off returns the number of the interrupt that was issued after “probe_on.” If no inter- rupts occurred, 0 is returned (therefore, IRQ 0 can’t be probed for, but no cus- tom device can use it on any of the supported architectures anyway). If more than one interrupt occurred (ambiguous detection), probe_irq_off returns a negative value.

#### Do-it-yourself probing
### Fast and Slow Handlers
Older versions of the Linux kernel took great pains to distinguish between “fast” and “slow” interrupts. Fast interrupts were those that could be handled very quickly, whereas handling slow interrupts took significantly longer. Slow interrupts could be sufficiently demanding of the processor, and it was worthwhile to reenable inter- rupts while they were being handled. Otherwise, tasks requiring quick attention could be delayed for too long.
In modern kernels, most of the differences between fast and slow interrupts have dis- appeared. There remains only one: fast interrupts (those that were requested with the SA_INTERRUPT flag) are executed with all other interrupts disabled on the current processor. Note that other processors can still handle interrupts, although you will never see two processors handling the same IRQ at the same time.
So, which type of interrupt should your driver use? On modern systems, SA_INTERRUPT is intended only for use in a few, specific situations such as timer interrupts. Unless you have a strong reason to run your interrupt handler with other interrupts disabled, you should not use SA_INTERRUPT.
This description should satisfy most readers, although someone with a taste for hard- ware and some experience with her computer might be interested in going deeper. If you don’t care about the internal details, you can skip to the next section.

#### Implementing a Handler

The only peculiarity is that a handler runs at interrupt time and, therefore, suffers some restrictions on what it can do. These restrictions are the same as those we saw with kernel timers. A handler can’t transfer data to or from user space, because it doesn’t execute in the context of a process. Handlers also cannot do anything that would sleep, such as calling wait_event, allocating memory with anything other than GFP_ATOMIC, or locking a semaphore. Finally, handlers cannot call schedule.

## USB Drivers
