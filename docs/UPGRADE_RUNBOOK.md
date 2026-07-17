# 升级手册:合并上游开源分支 + 重建部署

> 本文档记录本 fork(`helun03/sub2api`)从上游 `Wei-Shaw/sub2api` 同步新版本、重建镜像并重新部署的标准流程。
> 下次「merge 开源分支并升级」直接照此执行即可。最近一次执行:**0.1.151 → 0.1.160**(2026-07-18,422 提交,2 个冲突已解决)。

---

## 0. 本机环境关键事实(执行前先知道)

| 项 | 值 |
|----|----|
| origin 远程 | `git@github.com:helun03/sub2api.git`(你的 fork) |
| upstream 远程 | `https://github.com/Wei-Shaw/sub2api.git`(开源上游) |
| **本 fork 唯一自定义** | 只有 1 个提交,只改 `deploy/docker-compose.yml`:host bind mounts(`/var/lib/sub2api/{app,postgres,redis}`)+ 本地镜像 `sub2api:local` + `extra_hosts: host.docker.internal`。**别让它被上游覆盖** |
| 运行配置文件 | `/var/lib/sub2api/app/config.yaml`(权限 600,属主 `ubuntu:netdev`,**读写需 sudo**) |
| 配置加载方式 | **viper**(`SetDefault` + `ReadInConfig`)→ 局部配置块只覆盖写出来的 key,其余默认值不受影响 → **加局部 `gateway:` 块是安全的** |
| 配置热重载 | **无** → 改 config 必须重启容器才生效 |
| 镜像构建 | 根目录 `Dockerfile`,`-tags embed`(前端打进二进制);`deploy/docker-compose.yml` 里 `image: sub2api:local` **无 build 段** → 需手动 build 并打 tag |
| 主机 / SELinux | Ubuntu 24.04,**SELinux 已禁用** → bind mount **不需要 `:Z`**(上游给 dev/local compose 加的 `:Z` 对本机无意义,可不跟) |
| 账号上游代理 | 账号走 xray SOCKS5 `socks5h://...@172.17.0.1:10808`(docker 网桥网关),所以 compose 需要 `host.docker.internal` |
| 构建加速代理(可选) | xray:HTTP `172.17.0.1:10809` / SOCKS5 `172.17.0.1:10808`,配置在 `/usr/local/etc/xray/config.json` |

---

## 1. 获取上游并评估改动

```bash
cd /home/admin/sub2api

# 首次需添加 upstream(已加过可跳过)
git remote add upstream https://github.com/Wei-Shaw/sub2api.git 2>/dev/null || true
git fetch upstream --tags

# 看版本号 & 分叉情况
git show upstream/main:backend/cmd/server/VERSION    # 上游新版本
cat backend/cmd/server/VERSION                       # 你当前版本
git log --oneline upstream/main..HEAD                # 你领先上游的提交(应只有那个 deploy 提交)
git log --oneline HEAD..upstream/main                # 上游领先你的提交(本次要同步的)
```

## 2. 冲突预检(关键:确认上游有没有动你定制的文件)

```bash
BASE=$(git merge-base HEAD upstream/main)

# 上游相对共同祖先,改了 deploy/ 哪些文件
git diff --stat $BASE..upstream/main -- deploy/

# 上游是否动过你定制的 docker-compose.yml(理想情况:输出为空 = 没动 = 不冲突)
git diff $BASE..upstream/main -- deploy/docker-compose.yml

# 干跑合并,扫冲突
git merge-tree --write-tree HEAD upstream/main | grep -i conflict || echo "✅ 无冲突"
```

> 0.1.138 经验:上游只改了 `docker-compose.dev.yml` / `docker-compose.local.yml`(加 `:Z`)和 `config.example.yaml`(新增调度配置),**没碰 `docker-compose.yml`**,所以零冲突。
> 0.1.160 经验:上游修改了 `docker-compose.yml` 的镜像名并新增 `ENABLE_SERVER_TIMING`,与本地镜像定制产生冲突;解决时保留 `sub2api:local` / host bind mounts / `extra_hosts`,吸收新增环境变量。`.gitignore` 也因双方新增文档例外产生冲突,应保留双方规则。
> 若将来上游确实改了 `docker-compose.yml`,merge 时手动解冲突:**保留你的 host bind mounts / `sub2api:local` / `extra_hosts`,只吸收上游的其他改动**。

## 3. 合并并推送

```bash
git merge upstream/main --no-edit        # 本 fork 用 merge(保留双方历史,普通 push 即可)

# 校验合并结果
cat backend/cmd/server/VERSION                       # 应为新版本号
git log --oneline | grep "host bind mounts"          # 确认你的定制提交还在
grep -E "image:|var/lib/sub2api|host.docker.internal" deploy/docker-compose.yml   # 确认 compose 仍是你的版本
git status -s                                        # 应干净

git push origin main
```

## 4. (按需)同步上游新增的配置项到运行 config.yaml

上游每个版本可能在 `deploy/config.example.yaml` 新增配置项。**默认值通常保持旧行为**,不强制启用。要启用某项时:

```bash
# 看上游这次 config.example 新增了什么
git diff $BASE..upstream/main -- deploy/config.example.yaml

# 改运行配置前先备份(必须)
sudo cp -v /var/lib/sub2api/app/config.yaml \
           /var/lib/sub2api/app/config.yaml.bak.$(date +%Y%m%d-%H%M%S)

# 追加需要的配置块(viper 会合并默认值,只写要改的 key 即可)
sudo tee -a /var/lib/sub2api/app/config.yaml > /dev/null <<'EOF'
gateway:
    scheduling:
        prefer_soonest_reset: true        # anthropic/通用调度:use-it-or-lose-it
    openai_ws:
        scheduler_score_weights:
            reset: 0.5                     # OpenAI 调度器最早重置权重(>0 启用)
EOF
```

> **确认配置项作用域**:同名 key 可能分属不同调度器。查 viper key 与消费位置:
> ```bash
> grep -n "<配置名>" backend/internal/config/config.go        # 看 viper.SetDefault 的完整 key 路径
> grep -rn "<结构体字段名>" backend/internal/service/*.go      # 看哪条调度路径消费它
> ```
> 例:`prefer_soonest_reset` → `gateway.scheduling.*` → `gateway_service.go`(anthropic/通用);
> `scheduler_score_weights.reset` → `gateway.openai_ws.*` → `openai_account_scheduler.go`(仅 OpenAI)。

## 5. 重建镜像(带版本号区分)

```bash
cd /home/admin/sub2api
VER=$(tr -d '\r\n' < backend/cmd/server/VERSION)

# 双标签:版本 tag 追溯,:local 供 compose 使用。显式传 VERSION 保证 --version 输出确定
# (根 Dockerfile 已改为 scripts/resolve-version.sh:优先精确 git tag,否则回退 VERSION 文件;
#  merge 后的 HEAD 不在 tag 上,不传 VERSION 会回退到 VERSION 文件,但显式传更稳)
#
# ⚠️ 若 shell 里设了 http_proxy/https_proxy(比如刚才为了 git fetch 开了 xray),
#    docker CLI 会把它们自动转成 build-arg 塞进构建 → 踩下面的 goproxy.cn 坑!
#    构建时务必用 env -u 剥掉(git fetch 需要代理,docker build 要直连):
env -u http_proxy -u https_proxy -u all_proxy -u no_proxy \
    -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY -u NO_PROXY \
    docker build --build-arg VERSION=$VER -t sub2api:$VER -t sub2api:local -f Dockerfile .
# (构建久,可加 nohup ... > /tmp/sub2api-build-$VER.log 2>&1 & 后台跑再 tail 日志)
```

> ⚠️ **别给这个构建挂 xray 代理(踩过坑,0.1.142;0.1.151 再次确认)**。Dockerfile 里 `GOPROXY=https://goproxy.cn,direct`
> 本身就是国内镜像、直连最快;一旦 `HTTP_PROXY/HTTPS_PROXY` 生效(**注意:shell 里 export 的代理变量
> 会被 docker CLI 自动转发为 build-arg,不用显式 `--build-arg` 也会中招**),Go 会把对 goproxy.cn 的请求也塞进
> xray 隧道,`go mod download` 的大量并发连接直接把 xray 打到 `172.17.0.1:10809: connection refused`,
> 整个构建失败。**默认直连即可**(前端 npm 直连实测 ~1.2s,Go 走 goproxy.cn 也快)。
>
> 真遇到某个源直连卡住需要代理时,**必须把 goproxy.cn 及 Go module 后端排除**,否则重蹈覆辙:
> ```bash
> docker build --build-arg VERSION=$VER \
>   --build-arg HTTP_PROXY=http://172.17.0.1:10809 \
>   --build-arg HTTPS_PROXY=http://172.17.0.1:10809 \
>   --build-arg NO_PROXY=localhost,127.0.0.1,172.17.0.1,goproxy.cn,.goproxy.cn,proxy.golang.org,sum.golang.org \
>   -t sub2api:$VER -t sub2api:local -f Dockerfile .
> ```
> 判断是不是网络瓶颈:`curl -s -o /dev/null -w "%{time_total}s\n" https://registry.npmjs.org/`。直连够快就别上代理。

## 6. 重新部署

```bash
cd /home/admin/sub2api/deploy
docker compose up -d sub2api     # config 已通过 bind mount 挂载,无需额外操作
```

## 7. 验证

```bash
docker ps --filter name=sub2api --format '{{.Names}}\t{{.Image}}\t{{.Status}}'   # 应 healthy
docker exec sub2api /app/sub2api --version 2>&1 | head -1                          # 确认运行版本号
docker exec sub2api grep -E "prefer_soonest_reset|reset:" /app/data/config.yaml   # 确认新配置被加载
docker logs sub2api --since 90s 2>&1 | grep -iE "started|error|fatal|panic"       # 启动无报错
```

全部正常即升级完成。

---

## 8. 回滚

```bash
# 回滚配置
sudo cp /var/lib/sub2api/app/config.yaml.bak.<时间戳> /var/lib/sub2api/app/config.yaml

# 回滚镜像:用上一个版本 tag(如还在)重启
cd /home/admin/sub2api/deploy
# 临时把 docker-compose.yml 的 image 改成 sub2api:<旧版本>,或重新 tag:
#   docker tag sub2api:0.1.137 sub2api:local
docker compose up -d sub2api

# 回滚代码合并(尚未基于其继续开发时)
git reset --hard b8b45ed1   # 合并前的 HEAD;按需替换
```

---

## 注意点速查

- **重启有几十秒中断**(单实例),挑低峰期执行。
- **`prefer_soonest_reset` 的行为副作用**:同优先级下,有活跃会话窗口的 oauth(订阅)账号会被优先用尽,无窗口的 apikey 账号退为兜底。想两个号继续均衡轮换就别开,或给 apikey 账号设更大的 `priority` 数值(天然兜底)。
- **改 ent schema / interface 时**:照常 `cd backend && make generate` 并补全 test stub(见 CLAUDE.md)。
- **不要把 SELinux `:Z` 加到本机的 `docker-compose.yml`**:本机 SELinux 已禁用,加了无用。
- 运行 config.yaml 含明文 DB/Redis 密码,**备份文件别提交进 git**。
