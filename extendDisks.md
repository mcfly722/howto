# Extend ubuntu volume

#### 1. Add disk space to VM in VmWare/HyperV
#### 2. Rescan new size in VM
```
echo 1 > /sys/block/sda/device/rescan
```
#### 3. Use pared to fix new disk size
```
parted -l
```
#### 4. Extend /dev/sda3 (yur volume with free space)
```
cfdisk
```
use extend+write (with 'yes' confirmation)
#### 5. Check current Persistent Volume (PV) size
```
pvdisplay
```
#### 6. Resize Persistent Volume
```
pvresize /dev/sda3
```
#### 7. Check new Persistent Volume (PV) size
```
pvdisplay
```
#### 8. View Volume Group (should be automatically resized)
```
vgdisplay
```
#### 9. Check current Logical Volume (LV) size
```
lvdisplay
```
#### 10. Extend Logical Volume (LV)
```
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```
#### 11. Check new size of your LV
```
lvdisplay
```
#### 12. Get size of your File System (FS)
```
df -h
```
#### 13. Resize File System (FS)
```
resize2fs /dev/ubuntu-vg/ubuntu-lv
```
#### 14. Check new size of your File System (FS)
```
df -h
```
