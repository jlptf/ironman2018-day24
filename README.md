## Day 24 - 常見問題與建議 (5)

### 本日共賞

* 探測失敗 (Probe Failure)
* 容器映像檔未更新
* 小結

### 希望你知道

* [與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502)
* [yaml](https://ithelp.ithome.com.tw/articles/10193509)

<br/>

#### 探測失敗

> 容器正常運作不等於服務正常運作

所以該怎麼讓服務正常運作呢？答案是讓 k8s 知道容器目前正常運作。好像有點繞，但是這是使用 k8s 的時候特別重要的觀念：容器需要持續回報正常運作 (我還活著！)。

k8s 提供兩種探測方式：`Liveness Probe` 與 `Readiness Probe`。基本上，Probe 會定期的執行某些動作來確保應用程式在正常運作中。

> 某些動作可能是執行某個指令或者是傳送一個 Http Request

如果使用 `Liveness Probe`，當錯誤發生時 (無回應或回應錯誤)，k8s 會嘗試重新建立 (kill then create) 一個新的容器。

如果使用 `Readiness Probe`，當錯誤發生時，對應的 Service 物件就會將該容器標示為不可使用，所以任何的需求都不會導向該容器。

讓我們看看底下探測失敗的例子

```
# error-probe.yaml

---
apiVersion: v1
kind: Pod
metadata:
  name: err-probe-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    livenessProbe:
      httpGet:
        path: /healthy         <=== 探測一個不存在的位置
        port: 80
      initialDelaySeconds: 3   <=== 容器啟動後 3 秒開始探測
      periodSeconds: 3         <=== 探測間隔 3 秒
    readinessProbe:
      httpGet:
        path: /healthy         <=== 探測一個不存在的位置
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 3

```

接著部署到 k8s 並觀察狀態

```bash
$ kubectl apply -f error-probe.yaml
pod "err-probe-nginx" created

kubectl get pods
NAME              READY     STATUS    RESTARTS   AGE
err-probe-nginx   0/1       Running   0          11s
```

這時候會發現 `READY: 0/1`，過一下再次查看

```bash
$ kubectl get pods
NAME              READY     STATUS    RESTARTS   AGE
err-probe-nginx   0/1       Running   2          44s
```

發現 `RESTARTS: 2` 居然已經重啟兩次了，於是查看一下 Pod 內容

```bash
$ kubectl describe pods err-probe-nginx
...
Events:
  Type     Reason                 Age                From               Message
  ----     ------                 ----               ----               -------
  Normal   Scheduled              56s                default-scheduler  Successfully assigned err-probe-nginx to minikube
  Normal   SuccessfulMountVolume  56s                kubelet, minikube  MountVolume.SetUp succeeded for volume "default-token-ld9jt"
  Warning  Unhealthy              28s (x5 over 46s)  kubelet, minikube  Readiness probe failed: HTTP probe failed with statuscode: 404   <=== 錯誤在這裡！
  Normal   Pulling                27s (x3 over 55s)  kubelet, minikube  pulling image "nginx"
  Warning  Unhealthy              27s (x5 over 45s)  kubelet, minikube  Liveness probe failed: HTTP probe failed with statuscode: 404   <=== 錯誤在這裡！
  Normal   Killing                27s (x2 over 39s)  kubelet, minikube  Killing container with id docker://nginx:Container failed liveness probe.. Container will be killed and recreated.
...
```

`Liveness probe failed: HTTP probe failed` 因為 Liveness Probe 探測結果回應 `statuscode: 404`，即頁面不存在故 k8s 會嘗試重啟 Pod。

> 大於 200 小於 400 的 http code 皆判定為正常

*建議方案*

1. `Probe 指定的位置是否正確`：上面例子的 /healthy 就不存在
2. `是否太快進行探測`：服務還沒運行就探測所以無法正確回應
3. `服務錯誤無法回覆`：服務發生錯誤，也許是資料庫設定錯誤或某些檔案不存在

> 請注意，這邊提到的 Probe 是指 k8s 本身。如果套用雲端服務，k8s 外可能還會再包一層 Load Balancer (LB)，而 LB 本身也會需要探測服務是否正確運行，而探測的規則可能根據服務商而有所差異。
> 
> 例如 Google 的 LB 只判定 200 為正常運行，因此如果探測位置回覆介於 200 到 400 但不等於 200 的回覆碼，Google LB 就會判定為異常。

<br/>

#### 容器映像檔未更新

程式開發難免會遇到需要修改 bug，而在容器環境下修改 bug 後，通常會把修正後的內容包成映像檔後再發佈到像 k8s 這類的環境中使用。這時候會有機會發生一個問題：k8s 啟動 Pod 時，並未抓取更新後的映像檔。

要討論為什麼會有這個問題？先要知道 k8s 是根據何種策略抓取映像檔即 pull policy。在 pull policy 中可以有以下設定

1. `Always`：總是抓取
2. `Never`：不抓取
3. `IfNotPresent`：如果不存在的時候抓取

在不指定 :latest tag 下，預設會使用 `IfNotPresent` 這個抓取策略，如果有指定使用 :latest tag 則會套用 `Always` 策略。

> 關於 :latest tag 請參考 [Docker 官網說明](https://docs.docker.com/engine/reference/commandline/pull/)

接著預設一個情境

1. 部署一個使用 `jlptf/myapp:v1` 映像檔的 Deployment
2. 發現 myapp 有問題需要修正
3. 修正 myapp 後建立映像檔 `jlptf/myapp:v1`
4. 刪除原本使用 `jlptf/myapp:v1` 映像檔的 Pod
5. k8s 自動建立使用 `jlptf/myapp:v1` 映像檔的 Pod
6. 問題並沒有被修正

寫到這裡聰明的大家應該都知道發生什麼事情了，由於並未指定 :latest tag 所以抓取映像檔會使用 `IfNotPresent` 策略，而 `jlptf/myapp:v1` 早就存在系統中，因此 k8s 並不會抓取修改過後的映像檔

*建議方案*

1. 使用 :latest tag
2. 在部署 Deployment 的設定中指定 `ImagePullPolicy: Always`
3. 分配一個唯一的 tag

第一種方式不是很建議在正式環境下使用，因為只使用 :latest 無法追蹤映像檔變化，而且回復也會很困難。

使用第二種方式雖然可以解決映像檔未更新的問題，但是每次都需要重新抓取表示建構時間會加長。如果更新的內容與映像檔無關，那麼重新抓取映像檔就會浪費時間與流量。

> 如果使用雲端服務，流量是要算錢的

因此，如果能使用唯一的 tag，是較佳的解決方式。

<br/>

#### 小結

我們花了五天的時間分享了 k8s 常見的問題與建議方案。總結如下

* 善用 `kubectl get pods` 與 `kubectl describe pods/<name>` 觀察 Pod 狀態
* 記得 `kubectl describe deployment/<name>` 觀察 Deployment 狀態
* `kubectl logs` 可以幫你查看容器的訊息
* Event 區塊包含了很多對除錯有幫助的訊息

經過這五天的分享，相信各位在未來面對錯誤時會比較有概念該如何處理。

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day24/](https://jlptf.github.io/ironman2018-day24/)