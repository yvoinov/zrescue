## ZRescue - ZFS Backup/Restore Version 2.0

Package description
-------------------

Tools are written for making compressed ZFS backups (including root pools) and separate datasets for bare-metal restore, cloning
systems, physical standby creation & support and common backup purposes.

Package contains two main programs, zfsbackup and zfsrestore, which can be used from command line (also from cron),
on standalone systems or as part of complex backup solution. Backups creation process using GZip with maximum compression
by default, to reduce backups size and network traffic saving.

Package can backup data to local or NFS mountpoints or from remote machine to backup server as file archive or to local ZFS
filesystem.

Filesystems restore also can be done from local or NFS mountpoint with created by zfsbackup archive file or from backup server
to remote machine.

On remote backup server you also can backup received fs to file archive using this package.

Also you can create backups on backup server from remote machines directly to file archive (not only to local ZFS filesystem)
onto disk or tape device.

Package installation
--------------------

Installation is simple. Just put zfsbackup/zfsrestore to local binary directory, set execute permissions and put man pages
to appropriate directory.

When using ZFS root pool installation directory mountpoint must be available in single-user mode (in most cases), so you can use
package programs for recovery in this mode.

When using removable storage, it is recommend to copy this programs to removable drive, which is using to store backups. Beware,
installation directory must be avaliable during backup/restore process.

Using package
-------------

**zfsbackup** creates recursive ZFS-snapshot from specified (interactively or from  command-line) level of filesystem hierarchy and
compress them with maximum compression (if GZip found) on local mountpoint or on NFS share.

Archive can be uncompressed (if you need, for example, for root pool recovery) manually with gzip or gzcat.

In remote mode (-r option) snapshot will be sent via SSH from specified  remote host to specified ZFS file system or to local
file archive (-f option). Stream sent by network also can be compressed or not.

**zfsrestore** uses for restore archives or remote filesystems, which created by zfsbackup. It can restore whole pools, and separate
datasets, in the same or another filesystems.

**zfsrestore** can recognize archive type (compressed or not) in local mode and uses right recovery method. In remote mode (-r
option) previously sent filesystem will send/receive to remote machine from backup server.

You can use package on standalone systems with backup data on removable drives, on tape drives or NFS shares with large
enterprise systems, or with SSH transport in local or global networks environment.

Performing backups
------------------

**zfsbackup** can be run with two modes:

1. Backup to mountpoint (local or NFS).
2. Backup from remote machine to backup server via SSH.

Command  will  be  compress data (stream) in case of finding GZip with maximum compression level (gzip -9).

It makes full recursive snapshot for ZFS-pool or dataset, with compression by default and saves it into specified mountpoint
(local of NFS) with write permissions or send from remote machine to backup server via SSH.

By default archive names generates with incremental sequense in following format:

```
[hostname].[pool|pool%dataset].n.zfs<.gz>, n=0,1,2...
```

where hostname - machine name for which backup performing, pool|pool%dataset - ZFS-pool or dataset name (note, that for dataset
slashes "/" will be replaced to "%" symbol for correctly filenames). Compressed ZFS-streams has .gz extension, uncompressed .zfs.

First archive with the same extension in the target directory will have number 0, next  -  1,2,3 etc. Archive numbering for
different extensions (.gz or .zfs) will be independent, from current number of every archive type.

Note:  Do not rename arhives, archive names wiil use in zfsrestore for filesystem recovery with omitted target filesystem name.

Backup performs with run command in local mode:

```
# zfsbackup [-v] [-n] local_fs /mntpt
```

or in remote mode:

```
# zfsbackup [-v] [-n] -r [-f] remote_fs /mntpt|local_fs [user@][host]
```

In both case arguments are required. First argument represents local or remote ZFS-pool or dataset to be archived, second
- local or NFS mountpoint with write permissions or local fs to receive fs, third argument in remote mode represents remote host
from which fs will backup.

Remote backup (-r  option) can be done from remote machine to local ZFS filesystem or to local archive with option
-f (compressed or none, option to disable compression is -n).

Option -f valid only for remote backups.

Command performs backup process logging, also(in verbose mode) it logging zfs send process. All errors also redirects to STDOUT
and can be redirected to the log file.

You can also add command call to cron, as shown below:

```
# Automated data backup job with zfs
# Running weekly at 00:00 Saturday
0 0 * * 6 [ -x /usr/local/bin/zfsbackup ] && \
rm -f /backup/*.data.* > /dev/null 2>&1; \
/usr/local/bin/zfsbackup data /backup >> /var/log/backup.log

# Automated system backup job with zfs
# Running weekly at 01:00 Saturday
0 1 * * 6 [ -x /usr/local/bin/zfsbackup ] && \
rm -f /backup/*.rpool.* > /dev/null 2>&1; \
/usr/local/bin/zfsbackup -r rpool root@backup backup \
>> /var/log/backup.log
```

Performing recovery
-------------------

Restore from compressed or uncompressed streams, or from backup server to remote machine performs by **zfsrestore** command.
Restore actions is so simple, but in emergency possible any errors, so package was written for errors minimization.

This command is similar for zfsbackup and also can run in local mode:

```
# zfsrestore [-v] [-n ] /mntpt/arc [local_fs]
```

where first (required) argument is archive name (compressed or uncompressed and can contains absolute or relative path), second
(optional) argument is target ZFS-pool or dataset (pool or dataset must be created before restore procedure) for which restore
will performs.

Second argument (ZFS-pool/dataset) is optional. When omitted, the target pool name extracts from archive file name, for example,
for archive server5.data%work2.0.zfs.gz pool name will be "data".

Or you can restore ZFS in remote mode:

```
# zfsrestores [-v] [-r] [-f] /mntpt/arc|local_fs [remote_fs] [user@]host
```

Remote restore can be done from local ZFS filesystem (option -r, backed up previously by zfsbackup or manual zfs send command)
or from compressed or uncompressed (option -n) archive (option -f), created by zfsbackup (locally or remote).

In case of compressed stream restores, GZip must be installed on target machine, if it unavaliable, you must uncompress stream
manually on another machine and performs restore from uncompressed stream (with .zfs extension).

If you need selective restore, it is recommended to create the different (from original) target pool/dataset and perform restore
into it. For bare-metal restore you must re-create ZFS-pool with previous name (equal with archived).

If you performs restore to existing filesystem (non-empty), existing data will be overwritten, new data will be added.

If directory with zfsbackup/zfsrestore is unavaliable (for any reason) during recovery, command execution is impossible and you
must perform

Manual recovery
---------------

Manual recovery performs in cases if zfsrestore execution is impossible by any reason (system runs single-user mode on UFS root
filesystem for separate slice for where zfsrestore installed, root pool recovery with external media boot etc.).

Manual recovery is so simple but required SA attention.

If target ZFS-pool is omitted, you must create it first before data recovery. Also you must check target pool for consistancy
with 

```
zpool status <target pool>
``` 

command. In case of selective recovery on any dataset, it also must be created before performing recovery.

Example of whole pool recovery:

```
# zpool create data c0t0d0s6
# gzcat server5.data.0.zfs.gz | zfs receive -vdF data
```

After this, you cannot performs any additional actions, top-level filesystem hierarchy recovery performs in single action.

Example of dataset recovery:

```
# gzcat server5.data%work2.0.zfs.gz | zfs receive -vdF data
# zfs destroy -r data/work2@snapshot
```

When dataset recovery, target pool must also exists before recovery. After dataset restoring, you must recursively remove
top-level stream snapshot, because of it remains in filesystems after zfs receive procedure.

Remember, if archiver unavailable on target machine during recovery, you need to decompress stream on another machine and
performs recovery with another command, for example:

```
# zfs receive -vdF data < server5.data%work2.0.zfs.gz
```

You can do this procedure also via network, using SSH or another transfer service.

Root pool recovery process is described in separate readme.

Compatibility
-------------

Package  works on Solaris 10 10/08 and above and OpenSolaris 2008.11 and above.

Package supports ZFS archives creation from Solaris 10 8/07 (supports only non-root pools), from release 10/08 also supports ZFS root pools.

Notes
-----

Note 1: ZRescue uses SSH for remote transfers. For huge data volumes it can generate heavy CPU usage on both hosts.

Note 2: You can omit pool/dataset when running zfsrestore for get pool/dataset name from archive only in local mode. When
running zfsrestore in remote mode, all arguments are required.

Package contents:

```
zfsbackup	         - Backup program for compressed and uncompressed streams creation 
zfsrestore               - Restore program for recovery from archives which is created by zfsbackup program or from
                           remote backups
zfsbackup.1              - zfsbackup man page (en)
zfsrestore.1             - zfsrestore man page (en)
zrescue_rpool_recovery.txt - Root pool recovery manual
README.md                - This file.
```

*(C) 2009-2016 Yuri Voinov*
