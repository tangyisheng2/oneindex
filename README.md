# OneIndex
Onedrive Directory Index

## 功能：
不占用服务器空间，不走服务器流量，  

直接列出 OneDrive 目录，文件直链下载。  

Update：通过反代下载文件

```php
		//通过分页获取页面所有item
		static function dir_next_page($request, &$items, $retry=0){
			$resp = fetch::get($request);
			
			$data = json_decode($resp->content, true);
			if(empty($data) && $retry < 3){
				$retry += 1;
				return self::dir_next_page($request, $items, $retry);
			}
			
			foreach((array)$data['value'] as $item){
				//var_dump($item);
				$items[$item['name']] = array(
					'name'=>$item['name'],
					'size'=>$item['size'],
					'lastModifiedDateTime'=>strtotime($item['lastModifiedDateTime']),
					'downloadUrl'=>$item['@microsoft.graph.downloadUrl'],
					'folder'=>empty($item['folder'])?false:true
				);
			}

			if(!empty($data['@odata.nextLink'])){
				$request = self::request();
				$request['url'] = $data['@odata.nextLink'];
				return self::dir_next_page($request, $items);
			}
		}
```

将第16行替换为

```php
'downloadUrl'=>str_ireplace("截取的链接","反代的链接",$item['@microsoft.graph.downloadUrl']),
```

Nginx反代设置

```nginx
server
{
    listen 80;
    listen 443 ssl http2;
    server_name yours.domainname.com;#把这里改成你的站点域名（不能和FODI前端一样！）
    
    ssl_certificate    /usr/share/nginx/ssl/pem.crt;#修改成自己的ssl证书(PEM格式)的存放位置
    ssl_certificate_key    /usr/share/nginx/ssl/ssl.key;#修改成自己的ssl密钥(KEY)的存放位置
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
location  ~* \.(php|jsp|cgi|asp|aspx)$
    {
        proxy_pass https://yours.sharepoint.com;#把这里改成使用宝塔面板步骤3、步骤4的截选出的链接
        proxy_set_header Host yours.sharepoint.com;#把这里改成上面链接去掉https://
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header Range $http_range;#此项感谢评论区超音速的提醒
    }
    location /
    {
        proxy_pass https://yours.sharepoint.com;#把这里改成使用宝塔面板步骤3、步骤4的截选出的链接
        proxy_set_header Host yours.sharepoint.com;#把这里改成上面链接去掉https://
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header Range $http_range;#此项感谢评论区超音速的提醒
        
        add_header X-Cache $upstream_cache_status;
        add_header Cache-Control no-cache;
        expires 12h;
    }
}
```



Credit：https://www.nbmao.com/archives/3917

## Demo
[https://xn.tn](https://xn.tn)  

## 安装运行

### 源码安装运行：

#### 需求：
1、PHP空间，PHP 5.6+ 需打开curl支持  
2、OneDrive 账号 (个人、企业版或教育版/工作或学校帐户)  
3、OneIndex 程序   

## 配置：
<img width="658" alt="image" src="https://raw.githubusercontent.com/donwa/oneindex/files/images/install.gif">  

### 计划任务  
[可选]**推荐配置**，非必需。后台定时刷新缓存，可增加前台访问的速度。  
```
# 每小时刷新一次token
0 * * * * /具体路径/php /程序具体路径/one.php token:refresh

# 每十分钟后台刷新一遍缓存
*/10 * * * * /具体路径/php /程序具体路径/one.php cache:refresh
```

### Docker 安装运行

- 请参考[TimeBye/oneindex](https://github.com/TimeBye/oneindex)

## 特殊文件实现功能  
` README.md `、`HEAD.md` 、 `.password`特殊文件使用  

可以参考[https://github.com/donwa/oneindex/tree/files](https://github.com/donwa/oneindex/tree/files)  

**在文件夹底部添加说明:**  
>在 OneDrive 的文件夹中添加` README.md `文件，使用 Markdown 语法。  

**在文件夹头部添加说明:**  
>在 OneDrive 的文件夹中添加`HEAD.md` 文件，使用 Markdown 语法。  

**加密文件夹:**  
>在 OneDrive 的文件夹中添加`.password`文件，填入密码，密码不能为空。  

**直接输出网页:**  
>在 OneDrive 的文件夹中添加`index.html` 文件，程序会直接输出网页而不列目录。  
>配合 文件展示设置-直接输出 效果更佳。  

## 命令行功能  
仅能在PHP CLI模式下运行  

**清除缓存:**  
```
php one.php cache:clear
```
**刷新缓存:**  
```
php one.php cache:refresh
```
**刷新令牌:**  
```
php one.php token:refresh
```
**上传文件:**  
```
php one.php upload:file 本地文件 [OneDrive文件]
```


**上传文件夹:**  
```
php one.php upload:folder 本地文件夹 [OneDrive文件夹]
```

例如：  
```
//上传demo.zip 到OneDrive 根目录  
php one.php upload:file demo.zip  

//上传demo.zip 到OneDrive /test/目录  
php one.php upload:file demo.zip /test/  

//上传demo.zip 到OneDrive /test/目录并将其命名为 d.zip  
php one.php upload:file demo.zip /test/d.zip  

//上传up/ 到OneDrive /test/ 目录  
php one.php upload:file up/ /test/
```
