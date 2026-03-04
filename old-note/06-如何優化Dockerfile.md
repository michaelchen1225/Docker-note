# 06 如何優化 Dockerfile

## 目錄

* [第一招：把不常變動的 layer 放在前面，按照可能的變動頻率排序。](#第一招把不常變動的-layer-放在前面按照可能的變動頻率排序)

* [第二招：使用多階段 Dockerfile](#第二招使用多階段-dockerfile)

## 第一招：把不常變動的 layer 放在前面，按照可能的變動頻率排序。

Dockerfile 中的每一行指令都會產生一個 layer，如果其中一個 layer 有變動，後面的 layer 都要重新建立。

所以如果將不常變動的 layer 放在前面，可以避免後面的 layer 重複建立，例如：

**優化前：**

```dockerfile
FROM diamol/golang 

WORKDIR web
COPY index.html .
COPY main.go .

RUN go build -o /web/server
RUN chmod +x /web/server

CMD ["/web/server"]
ENV USER=sixeyed
EXPOSE 80
```

**優化後：**

```dockerfile
FROM diamol/golang 

WORKDIR web

COPY main.go .
RUN go build -o /web/server
RUN chmod +x /web/server

EXPOSE 80
ENV USER=sixeyed

CMD ["/web/server"] 
COPY index.html . # 這個最有可能變動
```

## 第二招：使用多階段 Dockerfile

也就是用多個 FROM，後續的 FROM 就從前面的 FROM 中複製檔案即可：

```dockerfile
FROM diamol/golang AS builder

COPY main.go .
RUN go build -o /server
RUN chmod +x /server

FROM alpine

EXPOSE 80
CMD ["./server"]
ENV USER=sixeyed

WORKDIR /test
COPY --from=builder /server .
COPY index.html .
```

> golang 需要的編譯環境就留在 builder 就好，最後的 image 只剩編譯出來的 binary 與 index.html。

> 同樣，編排方式也是從最不常變動的 layer 開始，到最常變動的 layer。


