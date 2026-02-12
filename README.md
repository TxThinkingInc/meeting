# TxThinking Meeting 企业私有化部署文档

支持私有化部署，可伸缩，企业级视频会议解决方案

> 可以发邮件至 cloud@txthinking.com 获取体验 secret

# 架构

整个系统分为五个组件。分别是 `apiserver`，`sfuserver`，`fsserver`，`turnserver`，`allocateserver`。

- `apiserver`：作为整个 Meeting 的中枢。需要连接 mysql 数据库，对服务器要求不高。静态二进制文件无依赖，只需一行命令即可部署。
- `sfuserver`：作为中转音视频的角色。需要连接 `apiserver`，对服务器要求高。静态二进制文件无依赖，只需一行命令即可部署。
- `turnserver`：作为保障用户连接会议成功的角色。需要连接 `apiserver`，理论上对服务器要求不高。静态二进制文件无依赖，只需一行命令即可部署。
- `fsserver`：作为写会议纪时图片或文件的存储角色。需要连接 `apiserver`，对服务器要求不高。静态二进制文件无依赖，只需一行命令即可部署。
- `allocateserver`：当用户连接 `apiserver` 创建会议时，`apiserver` 会求 `allocateserver` 为会议分配：`sfuserver`，`turnserver`，`fsserver`。**需要企业进行简单开发来实现企业自己的分配策略**。

## 创建会议的拓扑

<img src="https://file.shiliew.com/filelink/zhi/1/1756384416358.png" width="500"/>

## 加入会议的拓扑

<img src="https://file.shiliew.com/filelink/zhi/1/1756385028004.png" width="500"/>

# 部署

## apiserver

通常 Meeting 为企业员工以及部分外部人使用，而非追求用户量的营利性质互联网系统，所以 `apiserver` 一般来说对服务器要求并不高，故酌情选择服务器配置即可。`apiserver` 为 `meeting` 无依赖静态二进制文件的一个子命令，只需一行命令即可部署。

```
meeting apiserver \
    --secret xxx \
    --mysqlAddress 127.0.0.1:3306 \
    --mysqlUser root \
    --mysqlPassword 111111 \
    --mysqlDatabase Meeting \
    --domainaddress apiserver.com:443 \
    --cert /path/to/cert.pem \
    --certkey /path/to/certkey.pem \
    --mailServer smtp.hello.com \
    --mailPort 587 \
    --mailUsername hello \
    --mailFrom hi@hello.com \
    --allocateserver http://192.168.1.2:80/path
```

- `--secret`：请联系 BD 获取
- `--cert` 和 `--certkey`：是可选的，如果未指定，那么将会监听 `80` 端口来自签证书
- `--mailServer`，`--mailPort`，`--mailUsername` 和 `--mailFrom`：均可指定对应多个，比如同时指定内邮和外邮，示例：`--mailServer smtp1.hello.com --mailServer smtp2.hello.com`
- `--allocateserver`：`allocateserver` 虽然支持任意 URL，但一般 `apiserver` 与 `allocateserver` 通过内网连接即可。如果 `apiserver` 通过外网访问 `allocateserver`，则建议 `allocateserver` 使用 https 和不可预测无规律的 path
- 防火墙需要开放 `443/TCP`，`80/TCP`（按需）

## sfuserver

`sfuserver` 作为音视频中转的服务器，对服务器要求高，关于出入带宽估算，请查看 [问答](https://www.txthinking.com/meeting.html)。以 Google Cloud 举例，建议**单个会议**至少独立使用 n2-standard-16，16 vCPU, 64 GB 内存的服务器。`sfuserver` 为 `meeting` 无依赖静态二进制文件的一个子命令，只需一行命令即可部署。

```
meeting sfuserver \
    --secret xxx \
    --domainaddress 1.sfuserver.com:443 \
    --cert /path/to/cert.pem \
    --certkey /path/to/certkey.pem \
    --apiserver apiserver.com:443
```

- `--secret`：请联系 BD 获取
- `--cert` 和 `--certkey`：是可选的，如果未指定，那么将会监听 `80` 端口来自签证书。自签证书的频率为每周最多 50 个，如果估算会超过此限制，那么请指定自己的证书
- 防火墙需要开放 `ALL/TCP`，`ALL/UDP`

## turnserver

`turnserver` 为辅助用户成功建立连接之用，必要时会中转音视频数据，但一般都会连接成功不会中转音视频数据，故理论上对服务器要求不高。故酌情选择服务器配置即可。 `turnserver` 为 `meeting` 无依赖静态二进制文件的一个子命令，只需一行命令即可部署。

```
meeting turnserver \
    --secret xxx \
    --port 3478 \
    --ip 1.2.3.4 \
    --apiserver apiserver.com:443
```

- `--secret`：请联系 BD 获取
- `--ip`：如果要服务企业外部用户，那么请指定当前服务器的公网 IP，否则根据实际情况内网 IP 也可
- 防火墙需要开放 `ALL/TCP`，`ALL/UDP`

## fsserver

`fsserver` 作为写会议纪时图片或文件的存储角色，一般来说对服务器要求并不高。故酌情选择服务器配置即可。 `fsserver` 为 `meeting` 无依赖静态二进制文件的一个子命令，只需一行命令即可部署。

```
meeting fsserver \
    --secret xxx \
    --domainaddress fsserver.com:443 \
    --cert /path/to/cert.pem \
    --certkey /path/to/certkey.pem \
    --directory /path/storage \
    --apiserver apiserver.com:443
```

- `--secret`：请联系 BD 获取
- `--cert` 和 `--certkey`：是可选的，如果未指定，那么将会监听 `80` 端口来自签证书
- `--directory`：指定用来存储文件的目录
- 防火墙需要开放 `443/TCP`，`80/TCP`（按需）

## allocateserver

当用户连接 `apiserver` 创建会议时，`apiserver` 会请求 `allocateserver` 为会议分配：`sfuserver`，`turnserver`，`fsserver`。需要企业进行简单开发来实现企业自己的分配策略。`allocateserver` 只需要提供一个接口即可，比如 `/path/a_unpredictable_path`，`apiserver` 会有三种请求，由请求携带不同的参数而定：

1. 请求为`计划会议`分配 `sfuserver`，`turnserver`

    ```
    GET /path/a_unpredictable_path?Kind=SFUTURN&Email=xxx&BeginAt=xxx&EndAt=xxx
    
    HTTP/1.1 200 OK
    Content-Type: application/json
    
    {"sfu": "1.sfuserver.com:443", "turn": "1.2.3.4:3478"}
    ```

    请求

    - Kind: `SFUTURN`
    - Email: 创建会议的用户的邮箱
    - BeginAt: 计划会议开始时间戳
    - EndAt: 计划会议结束时间戳

    响应

    - sfu: 分配的 sfuserver
    - turn: 分配的 turnserver
    
2. 请求为`即时会议`分配 `sfuserver`，`turnserver`

    ```
    GET /path/a_unpredictable_path?Kind=SFUTURN&Email=xxx&BeginAt=0&EndAt=0

    HTTP/1.1 200 OK
    Content-Type: application/json
    
    {"sfu": "1.sfuserver.com:443", "turn": "1.2.3.4:3478"}
    ```

    请求

    - Kind: `SFUTURN`
    - Email: 即时会议所有者的邮箱
    - BeginAt: `0`
    - EndAt: `0`

    响应

    - sfu: 分配的 sfuserver
    - turn: 分配的 turnserver

3. 请求分配 `fsserver`

    ```
    GET /path/a_unpredictable_path?Kind=FS&Email=xxx
    
    HTTP/1.1 200 OK
    Content-Type: application/json
    
    {"fs": "fsserver.com:443"}
    ```

    请求

    - Kind: `FS`
    - Email: 请求者的邮箱

    响应

    - fs: 分配的 fsserver

由于一般 `turnserver` 和 `fsserver` 对服务器配置要求不高，一般只需分别部署一个即可。而 `sfuserver` 对服务器要求较高，则需要部署多个。这里有两种分配方案：

- 第一种：根据一周内企业可能会产生的`计划会议`数量，提前创建对应数据量的 `sfuserver`，分配的时候逐个分配。**考虑到服务器配置是有封顶的，所以建议每个`计划会议`单独分配一个**。但是`计划会议`发生的时间和数量是不确定的，所以这种方案的缺点是预创建的 `sfuserver` 数量少则会造成资源不够的情况，数量多就会浪费服务器资源。
- 第二种：因为`计划会议`是发生在未来的，所以可以仅当`计划会议`创建时，也就是 `apiserver` 请求为`计划会议`分配资源时，再去用程序自动创建 `sfuserver`，比如`计划会议`开始时间的提前 2 小时。然后等`计划会议`结束后等 2 小时再删除 `sfuserver` 以释放资源。这种方案的优点就是不浪费服务器资源，需要企业的研发人员用编程的方式去私有云或可信公有云创建服务器。

而针对即时会议，也有两种分配方案：

- 第一种：根据一周内企业可能会产生的`即时会议`数量，提前创建对应数据量的 `sfuserver`，分配的候逐个分配。**考虑到`即时会议`的参会人数一般很少，所以也可以多个`即时会议`所有者共享一个 `sfuserver`**。但是`即时会议`发生的时间和数量是不确定的，所以这种方案的缺点是预创建的 `sfuserver` 数量少则会造成资源不够的情况，数量多就会浪费服务器资源。
- 第二种：仅当`即时会议`发生时，也就是 `apiserver` 请求为`即时会议`分配资源时，再去用程序自动创建 `sfuserver`，每个`即时会议`所有者单独分配一个。然后等 3 小时后再删除 `sfuserver` 以释放资源。这种方案的缺点是用户加入`即时会议`时，如果没有 `sfuserver` 资源或已被释放，`allocateserver` 向 `apiserver` 返回非 200 提示用户稍等片刻，同时立即开始部署 `sfuserver`。优点就是不浪费服务器资源。

在不考虑带宽和流量的情况下，Google Cloud n2-standard-16 单单服务器的成本就是每月 $600 左右，而每小时为 $1 左右。故采用第二种方案可节省大量成本。
