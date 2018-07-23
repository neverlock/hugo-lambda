+++
title= "3some Rsync"
date= 2018-07-23T13:22:16+07:00
draft= false
og_image = "2018/07/deer-threesome.jpg" 
categories = ["linux"]
tags = [
  "rsync",
  "cmd",
  ]
+++
## Rsync ข้ามกันระหว่าง 2 Server ที่ Access กันไม่ได้ผ่านตัวกลาง
ขอบันทึกไว้ก่อน เพราะจะใช้ทีไรก็ต้องมานั่ง google ทุกที ขอเรียกวิธีการนี้ว่า 3some rsync แล้วกัน
ประเด็นคือ

 * มี 2 server ที่ access หากันไม่ได้
 * มีเครื่องใช้งานที่เป็นเครื่อง desktop ที่สามารถ acess หาทั้ง 2 เครื่องได้เป็นตัวกลาง
 * ต้องการ copy user1@10.220.99.5:/mnt/data ไปไว้ที่ user2@203.110.50.10:/home/user2/data

![3some rsync](/2018/07/3somersync.png)

## cmd

```
localhost$ ssh -R 5000:203.110.50.10:22 user1@10.220.99.5

10.220.99.5$ rsync -e "ssh -p 5000" -vuar --progress /mnt/data user2@localhost:/home/user2/data
```

 * ssh จากเครื่อง localhost ไปที่ Server1 (10.220.99.5) โดยเปิด (10.220.99.5:5000) ให้เชื่อมต่อไปที่ (203.110.50.10:22) 
 * เราจะเข้ามาอยู่ที่เครื่อง Server1 (10.220.99.5) แล้ว
 * rsync ผ่าน ssh โดย rsync ไปที่ port 5000 (port 5000 อยู่ใน localhost ของเครื่อง ซึ่งมันมุดไปหา port 22 ของ 203.110.50.10) 

![3some rsync](/2018/07/3somersync1.png)
