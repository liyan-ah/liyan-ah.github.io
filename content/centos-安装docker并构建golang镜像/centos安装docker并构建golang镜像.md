
<!--more-->

# 安装docker

```
sudo yum install -y yum-utils

sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo  
    
sudo yum install docker-ce docker-ce-cli containerd.io  
\# 这里报了一个错
\# (try to add '--allowerasing' to command line to replace conflicting packages or '--skip-broken' to skip uninstallable packages or '--nobest' to use not only best candidate packages)  
\# 重新执行  
sudo yum install docker-ce docker-ce-cli containerd.io --allowerasing

\# 启动 docker  
sudo systemctl start docker

\# 测试   
sudo docker run hello-world 
\# Hello from Docker!  
\# This message shows that your installation appears to be working correctly.  
```


# 构建golang服务镜像
先看下工作目录的结构：
```
.
├── Dockerfile
├── gin-srv
├── go.mod
├── go.sum
└── main.go
```
简单写一个`golang`的程序:
```
package main
import "time"
import "github.com/gin-gonic/gin"

type Resp struct{
	Errno int `json:"errno"`
	Data map[string]int64 `json:"data"`
}

func main(){
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context){
		resp := &Resp{Errno:0, Data: map[string]int64{
			"now": time.Now().Unix(),
		}}
		c.JSON(200, &resp)
	})
	r.Run("0.0.0.0:8080")
}
```

构建一个Dockerfile，以`centos`作为base以便能够正常登陆容器进行调试：
```
FROM centos:8
ADD . ./
EXPOSE 8080
ENTRYPOINT ["./gin-srv"]
```
启动容器：
```
# 构建镜像
sudo docker build -t gin_docker .
# 启动镜像
sudo docker run --name gin_docker -p 8080:8080 -d gin_docker
```
访问容器中的服务：
```
$ curl localhost:8080/ping
{"errno":0,"data":{"now":1646381863}}
```

容器起来了。可以继续后面的性能评估及agent启动工作了。

# 参考文献
[Install Docker Engine on CentOS](https://docs.docker.com/engine/install/centos/) 
[Golang应用打包docker镜像并运行](https://article.itxueyuan.com/7DreWj)