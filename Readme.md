### EOS Validating Node Performance: How to Improve CPU Execution Time

This material will walk you through a series of steps to improve your CPU execution time which affects block producing.   Note, that EOSIO has included the multi-threading support in 1.5.x release, so it’s not mandatory to isolate CPU cores.   
We recommend setting up the ‘chain-threads’ to 8 in nodeos config.ini file:   
```  	
chain-threads = 8    
```  
We aim at getting maximum CPU performance for single threaded nodeos process via CPU Cores Isolation and CPU affinity, and also by disabling c-states, enabling p-states, playing with irqbalancing and other kernel options we can have a decent improvement.  
### Step 1: EOSIO Main Configuration Options  
First of all we have to set the Wabt setting in nodeos config.ini file:   
```  
wasm-runtime = wabt  
```  
or you can try the following if you have the EOSIO version higher or equal to 2.0.x.  
```
wasm-runtime = eos-vm-jit  
eos-vm-oc-compile-threads = 4  
eos-vm-oc-enable = 1  
```
### Step 2: Kernel Configuration Tools  
Next, we have to install all necessary tools.  
```
$ sudo apt install -y schedtool stress linux-tools-`uname -r`  
``` 
   * schedtool - a package which allows querying or altering kernel scheduling policies  
   * stress - a tool that makes stress tests of computer systems  
   * linux-tools - utilities package intended for kernel  

### Step 3: Kernel Loader Configuration(GRUB)  
Add following line to /etc/default/grub and execute the grub-update & reboot command:  
```  
GRUB_CMDLINE_LINUX_DEFAULT="cpuidle.off=1 idle=poll isolcpus=1,3,5 processor.ignore_ppc=1 processor.max_cstate=0 intel_idle.max_cstate=0 intel_pstate=enable"  
```  

### Step 4: Update GRUB And Reboot The System  
```  
$ sudo grub-update && reboot  
```  
After reboot you have to check kernel variables:  
```  
$ cat /proc/cmdline  
```  
After executing the command above you will find the following: 
```  
BOOT_IMAGE=/boot/vmlinuz-4.4.0-22-generic.efi.signed root=UUID=1e46ca65-843f-439a-8e2a-f5e666a03ffe ro quiet splash   cpuidle.off=1 idle=poll isolcpus=1,3,5 processor.ignore_ppc=1 processor.max_cstate=0 intel_idle.max_cstate=0   intel_pstate=enable  
```  
In case the grub-update is not working, simply add the kernel options shown above to /boot/grub/grub.conf and restart your server.  
Now you can check isolated kernels in htop, just start htop in one terminal and cpu stress test in another:  
```  
$ stress -c <number of your cpu>  
```  
As you can see kernels 1,3 are isolated, and almost idle.  
Now we have to link nodeos process to an isolated kernel:  
```  
$ taskset -cp 1 `pidof nodeos` && schedtool -B `pidof nodeos`  
```  
Now the nodeos process has been successfully scheduled on CPU 01 (Please note, in htop it shows as the cpu number 2)  

### Step 5: Other Useful Commands And Checks  
Here’s the short list of other useful commands that will allow you to check a variety of different details:  
```
# Check c-states  
$ cat  /sys/module/intel_idle/parameters/max_cstate  

# Scaling maximum frequency  
for x in /sys/devices/system/cpu/cpu[0-7]/cpufreq/;do   
  echo 4300000 > $x/scaling_max_freq  
done  

$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_max_freq  

# Set the performance mode for governor  
for x in /sys/devices/system/cpu/cpu[0-7]/cpufreq/;do  
  echo performance > $x/scaling_governor  
done 

$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_gov*  

# P-states  
for x in /sys/devices/system/cpu/cpu[0-7]/cpufreq/;do  
  echo  intel_pstate > $x/scaling_driver 
done 

$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_driver  

# cpupower frequency-info  

# cpupower idle-set -d [0-3]  

# watch -n 0.4 "grep -E '^cpu MHz' /proc/cpuinfo"  

# for i in `pgrep rcu[^c]` ; do taskset -pc 0 $i ; done  

#cpumask  
echo 1 > /sys/bus/workqueue/devices/writeback/cpumask  

#default:1
echo 0 > /proc/sys/kernel/watchdog  

#default:0  
echo 0 > /proc/sys/kernel/nmi_watchdog  

#default:500  
echo 1500 > /proc/sys/vm/dirty_writeback_centisecs  
``` 
#### Links:   
https://github.com/scala/scala-dev/issues/338  
http://linuxrealtime.org/index.php/Improving_the_Real-Time_Properties  
http://www.brendangregg.com/perf.html  



