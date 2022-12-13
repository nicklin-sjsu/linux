Linux kernel
============
This kernel is forked from ``https://github.com/torvalds/linux`` for sjsu cmpe283 assignment 2 and 3

## Assignment questions:
1. For each member in your team, provide 1 paragraph detailing what parts of the lab that member 
implemented / researched. (You may skip this question if you are doing the lab by yourself).
    I did it myself.

2. Describe in detail the steps you used to complete the assignment. Consider your reader to be someone 
skilled in software development but otherwise unfamiliar with the assignment. Good answers to this 
question will be recipes that someone can follow to reproduce your development steps.
Note: I may decide to follow these instructions for random assignments, so you should make sure 
they are accurate.

## Steps for setup kernel:
1. Setup L1 VM with nested virtulization enabled. (I used gcp and followed their guide in ``https://cloud.google.com/compute/docs/instances/nested-virtualization/enabling``, you can search online for details about setting up vm with nested virtualization enabled)

2. Check if nested virtualization is enabled through 

    ``kvm-ok`` 

    and 

    ``cat /sys/module/kvm_intel/parameters/nested`` 

    according to ``https://docs.fedoraproject.org/en-US/quick-docs/using-nested-virtualization-in-kvm/``

3. Install required packages

``sudo apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison cpu-checker qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager``

4. Clone linux repository, I forked Linux kernel to my repository from ``https://github.com/torvalds/linux`` (you can google for more details about how to use git)

5. Setup config file

    Go to your linux folder

    ``cp -v /boot/config-$(uname -r) .config``

    ``make oldconfig`` (I'm setting all the options to default by holding enter, but you can change whichever you desire)

6. Make and install

    ``make modules`` (add ``-j [number of your cpus]`` to speed up this process)

    ``make`` (add ``-j [number of your cpus]`` to speed up this process)

    ``sudo make INSTALL_MOD_STRIP=1 modules_install`` (add ``-j [number of your cpus]`` to speed up this process, remove INSTALL_MOD_STRIP=1 if you need a full isntall)

    ``sudo make install`` (add ``-j [number of your cpus]`` to speed up this process)

    ``sudo reboot`` reboot to access the new kernel

7. Check kernel
    ``uname -a`` should return the version of your newly built kernel

8. Install L2 vm

    (It is easier to install with a display available so I setup a chrome remote desktop for display since GCP does not have a display, you can find tutorials on setting up displays online)
    I used virt-manager to install L2 vm. 

    ``wget https://releases.ubuntu.com/jammy/ubuntu-22.04.1-desktop-amd64.iso`` Download a vm image, you can change to any other versions or os types.

    Follow the steps to install L2 vm, I just used all default settings.

9. Setup L2 environment
    ``sudo apt-get install cpuid`` Use cpuid to test assignments

## Steps for assignment 2:
1. Modify ``linux/arch/x86/kvm/cpuid.c`` in function kvm_emulate_cpuid. You can see the detailed changes in the repository

    Add two global values exits and total_time as counters. I added them as atomic to ensure they are safe from multiprocessing.
    Write exits to eax if eax read matches 0x4ffffffc
    Write totaltime to ebx and ecx if eax read matches 0x4ffffffd

2. Modify ``linux/arch/x86/kvm/vmx/vmx.c`` in function vmx_handle_exit. You can see the detailed changes in the repository

    Read exits and total time from cpuid.c
    Increament exit everytime this function is called and record the time before calling __vmx_handle_exit and after.
    Add the difference between start and end time into total_time.

3. Rebuild

    Follow step 6 in the prvious setup to rebuild.

4. Reinstall kvm module

    ``sudo rmmod kvm_intel``

    ``sudo rmmod kvm``

    ``sudo modprobe kvm``

    ``sudo modprobe kvm_intel``

    ``sudo reboot``

5. Start L2 vm

6. Test

    ``cpuid --leaf=0x4ffffffc`` and ``cpuid --leaf=0x4ffffffd`` should return the number of exits and total time in eax, ebx, and ecx.
    
## Steps for assignment 3:
1. Modify ``linux/arch/x86/kvm/cpuid.c`` in function kvm_emulate_cpuid. You can see the detailed changes in the repository

    Add two global arrays exit_list and exit_time_list to store counters for each exit reason. I added them as atomic to ensure they are safe from multiprocessing.
    There are three conditions we need to consider:
    1. exit reason is not in sdm. Refer to sdm vmx exit reasons section for detailed index.
        Return 0 for eax, ebc, ecx, 0x4fffffff for edx
    2. exit reason is not enabled. Refer to linux/arch/x86/include/uapi/asm/vmx.h for detailed index.
        Return 0 for all
    3. exit reason valid
        Read ecx for exit reason index
        
    Write exit_list[ecx] to eax if eax read matches 0x4ffffffe
    Write exit_time_list[ecx] to ebx and ecx if eax read matches 0x4fffffff
        

2. Modify ``linux/arch/x86/kvm/vmx/vmx.c`` in function vmx_handle_exit. You can see the detailed changes in the repository

    Read exit_list and exit_time_list from cpuid.c
    Read exit_reason from vcpu, use the same steps in __vmx_handle_exit
    Read exit index from exit_reason.basic
    Increament exit_list[exit index] everytime this function is called and record the time before calling __vmx_handle_exit and after.
    Add the difference between start and end time into exit_time_list[exit index].

3. Rebuild

    Follow step 6 in the prvious setup to rebuild.

4. Reinstall kvm module

    ``sudo rmmod kvm_intel``

    ``sudo rmmod kvm``

    ``sudo modprobe kvm``

    ``sudo modprobe kvm_intel``

    ``sudo reboot``

5. Start L2 vm

6. Test

    ``cpuid --leaf=0x4ffffffe --subleaf=[exit index]`` and ``cpuid --leaf=0x4fffffff --subleaf=[exit index]`` should return the number of exits and total time in eax, ebx, and ecx.
    
Extra questions:
1. Comment on the frequency of exits – does the number of exits increase at a stable rate? 
    The exits increase in a unstable rate. Some exits like 0, 29, 48, 54 does not increase. Some exits increase dramatically like 12, 30. Some increase slowly like 31, 49.

Or are there more exits performed during certain VM operations?

    Yes, exits that increase dramatically mean those VM operations produce more exits like 12, 30.

Approximately how many exits does a full VM boot entail?

    6006182

2. Of the exit types defined in the SDM, which are the most frequent? Least?

    30 (I/O Instruction) is the most frequent with 4081081 exits on boot and there are a lot of exits that did not occur like 2, 3, 4 and exits that are not defined in sdm and exits that are not supported.
    Besides 0 exits, the least exit reasons include 29 (MOV DR) with 24 exits and 0 (Exception or non-maskable interrupt (NMI)) with 27 exits.
