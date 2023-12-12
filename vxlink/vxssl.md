# 微林 - 证书服务
## 自动化更新证书

微林的证书服务，在证书申请完成之后，您可以获得证书的私钥和证书文件，微林为其提供了 URL 供您下载，这个 URL 是不会改变的，即便证书更新。因此，您可以监听这个证书的 URL，当证书更新时，您可以自动下载新的证书。

下面是我们为您准备的证书更新监听脚本，它的作用是，当证书更新时，自动下载新的证书，并重启您的服务。

```bash
#!/bin/bash

# 定义 URL 和对应的本地存储位置
declare -A urls_and_paths=(
    ["https://ssl.vx.link/aaaaaaaaaaaaa.key"]="/path/to/local/aaaaaaaaaaaaa.key"]="/path/to/local/storage/file1.txt"
    ["https://ssl.vx.link/aaaaaaaaaaaaa.crt"]="/path/to/local/aaaaaaaaaaaaa.crt"
    # 添加更多的 URL 和对应的本地存储位置
)

# 定义特定的指令(如果要重启服务，可以使用 nginx -s reload)
command_to_execute="echo 'Download complete.'"

# 文件存储所有 URL 的 ETag
etag_file="/path/to/local/storage/etag.txt"

# 如果 ETag 文件不存在，说明是第一次运行，下载所有 URL 并存储 ETag
if [ ! -e "$etag_file" ]; then
    for url in "${!urls_and_paths[@]}"; do
        local_path="${urls_and_paths[$url]}"
        
        # 下载文件
        curl -o "$local_path" "$url"
        
        # 获取并存储 ETag
        etag=$(curl -sI "$url" | grep -i etag | awk -F' ' '{print $2}' | tr -d '\r\n')
        echo "$etag" > "$local_path.etag"
        
        echo "File downloaded from $url to $local_path"
    done
    
    # 将所有 ETag 写入单独的文件
    for url in "${!urls_and_paths[@]}"; do
        local_path="${urls_and_paths[$url]}"
        cat "$local_path.etag" >> "$etag_file"
        rm "$local_path.etag"
    done
    
    echo "ETags stored in $etag_file"
    
    # 执行特定的指令
    eval "$command_to_execute"
else
    # ETag 文件存在，比较当前 ETag 与之前存储的 ETag，如果发生变化则下载文件
    url_changed=false
    
    while read -r line; do
        index=0
        for url in "${!urls_and_paths[@]}"; do
            local_path="${urls_and_paths[$url]}"
            stored_etag=$(echo "$line" | awk -v i=$index '{print $i}')
            
            # 发送 HEAD 请求以获取当前 ETag
            etag=$(curl -sI "$url" | grep -i etag | awk -F' ' '{print $2}' | tr -d '\r\n')
            
            # 检查 ETag 是否发生变化
            if [ "$etag" != "$stored_etag" ]; then
                # ETag 发生变化，下载文件并更新本地存储
                curl -o "$local_path" "$url"
                echo "$etag" > "$local_path.etag"
                url_changed=true
                
                echo "File downloaded from $url to $local_path"
            else
                echo "No change for $url"
            fi
            
            index=$((index + 1))
        done
    done < "$etag_file"
    
    # 如果有 URL 发生变动，则执行特定的指令
    if [ "$url_changed" = true ]; then
        eval "$command_to_execute"
    else
        echo "No changes. "
    fi
fi
```

将这段脚本保存为 `vxSSLupdate.sh`（或者其它的文件名称），并赋予执行权限：

```bash
chmod +x vxSSLupdate.sh
```

然后，添加作为计划任务，每天执行一次：

```bash
echo "0 0 * * * /path/to/vxSSLupdate.sh" | crontab
```

这样，它就可以每天自动检查证书是否更新了，如果更新了，就会自动下载新的证书，并执行您指定的指令（比如 `nginx -s reload`）。