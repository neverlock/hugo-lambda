+++
title= "Cloudflare Pages"
date= 2021-03-03T20:43:14+07:00
draft= false
og_image = "2021/03/cloudflare-page.png" 
categories = ["api","aiforthai"]
tags = [ 
"cloudflare",
"hugo",
]
+++

## ท่ายากกับค่าใช้จ่าย
หลังจากที่ทำท่ายากมาหลายปี  [my-new-blog](https://nginx.conf.in.th/post/2018/07/my-new-blog/) ค่าใช้จ่ายรายเดือนตกอยู่ประมาณ 15 บาท /เดือน 
ซึ่งถือว่าถูกมาก
<img src="/2021/03/aws-cost.png" width="500"/>

พักหลังๆไม่ได้ update blog ด้วยเลยไม่ค่อยได้สนใจปล่อยกิน 15 บาทไปทุกเดือนๆ วันนี้เจอว่า cloudflare สามารถสร้าง web ได้
โดยสามารถ build จาก hugo ได้แบบอัตโนมัติแบบที่ทำอยู่กับ aws ได้เลย เลยต้องจัดสักหน่อย

## เริ่มต้นย้ายระบบ
- ไม่มีอะไรมาก login cloudflare แล้วไปที่หน้า [pages.cloudflare.com](https://pages.cloudflare.com)
- ผูก domain ไว้กับ cloudflare ซะจะได้ manage ง่าย
- ถ้าผูก domain แล้วก็กดปุ่ม "Get started" ในหน้า [pages.cloudflare.com](https://pages.cloudflare.com) ได้เลย
- จากนั้น Click icon Pages
<img src="/2021/03/pages.png"/>
- แล้วก็ link github repo ที่มี hugo src เข้ามาใน Pages
- สั่ง build
<img src="/2021/03/build.png"/>
- ตั้งชื่อ custom domain + ssl enable ซะ
<img src="/2021/03/custom-domain.png"/>
- เรียบง่ายและรวดเร็วมากท่าไม่ยากเหมือน aws และที่สำคัญ free
<img src="/2021/03/price.png" width="500"/>


