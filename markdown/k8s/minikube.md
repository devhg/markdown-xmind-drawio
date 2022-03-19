k8s安装 ：

* https://skyao.io/learning-kubernetes/docs/installation.html
* http://soulmz.me/2020/04/29/minikube-installed-for-mac/



## 安装minibube

https://minikube.sigs.k8s.io/docs/start/

```bash
$ minikube start --image-mirror-country='cn' --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers' --alsologtostderr


➜  ~ minikube dashboard
🔌  正在开启 dashboard ...
    ▪ Using image registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-scraper:v1.0.7
    ▪ Using image registry.cn-hangzhou.aliyuncs.com/google_containers/dashboard:v2.3.1
🤔  正在验证 dashboard 运行情况 ...
🚀  Launching proxy ...
🤔  正在验证 proxy 运行状况 ...
🎉  Opening http://127.0.0.1:65043/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...


```



