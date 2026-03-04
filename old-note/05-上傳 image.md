# 05 上傳 image

1. 登入 docker hub

```bash
docker login
```

2. tag image

```bash
docker tag <image_id> <docker_hub_username>/<image_name>:<tag>
```
例如：
```bash
docker tag  multi-stage michaelchenn1225/multi-stage:v1
```

現在，multi-stage image 有兩個 reference，一個是 multi-stage，另一個是 michaelchenn1225/multi-stage:v1：
```bash
docker image ls --filter reference=multi-stage --filter reference=*/multi-stage:v1
```
```text
REPOSITORY                     TAG       IMAGE ID       CREATED      SIZE
michaelchenn1225/multi-stage   v1        8a2c2ada0aa4   2 days ago   7.15MB
multi-stage                    latest    8a2c2ada0aa4   2 days ago   7.15MB
```

> 兩者都指向相同的 IMAGE ID

3. 上傳 image

```bash
docker push <docker_hub_username>/<image_name>:<tag>
```
例如：
```bash
docker push michaelchenn1225/multi-stage:v1
```

push image 也需要優化 dockerfile，因為如果 Image layer 有太多改動，docker hub 就會重新上傳。

### 本機的 registry

如果你想要在本機建立一個 registry，可以透過 docker run 來達成：

```bash
docker run -d -p 5000:5000 --restart=always --name local-registry registry
```

這樣我們就能將檔案 push 到 localhost:5000 了。

將 local-registry 命名：

* Windows：
```powershell
Add-Content -Value "127.0.0.1 registry.local" -Path "C:\Windows\System32\drivers\etc\hosts"
```

* Linux：
```bash
echo "\n127.0.0.1 registry.local" | sudo tee -a /etc/hosts
```

測試是否成功：
```bash
ping registry.local
```

push image 到 local registry：
```bash
docker tag multi-stage registry.local:5000/multi-stage:v1
docker push registry.local:5000/multi-stage:v1
```

這個 registry 沒有設置相關權限，所以任何人都可以 push image 到這個 registry，之後會再介紹如何設置權限。