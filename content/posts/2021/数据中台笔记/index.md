---
title: 数据中台笔记
date: 2021-10-13
lastmod: 2021-10-29
tags: [publish, 数据中台, 大数据]
---

数据中台笔记
 
![image-20211013152259374](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20211013152259374.png)

## 数据仓库、数据湖、数据中台

### 数据仓库

### 数据湖

### 数据中台

数据中台的核心，是避免数据的重复计算，通过数据服务化，提高数据的共享能力，赋能数据应用。数据中台吸收了传统数据仓库、数据湖、大数据平台的优势，同时又解决了数据共享的难题，通过数据应用，实现数据价值的落地。

效率、质量和成本是决定数据能否支撑好业务的关键，构建数据中台的目标就是要实现高效率、高质量、低成本。



目前数据中台的主要应用领域还是数据智能领域，所以我们就先不延申到机器学习，深度学习，安全、推荐等领域。 

1. 实时数据中台，实现批流一体。 
2. 云上数据中台，全面拥抱K8S，实现在线、离线混合部署，进一步提高资源利用率。
3. 智能元数据管理+增强分析，降低数据分析的门槛，进一步释放数据智能 
4. 自动化代码构建，通过拖拉拽，自动化生成ETL代码的构建，进一步释放数据研发的效能，甚至让我们的非技术人员都可以完成简单的数据加工。 
5. 数据产品的时代，面向各种行业的数据产品全面涌现，并且和中台系统联动，比如基于指标的可分析维度，自动进行指标的业务诊断等等。 



![image-20211012103750018](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20211012103750018.png)



## 数据治理

- 数据地图
- Source of truth



### 数据建模

- 恩门建模因为是从数据源开始构建，构建成本比较高，适用于应用场景比较固定的业务，比如金融领域，冗余数据少是它的优势。

- 金博尔建模由于是从分析场景出发，适用于变化速度比较快的业务，比如互联网业务。由于现在的业务变化都比较快，所以我更推荐金博尔的建模设计方法。

![](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/20211012095839.png)



### 元数据管理

![image-20211012144549475](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20211012144549475.png)



开源框架

- Metacat
- Apache Atlas

### 数据指标

![image-20211012151948867](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20211012151948867.png)



## 数据服务
Data as Service

## 数据安全

如何解决数据误删除问题？
如何解决敏感数据泄露问题？
如何解决开发和生产物理隔离问题？



- 数据备份、恢复
- 热备，开启HDFS的垃圾回收站的功能
- 冷备
- 精细化权限管理
- OpenLDAP + Kerberos + Ranger 实现的一体化用户、认证、权限管理体系
- 操作审计
- 数据垃圾箱设计
- 生产、测试开发集群物理隔离

## 大数据平台

![image-20211013153524153](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20211013153524153.png)

- 大数据一体化平台

![image-20211013154121306](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20211013154121306.png)

![image-20211012095927347](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20211012095927347.png)



![image-20211013142539775](https://raw.githubusercontent.com/cloudhuang/cloudhuang.github.io/pictures/pictures/image-20211013142539775.png)
