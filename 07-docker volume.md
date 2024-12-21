# 06 docker volume 

## 唯獨層 vs 可寫層

image 之間共享的 layer 是唯獨的，而每個 contianer 都會有各自的可寫層，可以寫入資料到某個檔案等等。

可寫層的資料會一直留著，除非容器被刪除。(exited 狀態的容器仍會保留可寫層的內容)

當我們試圖修改 image 中自帶的檔案時，其實 docker 是採用 copy-on-write 的方式，將檔案複製到可寫層，並在可寫層中進行修改。(所以不會影響到其他也用這個 image 的 container)

雖然留著容器可以保留可寫層的資料，但在正式的生產環境中，image 更新後通常會刪除舊的容器，並重新建立新的容器。但是刪除就容器的話就會丟失之前的資料，因此我們能透過 volume 讓資料保存下來。(即使在刪除容器以後)

## 使用 volume

方法有兩種：

1. 手動建立 volume 再掛載到 container
2. 在 dockerfile 中指定 volume：
```dockerfile
...(省略)...
FROM diamol/dotnet-aspnet
WORKDIR /app
ENTRYPOINT ["dotnet", "ToDoList.dll"]

VOLUME /data
COPY --from=builder /out/ .
```
> 上面掛載一個 volume 到 /data。

* 建立一個 Todo list 容器，並查看 volume：
```bash
docker run --name todo1 -d -p 8010:80 diamol/ch06-todo-list
```

* 查看 volume：
```bash
docker inspect --format '{{.Mounts}}' todo1
```
```text
[{volume ae882243a30050df4a2aea91bcb15fde93428681c3fc8057cc3df621b5dcefc1 /var/lib/docker/volumes/ae882243a30050df4a2aea91bcb15fde93428681c3fc8057cc3df621b5dcefc1/_data /data local  true }]
```

```bash
docker volume ls
```
```text
local     ae882243a30050df4a2aea91bcb15fde93428681c3fc8057cc3df621b5dcefc1
```

