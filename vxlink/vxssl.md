# 微林 - 证书服务
## 自动化更新证书

微林的证书服务，在证书申请完成之后，您可以获得证书的私钥和证书文件，微林为其提供了 URL 供您下载，这个 URL 是不会改变的，即便证书更新。因此，您可以监听这个证书的 URL，当证书更新时，您可以自动下载新的证书。

下面是我们为您准备的证书更新监听脚本，它的作用是，当证书更新时，自动下载新的证书，并重启您的服务。

```bash
#!/bin/bash
# 注意：此脚本需要 curl 与 md5sum 命令，如果没有请提前安装

# 定义 URL 和对应的本地存储位置
declare -A urls_and_paths=(
    # 下面是一个例子，如果是在 nginx 中，您应该根据 nginx.conf 或者自定义的配置文件来填写参数
    ["https://ssl.vx.link/657c302abeaaa.key"]="/root/test.key"
    ["https://ssl.vx.link/657c302abeaaa.crt"]="/root/test.crt"
    # 一行一份，添加更多的 URL 和对应的本地存储位置
    # 请注意， Key 和 Crt 文件应该都一起更新
)

# 遍历所有 URL，并下载文件
for url in "${!urls_and_paths[@]}"; do
    local_path="${urls_and_paths[$url]}"
    
    # 获取并存储 Xtag
    response_code=$(curl -sI "$url" | awk '/^HTTP/ {print $2}')
    if [ "$response_code" -eq 200 ]; then
        xtag=$(curl -sI "$url" | grep -i xtag | awk -F' ' '{print $2}' | tr -d '\r\n')

        # 检查 xtag 是否是标准的 md5 值，如果不是则跳过
        if [[ ! "$xtag" =~ ^[a-f0-9]{32}$ ]]; then
            echo "Warning: Xtag for $url is not a standard md5 value. Skipping Xtag check."
            continue
        fi

        # 检查是否 xtag 发生改变
        echo "Xtag for $url is $xtag | Local MD5 is $(md5sum "$local_path" | awk '{print $1}')"
        if [ "$xtag" != "$(md5sum "$local_path" | awk '{print $1}')" ]; then
            # 下载文件
            echo "File downloaded from $url to $local_path"
            curl -s -o "$local_path" "$url"
            execute_operation=true
        fi
    else
        echo "Warning: HTTP response code is not 200 for $url. Skipping Xtag check."
    fi
done

# 执行特定的指令，当有任意一个 URL 的 xtag 发生改变时才执行
if [ "$execute_operation" = true ]; then
    eval "$command_to_execute"
fi

```

将这段脚本保存为 `vxSSLupdate.sh`（或者其它的文件名称），并赋予执行权限：

```bash
chmod +x vxSSLupdate.sh
```
使用之前，记得修改脚本中的参数，比如 URL 和本地存储位置，以及需要执行的指令。  
然后，添加作为计划任务，每天执行一次：

```bash
(crontab -l 2>/dev/null; echo "0 0 * * * /path/to/vxSSLupdate.sh") | crontab -
```

这样，它就可以每天自动检查证书是否更新了，如果更新了，就会自动下载新的证书，并执行您指定的指令（比如 `nginx -s reload`）。

## Synology DiskStation Manager (DSM) 群晖自动化更新证书

先在 控制面板-安全性-证书-高级设置 中重置证书，然后在 控制面板-任务计划 中新增用户定义的脚本，用户为 root 链接替换为微林证书网址，执行时间根据情况设置：

```bash
cd /usr/syno/etc/certificate/_archive/
cd $(jq -r 'keys[0]' INFO)
wget https://ssl.vx.link/65.....AA.key -O privkey.pem
wget https://ssl.vx.link/65.....AA.crt -O cert.pem
cp cert.pem fullchain.pem
synow3tool --gen-all && systemctl reload nginx
```

## 关于安全性
理论上来说，证书服务提供的 Key 和 Crt 文件的 URL 不会被猜出，因为它是随机生成的。  
但是如果在某个环节导致了 URL 地址泄露，您可以随时删除证书，然后再添加回来，这个时候 URL 会重新生成，之前的 URL 就会失效。
