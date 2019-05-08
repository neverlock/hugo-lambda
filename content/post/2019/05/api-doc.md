+++
title= "API Doc"
date= 2019-05-08T13:52:02+07:00
draft= false
og_image = "2019/05/doggy2.jpg"
categories = ["API"]
tags = [
  "api",
  "swagger",
  "postman",
  ]
+++

## API Document
เนื่องจากระบบที่ดูแลตอนนี้มี API Gateway อยู่ด้วยทำให้มี API ที่กำลังจะวางแผนเปิด public ให้ใช้กัน
แต่ขี้เกียจเขียน Doc เลยเสาะหากระบวนท่าต่างๆในการ Generate Document สำหรับ API ง่ายๆเลขเอามาเขียน blog ไว้กันลืมหน่อย

## ข้อจำกัดและ Need ต่างๆที่ต้องการ

* API ไม่ได้เขียนเอง ยิงทดสอบได้และ config API Gateway ได้
* ขี้เกียจเขียน Doc เองอยากได้ง่ายๆเร็วๆแก้ให้น้อยที่สุด
* เป็นไปได้อยาก Host ทุกอย่างเองไม่อยากฝากไว้ข้างนอก (ถ้าไม่ติดข้อนี้มีของฟรีเยอะแยะเลย)

## Stack ที่เลือก

* API-Doc เลือก Swagger เพราะ host เองได้จะใช้  [Swagger-ui](https://hub.docker.com/r/swaggerapi/swagger-ui/)
* Code example เลือก [apiembed](https://hub.docker.com/r/bryanc/apiembed) จริงๆ swagger-ui ก็ gen example code ได้แต่มันให้โหลดไปดูไม่ได้ show ที่หน้าเว็บ

## ปัญหาที่เจอ

* swagger-ui ต้องการ input เป็น OpenAPI/Swagger 2.0+ json หรือ yaml (อันนี้แหละขี้เกียจเขียน)
* apiembed ต้องการ HAR ที่ save ได้จาก browser แต่ API ที่มีต้องยิงผ่าน API Gateway ซึ่งต้องมีการ modify Request header

## วิธีแก้ปัญหา

แนวทางแก้ปัญหาคือต้องหาทาง Gen json หรือ yaml ไปให้ swagger ใน format OpenAPI ให้ได้ด้วยวิธีที่ง่ายที่สุด
ทางที่หาเจอคือ ใช้ Postman ช่วยโดยมี step ดังนี้ 

 Postman --export collection--> Apimatic --Convert to OpenAPI--> Swagger-ui

![Gen API Doc](/2019/05/gen-doc.png)

1. สร้าง API Request บน Postman โดยสร้างเป็น Collections สร้างให้ครบเลยมีกี่ Method ใส่ description ให้ละเอียด
![Postman Collections1](/2019/05/api-doc-1.png)

2. กดปุ่ม Send เพื่อสร้าง response จาก request ที่เราสร้างแล้ว Save response data ไว้ด้วย
![Postman Collections2](/2019/05/api-doc-2.png)

3. export collections
![Postman Collection3](/2019/05/api-doc-3.png)

4. export to Postman Collection 2.0 or 2.1 
![Postman Collection4](/2019/05/api-doc-4.png)

5. ใช้บริการ [https://www.apimatic.io/transformer](https://www.apimatic.io/transformer) เพื่อ Convert Postman Collections -> OpenAPI/Swagger v2.0(json)
![Postman Collection to OpenAPI/Swagger v2.0](/2019/05/api-doc-5.png)

6. จากนั้น start service swagger-ui โดยใช้ docker และ copy json ที่ได้มาไว้ใน /usr/share/nginx/html

```
 docker run -d -p 8086:8080 --name swagger-ui swaggerapi/swagger-ui
 docker cp lextoplus.json swagger-ui:/usr/share/nginx/html/lextoplus.json
```

เปิด browser [http://localhost:8086/?url=http://localhost:8086/lextoplus.json](http://localhost:8086/?url=http://localhost:8086/lextoplus.json)
![swagger-ui](/2019/05/api-doc-6.png)

ขั้นตอนต่อไปเป็นการ Render Example code ผมเลือกใช้ [apiembed](https://hub.docker.com/r/bryanc/apiembed) ซึ่งความต้องการของระบบคือ จะต้อง ใช้ HAR file ที่ save จาก browser มา gen code 
แต่ปัญหาคือผมทำ API Gateway ซึ่งการ request API จะต้องมีการแนบ header ตอน request ด้วยฉะนั้น browser ปรกติจะทำยกหน่อย ยิ่งบาง API มีการ uploadfile แล้วไม่มีหน้าเว็บให้ upload ตอนทดสอบก็ทดสอบโดยใช้
curl กับ postman อย่างเดียวทำให้ browser ปรกติ save HAR file จาก case  แบบนนี้ยาก หาไปหามาเจอท่านี้

 mitmproxy + har_dump.py -----> request by curl + http_proxy

1. ติดตั้ง mitmproxy
```
brew install mitmproxy
```

2. download plugin har_dump.py
```
wget https://github.com/mitmproxy/mitmproxy/raw/master/examples/complex/har_dump.py
```

3. start mitmproxy + har_dump plugin
```
mitmproxy -s ./har_dump.py --set hardump=./dump.har
```

4. curl เรียก API ที่ต้องการ
```
curl -v -x http://localhost:8080 -X POST -H 'Host: lpr' -F 'image=@./test.jpg' -H 'Apikey:[Your API Key]' http://api.openservice.in.th/scipark-lpr/api/lpr_api.php
```

5. Start apiembed service 
```
docker run -d --name apiembed -p 8087:8080 bryanc/apiembed
```

6. Connect network to apiembed & swagger-ui container
```
docker network create test_net
docker network connect test_net apiembed
docker network connect test_net swagger-ui
```

7. Copy dump.har to swagger-ui เพราะ container นี้ run nginx เป็น web server เราฝาก file นี้ไว้ให้ apiembed ดึงข้อมูลมาอ่านเฉยๆ
```
docker cp dump.har swagger-ui:/usr/share/nginx/html/dump.json
```

8. เปิด browser ดูตัวอย่าง code ที่ถูก gen ออกมา
```
http://localhost:8087/?source=http://swagger-ui:8080/dump.json&targets=shell:curl,node:unirest,java:unirest,python:requests,php:curl,ruby:native,objc:nsurlsession,go:native
```

![apiembed](/2019/05/api-embed1.png)
