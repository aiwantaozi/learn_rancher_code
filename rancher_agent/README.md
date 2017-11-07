# Rancher agent
Rancher agent 是rancher服务的一部分，最终形态是一个容器，部署在每一台需要被纳管的host上，负责与rancher server通信。

## Rancher agent主要功能
* 与rancher server建立websocket链接，接收来自rancher server的事件，事件的内容包括rancher纳管的各个组件的状态变化，比如container，host，volume等的状态变化。
* 接受rancher server的事件，根据对应事件操作本地的docker，执行拉镜像，容器启停等操作。

## 代码结构
* events
* handlers
* host_info
* ping
* progress
* register
* runtime
* service
* utils
* main


### main

* 通过flag包，获取一系列解析命令行参数
```
    version := flag.Bool("version", false, "go-agent version")
	rurl := flag.String("url", "", "registration url")
	registerService := flag.String("register-service", "", "register rancher-agent service")
	unregisterService := flag.Bool("unregister-service", false, "unregister rancher-agent service")
	flag.Parse()
	if *version {
		fmt.Printf("go-agent version %s \n", VERSION)
		os.Exit(0)
	}
```

* 从rancher server下载证书
```

func DownloadAPICrt() error {
	if _, err := os.Stat(apiCrtFile); err == nil {
		os.Remove(apiCrtFile)
	}
	if err := os.MkdirAll(path.Dir(apiCrtFile), 0755); err != nil {
		return err
	}
	file, err := os.Create(apiCrtFile)
	if err != nil {
		return err
	}
	defer file.Close()
	response, err1 := http.Get(os.Getenv(cattleURLEnv) + "/scripts/api.crt")
	if err1 != nil {
		logrus.Error(fmt.Sprintf("Error while downloading error: %s", err1))
		return err1
	}
	defer response.Body.Close()
	_, err = io.Copy(file, response.Body)
	if err != nil {
		logrus.Error(fmt.Sprintf("Error while copy file: %s", err))
		return err
	}
	return nil
}
```

* 设置环境变量

在rancher server中执行添加customer host时，需要在主机上运行一段命令，例如：
```
sudo docker run -d --privileged -v /var/run/docker.sock:/var/run/docker.sock -v /var/lib/rancher:/var/lib/rancher rancher/agent:v1.2.0 http://10.255.111.116:8082/v1/scripts/&&&CA7053F38E9740AF6:148314240000:PkpeO4E8tJpeYksUuizs8el9Y
```
最后的url是一个脚本，其中包含环境变量的值，下面是获取到的脚本：

```
#!/bin/sh
export CATTLE_REGISTRATION_ACCESS_KEY="registrationToken"
export CATTLE_REGISTRATION_SECRET_KEY="CA7053F38E9740AF6:1483142400000:PkpeO4E8tJpeYksUuizs8el9Y"
export CATTLE_URL="http://10.255.111.116:8082/v1"
export DETECTED_CATTLE_AGENT_IP="10.39.171.97"
```

```
const (
	cattleAgentIP   = "CATTLE_AGENT_IP"
	cattleURLEnv    = "CATTLE_URL"
	cattleAccessKey = "CATTLE_ACCESS_KEY"
	cattleSecretKey = "CATTLE_SECRET_KEY"
)

func RunRegistration(url string) error {
	accessKey, secretKey, cattleURL, agentIP := loadEnv(url)
	os.Setenv(cattleAgentIP, agentIP)
	os.Setenv(cattleURLEnv, cattleURL)
	return register(accessKey, secretKey, cattleURL)
}

func loadEnv(url string) (string, string, string, string) {
	accessKey, secretKey, cattleURL, agentIP := "", "", "", ""
	resp, err := http.Get(url) //此处的url获取上面的脚本，并设置到环境变量中
	if err != nil {
		return "", "", "", ""
	}
	defer resp.Body.Close()
	scanner := bufio.NewScanner(resp.Body)
	for scanner.Scan() {
		line := scanner.Text()
		if strings.Contains(line, "CATTLE_REGISTRATION_ACCESS_KEY") {
			str := strings.Split(line, "=")[1]
			accessKey = str[1 : len(str)-1]
		} else if strings.Contains(line, "CATTLE_REGISTRATION_SECRET_KEY") {
			str := strings.Split(line, "=")[1]
			secretKey = str[1 : len(str)-1]
		} else if strings.Contains(line, "CATTLE_URL") {
			str := strings.Split(line, "=")[1]
			cattleURL = str[1 : len(str)-1]
		} else if strings.Contains(line, "DETECTED_CATTLE_AGENT_IP") {
			str := strings.Split(line, "=")[1]
			agentIP = str[1 : len(str)-1]
		}
	}
	return accessKey, secretKey, cattleURL, agentIP
}
```

* 监听rancher server的事件
```
err := events.Listen(url, accessKey, secretKey, workerCount)
```

### events listen

详细介绍一下event listen

```
func Listen(eventURL, accessKey, secretKey string, workerCount int) error {
	...
	eventHandlers, err := handlers.GetHandlers() //获取所有的event handler，接收到rancher的event之后会转到相应的event handler进行处理
	if err != nil {
		return err
	}

	...
	router, err := revents.NewEventRouter("", 0, eventURL, accessKey, secretKey, nil, eventHandlers, "", workerCount, pingConfig)
	if err != nil {
		return err
	}
	err = router.StartWithoutCreate(nil)
	return err
}
```

GetHandlers中定义了监听的事件，以及对应的处理方法
```
func GetHandlers() (map[string]revents.EventHandler, error) {
	handler := initializeHandlers()

	utils.SerializeCompute = hostInfo.DockerData.Info.Driver == "devicemapper"
	logrus.Infof("Serialize compute requests: %v, driver: %s", utils.SerializeCompute, hostInfo.DockerData.Info.Driver)

	return map[string]revents.EventHandler{
		"compute.instance.activate":   cleanLog(logRequest(handler.compute.InstanceActivate)),
		"compute.instance.deactivate": utils.SerializeHandler(cleanLog(logRequest(handler.compute.InstanceDeactivate))),
		"compute.instance.inspect":    cleanLog(logRequest(handler.compute.InstanceInspect)),
		"compute.instance.pull":       cleanLog(logRequest(handler.compute.InstancePull)),
		"compute.instance.remove":     utils.SerializeHandler(cleanLog(logRequest(handler.compute.InstanceRemove))),
		"storage.volume.remove":       cleanLog(logRequest(handler.storage.VolumeRemove)),
		"ping":                        cleanLog(handler.ping.Ping),
	}, nil
}
```