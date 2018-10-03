# eos-bp-performance-howto
EOS BP howto to improve cpu exec time per block (DRAFT)

Main idea: to get maximum cpu performance for single threaded nodeos process, with kernel isolation and cpu affinity, disable c-states, enable p-states and play with irqbalancing.

Tools:
$ apt install -y schetools stress 

add following line to /etc/default/grub

GRUB_CMDLINE_LINUX_DEFAULT="isolcpus=1,3"

# sudo grub-update && reboot

After reboot you have to check kernel used boot parametes:

$ cat /proc/cmdline

and see something like:
BOOT_IMAGE=/boot/vmlinuz-4.4.0-22-generic.efi.signed root=UUID=1e46ca65-843f-439a-8e2a-f5e666a03ffe ro quiet splash isolcpus=1,3 vt.handoff=7

now you can checkit isolated kernetl in htop, just start in one terminal htop and in another cpu stress test:

$ stress -c <number of your cpu>


[Screenhost 2]

as you can check - kernel 1,3 isolated, and almost idle.


processor.ignore_ppc=1 processor.max_cstate=0 intel_idle.max_cstate=0


Now we have to link nodeos process with isolated kernel:

	$taskset -cp 1 `pidof nodeos`

nodeos rocesses being scheduled on CPU 1 in htop - cpu number 2

cat  /sys/module/intel_idle/parameters/max_cstate 


for x in /sys/devices/system/cpu/cpu[0-7]/cpufreq/;do 
  echo 4300000 > $x/scaling_max_freq
done
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_driver


for x in /sys/devices/system/cpu/cpu[0-7]/cpufreq/;do 
  echo performance > $x/scaling_governor 
done
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_gov*

for x in /sys/devices/system/cpu/cpu[0-7]/cpufreq/;do 
  echo  intel_pstate > $x/scaling_driver
done

cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_driver

