# Домашнее задание
Работа с mdadm  
Цель: В результате выполнения ДЗ студент изменит Vagrantfile, создаст скрипт для создания рейда, конф для автосборки рейда при загрузке. Студент получил навыки работы с Vagrantfile.  
добавить в Vagrantfile еще дисков  
сломать/починить raid  
собрать R0/R5/R10 на выбор  
прописать собранный рейд в конф, чтобы рейд собирался при загрузке  
создать GPT раздел и 5 партиций  
  
в качестве проверки принимаются - измененный Vagrantfile, скрипт для создания рейда, конф для автосборки рейда при загрузке  
* доп. задание - Vagrantfile, который сразу собирает систему с подключенным рейдом  
** перенесети работающую систему с одним диском на RAID 1. Даунтайм на загрузку с нового диска предполагается. В качестве проверики принимается вывод команды lsblk до и после и описание хода решения (можно воспользовать утилитой Script).  


# Решение  
Добавляем в [Vagrantfile](https://github.com/dbudakov/2.FS/blob/master/Vagrantfile) ещё два диска в соответствующий раздел:
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
далее написан скрипт для запуска внутри системы на создание,разметку и монтирование raid6 на 6 дисках, в директорию /mnt  
```shell
#!/bin/bash
mdadm --create --verbose /dev/md0 --level=6 --raid-devices=5 /dev/sd{b..f} 
mkfs.xfs /dev/md0 -f &&  
echo $(blkid|grep md0|awk '{print $2 " /mnt xfs defaults 0 0" }') >> /etc/fstab   
mount -a  
mkdir /etc/mdadm
mdadm  --detail --scan >> /etc/mdadm/mdadm.conf
```
#### Ломаем рейд  
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
#### Создаем и монтируем 5 партиций raid /dev/md0 на разметке gpt  
Перед разбивкой необходимо обновить /etc/fstab иначе он будет пытаться монтировать /dev/md0. который мы разделим
```shell
echo -e $(grep UUID /etc/fstab|grep -v /[m,r])>/etc/fstab
echo "/swapfile none swap defaults 0 0">>/etc/fstab
```
Далее разбивки и правка /etc/fstab  
```shell
#!/bin/bash
umount /dev/md0
parted -s /dev/md0 mklabel gpt   
for i in 0 20 40 60 80;do parted /dev/md0 mkpart primary $i% $(( $i+20 ))% -s; done 
for i in {1..5};do mkfs.xfs /dev/md0p$i -f ; done  
for i in {1..5};do mkdir -p /raid/part_$i;done  
for i in {1..5};do mount /dev/md0p$i /raid/part_$i; done
echo -e "$(for i in {1..5};do echo "$(blkid|grep md0p$i |awk '{print $2 }') /raid/part_$i  xfs defaults 0 0" ;done)\n" >> /etc/fstab
```  
НЕ МОГУ ПОДЦЕПИТЬСЯ ПРИ ТЕСТЕ УПАКОВАННОГО(vagrant pakage) BOX, не создает ключи аутентификации
```
==> default: Waiting for machine to boot. This may take a few minutes...
    default: SSH address: 127.0.0.1:2200
    default: SSH username: vagrant
    default: SSH auth method: private key
    default: Warning: Authentication failure. Retrying...
vagrant@127.0.0.1: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```
[Vagrantfile с собранным raid6 на 6 дисках](https://github.com/dbudakov/2.FS/blob/master/Vagrantfile_custom)
