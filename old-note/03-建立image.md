# 03 建立 image

## 目錄

* [Image layer](#image-layer)

* [作者的 Image：ping 特定的網站](#作者的-imageping-特定的網站)

* [環境變數](#環境變數)

* [撰寫 Dockerfile](#撰寫-Dockerfile)
  * [CMD vs ENTRYPOINT](#CMD-vs-ENTRYPOINT)
  * [ENTRYPOINT 的用途](#ENTRYPOINT-的用途)

* [了解 image 及 image layer](#了解-image-及-image-layer)

* [用 image layer cache 優化 Dockerfile](#用-image-layer-cache-優化-Dockerfile)

* [補充：將當前容器的狀態保存成新的 image](#補充將當前容器的狀態保存成新的-image)

## Image layer

一個 image 由多個小檔案分層組成，Docer 將這些小檔案組成一個容器的檔案系統。

## 作者的 Image：ping 特定的網站

先把 image 拉下來：

```bash
docker image pull diamol/ch03-web-ping
```

> 可以看到下載了很多 Layer，而這個 image 具有 Node.js 的執行環境與程式碼

把容器在後台跑起來，取名叫 web-ping：

```bash
docker run -d --name web-ping diamol/ch03-web-ping
```

看容器的 log：

```bash
docker logs web-ping
```
> 可以看到一直在 ping 一個網站

## 環境變數

上面 ping 容器的 target 是用環境變數設定的，預設 TARGET 在 dockerfile 裡先定義好了。

透過以下方式修改該環境變數的值：
```bash
docker rm -f web-ping
docker run -e TARGET=www.google.com diamol/ch03-web-ping
```
> 可以看到 ping 的目標變成了 www.google.com

## 撰寫 Dockerfile

上面 web-ping  的 Dockerfile 如下：

```Dockerfile
FROM diamol/node
ENV TARGET="blog.sixeyed.com"
ENV METHOD="HEAD"
ENV INTERVAL="3000"

WORKDIR /web-ping
COPY app.js .

CMD ["node", "/web-ping/app.js"]
```

**FROM**：指定 base image，這裡是 diamol/node，這個 image 具有 Node.js 的執行環境。

**ENV**：設定環境變數，語法為「key="value"」。這裡設定了 TARGET、METHOD、INTERVAL 三個環墫變數。

**WORKDIR**：會在容器檔案系統中建立一個目錄，並設定為工作目錄。

**COPY**：將本機的 app.js 複製到容器的 /web-ping 目錄。語法：「原始路徑 目的路徑」。

**CMD**：設定容器啟動後要執行的指令。

進入 ch03/exercises/web-ping 目錄，發先所有需的檔案都在裡面，所以可以直接建立 image：

```bash
docker image build -t web-ping .
```

* -t (--tag)：指定 image 的名稱

列出開頭為 w 的 image：

```bash
docker image ls w*
```
```text
REPOSITORY   TAG       IMAGE ID       CREATED              SIZE
web-ping     latest    62cd6dd6d411   About a minute ago   75.3MB
```

執行 image，變成每 5 秒 ping 一次 docker.com：

```bash
docker run -e TARGET=docker.com -e INTERVAL=5000 web-ping
```

### CMD vs ENTRYPOINT

* CMD：設定容器啟動後要執行的指令，***可以被 docker run 後面的 argument 覆蓋**。例如原本的 CMD 是「echo hello」，可以執行 docker run my-echo ls 後，CMD 就會被覆蓋成 ls。

* ENTRYPOINT：設定容器啟動後要執行的指令，**不會被 docker run 後面的 argument 覆蓋**。例如原本的 ENTRYPOINT 是「echo hello」，可以執行 docker run -it my-echo ls 後，容器仍然執行 echo hello。

> 可以理解為 ENTRYPOINT 等於 Linux 的執行檔，CMD 等於執行檔的參數。

如果真的想替換 ENTRYPOINT，可以在 docker run 後面加上 --entrypoint 參數：

```bash
docker run --entrypoint ls my-echo
```

### ENTRYPOINT 的用途

1. 替換經常變動的參數

例如非常確定這個容器用來做 echo，可以使用 ENTRYPOINT 設定 echo，而 CMD 設定要 echo 的內容：

```Dockerfile
FROM alpine
ENTRYPOINT ["echo"]
CMD ["hello"]
```

這樣預設會 echo hello，但是可以透過 docker run my-echo hi 來 echo hi。

### shell form vs exec form

> CMD & ENTRYPOINT 有兩種寫法：**shell form** 和 **exec form**

**shell form**

格式：

```Dockerfile
ENTRYPOINT command param1 param2
```

優點：預設以 shell 執行，因此可以取得環境變數。
缺點：無法與 CMD 結合，因為 shell form 會被視為一個指令。例如：

```Dockerfile
FROM alpine
ENV NAME=Michael
ENTRYPOINT echo $NAME
CMD ["hello"]
```
> 只會輸出 Micheal，不會有 hello。

**exec form**

格式：

```Dockerfile
ENTRYPOINT ["command", "param1", "param2"]
```

優點：可以與 CMD 結合：

```Dockerfile
FROM alpine
ENTRYPOINT ["echo", "Michael"]
CMD ["hello"]
```

> 輸出 Michael hello

缺點：無法取得環境變數。

```Dockerfile
FROM alpine
ENV NAME=Michael
ENTRYPOINT ["echo", "$NAME"]
CMD ["hello"]
```

> 輸出 $NAME hello

解決方式：在 ENTRYPOINT 中指定 shell：

```Dockerfile
FROM alpine
ENV NAME=Michael
ENTRYPOINT ["/bin/sh", "-c", "echo $NAME"]
CMD ["hello"]
```
> 但 CMD 就會被忽略


## 了解 image 及 image layer

Docker 打包 image 的過程中會將相關資訊存在 metadata 中，可以用以下指令來看：

```bash
docker image history web-ping
```

image 由 image layer 堆疊而成，每一層都是一個 image layer，每一層存在 Docker engine 的 cache 中。
> 假如有多個 image 都需要用到 Node.js 才能執行，他們會共用包含 Node.js 的 Layer

在作者的 image 中，diamol/node 映像檔有一個作業系統層(base os)，作業系統層上面有一個 Node.js 的映像層，然後 web-ping 建立在 diamol/node 之上，所以先透過 FROM 指令引入 diamol/node，再透過 COPY 指令將 app.js 複製到 web-ping 的映像層中。

查看 image 有多大：
```bash
docker image ls
```

其實列出的 size 是邏輯大小，也就是不考慮多個映像檔間共用 layer 的情況，所以實際上的大小可能會比邏輯大小小一點。我們可以用以下指令看出所有 image 的總大小：
```bash
docker system df
```
```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          25        11        31.89GB   31.23GB (97%)
Containers      37        18        360B      180B (50%)
Local Volumes   0         0         0B        0B
Build Cache     75        0         13.45GB   13.45GB
```

如果有 image 中的 image layer 是共用的，就不能隨意修改此 image，否則會影響所有使用到此 layer 的 image。
> 所以 Docker 將共用的 image layer 設定為唯讀

當 image 被建立時，就會自動建立 image layer，這些 Layer 可被共用但不能修改，這樣的設計可以節省 image 占用的空間。

## 用 image layer cache 優化 Dockerfile

web-ping 映像檔有一層 image layer 包含 javascript 的檔案，如果修改這個 javascript，並重新 build image，就會多出新的 image layer。

> 當某一個 layer 被動了，Docker 會認為順序亂掉，所以之後在那之上的每一層都不能重複使用

修改 ch03/exercises/web-ping/app.js，在結尾多個空白，然後建立新的 Image：
```bash
docker image build -t web-ping:v2 .
```
> 在 COPY 之上的所有 Layer 都會被重新建立，因為 COPY 的 JavaScript 有變動

所以我們可以將不常變動的 layer 寫到 Dockerfile 的最前面，也可以將三個 ENV 合併，以此優化 Dockerfile：

```Dockerfile
FROM diamol/node

CMD ["node", "/web-ping/app.js"]

ENV TARGET="blog.sixeyed.com" \
    METHOD="HEAD" \
    INTERVAL="3000"

WORKDIR /web-ping
COPY app.js .
```

重新 build image：
```bash
docker image build -t web-ping:v3 .
```

可以發現建置 image 需要的步驟減少了，因為 ENV 合併了。除此之外，後續修改 app.js 也不會影響到前面的 Layer。

## 補充：將當前容器的狀態保存成新的 image

假設容器有個檔案，我們在容器跑起來時修改了檔案內容，退出區後可以用以下方是將容器的狀態保存成新的 image：

```bash
docker container commit web-ping web-ping:v4
```