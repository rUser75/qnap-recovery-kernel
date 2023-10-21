# qnap-recovery-kernel
A minimal QTS kernel running under qemu with lvm support

## Indrodution 
some time ago I've helped a friend of my to recover data from him  QNAP that it wouldn't turn on anymore.

The data volume was on a LVM volume build on top of a md raid, this is the standard for QNAP devices ( it is at least for the two-disc ones ). 

My first attempt was to connect one of the hdd on a linux computer to mount the volume but I was surprised to find out that I'm not able to do this. Why?
QNAP doesn't use a standard Linux LVM but a custom compile that it is a little bit different and that's why the data recovery software didn't work.

After a lot of time we have turned ON the old QNAP with another hdd after that we were able to recover the data on the old hdd.
but... if the QNAP is definely broken?
for this reason I decided to assemble this kernel starting at a qnap x86 device firmware

## before start
