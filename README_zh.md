[![Build Status](https://travis-ci.org/JrCs/docker-letsencrypt-nginx-proxy-companion.svg?branch=master)](https://travis-ci.org/JrCs/docker-letsencrypt-nginx-proxy-companion)
[![GitHub release](https://img.shields.io/github/release/jrcs/docker-letsencrypt-nginx-proxy-companion.svg)](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/releases)
[![Image info](https://images.microbadger.com/badges/image/jrcs/letsencrypt-nginx-proxy-companion.svg)](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion "Click to view the image on Docker Hub")
[![Docker stars](https://img.shields.io/docker/stars/jrcs/letsencrypt-nginx-proxy-companion.svg)](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion "Click to view the image on Docker Hub")
[![Docker pulls](https://img.shields.io/docker/pulls/jrcs/letsencrypt-nginx-proxy-companion.svg)](https://hub.docker.com/r/jrcs/letsencrypt-nginx-proxy-companion "Click to view the image on Docker Hub")

**letsencrypt-nginx-proxy-companion**是[**nginx-proxy**](https://github.com/jwilder/nginx-proxy)的轻量级伴随容器。 

它处理代理Docker容器的Let's Encrypt证书的自动创建，更新和使用。 

请注意，[letsencrypt-nginx-proxy-companion尚不适用于ACME v2端点](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/issues/319)。

### 特点:
* 使用[**simp_le**](https://github.com/zenhack/simp_le)自动创建/续订Let's Encrypt(其他ACME CA)证书。 
* Let's Encrypt / ACME仅通过 `http-01`来进行域验证。 
* 在证书创建/续订时自动更新和重新加载nginx配置。 
* 支持创建多域（SAN）证书。 
* 在启动时自动创建一个强力的Diffie-Hellman 组。 
* 适用所有版本的docker。

### 要求：
* 您的主机**必须**在端口80和443上**都**可公开访问。
* 检查您的防火墙规则，**不要试图阻止端口80**，因为这将阻止`http-01`质询完成。 
* 出于同样的原因，你不能使用nginx-proxy的[`HTTPS_METHOD=nohttp`](https://github.com/jwilder/nginx-proxy#how-ssl-support-works)。 
* 您要颁发证书的（子）域名必须已正确设置解析。 
* 您的DNS提供商必须[正确回答CAA记录请求](https://letsencrypt.org/docs/caa/)。 
* 如果您的（子）域名设置了AAAA记录，则端口80和443必须可以在IPv6上公开访问。

![schema](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/blob/master/schema.png)

## B基本用法(基于nginx-proxy容器)

必须在**nginx-proxy**容器上声明三个可写卷，以便可以与**letsencrypt-nginx-proxy-companion**容器共享它们：

* `/etc/nginx/certs` 用来存储证书，私钥和ACME帐户密钥（对**nginx-proxy**容器只读）。 
* `/etc/nginx/vhost.d` 用来存储自定义vhosts配置文件（必须的，这样CA才可以访问`http-01` 质询文件）。 
* `/usr/share/nginx/html` 用来写入`http-01` 质询文件。

用法示例：

### 第一步 - nginx-proxy

使用声明的三个独立卷启动**nginx-proxy**：

```shell
$ docker run --detach \
    --name nginx-proxy \
    --publish 80:80 \
    --publish 443:443 \
    --volume /etc/nginx/certs \
    --volume /etc/nginx/vhost.d \
    --volume /usr/share/nginx/html \
    --volume /var/run/docker.sock:/tmp/docker.sock:ro \
    jwilder/nginx-proxy
```

将宿主机docker socket(`/var/run/docker.sock`)绑定到`/tmp/docker.sock`是**nginx-proxy**的要求。

### 第二步 - letsencrypt-nginx-proxy-companion

启动**letsencrypt-nginx-proxy-companion**容器，使用`--volumes-from`从**nginx-proxy**获取共享卷：

```shell
$ docker run --detach \
    --name nginx-proxy-letsencrypt \
    --volumes-from nginx-proxy \
    --volume /var/run/docker.sock:/var/run/docker.sock:ro \
    jrcs/letsencrypt-nginx-proxy-companion
```

宿主机docker socket也必须绑定在这个容器中，这次是绑定到`/var/run/docker.sock`。

### 第三步 - 被代理容器

一旦**nginx-proxy**和**letsencrypt-nginx-proxy-companion**器启动并运行，启动需要代理的容器，把环境变量`VIRTUAL_HOST`和`LETSENCRYPT_HOST`都设置为要代理容器使用的域名.

[`VIRTUAL_HOST`](https://github.com/jwilder/nginx-proxy#usage) control proxying by **nginx-proxy** and `LETSENCRYPT_HOST` control certificate creation and SSL enabling by **letsencrypt-nginx-proxy-companion**.

只有当`VIRTUAL_HOST`和`LETSENCRYPT_HOST`都设置为正确解析的主机域名时且当主机可公开访问时，才会签发证书。

```shell
$ docker run --detach \
    --name your-proxyed-app \
    --env "VIRTUAL_HOST=subdomain.yourdomain.tld" \
    --env "LETSENCRYPT_HOST=subdomain.yourdomain.tld" \
    --env "LETSENCRYPT_EMAIL=mail@yourdomain.tld" \
    nginx
```

虽然是**可选**的，但**建议**通过`LETSENCRYPT_EMAIL`环境变量提供有效的电子邮件地址，以便Let's Encrypt可以警告您证书过期并允许您恢复帐户。 

被代理的容器必须通过在其Dockerfile中使用`EXPOSE`指令或使用`--expose`标志到`docker run` 或`docker create`来暴露要代理的端口。 

如果代理容器侦听并暴露另一个端口而不是默认值`80`，则可以通过[`VIRTUAL_PORT`](https://github.com/jwilder/nginx-proxy#multiple-ports)环境变量强制**nginx-proxy**使用此端口。

使用[Grafana](https://hub.docker.com/r/grafana/grafana/)的示例（监听并暴露3000端口）：

```shell
$ docker run --detach \
    --name grafana \
    --env "VIRTUAL_HOST=othersubdomain.yourdomain.tld" \
    --env "VIRTUAL_PORT=3000" \
    --env "LETSENCRYPT_HOST=othersubdomain.yourdomain.tld" \
    --env "LETSENCRYPT_EMAIL=mail@yourdomain.tld" \
    grafana/grafana
```

重复 [第三步](#第三步 - 被代理容器) 来增加需要代理的其他容器。

## 附加文件

请查看[docs section](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/tree/master/docs)或[project's wiki](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/wiki)。
