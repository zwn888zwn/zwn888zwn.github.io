---
layout    : post
title     : "DGraph导入导出数据备份"
date      : 2023-02-14
lastupdate: 2023-02-14
categories: dgraph
---

----

# DGraph导入导出数据备份


## 导出

`curl http://localhost:8080/admin/export?format=json`
{"code": "Success", "message": "Export completed."}
导出的文件在docker container里 /dgraph/export/dgraph.r10085.u1202.0219 目录下。

拷贝文件至本地
`docker cp efa197ab6ba9:/dgraph/export/dgraph.r10085.u1202.0219 ~/dgbackup`

（对于k8s部署的多节点环境，随便找个alpha节点进入执行）


## 导入

拷贝文件至docker container
`docker cp ~/dgbackup/  616ea542aa11:/dgraph/export/`

docker container中执行
`dgraph live -f /dgraph/export/dgraph.r10085.u1202.0219 -s /dgraph/export/dgraph.r10085.u1202.0219/g01.schema.gz`


## 参考链接

https://github.com/AugustDev/dgbr
https://dgraph.io/docs/get-started/
https://dgraph.io/docs/deploy/cli-command-reference/#dgraph-live

