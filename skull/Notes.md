# Some Notes about scull driver
the major number identifies the driver associated with the device
The minor number is used by the kernel to determine exactly which device is being
referred to.


°°°°°°°°°°°°°°°°°°°°°°°°°°°

function to obtain  one  or  more  device  numbers  to  work  with. 

first ->is the beginning device number of the range you would like to allocate.  
count ->is the total number of contiguous device numbers you are requesting  
name -> is the name of the device that should be associated with this number range; it will appear in /proc/devices and sysfs.  

the return value from register_chrdev_region will be 0 if the allocation was successfully performed. In case of error, a negative error code
will be returned, and you will not have access to the requested region.

```
int register_chrdev_region(dev_t first, unsigned int count,
                           char *name);
```



*register_chrdev_region works* well  if  you  know  ahead  of  time  exactly  which  device numbers you want. Often, however, you will not know which major numbers your
device will use; there is a constant effort within the Linux kernel development community to move over to the use of dynamicly-allocated device numbers. The kernel
will happily allocate a major number for you on the fly, but you must request this allocation by using a different function  
°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°  
*dev->* is an output-only parameter that will, on successful completion,  hold  the  first  number  in  your  allocated  range  
*firstminor ->* should  be  the requested first minor number to use; 

```
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor,
                        unsigned int count, char *name);
```


Regardless  of  how  you  allocate  your  device  numbers,  you  should  free  them  when
they are no longer in use. Device numbers are freed with:
```
void unregister_chrdev_region(dev_t first, unsigned int count);
```
°°°°°°°°°°°°°°°°°°°°°°°°°°°°  

### File Structure:

The file structure represents an open file . (It is not specific to device drivers; every open file in the system has an associated
struct file in kernel space.) It is created by the  kernel  on open and  is  passed  to  any  function  that  operates  on  the  file,  until
the last close.  After  all  instances  of  the  file  are  closed,  the  kernel  releases  the  data structure.

Some field:
```
void *private_data;
* this field is set by default NULL but it can be used to point to allocated data; it is used to preserve state information across system calls
```
# Some Scull Charachteristics
Internally,scull represents each device with a structure of type struct scull_dev. This structure is defined as:
```
struct scull_dev {
    struct scull_qset *data;  /* Pointer to first quantum set */
    int quantum;              /* the current quantum size */
    int qset;                 /* the current array size */
    unsigned long size;       /* amount of data stored here */
    unsigned int access_key;  /* used by sculluid and scullpriv */
    struct semaphore sem;     /* mutual exclusion semaphore     */
    struct cdev cdev;     /* Char device structure      */
};
```

cdev is the struct that interfaces our device to the kernel

°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°°



The scull driver introduces two core functions used to manage memory in the Linux kernel. These functions, defined in
<linux/slab.h>, are:
```
void *kmalloc(size_t size, int flags);
void kfree(void *ptr);
```
A  call  to kmalloc attempts  to  allocate size bytes  of  memory;  the  return  value  is  a pointer to that memory or
NULL if the allocation fails. The flags argument is used to describe how the memory should be allocated;


In scull, each device is a linked list of pointers, each of which points to a scull_dev
structure. Each such structure can refer, by default, to at most four million bytes,
through an array of intermediate pointers. The released source uses an array of 1000
pointers to areas of 4000 bytes. We call each memory area a quantum and the array
(or its length) a quantum set.

////////

to copy data from kernel space to userspace and viceversa we use two functions defined in <asm/uaccess.h>:
```
unsigned long copy_to_user(void __user *to,const void *from,unsigned long count);


unsigned long copy_from_user(void *to,const void __user *from,unsigned long count);
```
The role of the two functions is not limited to copying data to and from user-space:
they also check whether the user space pointer is valid. If the pointer is invalid, no copy
is performed; if an invalid address is encountered during the copy, on the other hand,
only part of the data is copied. In both cases, the return value is the amount of mem-
ory still to be copied. The scull code looks for this error return, and returns -EFAULT to
the user if it’s not 0


### The read Method

The return value for read is interpreted by the calling application program:
* If the value equals the count argument passed to the read system call, the
requested number of bytes has been transferred. This is the optimal case.
* If the value is positive, but smaller than count , only part of the data has been
transferred. This may happen for a number of reasons, depending on the device.
Most often, the application program retries the read. For instance, if you read
using the fread function, the library function reissues the system call until com-
pletion of the requested data transfer.
* If the value is 0 , end-of-file was reached (and no data was read).
* A negative value means there was an error. The value specifies what the error
was, according to <linux/errno.h>. Typical values returned on error include -EINTR
(interrupted system call) or -EFAULT (bad address).


### The write Method

write, like read, can transfer less data than was requested, according to the following
rules for the return value:
* If the value equals count , the requested number of bytes has been transferred.
* If the value is positive, but smaller than count , only part of the data has been
transferred. The program will most likely retry writing the rest of the data.
* If the value is 0 , nothing was written. This result is not an error, and there is no
reason to return an error code. Once again, the standard library retries the call to
write. We’ll examine the exact meaning of this case in Chapter 6, where block-
ing write is introduced.
* A negative value means an error occurred; as for read, valid error values are
those defined in <linux/errno.h>.


