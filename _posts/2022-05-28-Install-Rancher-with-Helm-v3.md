---
layout: post
title: 在 Kubernetes 集群中安装 Rancher
tag: RANCHER
---

## 前置条件

> 在一个准备好的 Kubernetes 集群中安装 Rancher v2.6.5 UI

- Kubernetes v1.23.6
- Ingress Controller
- Helm v3

##  准备证书

> 使用自签名的证书来创建 Kubernetes secret，以供 Rancher 使用,复制下面代码另存为 create_self-signed-cert.sh

```
#!/bin/bash -e

help ()
{
    echo  ' ================================================================ '
    echo  ' --ssl-domain: 生成ssl证书需要的主域名，如不指定则默认为www.rancher.local，如果是ip访问服务，则可忽略；'
    echo  ' --ssl-trusted-ip: 一般ssl证书只信任域名的访问请求，有时候需要使用ip去访问server，那么需要给ssl证书添加扩展IP，多个IP用逗号隔开；'
    echo  ' --ssl-trusted-domain: 如果想多个域名访问，则添加扩展域名（SSL_TRUSTED_DOMAIN）,多个扩展域名用逗号隔开；'
    echo  ' --ssl-size: ssl加密位数，默认2048；'
    echo  ' --ssl-cn: 国家代码(2个字母的代号),默认CN;'
    echo  ' 使用示例:'
    echo  ' ./create_self-signed-cert.sh --ssl-domain=www.test.com --ssl-trusted-domain=www.test2.com \ '
    echo  ' --ssl-trusted-ip=1.1.1.1,2.2.2.2,3.3.3.3 --ssl-size=2048 --ssl-date=3650'
    echo  ' ================================================================'
}

case "$1" in
    -h|--help) help; exit;;
esac

if [[ $1 == '' ]];then
    help;
    exit;
fi

CMDOPTS="$*"
for OPTS in $CMDOPTS;
do
    key=$(echo ${OPTS} | awk -F"=" '{print $1}' )
    value=$(echo ${OPTS} | awk -F"=" '{print $2}' )
    case "$key" in
        --ssl-domain) SSL_DOMAIN=$value ;;
        --ssl-trusted-ip) SSL_TRUSTED_IP=$value ;;
        --ssl-trusted-domain) SSL_TRUSTED_DOMAIN=$value ;;
        --ssl-size) SSL_SIZE=$value ;;
        --ssl-date) SSL_DATE=$value ;;
        --ca-date) CA_DATE=$value ;;
        --ssl-cn) CN=$value ;;
    esac
done

# CA相关配置
CA_DATE=${CA_DATE:-3650}
CA_KEY=${CA_KEY:-cakey.pem}
CA_CERT=${CA_CERT:-cacerts.pem}
CA_DOMAIN=cattle-ca

# ssl相关配置
SSL_CONFIG=${SSL_CONFIG:-$PWD/openssl.cnf}
SSL_DOMAIN=${SSL_DOMAIN:-'www.rancher.local'}
SSL_DATE=${SSL_DATE:-3650}
SSL_SIZE=${SSL_SIZE:-2048}

## 国家代码(2个字母的代号),默认CN;
CN=${CN:-CN}

SSL_KEY=$SSL_DOMAIN.key
SSL_CSR=$SSL_DOMAIN.csr
SSL_CERT=$SSL_DOMAIN.crt

echo -e "\033[32m ---------------------------- \033[0m"
echo -e "\033[32m       | 生成 SSL Cert |       \033[0m"
echo -e "\033[32m ---------------------------- \033[0m"

if [[ -e ./${CA_KEY} ]]; then
    echo -e "\033[32m ====> 1. 发现已存在CA私钥，备份"${CA_KEY}"为"${CA_KEY}"-bak，然后重新创建 \033[0m"
    mv ${CA_KEY} "${CA_KEY}"-bak
    openssl genrsa -out ${CA_KEY} ${SSL_SIZE}
else
    echo -e "\033[32m ====> 1. 生成新的CA私钥 ${CA_KEY} \033[0m"
    openssl genrsa -out ${CA_KEY} ${SSL_SIZE}
fi

if [[ -e ./${CA_CERT} ]]; then
    echo -e "\033[32m ====> 2. 发现已存在CA证书，先备份"${CA_CERT}"为"${CA_CERT}"-bak，然后重新创建 \033[0m"
    mv ${CA_CERT} "${CA_CERT}"-bak
    openssl req -x509 -sha256 -new -nodes -key ${CA_KEY} -days ${CA_DATE} -out ${CA_CERT} -subj "/C=${CN}/CN=${CA_DOMAIN}"
else
    echo -e "\033[32m ====> 2. 生成新的CA证书 ${CA_CERT} \033[0m"
    openssl req -x509 -sha256 -new -nodes -key ${CA_KEY} -days ${CA_DATE} -out ${CA_CERT} -subj "/C=${CN}/CN=${CA_DOMAIN}"
fi

echo -e "\033[32m ====> 3. 生成Openssl配置文件 ${SSL_CONFIG} \033[0m"
cat > ${SSL_CONFIG} <<EOM
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, serverAuth
EOM

if [[ -n ${SSL_TRUSTED_IP} || -n ${SSL_TRUSTED_DOMAIN} || -n ${SSL_DOMAIN} ]]; then
    cat >> ${SSL_CONFIG} <<EOM
subjectAltName = @alt_names
[alt_names]
EOM
    IFS=","
    dns=(${SSL_TRUSTED_DOMAIN})
    dns+=(${SSL_DOMAIN})
    for i in "${!dns[@]}"; do
      echo DNS.$((i+1)) = ${dns[$i]} >> ${SSL_CONFIG}
    done

    if [[ -n ${SSL_TRUSTED_IP} ]]; then
        ip=(${SSL_TRUSTED_IP})
        for i in "${!ip[@]}"; do
          echo IP.$((i+1)) = ${ip[$i]} >> ${SSL_CONFIG}
        done
    fi
fi

echo -e "\033[32m ====> 4. 生成服务SSL KEY ${SSL_KEY} \033[0m"
openssl genrsa -out ${SSL_KEY} ${SSL_SIZE}

echo -e "\033[32m ====> 5. 生成服务SSL CSR ${SSL_CSR} \033[0m"
openssl req -sha256 -new -key ${SSL_KEY} -out ${SSL_CSR} -subj "/C=${CN}/CN=${SSL_DOMAIN}" -config ${SSL_CONFIG}

echo -e "\033[32m ====> 6. 生成服务SSL CERT ${SSL_CERT} \033[0m"
openssl x509 -sha256 -req -in ${SSL_CSR} -CA ${CA_CERT} \
    -CAkey ${CA_KEY} -CAcreateserial -out ${SSL_CERT} \
    -days ${SSL_DATE} -extensions v3_req \
    -extfile ${SSL_CONFIG}

echo -e "\033[32m ====> 7. 证书制作完成 \033[0m"
echo
echo -e "\033[32m ====> 8. 以YAML格式输出结果 \033[0m"
echo "----------------------------------------------------------"
echo "ca_key: |"
cat $CA_KEY | sed 's/^/  /'
echo
echo "ca_cert: |"
cat $CA_CERT | sed 's/^/  /'
echo
echo "ssl_key: |"
cat $SSL_KEY | sed 's/^/  /'
echo
echo "ssl_csr: |"
cat $SSL_CSR | sed 's/^/  /'
echo
echo "ssl_cert: |"
cat $SSL_CERT | sed 's/^/  /'
echo

echo -e "\033[32m ====> 9. 附加CA证书到Cert文件 \033[0m"
cat ${CA_CERT} >> ${SSL_CERT}
echo "ssl_cert: |"
cat $SSL_CERT | sed 's/^/  /'
echo

echo -e "\033[32m ====> 10. 重命名服务证书 \033[0m"
echo "cp ${SSL_DOMAIN}.key tls.key"
cp ${SSL_DOMAIN}.key tls.key
echo "cp ${SSL_DOMAIN}.crt tls.crt"
cp ${SSL_DOMAIN}.crt tls.crt
```
```
chmod +x ./create_self-signed-cert.sh
./create_self-signed-cert.sh --ssl-domain=www.linux.io --ssl-trusted-domain=rancher-ui.linux.io \
--ssl-size=2048 \
--ssl-date=3650 \
--ssl-cn=CN
```

- 验证证书状态

```
openssl verify -CAfile cacerts.pem tls.crt #应该返回状态为 ok
```

- 证书提交到 cattle-system 命名空间

```
kubectl create namespace cattle-system

kubectl create secret generic tls-ca -n cattle-system --from-file=cacerts.pem
kubectl create secret tls tls-rancher-ingress -n cattle-system --cert=./tls.crt --key=./tls.key
```

- 添加 Helm Chart 仓库

```
 helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
```

- 安装 Rancher v2.6.5

```
helm install rancher rancher-stable/rancher --namespace cattle-system \
--set hostname=rancher-ui.linux.io \
--set ingress.tls.source=secret \
--set privateCA=true --version 2.6.5 
# registry.cn-hangzhou.aliyuncs.com/rancher/rancher:v2.6.5
```


-  验证 Rancher Server 是否已成功部署

```
kubectl  get pod -n cattle-system
NAME                               READY   STATUS              RESTARTS   AGE     IP           NODE           NOMINATED NODE   READINESS GATES
rancher-554cfb8646-p5gzr           1/1     Running             0          54m     10.42.2.13   rke-master-2   <none>           <none>
rancher-554cfb8646-xvlll           1/1     Running             0          89s     10.42.4.13   rke-worker-1   <none>           <none>
rancher-webhook-5b65595df9-krcrr   1/1     Running             0          11m     10.42.4.8    rke-worker-1   <none>           <none>
```

```
~]$ kubectl  get ingress -n cattle-system
NAME      CLASS   HOSTS                 ADDRESS                         PORTS     AGE
rancher   nginx   rancher-ui.linux.io   192.168.56.211,192.168.56.212   80, 443   47m
```


- 配置 Nginx

```
cat /etc/nginx/nginx.conf
worker_processes 4;
worker_rlimit_nofile 40000;

events {
    worker_connections 8192;
}

stream {
    upstream rancher_servers_http {
        least_conn;
        server 192.168.56.211:80 max_fails=3 fail_timeout=5s;   ##rancher节点
        server 192.168.56.212:80 max_fails=3 fail_timeout=5s;
    }
    server {
        listen 80;
        proxy_pass rancher_servers_http;
    }

    upstream rancher_servers_https {
        least_conn;
        server 192.168.56.211:443 max_fails=3 fail_timeout=5s;
        server 192.168.56.212:443 max_fails=3 fail_timeout=5s;
    }
    server {
        listen     443;
        proxy_pass rancher_servers_https;
    }

}
```

```
docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 \
  -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf \
  nginx:1.14
```

- 访问 UI

> https://rancher-ui.linux.io/dashboard/?setup=

```
./kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
```

## 参考

- 支持矩阵: https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-6-5/
- 自签证书: https://docs.rancher.cn/docs/rancher2.5/installation/resources/advanced/self-signed-ssl/_index/#41-%E4%B8%80%E9%94%AE%E7%94%9F%E6%88%90-ssl-%E8%87%AA%E7%AD%BE%E5%90%8D%E8%AF%81%E4%B9%A6%E8%84%9A%E6%9C%AC
- 如何在国内使用Rancher: https://docs.rancher.cn/docs/rancher2/best-practices/use-in-china/_index
- 通过 helm 安装 rancher: https://docs.rancher.cn/docs/rancher2.5/trending-topics/install-rancher-on-k8s/_index#5-%E6%A0%B9%E6%8D%AE%E4%BD%A0%E9%80%89%E6%8B%A9%E7%9A%84-ssl-%E9%80%89%E9%A1%B9%EF%BC%8C%E9%80%9A%E8%BF%87-helm-%E5%AE%89%E8%A3%85-rancher

