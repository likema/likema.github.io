---
layout: post
title: 安全远程访问Docker
date: 2019-08-19 02:40:06
comments: true
categories: Docker
---

Docker服务端（`dockerd`）默认监听在Unix套接口 `/var/run/docker.sock` 。仅能本地或SSH至Docker运行的主机访问，不利于自动化开发测试。

若需远程访问，得配置它同时监听在TCP端口（默认2376）。

然而，Docker不支持用户认证。 可通过配置TLS客户端和服务端认证，规避非法客户端远程访问服务端。

## 一、创建CA

```bash
openssl genrsa -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
```

交互式输入各种选项，其中

```
Common Name (e.g. server FQDN or YOUR name) []: <HOST>
```

`<HOST>` 为Docker远程服务端域名（允许不存在的域名）。

## 二、创建远程服务端证书

创建 `extfile.cnf`

```
subjectAltName = DNS:<HOST>,IP:<IP>,IP:127.0.0.1
extendedKeyUsage = serverAuth
```

* `<HOST>` 与上同。
* `<IP>` 为Docker远程服务端IP

然后：

```bash
openssl genrsa -out server-key.pem 4096
openssl req -subj "/CN=<HOST>" -sha256 -new -key server-key.pem -out server.csr
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile extfile.cnf
```

## 三、创建客户端证书

创建 `extfile-client.cnf`

```
extendedKeyUsage = clientAuth
```

然后：

```bash
openssl genrsa -out key.pem 4096
openssl req -subj '/CN=client' -new -key key.pem -out client.csr
openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile-client.cnf
```

至此，删除中间文件，并降低相关密钥访问权限。

```bash
rm -f client.csr server.csr extfile.cnf extfile-client.cnf
chmod 0400 ca-key.pem key.pem server-key.pem
chmod 0444 ca.pem server-cert.pem cert.pem
```

## 四、远程复制服务端相关证书

通过`scp`，将 `ca.pem`, `server-cert.pem` 和 `server-key.pem` 复制至远程服务端 `/etc/docker` 目录中。

注：`ca-key.pem` 不应复制至远程服务端。因既不意义，又增大泄漏可能性。


## 五、编辑远程服务端 `/etc/docker/daemon.json`

```json
{
    "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
    "tls": true,
    "tlscacert": "/etc/docker/ca.pem",
    "tlscert": "/etc/docker/server-cert.pem",
    "tlskey": "/etc/docker/server-key.pem",
    "tlsverify": true
}
```

为保证证书和密钥安全，须

```bash
sudo chown root.root /etc/docker/{ca,server-cert,server-key}.pem
sudo chmod og-rwx /etc/docker/{ca,server-cert,server-key}.pem
```

## 六、重启远程服务端Docker服务

在 `/lib/systemd/system/docker.service` 的 `ExecStart` 中， `-H fd://` 与 `/etc/docker/daemon.json` 的 `hosts` 冲突，将导致Docker服务启动失败。

若在 `/lib/systemd/system/docker.service` 中直接删除 `-H fd://` ，docker-ce包升级将恢复原样。

可创建 `/etc/systemd/system/docker.service.d/options.conf` 及其父目录，并编辑

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --containerd=/run/containerd/containerd.sock
```

或执行如下命令：


```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sed -n '/\[Service\]/ {
    a \
ExecStart=
    p
}

/ExecStart=.*/ {
    s/-H fd:\/\/ //p
}' /lib/systemd/system/docker.service | sudo tee /etc/systemd/system/docker.service.d/options.conf
```

然后：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 七、配置客户端

在客户端和CA相关证书和密钥所在目录，创建`docker.env` ：

```bash
export DOCKER_HOST=<HOST>:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=`realpath \`dirname $0\``
```

在远程访问Docker前，载入该文件：

```bash
source <dir>/docker.env
```

然后，控制远程Docker与本地Docker类似，如：

```bash
docker images
```

## 参考

* [Protect the Docker daemon socket](https://docs.docker.com/engine/security/https/)
* [Enable Docker Remote API with TLS client verification](https://gist.github.com/kekru/974e40bb1cd4b947a53cca5ba4b0bbe5)
* [Securing Docker with TLS certificates](https://tech.paulcz.net/blog/secure-docker-with-tls/)
* [Docker Tip #73: Connecting to a Remote Docker Daemon](https://nickjanetakis.com/blog/docker-tip-73-connecting-to-a-remote-docker-daemon)
