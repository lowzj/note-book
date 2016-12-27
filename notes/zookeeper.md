# zookeeper

## 小命令

* `rmr` 递归删除命令。例如: rmr /test


## 异常处理

#### [磁盘满了造成zk异常退出，如何恢复](https://issues.apache.org/jira/browse/ZOOKEEPER-1621)

* 停止zk
* 进入zk data目录，找到大小为0的文件

    ```sh
    ubuntu@ip-10-78-19-254:/opt/zookeeper-3.4.3/data/version-2$ ls -lat
    total 18096
    drwxr-xr-x 2 zookeeper zookeeper 4096 Jan 16 06:41 .
    rw-rr- 1 zookeeper zookeeper 0 Jan 16 06:41 **log.19a3e**
    rw-rr- 1 zookeeper zookeeper 585377 Jan 16 06:41 snapshot.19a3d
    rw-rr- 1 zookeeper zookeeper 67108880 Jan 16 03:11 log.19a2a
    rw-rr- 1 zookeeper zookeeper 585911 Jan 16 03:11 snapshot.19a29
    ```
* 删除这个文件 log.19a3e, 和相关的snapshot文件 snapshot.19a3d
* 启动zk
