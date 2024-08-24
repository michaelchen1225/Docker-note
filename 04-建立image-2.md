# Build server 的困境

開發通常會用版控軟體來管理程式碼，當程式碼變動後，通常會轉到專門建置整個軟體的伺服器上，進行編譯、打包、測試等等的工作，這個伺服器就是 Build server。

問題：

* 需額外維護一台伺服器
* 所有人的開發環境安裝的版本都要和 Build server 一樣，不然可能會有編譯錯誤
* 新人加入時，要花時間設定與 Build server 一樣的環境

用 Docker 將這個專案所需的工具包好，分享給所有人即可，團隊即可用 image 中的環境來編譯程式碼。

## 多階段 Dockerfile

```dockerfile
FROM diamol/base As build-stage
RUN echo "Building..." > build.txt

FROM diamol/base As test-stage
COPY --from=build-stage /build.txt /build.txt
RUN echo "Testing..." >> /build.txt

FROM diamol/base
COPY --from=test-stage /build.txt /build.txt
CMD cat /build.txt
```

* 多階段 Dockerfile 以多個 FROM 組成
  * 用 As 命名每個階段
  * 多個階段，最終產生的只有最後一個階段的 image
  * 如果任何階段有錯誤，整個 Dockerfile 會失敗

* COPY --from 可以從先前的階段複製檔案

* RUN 是在製作 image 時執行的指令，結果會留在 image 中
  * RUN 所執行的指令必須由 FROM 映像檔中提供

進入 ch04/exercises/multi-stage 目錄，執行以下指令：

```bash
docker build -t multi-stage .
```

這種多階段的 Dockefile 可以這樣設計：

* 先安裝應用程式所需的工具
* 編譯程式碼
* 單元測試
* 將編譯好的 binary 進行最終測試
* 將最終測試的結果複製到最終 image 中

> Build server 只需安裝 Docker，因為建置工具都在 Dockerfile 中

## Java Spring Boot 範例

一般建置多階段 java 應用程式的流程如下：

1. 下載相依函式庫並建置應用程式
2. 複製建置好的應用程式並測試
3. 複製測試完成的應用程式到最終 image

下面的應用程式需要一套 Java 工具：
* Maven(定義建置過程、取得相依函式庫)
* OpenJDK(Java 執行環境和工具包)

```dockerfile
FROM diamol/maven As builder

WORKDIR /usr/src/iotd
COPY pom.xml .
RUN mvn -B dependency:go-offline #  取得相依函式庫

COPY . .
RUN mvn package # 執行 mvn package

FROM diamol/openjdk
WORKDIR /app
COPY --from=builder /usr/src/iotd/target/iotd-service-0.1.0.jar .

EXPOSE 80
ENTRYPOINT ["java", "-jar", "/app/iotd-service-0.1.0.jar"]
```

* ENTRYPOINT 是容器啟動時執行的指令

到 ch04/exercises/image-of-the-day 目錄，執行以下指令：

```bash
docker build -t image-of-the-day .
```

## Docker 虛擬網路

建好 image 只是第一步，Docker 建立容器後會分配虛擬 IP，讓容器之間可以互相溝通。

* 建立虛擬網路：
```bash
docker network create nat
```

* 用剛剛的 image 執行一個容器，開放 80 port，並加入 nat 網路：
```bash
docker run --name iotd -d -p 800:80 --network nat image-of-the-day
```

前往瀏覽器輸入 http://localhost:800/image 即可看到畫面 NASA 的每日圖片。

> 提醒；最終階段的 image 不包含 maven，因為最終階段使用的是 diamol/openjdk 映像檔

## 重點：

總之，用多階段 Dockerfile 可以優化 image 的大小：

> 原理：不僅可以將建置工具包裝在 Dockerfile 中，讓團隊可以共享，且若前面執行的成功，最終產出的就是最後一個階段包含最終測試程式碼的 image。