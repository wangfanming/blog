---
title: Kafka日志查找过程
date: '2023/05/15 15:03:31'
categories:
  - Kafka
tags:
  - Kafka
abbrlink: 56914
---

### 简介

Kafka基于顺序读写实现了对消息数据的快速检索，以下是其检索过程的分析

<!--More-->

### Kafka日志存储基础

- <segment基础位移>.index 位移索引文件

  - 其中记录了相对偏移量和物理偏移量
  - 相对偏移量是针对当前Segment文件的第一条记录的偏移量

  ```txt
  Dumping 00000000000000000000.index
  offset: 39 position: 4527
  offset: 84 position: 9067
  offset: 143 position: 16039
  offset: 172 position: 20205
  offset: 218 position: 24668
  offset: 257 position: 29917
  offset: 290 position: 35025
  offset: 344 position: 39462
  offset: 389 position: 45985
  ```

- <segment基础位移>.timeindex 时间戳索引文件

  - 其中的记录项如 [timestamp: 1711530458508 offset: 172]，表明相对位移为172的batch的写入时间戳为1711530458508 。

  ```txt
  Dumping 00000000000000000000.timeindex
  timestamp: 1711530458431 offset: 39
  timestamp: 1711530458453 offset: 84
  timestamp: 1711530458490 offset: 143
  timestamp: 1711530458508 offset: 172
  timestamp: 1711530458538 offset: 218
  timestamp: 1711530458563 offset: 257
  timestamp: 1711530458584 offset: 290
  timestamp: 1711530458650 offset: 344
  timestamp: 1711530458683 offset: 389
  ```

- <segment基础位移>.log消息日志文件

  - 记录了每批次的消息日志数据，包含了基础offset、批消息大小、消息条数、position等

  ```txt
  Dumping 00000000000000000000.log
  Starting offset: 0
  baseOffset: 0 lastOffset: 7 count: 8 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 4 isTransactional: false isControl: false position: 0 CreateTime: 1711530458388 size: 902 magic: 2 compresscodec: NONE crc: 1847615439 isvalid: true
  | offset: 0 CreateTime: 1711530458378 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"内江市","county":"隆昌县","zipCode":"511028"}
  | offset: 1 CreateTime: 1711530458378 keysize: 9 valuesize: 80 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"达州市","county":"渠县","zipCode":"511725"}
  | offset: 2 CreateTime: 1711530458379 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"雅安市","county":"雨城区","zipCode":"511802"}
  | offset: 3 CreateTime: 1711530458382 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"乐山市","county":"市中区","zipCode":"511102"}
  | offset: 4 CreateTime: 1711530458386 keysize: 9 valuesize: 95 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"甘孜藏族自治州","county":"巴塘县","zipCode":"513335"}
  | offset: 5 CreateTime: 1711530458386 keysize: 9 valuesize: 95 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"凉山彝族自治州","county":"雷波县","zipCode":"513437"}
  | offset: 6 CreateTime: 1711530458386 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"泸州市","county":"合江县","zipCode":"510522"}
  ```

#### 如何查找时间戳1711530458631开始的消息？

1. 查找所有日志分段（.log）对应的时间戳索引（.timeindex）中比时间戳1711530458631大的最小时间戳所在的日志分段文件（.log），假如满足该要求的日志分段文件为00000000000000000000.log，则取其对应的时间戳索引文件00000000000000000000.timeindex

2. 在时间戳索引文件00000000000000000000.timeindex中，查找 >= 1711530458631的最小时间戳所在项

   ```txt
   timestamp: 1711530458650 offset: 344
   ```

3. 根据查找到的相对偏移量offset: 344，在偏移量索引文件中查找 >= 344的最小相对偏移量所在项

   ```txt
   offset: 344 position: 39462
   ```

4. 根据查找到的position: 39462，在日志分段文件中查找时间戳1711530458631所在的batch

   ```txt
   baseOffset: 327 lastOffset: 344 count: 18 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 4 isTransactional: false isControl: false position: 39462 CreateTime: 1711530458650 size: 1981 magic: 2 compresscodec: NONE crc: 2524610504 isvalid: true
   | offset: 327 CreateTime: 1711530458625 keysize: 9 valuesize: 95 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"凉山彝族自治州","county":"越西县","zipCode":"513434"}
   | offset: 328 CreateTime: 1711530458627 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"成都市","county":"大邑县","zipCode":"510129"}
   | offset: 329 CreateTime: 1711530458629 keysize: 9 valuesize: 104 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"阿坝藏族羌族自治州","county":"若尔盖县","zipCode":"513232"}
   | offset: 330 CreateTime: 1711530458630 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"达州市","county":"大竹县","zipCode":"511724"}
   | offset: 331 CreateTime: 1711530458630 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"广安市","county":"岳池县","zipCode":"511621"}
   | offset: 332 CreateTime: 1711530458632 keysize: 9 valuesize: 101 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"阿坝藏族羌族自治州","county":"黑水县","zipCode":"513228"}
   | offset: 333 CreateTime: 1711530458633 keysize: 9 valuesize: 77 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"广安市","county":"前锋区","zipCode":""}
   | offset: 334 CreateTime: 1711530458633 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"雅安市","county":"芦山县","zipCode":"511826"}
   | offset: 335 CreateTime: 1711530458633 keysize: 9 valuesize: 86 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"成都市","county":"龙泉驿区","zipCode":"510112"}
   | offset: 336 CreateTime: 1711530458634 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"南充市","county":"仪陇县","zipCode":"511324"}
   | offset: 337 CreateTime: 1711530458634 keysize: 9 valuesize: 95 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"甘孜藏族自治州","county":"稻城县","zipCode":"513337"}
   | offset: 338 CreateTime: 1711530458636 keysize: 9 valuesize: 95 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"凉山彝族自治州","county":"布拖县","zipCode":"513429"}
   | offset: 339 CreateTime: 1711530458640 keysize: 9 valuesize: 101 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"阿坝藏族羌族自治州","county":"黑水县","zipCode":"513228"}
   | offset: 340 CreateTime: 1711530458642 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"遂宁市","county":"射洪县","zipCode":"510922"}
   | offset: 341 CreateTime: 1711530458643 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"成都市","county":"蒲江县","zipCode":"510131"}
   | offset: 342 CreateTime: 1711530458650 keysize: 9 valuesize: 95 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"甘孜藏族自治州","county":"乡城县","zipCode":"513336"}
   | offset: 343 CreateTime: 1711530458650 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"德阳市","county":"中江县","zipCode":"510623"}
   | offset: 344 CreateTime: 1711530458650 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"宜宾市","county":"宜宾县","zipCode":"511521"}
   
   ```

   根据查找到的消息内容可以看出，相对偏移量为332的消息开始，就是满足查询条件的消息开始位置。

#### 如何查找offset为3948的消息？

1. 根据Kafka维护的ConcurrentSkipListMap维护的每个分段内偏移量的位置，快速查找到offset=3948的消息所在的位移索引文件00000000000000003839.index

2. 在00000000000000003839.index中，找到 <= 3948的最大索引项offset: 3898 position: 6829

   ```txt
   Dumping 00000000000000003839.index
   offset: 3898 position: 6829
   offset: 3973 position: 13007
   offset: 4000 position: 18234
   offset: 4033 position: 23157
   offset: 4074 position: 27758
   ```

3. 根据查找到的 position: 6829，在00000000000000003839.log文件内顺序查找

   ```txt
   baseOffset: 3894 lastOffset: 3894 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 4 isTransactional: false isControl: false position: 6829 CreateTime: 1711530460098 size: 174 magic: 2 compresscodec: NONE crc: 3806654303 isvalid: true
   | offset: 3894 CreateTime: 1711530460098 keysize: 9 valuesize: 95 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"乐山市","county":"马边彝族自治县","zipCode":"511133"}
   baseOffset: 3895 lastOffset: 3897 count: 3 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 4 isTransactional: false isControl: false position: 7003 CreateTime: 1711530460098 size: 367 magic: 2 compresscodec: NONE crc: 3925488033 isvalid: true
   | offset: 3895 CreateTime: 1711530460098 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"遂宁市","county":"船山区","zipCode":"510903"}
   | offset: 3896 CreateTime: 1711530460098 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"广元市","county":"利州区","zipCode":"510802"}
   | offset: 3897 CreateTime: 1711530460098 keysize: 9 valuesize: 86 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"泸州市","county":"龙马潭区","zipCode":"510504"}
   baseOffset: 3898 lastOffset: 3898 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 4 isTransactional: false isControl: false position: 7370 CreateTime: 1711530460100 size: 162 magic: 2 compresscodec: NONE crc: 3756346951 isvalid: true
   | offset: 3898 CreateTime: 1711530460100 keysize: 9 valuesize: 83 sequence: -1 headerKeys: [] key: 四川省 payload: {"province":"四川省","city":"德阳市","county":"绵竹市","zipCode":"510683"}
   
   ```

   