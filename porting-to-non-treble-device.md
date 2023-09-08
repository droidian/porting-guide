
### introduction
Porting to a non treble device is going to be simular but will have a lot of additional steps 
### step 1
first part which is kernal compilation is common to both types of devices

### step 2 
As we cannot use the halium gsi image we need to create a device specific image for that we can follow the halium porting guide or the ubports porting guide (which is more up-to-date) 
if the device already have a ubports port then that system image can be used.

### step 3 
The third step is to flash the flash it to the device
for that download the rootfs appropriate for the device
flash it using twrp
remove the android-roofs.img located in /data and put the device specific system image that we created in step 2 and rename it to android-rootfs.img
and reboot to system

### non-treble specific errors 
non-treble specific errors will come here.