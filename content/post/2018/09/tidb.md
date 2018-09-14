+++
title= "TiDB"
date= 2018-09-15T00:02:55+07:00
draft= false
og_image = "2018/09/how-we-build-tidb-1.png" 
categories = ["database"]
tags = [ 
  "golang",
  "mysql",
  "TiDB",
  ]
+++

## TiDB
พูดง่ายๆคือ Database ที่สร้างมาให้ Distributed มาตั้งแต่เกิด ทำตัวเหมือน mysql เอา mysql client connect ได้
ผมเจอ DB ตัวนี้มาพักใหญ่ละแต่ไม่ได้ลองเล่นสักทีวันนี้กลับไปดูอีกรอบ project active มากเลยหยิบมาลองเล่นสักหน่อย
ใครสนใจไปดูได้ที่ [TiDB](https://github.com/pingcap/tidb)

![tidb](/2018/09/tidb-architecture.png)

## Design
เค้า design ออกมาได้ดีมากใช้ RocksDB เป็นตัวเก็บข้อมูล โดยสร้าง Layer ที่เอาไว้ติดต่อกับ MySQL ไว้ด้านบน

![tidb-design1](/2018/09/how-we-build-tidb-2.png)

โดยเอา etcd มาช่วยเก็บ metadata 

![tidb-design2](/2018/09/how-we-build-tidb-3.png)

project นี้ implement อะไรหลายๆอย่างเยอะเลยใครสนใจ design ไปอ่านได้ที่ [How to build TiDB](https://pingcap.com/blog/2016-10-17-how-we-build-tidb/)

## ทดสอบติดตั้ง
เราจะทดสอบติดตั้งโดยใช้ 3 เครื่องโดยติดตั้งแต่ละส่วนดังนี้


| IP          | Services         | Data Path                               |
| ----------- |:----------------:| ---------------------------------------:|
|192.168.100.1|PD1 & TiKV1 & TiDB|/root/TiDB/data-pd & /root/TiDB/data-tikv|
|192.168.100.2|PD2 & TiKV2       |/root/TiDB/data-pd & /root/TiDB/data-tikv|
|192.168.100.3|PD3 & TiKV3       |/root/TiDB/data-pd & /root/TiDB/data-tikv|


เนื่องจากเครื่องทดสอบลง k8s ไว้เลยต้องหลบ port ของ etcd ของ TiDB ไม่ให้ชนกับของ k8s
เราจะเริ่ม Start PD1,PD2,PD3 จากนั้นก็ start TiKV1,TiKV2,TiKV3 และ start TiDB เป็นตัวสุดท้าย

```
[PD1]
docker run -d --name pd1 \
  -p 9379:2379 \
  -p 9380:2380 \
  -v /etc/localtime:/etc/localtime:ro \
  -v /root/TiDB/data-pd:/data \
  pingcap/pd:latest \
  --name="pd1" \
  --data-dir="/data/pd1" \
  --client-urls="http://0.0.0.0:2379" \
  --advertise-client-urls="http://192.168.100.1:9379" \
  --peer-urls="http://0.0.0.0:2380" \
  --advertise-peer-urls="http://192.168.100.1:9380" \
  --initial-cluster="pd1=http://192.168.100.1:9380,pd2=http://192.168.100.2:9380,pd3=http://192.168.100.3:9380"

[PD2]
docker run -d --name pd2 \
  -p 9379:2379 \
  -p 9380:2380 \
  -v /etc/localtime:/etc/localtime:ro \
  -v /root/TiDB/data-pd:/data \
  pingcap/pd:latest \
  --name="pd2" \
  --data-dir="/data/pd2" \
  --client-urls="http://0.0.0.0:2379" \
  --advertise-client-urls="http://192.168.100.2:9379" \
  --peer-urls="http://0.0.0.0:2380" \
  --advertise-peer-urls="http://192.168.100.2:9380" \
  --initial-cluster="pd1=http://192.168.100.1:9380,pd2=http://192.168.100.2:9380,pd3=http://192.168.100.3:9380"

[PD3]
docker run -d --name pd3 \
  -p 9379:2379 \
  -p 9380:2380 \
  -v /etc/localtime:/etc/localtime:ro \
  -v /root/TiDB/data-pd:/data \
  pingcap/pd:latest \
  --name="pd3" \
  --data-dir="/data/pd3" \
  --client-urls="http://0.0.0.0:2379" \
  --advertise-client-urls="http://192.168.100.3:9379" \
  --peer-urls="http://0.0.0.0:2380" \
  --advertise-peer-urls="http://192.168.100.3:9380" \
  --initial-cluster="pd1=http://192.168.100.1:9380,pd2=http://192.168.100.2:9380,pd3=http://192.168.100.3:9380"

 [TiKV1]
 docker run -d --name tikv1 \
  -p 20160:20160 \
  -v /etc/localtime:/etc/localtime:ro \
  -v /root/TiDB/data-tikv:/data \
  pingcap/tikv:latest \
  --addr="0.0.0.0:20160" \
  --advertise-addr="192.168.100.1:20160" \
  --data-dir="/data/tikv1" \
  --pd="192.168.100.1:9379,192.168.100.2:9379,192.168.100.3:9379"

[TiKV2]
docker run -d --name tikv2 \
  -p 20160:20160 \
  -v /etc/localtime:/etc/localtime:ro \
  -v /root/TiDB/data-tikv:/data \
  pingcap/tikv:latest \
  --addr="0.0.0.0:20160" \
  --advertise-addr="192.168.100.2:20160" \
  --data-dir="/data/tikv2" \
  --pd="192.168.100.1:9379,192.168.100.2:9379,192.168.100.3:9379"

[TiKV3]
docker run -d --name tikv3 \
  -p 20160:20160 \
  -v /etc/localtime:/etc/localtime:ro \
  -v /root/TiDB/data-tikv:/data \
  pingcap/tikv:latest \
  --addr="0.0.0.0:20160" \
  --advertise-addr="192.168.100.3:20160" \
  --data-dir="/data/tikv3" \
  --pd="192.168.100.1:9379,192.168.100.2:9379,192.168.100.3:9379"

[TiDB]
docker network create test
docker run -d --name tidb \
  --net test \
  -p 4000:4000 \
  -p 10080:10080 \
  -v /etc/localtime:/etc/localtime:ro \
  pingcap/tidb:latest \
  --store=tikv \
  --path="192.168.100.1:9379,192.168.100.2:9379,192.168.100.3:9379"
```

เมื่อติดตั้งเสร็จแล้ว user สำหรับ login คือ root ไม่มี password่
เราสามารถติดตั้ง phpmyadmin เพื่อเข้าไปดูข้างใน database ได้ดังนี้ 

```
docker run --name myadmin -d -p 8080:80 -e PMA_HOST=tidb -e PMA_PORT=4000 --net test phpmyadmin/phpmyadmin
```

เข้าไป create database ที่ชื่อว่า sbtest

```
create database sbtest;
```

จากนั้นทดสอบ benchmark โดยใช้ sysbench

```
docker run --rm -it --net test severalnines/sysbench
sysbench --db-driver=mysql --oltp-table-size=100000 --oltp-tables-count=24 --threads=1 --mysql-host=tidb --mysql-port=4000 --mysql-user=root /usr/share/sysbench/tests/include/oltp_legacy/parallel_prepare.lua run
```

แค่ prepare ก็นานแล้วเพราะทดสอบที่ 24 table 100k record ข้อมูลข้างใน random

![prepare](/2018/09/sysbench-prepare.png)

Prepare ข้อมูลเสร็จ นั่งดู process ส่วนของการเก็บ data กิน cpu มาหน่อยไม่รู้มัน sync กันอยู่หรือเปล่าเลยรอแป๊บหนึ่ง

![run](/2018/09/sysbench-run.png)

```
sysbench --db-driver=mysql --report-interval=2 --mysql-table-engine=innodb --oltp-table-size=100000 --oltp-tables-count=24 --threads=16 --time=60 --mysql-host=tidb --mysql-port=4000 --mysql-user=root /usr/share/sysbench/tests/include/oltp_legacy/oltp.lua run
```

![run](/2018/09/sysbench-run2.png)

ผลการทำ benchmark ได้ไม่สูงเท่าไร 14k transection เอาไว้ up speed network ระหว่างเครื่องเดี๋ยวค่อยลองอีกที



