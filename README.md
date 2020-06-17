## 注册流程

### 客户端

`处理用户输入进行数据校验，获取用户的注册信息`

```
用户名(username)：
昵称(nickname)：
密码(password)：
确认密码：
电话(phone)：
邮箱(email)：
```

​	**客户端请求**：

`url：http//192.168.142.228:80/reg`

```
携带数据：post：json格式
{
	“userName”:"xxx",
	"nickName":"xxx",
	"password":"xxx",
	"phone":"xxx",
	"email":"xxx"
}
```

**服务器回复:**

`JSON格式`

```
成功：{“code”:"002"} --> 跳转到登录界面
该用户已存在：{“code”:"003"}
失败：{“code":"004"}
```

### 服务器端

`接受客户端post请求，转接到FastCGI函数（reg_cgi）处理`

`nginx配置`

```
location /reg{
    fastcgi_pass 127.0.0.1:10001;
    include fastcgi.conf;
}
```

`reg_cgi处理函数处理流程`

```
1. 阻塞等待用户连接
2. 获取客户端发送过来的json数据 （接收从nginx转接过来的用户注册信息）
3. 解析用户注册信息的json包，获取用户的注册信息
4. 读取配置文件，登录数据库，查询解析出来的用户是否在数据库存在，发送对应的“code”
5. 用户名不存在则在数据库中创建对应的用户信息及用户创建时间
```



## 登录流程

### 客户端

```
1. 获取用户登录信息&&数据校验
2. 对密码进行MD5加密并放入客户端配置文件
```

`url：http//192.168.142.228:80/login`

```
post:
{
	“username”:"xxx"
	"password":"xxx" --> MD5加密
}
```

**接受服务器端的回复：**

```
成功：{"code":"000"} --> token
失败：{"code":"001"}

将当前的用户信息（用户名，密码，token）通过单例模式类LoginInfoInstance保存在本地                                                                                       
```



### 服务器端

`接受客户端post请求，转接到FastCGI函数（login_cgi）处理`

`nginx配置`

```
location /login{
    fastcgi_pass 127.0.0.1:10000;
    include fastcgi.conf;
}
```

`login_cgi`

```
1. 阻塞等待用户连接
2. 获取用户登录数据及长度
3. 解析用户登录信息的json包，获取用户名及密码
4. 读取配置信息，登录数据库，查询数据库，校验用户名及密码
5. 校验成功，发送token给客户端
```

## 上传流程

### 客户端

#### 文件上传

`url：http//192.168.142.228:80/upload`

```
post数据：---得到上传的文件数据的MD5加密序列
------WebKitFormBoundary88asdgewtgewx\r\n
Content-Disposition: form-data; user="mike";filename="xxx.jpg"; md5="xxxx"; size="xxx"\r\n
Content-Type: application/octet-stream\r\n
\r\n
真正的文件内容\r\n
------WebKitFormBoundary88asdgewtgewx
```

**服务端回传数据**：

```
{
	成功：{"code":"008"}
    失败：{"code":"009"}
}
```

#### MD5秒传

`url：http//192.168.142.228:80/md5`

```
post数据
{
    "user":xxxx,
    "token": xxxx,
    "md5":xxx,
    "fileName": xxx
}
```

**服务端回复：**

```
文件已存在：{"code":"004"}
秒传成功：  {"code":"005"}
秒传失败：  {"code":"006"} -->完整的文件上传
```



### 服务器端

#### 	文件上传

`接受客户端post请求，转接到FastCGI函数（upload_cgi）处理`

`nginx配置`

```
location /upload{
    fastcgi_pass 127.0.0.1:10002;
    include fastcgi.conf;
}
```

`upload_cgi`

```
1. 读取数据库配置信息
2. 等待nginx发送数据
3. 从标准输入(web服务器)读取内容
4. 解析post数据，得到上传文件信息-->user,filename,md5,size
5. 将该文件存入fastDFS中,并得到文件的file_id
6. 删除本地临时存放的上传文件
7. 得到文件所存放storage的host_name,拼接文件的URL
8. 将该文件的FastDFS相关信息及文件的md5存入mysql中
9. 给前端反馈信息
```

#### 	md5秒传

`接受客户端post请求，转接到FastCGI函数（md5_cgi）处理`

`nginx配置`

```
location /md5{
    fastcgi_pass 127.0.0.1:10003;
    include fastcgi.conf;
}
```

`md5_cgi`

```
1. 读取数据库配置信息
2. 等待nginx发送数据
3. 从标准输入(web服务器)读取内容
4. 解析json数据，验证登陆token
5. 登录数据库，查看是否有此文件的md5，
	如果没有
		-->返回 006
	如果有
		-->查看此用户是否已经有此文件，如果存在说明此文件已上传，无需再上传 返回004
 		-->如果用户没有此文件，修改file_info中的count字段，+1 （count 文件引用计数）
 			\在user_file_list中插入一条数据，更新user_file_count  返回005
6. 反馈前端
```



## 用户列表相关

### 客户端

`url：http//192.168.142.228:80/myfiles?cmd=count`--->获取用户文件个数

```
post数据
{
	"username"="xxx",
	"token"="xxx"
}
```

**recv:**

```
{
	"num":"xxx"
	"code":"110" //token 验证
}
```

`url：http//192.168.142.228:80/myfiles?cmd=normal`--->获取用户文件信息

`url：http//192.168.142.228:80/myfiles?cmd=pvasc/pvdesc`--->按下载量升序/降序

```
{
    "user": "xxxx"
    "token": "xxxx"
    "start": 0
    "count": 10
}
```

**recv:**

```
 成功：
 {
  "file":[
  	{
  		"user":"yoyo",
 		"md5":"e8ea6031b779ac26c319ddf949ad9d8d",
 		"time":"2020-05-10 21:35:25",
 		"filename":"test.mp4",
		"share_status":0,
 		"pv":0,
 		"url":"http://192.168.31.109:80/group1/M00/00/00/wsksadfbViy2Z2As-g3Z526.mp4",
 		"size":274666,
 		"type":"mp4"
 	},
 	{
  		"user":"yoyo",
 		"md5":"esdfsa031b779ac26c319ddsfsfsdfd",
 		"time":"2020-04-29 19:56:05",
 		"filename":"test.mp4",
		"share_status":0,
 		"pv":0,
 		"url":"http://192.168.31.109:80/group1/M00/00/00/wKgfbViy2Z2As-g3Z0782.mp4",
 		"size":27473666,
 		"type":"mp4"
 	}	
  ]
 	
}
失败：
{
	"code":"015"
}
```

`显示文件列表Qt概述`

> 1. 获取登录信息的单例模式类--LoginInfoInstance
> 2. 通过单例类存储的用户信息访问服务器获取用户文件数目，判断用户的文件个数是否>0
> 3. 获取文件列表信息--通过单例类存储信息访问服务器，递归地获取用户文件信息，并将每个用户文件信息封装到FileInfo结构体中，同时用`QList<FileInfo *> m_fileList` 保存用户结构体信息
> 4. 通过遍历 `m_fileList` 来获取每个文件信息，让 `QListWidget`进行显示

### 服务器端

`接受客户端post请求，转接到FastCGI函数（myfiles_cgi）处理`

`nginx配置`

```
location /myfiles{
    fastcgi_pass 127.0.0.1:10004;
    include fastcgi.conf;
}
```

`myfiles_cgi`

```
1. 获取URL地址 “？” 后面的内容 char* query = getenv("QUERY_STRING");
2. 解析cmd，获取cmd的指令内容
3. 获取post提交内容 fread(buf, 1, len, stdin) -- 从标准输入(web服务器)读取内容
4. if cmd == count //count 获取用户文件个数
		--通过json包获取用户名, token
		--验证登录token
		--登录数据库获取用户文件个数
   else
   		--解析json包{user，token，start，count}
   		--验证登录token
   		--获取用户文件列表
   			--cmd==normal 获取用户文件信息
   			--cmd==pvasc  按下载量升序
   			--cmd==pvdesc 按下载量降序
   		
   
```



## 下载流程

### 客户端

```
1. 将文件添加下载列表中，其中创建下载任务列表类 DownloadTask，单例模式，一个程序只能有一个下载任务列表，DownloadTask 类中包含储存下载文件的列表 QList <DownloadInfo *> list，DownloadInfo 结构体储存的是文件属性信息，包含文件的URL
2. 线程池处理下载队列
3. 文件下载URL
```

`请求服务器 URL= http//192.168.142.228:80/myfiles?cmd=pv`

```
post
{
	"user": "yoyo",
	"token": "xxx",
	"md5": "xxx",
	"filename": "xxx"
}
```

**recv:**

```
成功
{"code":"016"}
失败
{"code":"017"}
```



### 服务器端

`接受客户端post请求，转接到FastCGI函数（dealfile_cgi）处理`

```
1. 读取数据库(mysql,redis)配置信息
2. 获取URL地址 "?" 后面的内容
3. 从标准输入(web服务器)读取内容
4. 解析json信息
5. 验证token
6. if cmd == pv
	--更新user_file_list pv字段(下载量)
```

## 分享和删除



#### 客户端

`url：http//192.168.142.228:80/dealfile?cmd=share`

`url：http//192.168.142.228:80/dealfile?cmd=del`

```
post数据
{
	"user": "yoyo",
	"token": "xxxx",
	"md5": "xxx",
	"filename": "xxx"
}
```

**recv:**

`分享`

```
成功：
{"code":"010"}
失败
{"code":"011"}
别人已经分享此文件
{"code":"012"}
```

`删除`

```
成功：
{"code":"013"}
失败
{"code":"014"}
```



#### 服务器端

`接受客户端post请求，转接到FastCGI函数（dealfile_cgi）处理`

```
1. 读取数据库(mysql,redis)配置信息
2. 获取URL地址 "?" 后面的内容
3. 从标准输入(web服务器)读取内容
4. 解析json信息
5. 验证token
6. if cmd == share
	a)先判断此文件是否已经分享，集合有没有这个文件，如果有，说明别人已经分享此文件，中断操作(redis操作)
    b)如果集合没有此元素，可能因为redis中没有记录，再从mysql中查询，如果也没有，说明真没有(mysql操作)
    c)如果mysql有记录，而redis没有，说明redis没有存此文件，redis保存此文件信息后，再中断操作(redis)
    d)如果此文件没有被分享，mysql保存一份持久化操作(mysql操作)
    e)redis集合中增加一个元素(redis操作)
    f)redis对应的hash也需要变化 (redis操作)
    
   if cmd == del
    a)先判断此文件是否已经分享
    b)判断集合有没有这个文件，如果有，说明别人已经分享此文件(redis操作)
    c)如果集合无此元素，可能因为redis中没有记录，再从mysql中查询，如果mysql也无，说明真没有(mysql操作)
    d)如果mysql有记录，而redis没有记录，那么分享文件处理只需要处理mysql (mysql操作)
    e)如果redis有记录，mysql和redis都需要处理，删除相关记录
    
    
    note:{
    	redis 存取内容：
    		SortSet类型：FILE_PUBLIC_ZSET:md5+文件名
    		hash类型：FILE_NAME_HASH：md5+文件名:文件名
    		string类型：用户名：token
    }
```



## 共享列表

### 客户端

1. 获取共享文件数目

`url：http//192.168.142.228:80/sharefiles?cmd=count`--> get请求

2. 共享文件信息
   - 获取普通共享共享文件信息

`url：http//192.168.142.228:80/sharefiles?cmd=normal`

`url：http//192.168.142.228:80/sharefiles?cmd=pvasc`

`url：http//192.168.142.228:80/sharefiles?cmd=pvdesc`

```
post数据
{
    "start": 0,
    "count": 10
}
```

- recv

```
cmd == normal
{
        "user": "yoyo",
        "md5": "e8ea6031b779ac26c319ddf949ad9d8d",
        "time": "2017-02-26 21:35:25",
        "filename": "test.mp4",
        "share_status": 1,
        "pv": 0,
        "url": "http://192.168.31.109:80/group1/M00/00/00/wKgfbViy2Z2AJ-FTAaM3As-g3Z0782.mp4",
        "size": 27473666,
         "type": "mp4"
}

cmd == pvasc || pvdesc
{
	"filename":"XXXX",
	"pv":x
}
```



3. 取消分享文件

`url：http//192.168.142.228:80/dealsharefile?cmd=cancel`

```
post数据
{
	"user": "yoyo",
	"md5": "xxx",
	"filename": "xxx"
}
```

- **recv**

```
成功
{"code":"018"}
失败
{"code":"019"}
```

4. 转存文件

`url：http//192.168.142.228:80/dealsharefile?cmd=save`

```
post数据
{
	"user": "yoyo",
	"md5": "xxx",
	"filename": "xxx"
}
```

- **recv**

```
成功
{"code":"020"}
文件已存在
{"code":"021"}
失败
{"code":"022"}
```

### 服务器端

`获取共享文件信息`

```
1. 获取URL地址 "?" 后面的内容，解析cmd
2. 从标准输入(web服务器)读取内容
3. 解析json包
3. if cmd == count 
		--获取共享文件个数
   if cmd == normal 
   		-- 查询数据库（share_file_list）获取共享文件列表
   if cmd == pvasc || pvdesc
        a) mysql共享文件数量和redis共享文件数量对比，判断是否相等
        b) 如果不相等，清空redis数据，从mysql中导入数据到redis (mysql和redis交互)
        c) 从redis读取数据，给前端反馈相应信息
```

`共享文件pv字段处理、取消分享、转存文件cgi程序`



```
1. 获取URL地址 "?" 后面的内容，解析cmd
2. 从标准输入(web服务器)读取内容
3. 解析json包
3. if cmd == pv 
		--文件下载标志处理
		下载文件pv字段处理
			成功：{"code":"016"}
			失败：{"code":"017"}
   if cmd == cancel
   		-- 取消分享文件
   if cmd == save 
        -- 转存文件
```

**项目描述：**基于Nginx作为反向代理和轻量级的Web服务器，用C/C++编写应用程序通过FastCGI与Nginx交互处理动态页面，采用FastDFS作为后台的分布式文件系统实现文件的存储。客户端基于QT环境开发，与后台服务器以redis/MySQL数据库进行交互，实现文件的上传、秒传、下载、共享等功能。