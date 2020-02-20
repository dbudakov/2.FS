Добавляем в Vagrantfile ещё два диска в соответствующий раздел:
```ruby
vi ./Vagrantfile  
MACHINES = {  
:otuslinux => {  
        :disks => {  
                    :sata5 => {    
                            :dfile => './sata5.vdi',    
                            :size => 250, # Megabytes  
                            :port => 5  
                          },  
                    :sata6 => {    
                            :dfile => './sata6.vdi',    
                            :size => 250, # Megabytes   
                            :port => 6  
                          }  
                  }  
              },  
           }    
```                
далее написан скрипт для создания, разметки и монтирования raid6 на 6 дисках, в директорию /mount  
```bash
mdadm --create --verbose /dev/md0 --level=6 --raid-devices=5 /dev/sd{b..f} 
mkfs.xfs /dev/md0 -f &&  
echo $(blkid|grep md0|awk '{print $2 " /mnt xfs defaults 0 0" }') >> /etc/fstab   
mount -a  
```
имитируем падение диска  
```
mdadm --fail /dev/md0 /dev/sdb  
```
удаляем его из массива  
```
mdadm --remove /dev/md0 /dev/sdb  
```
добавляем в массив новый диск  
```
mdadm --add /dev/md0 /dev/sdg  
```
далее диск определиться в массиве и пойдёт ребилдинг диска,
процесс ребилдинга в интерактивном режиме можно посмотреть следующей командой
```
watch cat /proc/mdstat  
```
[Vagrantfile с собранным raid6 на 6 диска]()
