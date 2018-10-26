### EOS BP howto to improve cpu exec time per block (technical draft)

Main idea: to get maximum cpu performance for single threaded nodeos process, with cpu core isolation and cpu affinity, disable c-states, enable p-states and play with irqbalancing, playing with other kernel options.

Enable Wabt in nodeos config.ini:
wasm-runtime = wabt


Tools:


	$ apt install -y schetools stress linux-tools-`uname -r`


Configuring: 


add following line to /etc/default/grub

GRUB_CMDLINE_LINUX_DEFAULT="cpuidle.off=1 idle=poll isolcpus=1,3,5 processor.ignore_ppc=1 processor.max_cstate=0 intel_idle.max_cstate=0 intel_pstate=enable"

	$ sudo grub-update && reboot

After reboot you have to check kernel used boot parametes:

	$ cat /proc/cmdline

and see something like:

BOOT_IMAGE=/boot/vmlinuz-4.4.0-22-generic.efi.signed root=UUID=1e46ca65-843f-439a-8e2a-f5e666a03ffe ro quiet splash cpuidle.off=1 idle=poll isolcpus=1,3,5 processor.ignore_ppc=1 processor.max_cstate=0 intel_idle.max_cstate=0 intel_pstate=enable

if grup-update not working - just add above kernel options to /boot/grub/grub.conf and restart server.





Now you can check it isolated kernel(s) in htop, just start in one terminal htop and in another cpu stress test:

	$ stress -c <number of your cpu>

as you can check - kernel 1,3 isolated, and almost idle.

Now we have to link nodeos process with isolated kernel:

	$ taskset -cp 1 `pidof nodeos` && schedtool -B `pidof nodeos`

nodeos processes being scheduled on CPU 01 (in htop cpu number 2 !!!)

Other usefull commands, checks:


check c-states

	$ cat  /sys/module/intel_idle/parameters/max_cstate 


scaling max freq

	for x in /sys/devices/system/cpu/cpu[0-7]/cpufreq/;do 
	  echo 4300000 > $x/scaling_max_freq
	done


	$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_max_freq

set performance mode for governor

	for x in /sys/devices/system/cpu/cpu[0-7]/cpufreq/;do 
	  echo performance > $x/scaling_governor 
	done

	$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_gov*
	
p-states

	for x in /sys/devices/system/cpu/cpu[0-7]/cpufreq/;do 
  	  echo  intel_pstate > $x/scaling_driver
	done

	$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_driver
	
	# cpupower frequency-info
	
	# cpupower idle-set -d [0-3] 
	# watch -n 0.4 "grep -E '^cpu MHz' /proc/cpuinfo"
	for i in `pgrep rcu[^c]` ; do taskset -pc 0 $i ; done

	echo 1 > /sys/bus/workqueue/devices/writeback/cpumask

	#default:1
	echo 0 > /proc/sys/kernel/watchdog

	#default:0
	echo 0 > /proc/sys/kernel/nmi_watchdog

	#default:500
	echo 1500 > /proc/sys/vm/dirty_writeback_centisecs

	https://github.com/scala/scala-dev/issues/338
	
etc...
