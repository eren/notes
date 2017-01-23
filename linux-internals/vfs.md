# VFS
The VFS is object-oriented[3]. A family of data structures represents the common file model. These data structures are akin to objects. Because the kernel is programmed strictly in C, without the benefit of a language directly supporting object-oriented paradigms, the data structures are represented as C structures. The structures contain both data and pointers to filesystem-implemented functions that operate on the data.

![](http://www.makelinux.net/books/lkd2/graphics/12fig02.gif)

# Data Structures
## 4 Main Objects in VFS
- The `superblock` object, which represents a specific mounted filesystem.
- The `inode` object, which represents a specific file.
- The `dentry` object, which represents a directory entry, a single component of a path.
- The `file` object, which represents an open file as associated with a process.

Note that because the VFS treats directories as normal files, there is not a specific directory object. Recall from earlier in this chapter that a `dentry` represents a component in a path, which might include a regular file. In other words, **a `dentry` is not the same as a directory, but a directory is the same as a file.**

## Operations Object
An operations object is contained within each of these primary objects. These objects describe the methods that the kernel invokes against the primary objects. 

- The `super_operation`s object, which contains the methods that the kernel can invoke on a specific filesystem, such as read_inode() and sync_fs().
- The `inode_operations` object, which contains the methods that the kernel can invoke on a specific file, such as create() and link().
- The `dentry_operations` object, which contains the methods that the kernel can invoke on a specific directory entry, such as d_compare() and d_delete().
- The `file` object, which contains the methods that a process can invoke on an open file, such as read() and write().

The operations objects are implemented as a structure of pointers to functions that operate on the parent object.

## Other Objects
- Each registered filesystem is represented by a `file_system_type` structure.

  This object describes the filesystem and its capabilities

- Each mount point is represented by the `vfsmount` structure.

  This structure contains information about the mount point, such as its location and mount flags.

There are **three** per-process structures. Namely;

- `files_struct`: files associated with the process

  ```c
  struct files_struct {
    int count;
    fd_set close_on_exec;
    fd_set open_fds;
    struct file * fd[NR_OPEN];
  };
  ```

- `fs_struct`: describes mountpoint

  ```c
  struct fs_struct {
    int count;
    unsigned short umask;
    struct inode * root, * pwd;
  };
  ```
  
- `namespace`: namespace of the process (network, file, etc)

# Superblock
The superblock object is implemented by each filesystem, and is used to store information describing that specific filesystem. This object usually corresponds to the filesystem superblock or the filesystem control block, which is stored in a special sector on disk (hence the object's name). Filesystems that are not disk-based (a virtual memorybased filesystem, such as sysfs, for example) generate the superblock on the fly and store it in memory.

A superblock object is created and initialized via the `alloc_super()` function. **When mounted, a filesystem invokes this function, reads its superblock off of the disk, and fills in its superblock object.**

```c
struct super_block {
        struct list_head        s_list;            /* list of all superblocks */
        dev_t                   s_dev;             /* identifier */
        unsigned long           s_blocksize;       /* block size in bytes */
        unsigned long           s_old_blocksize;   /* old block size in bytes */
        unsigned char           s_blocksize_bits;  /* block size in bits */
        unsigned char           s_dirt;            /* dirty flag */
        unsigned long long      s_maxbytes;        /* max file size */
        struct file_system_type s_type;            /* filesystem type */
        struct super_operations s_op;              /* superblock methods */
        struct dquot_operations *dq_op;            /* quota methods */
        struct quotactl_ops     *s_qcop;           /* quota control methods */
        struct export_operations *s_export_op;     /* export methods */
        unsigned long            s_flags;          /* mount flags */
        unsigned long            s_magic;          /* filesystem's magic number */
        struct dentry            *s_root;          /* directory mount point */
        struct rw_semaphore      s_umount;         /* unmount semaphore */
        struct semaphore         s_lock;           /* superblock semaphore */
        int                      s_count;          /* superblock ref count */
        int                      s_syncing;        /* filesystem syncing flag */
        int                      s_need_sync_fs;   /* not-yet-synced flag */
        atomic_t                 s_active;         /* active reference count */
        void                     *s_security;      /* security module */
        struct list_head         s_dirty;          /* list of dirty inodes */
        struct list_head         s_io;             /* list of writebacks */
        struct hlist_head        s_anon;           /* anonymous dentries */
        struct list_head         s_files;          /* list of assigned files */
        struct block_device      *s_bdev;          /* associated block device */
        struct list_head         s_instances;      /* instances of this fs */
        struct quota_info        s_dquot;          /* quota-specific options */
        char                     s_id[32];         /* text name */
        void                     *s_fs_info;       /* filesystem-specific info */
        struct semaphore         s_vfs_rename_sem; /* rename semaphore */
};
```

The most importnat object is `s_op` which is a superblock operations table. 

```c
struct super_operations {
        // Creates and initializes a new inode object under the given superblock.
        struct inode *(*alloc_inode) (struct super_block *sb);
        
        //  Deallocates the given inode.
        void (*destroy_inode) (struct inode *);
        
        //  Reads the inode specified by inode->i_ino from disk and fills in the rest of the inode structure.
        void (*read_inode) (struct inode *);
        
        // Invoked by the VFS when an inode is dirtied (modified).
        // Journaling filesystems (such as ext3) use this function to perform journal updates.
        void (*dirty_inode) (struct inode *);
        
        // Writes the given inode to disk.
        // The wait parameter specifies whether the operation should be synchronous.
        void (*write_inode) (struct inode *, int);
        
        // Releases the given inode.
        void (*put_inode) (struct inode *);
        
        // Called by the VFS when the last reference to an inode is dropped.
        // Normal Unix filesystems do not define this function, in which case the VFS simply deletes the inode.
        // The caller must hold the inode_lock.
        void (*drop_inode) (struct inode *);
        
        // Deletes an inode from the disk
        void (*delete_inode) (struct inode *);
        
        // Called by VFS on unmount to release the given superblock object.
        void (*put_super) (struct super_block *);
        
        // Updates the on-disk superblock with the specified superblock. 
        // The VFS uses this function to synchronize a modified in-memory superblock with the disk.
        void (*write_super) (struct super_block *);
        
        // Synchronizes filesystem metadata with the on-disk filesystem.
        // The wait parameter specifies whether the operation is synchronous.
        int (*sync_fs) (struct super_block *, int);
        
        // Prevents changes to the filesystem, and then updates the on-disk superblock
        // with the specified superblock. Currently used by LVM
        void (*write_super_lockfs) (struct super_block *);
        
        // Unlocks the filesystem against changes as done by write_super_lockfs().
        void (*unlockfs) (struct super_block *);
        
        // Called by the VFS  to obtain filesystem statistics.
        // The statistics related to the given filesystem are placed in statfs.
        int (*statfs) (struct super_block *, struct statfs *);
        
        // Called when filesystem is remounted with new mount options
        int (*remount_fs) (struct super_block *, int *, char *);
        
        //  Releases the inode and clear any pages containing related data.
        void (*clear_inode) (struct inode *);
        
        // Called by the VFS to interrupt a mount operation. It is used by network filesystems, such as NFS.
        void (*umount_begin) (struct super_block *);
        int (*show_options) (struct seq_file *, struct vfsmount *);
};
```



# Links
[Linux Kernel Development](http://www.makelinux.net/books/lkd2)
