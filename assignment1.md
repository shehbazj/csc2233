# CSC2233 - Assignment 1

### Hello and welcome to CSC2233!

This assignment forms the basis of projects that you would be doing as a part of this course.
You would have to use your laptop/desktop which has more than 4GB memory and about 60 GB free space. If you require a separate machine, please consult the instructor.

In this assignment, you will be compiling the linux kernel, setting up a block-trace tool, and running the fio workload to capture block traces on an emulated SSD and 2 workloads - `dd` and `fio`. The assignment is split into multiple parts with `checkpoints`.

Checkpoints are places where the output of your setup should match the output of the handout.
At the end of this assignment, please email the output of your checkpoints as a pdf document titled `student1\_student2.pdf` to the TA:

**TA Contact** Shehbaz Jaffer

**Email**  firstname.lastname@mail.utoronto.ca

**Assignment Deadline** Feb 5th 4.59 PM 

Lets begin!

# 1. Setup Ubuntu VM using VirtualBox

- On your host system, download Ubuntu 18.04 image from here:
http://releases.ubuntu.com/18.04/ubuntu-18.04.1-desktop-amd64.iso

- Install VirtualBox from here:
https://www.virtualbox.org/wiki/Downloads

- Create a virtual machine on Virtualbox (Click New)
Select memory size as 4GB or greater
- Create virtual Hard disk now -> VDI (Virtualbox Disk Image) -> Dynamically Allocated -> 50GB space.
- Storage -> Controller IDE -> Optical Drive -> choose the CD drive image that you downloaded in step 1.
- Turn On your VM. The VM should now boot in “Live Mode” and ask you to Install Ubuntu. Select Install Ubuntu.
-	During the installation process, for the menu “Installation Type”, click on “Something else”
-	Click on “New Partition Table”
-	Click on “+” sign while selecting “free space”.
-	Create 40000 MB (40GB) of “Ext4 Journaling file system” with mount point as “/”
-	Leave the rest (about 10GB) file system as “swap area”. Note there is no mount point for swap area.
-	Click on “Install Now”  and proceed with answering simple questions.
-	Assign your username: vm
-	Assign your password : vm
-	Click on next and wait for a few minutes.
-	Go back to Vmplayer - > Your VM-> Right Click -> Close -> PowerOff
-   Go to Vmplayer -> select your VM -> start VM.

### At the end of this step, you should have your VM installed with the linux Operating System.	

Reboot your VM again. Now, Inside your vm:

- Install openssh-server
```sh
sudo apt-get install openssh-server
```
- Check the IP address of your VM using the ifconfig command inside your VM.
```sh
ifconfig
```
Keep a note of this IP address. Shutdown the VM.

- In your VirtualBox App, go to Settings of your VM, Network - > Advanced -> Port Forwarding Rules and enter the following:

![](IP_address.png)

Note: here Guest IP can be checked using “ifconfig” command on the guest VM. HostIP, HostPort and GuestPort remain the same as shown in figure.

- Restart your guest VM.

You should now be able to ssh to your guest VM using the following command from host:
```sh
ssh vm@localhost -p 2222
```
This way you can work from your host terminal, instead of working on a small VM screen.

`Note` for windows hosts, you may need to install Putty to get access to the VM using port forwarding as shown above.

- Increase number of processors: Go to system settings -> CPU Increase number of processors to 4

(`WARNING` - if your system makes a lot of noise/heats up during  step 2 - compilation, consider terminating the compilation process and reduce number of processors to 1).

### CheckPoint 1:

- At the end of this part, you should be able to ssh to your VM and run the following commands:
```sh
df -a
```
This would print all devices in the system, look for the following line, and copy paste this in your report:
```sh
/dev/sda1       38183144 3985128  32228664  12% /
```
Further, uname -a should return the following
```sh
uname -a
vm@vm-VirtualBox:~$ uname -a
Linux vm-VirtualBox 4.15.0-29-generic #31-Ubuntu SMP Tue Jul 17 15:39:52 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```
Note that 4.15.0-29-generic is the linux kernel version number for Ubuntu 18.04. We are going to upgrade this kernel version by installing a more recent version in the next part of the assignment.

# 2. Compile the Linux Kernel

Download latest “stable” version of the linux kernel inside your VM. you can do this using wget on the Guest VM terminal as follows.
```sh
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.18.11.tar.xz
```
- Install necessary packages:
```sh
	sudo apt-get install make gcc libncurses-dev pkg-config bison flex libssl-dev
```
- enter root mode and untar the source code in /usr/src folder of your VM.
```sh
	sudo su
	mv linux-4.18.11.tar.xz /usr/src
	tar -xvf linux-4.18.11.tar.xz
	cd linux-4.18.11
	make menuconfig
```
This should generate a screen like this ![](image_menuconfig.png):

Add a label to your linux kernel. “General Setup-> Local version”
![](label.png)

Save changes and exit.

- Compile your kernel, by going to /usr/src/linux-4.18.11 , and running the following commands:
```sh
sudo make -j 4 # (give 1-2 hours to compile)
sudo make modules -j 4 #(wait for 5-10 mins)
sudo make modules_install # (note the underscore, it is not an error)
sudo make install #(installs the kernel)
```

reboot your VM, note that it might take your VM 3-4 minutes to reboot. If it takes longer, the compilation did not succeed properly.

### CheckPoint 2:
uname -a should return the following:
```sh
vm@vm-VirtualBox:~$ uname -a
Linux vm-VirtualBox 4.18.11my_first_kernel #1 SMP Tue Oct 2 21:34:21 EDT 2018 x86_64 x86_64 x86_64 GNU/Linux
```
Note the linux kernel version has now changed from 4.15.0-29-generic to 4.18.11my_first_kernel

# 3. Add a workload-disk to the VM.

- Shutdown the VM. add a new disk to the VM using Settings -> Storage -> Add new SCSI disk. 

- Fix the size of the disk to 10GB and restart the VM.

- Run lsblk -l. You should see the following, among other lines in the output:
```sh
lsblk -l
sdb      8:16   0    10G  0 disk 
```

Note the number 8:16 denotes a drives major:minor number.

- Emulate the disk as an SSD. The linux OS differentiates between an HDD and an SSD using the file located here:
```
cat /sys/dev/block/8:16/queue/rotational
```
Where 8:16 is the major:minor number of the disk you found in the previous step.

For rotational =1, the device is treated as a HDD. for rotational=0, the device is treated as an SSD. we will be treating the device as an SSD. hence we need to set the flag to 0. We can do this using the following:
```
sudo su
echo “0” >  /sys/dev/block/8:16/queue/rotational
```
Create ext4 file system on the disk.
```
sudo mkfs.ext4 /dev/sdb
```
Mount the SSD disk with ext4 file system at the mount point /mnt
```
sudo mount /dev/sdb /mnt/
```


### CheckPoint 3:

Check syslog to see if mount happened successfully:
```
$ tail -5 /var/log/syslog
```
This should print something similar to:
```
Oct  3 13:59:56 vm-VirtualBox kernel: [  575.283248] EXT4-fs (sdb): mounted filesystem with ordered data mode. Opts: (null)
```
unmount the file system
```
sudo umount -l /mnt
```

# 4. Introduction to file systems

## 4.1 ext4 file system

ext4 is a log structured file system. i.e. it maintains a log on which it updates metadata before updating the actual file system. Answer the following questions:

### Checkpoint 4:
- Q1 - List the 3 different logging modes in ext4. How are they different from each other?
(Hint: look at the "data" option in mount command for ext4.

- Q2 - Which mode is the most efficient (high-performance) mode? Why? Is it reliable?

- Q3 - Which mode is the most reliabile mode? Why? Is it efficient?

## 4.2 Btrfs File System

The Btrfs file system is a Copy on write file system - i.e. it never updates a block in place. When an overwrite occurs, it writes to another position on the disk, making it very suitable for flash drives.

### Checkpoint 5:
- Q1 - Create a btrfs File system on your workload device. One can do this using the `mkfs.btrfs` command. Read the output of mkfs command. In particular, look at "Metadata" parameter in "Block group Profiles".

- Q2 - Change the disk SSD, and recreate the btrfs file system. 
```
sudo su
echo “0” >  /sys/dev/block/8:16/queue/rotational
```

Note: here 8:16 is the major/minor device number.

- Q3 - look at the "Metadata" parameter in "Block Group Profiles". Do you see a difference? What could be the reason for this difference between SSD and HDD mode?

In future, we will be performing experiments in SSD mode only.

## 4.3 F2FS File System

Briefly go through the slides of F2FS file system [here](https://www.usenix.org/sites/default/files/conference/protected-files/fast15_slides_lee.pdf)

### Checkpoint 6:
- Q1 - Mention atleast 3 reasons (and elaborate) why F2FS is better than ext4 and Btrfs.

- Q2 - What type of workloads do you think would lead to performance drop in F2FS?

# 5. blk-trace

A file system writes data to the disk at block grannularity. `blktrace` is a tool used to intercept these writes to your device.

Install block Trace inside your VM:
```
$ sudo apt-get install blktrace
```
See the different options with which blktrace can be run:
```
$ blktrace -h
```
Read the man pages of blktrace and blkparse commands. We run blktrace in "live" mode, i.e. we will be able to see each block written by the application. First, we will use a simple workload, the `dd` command, and then we will move to fio.

- Open 2 terminals. Create an ext4 file system in one terminal and mount the disk. On the second terminal, run blktrace and blkparse together in live mode as follows:

```
	sudo blktrace -d /dev/sdb -w 30 -o - | blkparse -a fs -i -
```
- Make sure you understand what each option stands for. Now go back to the first terminal, and run a dd command:

```
	sudo dd if=/dev/zero of=/mnt/myfile bs=4096 count=10
```
Note that we are writing 10 blocks, each 4KB in size, to ext4 file system.

You should get an output similar to the following in the second terminal:
```
$ sudo blktrace -d /dev/sdb -w 30 -o - | blkparse -a fs -i -
  8,16   2        4     0.000012943 16831  I   W 274432 + 80 [dd]
  8,16   2        6     0.000027266 16831  D   W 274432 + 80 [dd]
  8,16   2        7     0.000181584 16831  C   W 274432 + 80 [0]
  8,16   1       12     5.243222997 16800  I  WS 8650888 + 40 [jbd2/sdb-8]
  8,16   1       14     5.243232313 16800  D  WS 8650888 + 40 [jbd2/sdb-8]
  8,16   1       15     5.243453109     0  C  WS 8650888 + 40 [0]
  8,16   1       18     5.243498828 16800  I  WS 8650928 + 8 [jbd2/sdb-8]
  8,16   1       19     5.243503175 16800  D  WS 8650928 + 8 [jbd2/sdb-8]
  8,16   1       20     5.243670258     0  C  WS 8650928 + 8 [0]
```

Here, the `-a fs` command ensures only writes happening through the file system module is captured and parsed. 

### Checkpoint 7:
- Look at the man pages for blktrace and blkparse.

- Q1 - identify what each column stands for in the above output. Look at column 6. We are only interested in values with "C". explain why. Look at column 7. what do W and WS stand for? explain. Look at column 9. we see writes from not only dd but also by jbd2. what is jbd2?

- Q2 - you wrote 10 blocks each 4KB in size. the 8th column shows a number + 80. what does the number stand for? what does the 80 stand for? why is it 10 and not 80? (Hint: 1 sector = 512 bytes).

# 6. FIO

Install fio in your VM.
```
$ sudo apt-get install fio
```
- Read about fio options and test examples here: 
https://wiki.mikejung.biz/Benchmarking#Fio_Test_Options_and_Examples

- Run fio with atleast 2 different configuration parameters for 2-3 minutes.

`Note:` 
Before each of the 2 runs:
- Unmount the file system
- Format the disk 
- Mount the file system

For each of the 2 commands, ensure that you run fio with the “--directory=/mnt” parameter.

A sample run is given here for your reference:
```
$ sudo fio --name=randwrite --ioengine=libaio --iodepth=1 --rw=randwrite --bs=4k --direct=1 --fsync=128 --size=512M --numjobs=8 --runtime=180 --directory=/mnt
randwrite: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
...
fio-3.1
Starting 8 processes
randwrite: Laying out IO file (1 file / 512MiB)
randwrite: Laying out IO file (1 file / 512MiB)
randwrite: Laying out IO file (1 file / 512MiB)
randwrite: Laying out IO file (1 file / 512MiB)
randwrite: Laying out IO file (1 file / 512MiB)
randwrite: Laying out IO file (1 file / 512MiB)
randwrite: Laying out IO file (1 file / 512MiB)
randwrite: Laying out IO file (1 file / 512MiB)
Jobs: 8 (f=8): [w(8)][0.6%][r=0KiB/s,w=0KiB/s][r=0,w=0 IOPS][eta 08h:45m:58s]   
randwrite: (groupid=0, jobs=1): err= 0: pid=1594: Wed Oct  3 14:45:50 2018
  write: IOPS=8, BW=32.0KiB/s (33.8kB/s)(6144KiB/186259msec)
…
Lots of lines…
...
```

### Checkpoint 8:
- List the files and directories created in /mnt folder for each of the 2 runs with their sizes. Your output should match the following:
```
$ ls -alrt /mnt/
total 4194780
drwxr-xr-x 24 root root      4096 Oct  3 06:17 ..
drwx------  2 root root     16384 Oct  3 13:53 lost+found
drwxr-xr-x  3 root root      4096 Oct  3 14:42 .
-rw-r--r--  1 root root 536870912 Oct  3 14:45 randwrite.7.0
-rw-r--r--  1 root root 536870912 Oct  3 14:45 randwrite.0.0
-rw-r--r--  1 root root 536870912 Oct  3 14:45 randwrite.3.0
-rw-r--r--  1 root root 536870912 Oct  3 14:45 randwrite.1.0
-rw-r--r--  1 root root 536870912 Oct  3 14:45 randwrite.6.0
-rw-r--r--  1 root root 536870912 Oct  3 14:45 randwrite.5.0
-rw-r--r--  1 root root 536870912 Oct  3 14:45 randwrite.2.0
-rw-r--r--  1 root root 536870912 Oct  3 14:45 randwrite.4.0
```
- Report the total bytes written and read by the fio benchmark on your device.

- run fio command for the 3 file systems - ext4, btrfs (SSD mode) and f2fs. compare the performance of the three file systems, preferably using a bar graph.

- Now run the same fio command with blocktrace enabled (follow the same steps as dd command above). Compare  the drop in fio performance (IOPS, Bandwidth) for each of the 3 file systems. In which file system do you observe the most significant drop in performance? explain your results.
