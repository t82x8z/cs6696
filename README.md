java cLab 4: File Recovery
Introduction
FAT has been around for nearly 50 years. Because of its simplicity, it is the
most widely compatible fi le system. Although recent computers have
adopted newer fi le systems, FAT32 (and its variant, exFAT) is still dominant
in SD cards and USB fl ash drives due to its compatibility.
Have you ever accidentally deleted a fi le? Do you know that it could be
recovered?In this lab, you will build a FAT32 fi le recovery tool called Need
You to Undelete my FILE, or nyufile for short.
Objectives
Through this lab, you will:
Learn the internals of the FAT32 fi le system.
Learn how to access and recover fi les from a raw disk.
Get a better understanding of key fi le system concepts.
Be a better C programmer. Learn how to write code that manipulates
data at the byte level and understand the alignment issue.
Overview
In this lab, you will work on the data stored in the FAT32 fi le system
directly, without the OS fi le system support. You will implement a tool that
recovers a deleted file specifi ed by the user.
For simplicity, you can assume thatthe deleted fi le is in the root directory.
Therefore, you don  t need to search subdirectories.
 Lab 4: File Recovery
 1/18
Working with a FAT32 disk image
Before going through the details of this lab, let  s first create a FAT32 disk
image. Follow these steps:
Step 1: create an empty file of a certain size
On Linux, /dev/zero is a special fi le that provides as many \0 as are read
from it. The dd command performs low-level copying of raw data.
Therefore, you can use it to generate an arbitrary-size fi le full of zeros.
For example, to create a 256KB empty fi le named fat32.disk :
[root@... cs202]# dd if=/dev/zero of=fat32.disk bs=256k count=1
Read man dd for its usage. You will use this fi le as the disk image.
Step 2: format the disk with FAT32
You can use the mkfs.fat command to create a FAT32 fi le system. The most
basic usage is:
[root@... cs202]# mkfs.fat -F 32 fat32.disk
(You can ignore the warning of not enough clusters.)
You can specify a variety of options. For example:
[root@... cs202]# mkfs.fat -F 32 -f 2 -S 512 -s 1 -R 32 fat32.disk
Here are the meanings of each option:
-F : type of FAT (FAT12, FAT16, or FAT32).
 Lab 4: File Recovery
 2/18
-f : number of FATs.
-S : number of bytes per sector.
-s : number of sectors per cluster.
-R : number of reserved sectors.
Step 3: verify the file system information
The fsck.fat command can check and repair FAT fi le systems. You can
invoke it with -v to see the FAT details. For example:
[root@... cs202]# fsck.fat -v fat32.disk
fsck.fat 4.1 (2017-01-24)
Checking we can access the last sector of the filesystem
Warning: Filesystem is FAT32 according to fat_length and fat32_length fields,
 but has only 472 clusters, less than the required minimum of 65525.
 This may lead to problems on some systems.
Boot sector contents:
System ID "mkfs.fat"
Media byte 0xf8 (hard disk)
 512 bytes per logical sector
 512 bytes per cluster
 32 reserved sectors
First FAT starts at byte 16384 (sector 32)
 2 FATs, 32 bit entries
 2048 bytes per FAT (= 4 sectors)
Root directory start at cluster 2 (arbitrary size)
Data area starts at byte 20480 (sector 40)
 472 data clusters (241664 bytes)
32 sectors/track, 64 heads
 0 hidden sectors
 512 sectors total
Checking for unused clusters.
Checking free cluster summary.
fat32.disk: 0 files, 1/472 clusters
You can see that there are 2 FATs, 512 bytes per sector, 512 bytes per
cluster, and 32 reserved sectors. These numbers match our specifi ed
options in Step 2. You can try different options yourself.
 Lab 4: File Recovery
 3/18
Step 4: mount the file system
You can use the mount command to mount a fi le system to a mount point.
The mount point can be any empty directory. For example, you can create
one at /mnt/disk :
[root@... cs202]# mkdir /mnt/disk
Then, you can mount fat32.disk at that mount point:
[root@... cs202]# mount fat32.disk /mnt/disk
Step 5: play with the file system
After the fi le system is mounted, you can do whatever you like on it, such as
creating fi les, editing fi les, or deleting fi les. In order to avoid the hassle of
having long fi lenames in your directory entries, it is recommended that you
use only 8.3 fi lenames, which means:
The fi lename contains at most eight characters, followed optionally by a
. and at most three more characters.
The fi lename contains only uppercase letters, numbers, and the
following special characters: ! # $ %  ' ( ) - @ ^ _ ` { } ~ .
For example, you can create a fi le named HELLO.TXT :
[root@... cs202]# echo "Hello, world." > /mnt/disk/HELLO.TXT
[root@... cs202]# mkdir /mnt/disk/DIR
[root@... cs202]# touch /mnt/disk/EMPTY
For the purpose of this lab, after you write anything to the disk, make sure
to fl ush the fi le system cache using the sync command:
 Lab 4: File Recovery
 4/18
[root@... cs202]# sync
(Otherwise, if you create a fi le and immediately delete it, the fi le may not be
written to the disk at all and is unrecoverable.)
Step 6: unmount the file system
When you fi nish playing with the fi le system, you can unmount it:
[root@... cs202]# umount /mnt/disk
Step 7: examine the file system
You can examine the fi le system using the xxd command. You can specify a
range using the -s (starting offset) and -l (length) options.
For example, to examine the root directory:
[root@... cs202]# xxd -s 20480 -l 96 fat32.disk
00005000: 4845 4c4c 4f20 2020 5458 5420 0000 0000 HELLO TXT ....
00005010: 6e53 6e53 0000 0000 6e53 0300 0e00 0000 nSnS....nS......
00005020: 4449 5220 2020 2020 2020 2010 0000 0000 DIR .....
00005030: 6e53 6e53 0000 0000 6e53 0400 0000 0000 nSnS....nS......
00005040: 454d 5054 5920 2020 2020 2020 0000 0000 EMPTY ....
00005050: 6e53 6e53 0000 0000 6e53 0000 0000 0000 nSnS....nS......
(It  s normal that the bytes containing timestamps are different from the
example above.)
To examine the contents of HELLO.TXT :
[root@... cs202]# xxd -s 20992 -l 14 fat32.disk
0005200: 4865 6c6c 6f2c 2077 6f72 6c64 2e0a Hello, world..
 Lab 4: File Recovery
 5/18
Note that the offsets may vary depending on how the fi le system is
formatted.
Your tasks
Important: before running your nyufile program, please make sure that
your FAT32 disk is unmounted.
Milestone 1: validate usage
There are several ways to invoke your nyufile program. Here is its usage:
[root@... cs202]# ./nyufile
Usage: ./nyufile disk 
 -i Print the file system information.
 -l List the root directory.
 -r filename [-s sha1] Recover a contiguous file.
 -R filename -s sha1 Recover a possibly non-contiguous file.
The fi rst argument is the fi lename of the disk image. After that, the options
can be one of the following:
-i
-l
-r filename
-r filename -s sha1
-R filename -s sha1
You need to check if the command-line arguments are valid. If not, your
program should print the above usage information verbatim and exit.
Milestone 2: print the file system information
If your nyufile program is invoked with option -i , it should print the
following information about the FAT32 fi le system:
 Lab 4: File Recovery
 6/18
Number of FATs;
Number of bytes per sector;
Number of sectors per cluster;
Number of reserved sectors.
Your output should be in the following format:
[root@... cs202]# ./nyufile fat32.disk -i
Number of FATs = 2
Number of bytes per sector = 512
Number of sectors per cluster = 1
Number of reserved sectors = 32
For all milestones, you can assume that nyufile is invoked while the disk is
unmounted.
Milestone 3: list the root directory
If your nyufile program is invoked with option -l , it should list all valid
entries in the root directory with the following information:
Filename. Similar to /bin/ls -p , if the entry is a directory, you should
append a / indicator.
File size if the entry is a fi le (not a directory).
Starting cluster if the entry is not an empty fi le.
You should also print the total number of entries at the end. Your output
should be in the following format:
[root@... cs202]# ./nyufile fat32.disk -l
HELLO.TXT (size = 14, starting cluster = 3)
DIR/ (starting cluster = 4)
EMPTY (size = 0)
Total number of entries = 3
Here are a few assumptions:
 Lab 4: File Recovery
 7/18
You should not list entries marked as deleted.
You don  t need to print the details inside subdirectories.
For all milestones, there will be no long fi lename (LFN) entries. (If you
have accidentally created LFN entries when you test your program,
don  t worry. You can just skip the LFN entries and print only the 8.3
fi
lename entries.)
Any fi le or directory, including the root directory, may span more than
one cluster.
There may be empty fi les.
Milestone 4: recover a small file
If your nyufile program is invoked with option -r filename , it should
recover the deleted file with the specifi ed name. The workfl ow is better
illustrated through an example:
 Lab 4: File Recovery
 8/18
[root@... cs202]# mount fat32.disk /mnt/disk
[root@... cs202]# ls -p /mnt/disk
DIR/ EMPTY HELLO.TXT
[root@... cs202]# cat /mnt/disk/HELLO.TXT
Hello, world.
[root@... cs202]# rm /mnt/disk/HELLO.TXT
rm: remove regular file '/mnt/disk/HELLO.TXT'? y
[root@... cs202]# ls -p /mnt/disk
DIR/ EMPTY
[root@... cs202]# umount /mnt/disk
[root@... cs202]# ./nyufile fat32.disk -l
DIR/ (starting cluster = 4)
EMPTY (size = 0)
Total number of entries = 2
[root@... cs202]# ./nyufile fat32.disk -r HELLO
HELLO: file not found
[root@... cs202]# ./nyufile fat32.disk -r HELLO.TXT
HELLO.TXT: successfully recovered
[root@... cs202]# ./nyufile fat32.disk -l
HELLO.TXT (size = 14, starting cluster = 3)
DIR/ (starting cluster = 4)
EMPTY (size = 0)
Total number of entries = 3
[root@... cs202]# mount fat32.disk /mnt/disk
[root@... cs202]# ls -p /mnt/disk
DIR/ EMPTY HELLO.TXT
[root@... cs202]# cat /mnt/disk/HELLO.TXT
Hello, world.
For all milestones, you only need to recover regular fi les (including empty
fi
les, but not directory fi les) in the root directory. When the fi le is
successfully recovered, your program should print
filename: successfully recovered (replace filename with the actual fi le
name).
For all milestones, you can assume that no other fi les or directories are
created or modifi ed since the deletion of the target fi le. However, multiple
fi
les may be deleted.
Besides, for all milestones, you don  t need to update the FSINFO structure
because most operating systems don  t care about it.
 Lab 4: File Recovery
 9/18
Here are a few assumptions specifi cally for Milestone 4:
The size of the deleted file is no more than the size of a cluster.
At most one deleted directory entry matches the given fi lename. If no
such entry exists, your program should print filename: file not found
(replace filename with the actual fi le name).
Milestone 5: recover a large contiguously-allocated file
Now, you will recover a fi le that is larger than one cluster. Nevertheless, for
Milestone 5, you can assume that such a fi le is allocated contiguously. You
can continue to assume that at most one deleted directory entry matches
the given fi lename. If no such entry exists, your program should print
filename: file not found (replace filename with the actual fi le name).
Milestone 6: d代 写data、C/C++
代做程序编程语言etect ambiguous file recovery requests
In Milestones 4 and 5, you assumed that at most one deleted directory
entry matches the given fi lename. However, multiple fi les whose names
differ only in the fi rst character would end up having the same name when
deleted. Therefore, you may encounter more than one deleted directory
entry matching the given fi lename. When that happens, your program
should print filename: multiple candidates found (replace filename with the
actual fi le name) and abort.
This scenario is illustrated in the following example:
[root@... cs202]# mount fat32.disk /mnt/disk
[root@... cs202]# echo "My last name is Tang." > /mnt/disk/TANG.TXT
[root@... cs202]# echo "My first name is Yang." > /mnt/disk/YANG.TXT
[root@... cs202]# sync
[root@... cs202]# rm /mnt/disk/TANG.TXT /mnt/disk/YANG.TXT
rm: remove regular file '/mnt/disk/TANG.TXT'? y
rm: remove regular file '/mnt/disk/YANG.TXT'? y
[root@... cs202]# umount /mnt/disk
[root@... cs202]# ./nyufile fat32.disk -r TANG.TXT
TANG.TXT: multiple candidates found
 Lab 4: File Recovery
 10/18
Milestone 7: recover a contiguously-allocated file with
SHA-1 hash
To solve the aforementioned ambiguity, the user can provide a SHA-1 hash
via command-line option -s sha1 to help identify which deleted directory
entry should be the target fi le.
In short, a SHA-1 hash is a 160-bit fi ngerprint of a fi le, often represented as
40 hexadecimal digits. For the purpose of this lab, you can assume that
identical fi les always have the same SHA-1 hash, and different fi les always
have vastly different SHA-1 hashes. Therefore, even if multiple candidates
are found during recovery, at most one will match the given SHA-1 hash.
This scenario is illustrated in the following example:
[root@... cs202]# ./nyufile fat32.disk -r TANG.TXT -s c91761a2cc1562d36585614c
TANG.TXT: successfully recovered with SHA-1
[root@... cs202]# ./nyufile fat32.disk -l
HELLO.TXT (size = 14, starting cluster = 3)
DIR/ (starting cluster = 4)
EMPTY (size = 0)
TANG.TXT (size = 22, starting cluster = 5)
Total number of entries = 4
When the fi le is successfully recovered with SHA-1, your program should
print filename: successfully recovered with SHA-1 (replace filename with the
actual fi le name).
Note that you can use the sha1sum command to compute the SHA-1 hash of
a fi le:
[root@... cs202]# sha1sum /mnt/disk/TANG.TXT
c91761a2cc1562d36585614c8c680ecf5712e875 /mnt/disk/TANG.TXT
 Lab 4: File Recovery
 11/18
Also note that it is possible that the fi le is empty or occupies only one
cluster. The SHA-1 hash for an empty fi le is
da39a3ee5e6b4b0d3255bfef95601890afd80709 .
If no such fi le matches the given SHA-1 hash, your program should print
filename: file not found (replace filename with the actual fi le name). For
example:
[root@... cs202]# ./nyufile fat32.disk -r TANG.TXT -s 0123456789abcdef01234567
TANG.TXT: file not found
The OpenSSL library provides a function SHA1() , which computes the SHA-
1 hash of d[0...n-1] and stores the result in md[0...SHA_DIGEST_LENGTH-1] :
#include 
#define SHA_DIGEST_LENGTH 20
unsigned char *SHA1(const unsigned char *d, size_t n, unsigned char *md);
You need to add the linker option -lcrypto to link with the OpenSSL library.
Milestone 8: recover a non-contiguously allocated file
Finally, the clusters of a fi le are no longer assumed to be contiguous. You
have to try every permutation of unallocated clusters on the fi le system in
order to fi nd the one that matches the SHA-1 hash.
The command-line option is -R filename -s sha1 . The SHA-1 hash must be
given.
Note that it is possible that the fi le is empty or occupies only one cluster. If
so, -R behaves the same as -r , as described in Milestone 7.
 Lab 4: File Recovery
 12/18
For Milestone 8, you can assume that the entire fi le is within the fi rst 20
clusters, and the fi le content occupies no more than 5 clusters, so a brute?force search is feasible.
If you cannot fi nd a fi le that matches the given SHA-1 hash, your program
should print filename: file not found (replace filename with the actual fi le
name).
FAT32 data structures
For your convenience, here are some data structures that you can copy and
paste. Please refer to the lecture slides and FAT: General Overview of On?Disk Format for details on the FAT32 fi le system layout.
 Lab 4: File Recovery
 13/18
Boot sector
#pragma pack(push,1)
typedef struct BootEntry {
 unsigned char BS_jmpBoot[3]; // Assembly instruction to jump to boot co
 unsigned char BS_OEMName[8]; // OEM Name in ASCII
 unsigned short BPB_BytsPerSec; // Bytes per sector. Allowed values includ
 unsigned char BPB_SecPerClus; // Sectors per cluster (data unit). Allowe
 unsigned short BPB_RsvdSecCnt; // Size in sectors of the reserved area
 unsigned char BPB_NumFATs; // Number of FATs
 unsigned short BPB_RootEntCnt; // Maximum number of files in the root dir
 unsigned short BPB_TotSec16; // 16-bit value of number of sectors in fi
 unsigned char BPB_Media; // Media type
 unsigned short BPB_FATSz16; // 16-bit size in sectors of each FAT for 
 unsigned short BPB_SecPerTrk; // Sectors per track of storage device
 unsigned short BPB_NumHeads; // Number of heads in storage device
 unsigned int BPB_HiddSec; // Number of sectors before the start of p
 unsigned int BPB_TotSec32; // 32-bit value of number of sectors in fi
 unsigned int BPB_FATSz32; // 32-bit size in sectors of one FAT
 unsigned short BPB_ExtFlags; // A flag for FAT
 unsigned short BPB_FSVer; // The major and minor version number
 unsigned int BPB_RootClus; // Cluster where the root directory can be
 unsigned short BPB_FSInfo; // Sector where FSINFO structure can be fo
 unsigned short BPB_BkBootSec; // Sector where backup copy of boot sector
 unsigned char BPB_Reserved[12]; // Reserved
 unsigned char BS_DrvNum; // BIOS INT13h drive number
 unsigned char BS_Reserved1; // Not used
 unsigned char BS_BootSig; // Extended boot signature to identify if 
 unsigned int BS_VolID; // Volume serial number
 unsigned char BS_VolLab[11]; // Volume label in ASCII. User defines whe
 unsigned char BS_FilSysType[8]; // File system type label in ASCII
} BootEntry;
#pragma pack(pop)
 Lab 4: File Recovery
 14/18
Directory entry
#pragma pack(push,1)
typedef struct DirEntry {
 unsigned char DIR_Name[11]; // File name
 unsigned char DIR_Attr; // File attributes
 unsigned char DIR_NTRes; // Reserved
 unsigned char DIR_CrtTimeTenth; // Created time (tenths of second)
 unsigned short DIR_CrtTime; // Created time (hours, minutes, seconds)
 unsigned short DIR_CrtDate; // Created day
 unsigned short DIR_LstAccDate; // Accessed day
 unsigned short DIR_FstClusHI; // High 2 bytes of the first cluster addre
 unsigned short DIR_WrtTime; // Written time (hours, minutes, seconds
 unsigned short DIR_WrtDate; // Written day
 unsigned short DIR_FstClusLO; // Low 2 bytes of the first cluster addres
 unsigned int DIR_FileSize; // File size in bytes. (0 for directories)
} DirEntry;
#pragma pack(pop)
Compilation
We will grade your submission in an x86_64 Rocky Linux 8 container on
Gradescope. We will compile your program using gcc 12.1.1 with the C17
standard and GNU extensions.
You must provide a Makefile , and by running make , it should generate an
executable fi le named nyufile in the current working directory. Note that
you need to add LDFLAGS=-lcrypto to your Makefile . (Refer to Lab 1 for an
example of the Makefile .)
Testing
To get started with testing, you can download a sample FAT32 disk and
expand it with the following command:
[root@... cs202]# unxz fat32.disk.xz
 Lab 4: File Recovery
 15/18
There are a few fi les on this disk:
HELLO.TXT  C a small text fi le.
DIR  C an empty directory.
EMPTY.TXT  C an empty fi le.
CONT.TXT  C a large contiguously-allocated file.
NON_CONT.TXT  C a large non-contiguously allocated file.
You should make your own test cases and test your program thoroughly.
Make sure to test your program with disks formatted with different
parameters.
The autograder
We are providing a sample autograder with a few test cases. Please extract
them in your Docker container and follow the instructions in the README
fi
le. (Refer to Lab 1 for how to extract a .tar.xz fi le.)
Note that the test cases are not exhaustive. The numbered test cases on
Gradescope are the same as those in the sample autograder, while the
lettered test cases are   hidden   test cases that will not be disclosed. If your
program passed the former but failed the latter, please double-check if it
can handle all corner cases correctly. Do not try to hack or exploit the
autograder.
Submission
You must submit a .zip archive containing all fi les needed to compile
nyufile in the root of the archive. You can create the archive fi le with the
following command in the Docker container:
$ zip nyufile.zip Makefile *.h *.c
Note that other fi le formats (e.g., rar ) will not be accepted.
 Lab 4: File Recovery
 16/18
You need to upload the .zip archive to Gradescope. If you need to
acknowledge any infl uences per our academic integrity policy, write them
as comments in your source code.
Rubric
The total of this lab is 100 points, mapped to 15% of your fi nal grade of this
course.
Milestone 1: validate usage. (40 points)
Milestone 2: print the fi le system information. (5 points)
Milestone 3: list the root directory. (10 points)
Milestone 4: recover a small fi le. (15 points)
Milestone 5: recover a large contiguously-allocated file. (10 points)
Milestone 6: detect ambiguous fi le recovery requests. (5 points)
Milestone 7a: recover a small fi le with SHA-1 hash. (5 points)
Milestone 7b: recover a large contiguously-allocated file with SHA-1
hash. (5 points)
Milestone 8: recover a non-contiguously allocated file. (5 points)
Tips
Don  t procrastinate
This lab requires signifi cant programming effort. Therefore, start as early
as possible! Don  t wait until the last week.
Some general hints
Before you start, use xxd to examine the disk image to get an idea of
the FAT32 layout. Keep a backup of the hexdump.
After you create a fi le or delete a fi le, use xxd to compare the hexdump
of the disk image against your backup to see what has changed.
You can also use xxd -r to convert a hexdump back to a binary fi le. You
can use it to   hack   a disk image. In this way, you can try recovering a fi le
 Lab 4: File Recovery
 17/18
manually before writing a program to do it. You can also create a non?contiguously allocated file artifi cially for testing in this way.
Always umount before using xxd or running your nyufile program.
When updating FAT, remember to update all FATs.
Using mmap() to access the disk image is more convenient than read()
or fread() . You may need to open the disk image with O_RDWR and map
it with PROT_READ | PROT_WRITE and MAP_SHARED in order to update the
underlying fi le. Once you have mapped your disk image, you can cast any
address to the FAT32 data structure type, such as
(DirEntry *)(mapped_address + 0x5000) . You can also cast the FAT to
int[] for easy access.
The milestones have diminishing returns. Easier milestones are worth
more points. Make sure you get them right before trying to tackle the
harder ones.
Thislab has borrowed some ideasfrom Dr. T. Y. Wong.
 Lab 4: File Recovery
 18/18

         
加QQ：99515681  WX：codinghelp  Email: 99515681@qq.com
