# Webrouter

webrouter是所有lain app的入口，包括layer1层的registry和console等。基于Tengine实现。

## Scale

webrouter可以启多个instance来实现High Avaliable。 由于webrouter的特殊身份，无法像普通的lain app简单的scale，而且其在scale过程中所有的web类型的lain app都可能会出现短暂的无法访问。

因 swarm 是动态决定一个 container 部署到哪台机器，而 webrouter 在启动之前是需要配置文件存在的。因此我们需要确保集群的所有 node 上有一份 webrouter 的配置，以确保 swarm 无论在哪台机器上启动 webrouter ，能都能正常工作。

或者，我们可以通过 `docker -H swarm.lain:2376 info` 查看各个节点的资源使用情况，预测下 webrouter 会被部署到哪个节点上。

具体操作步骤如下:

1. 在所有 node 上执行:
   ```sh
   cat /etc/rsyncd.secrets  | cut -d ':' -f 2 > /tmp/pass && chmod 600 /tmp/pass
   mkdir -p /data/lain/volumes/webrouter/webrouter.worker.worker/2/
   rsync -az --password-file  /tmp/pass rsync://lain@BOOTSTRAP_NODE/lain_volume/webrouter/webrouter.worker.worker/1/ /data/lain/volumes/webrouter/webrouter.worker.worker/2/
   ```

2. 利用swarm，在所有node上pull webrouter image，确保每个节点上都有webrouter image
   ```sh
   docker -H swarm.lain:2376 pull registry.DOMAIN/webrouter:VERSION
   ```
3. 使用laincli扩充instance数量(在webrouter项目目录下)
   ```sh
   lain scale -n NUMBER PHASE worker
   ```

## 故障排除

webrouter 异常时，会导致某些 web 类型的 pod 无法工作。这里我们以 appname 为 demo，procname 为 web 的 app 为例子。

大致的问题排查步骤如下:

1. 确保 demo 的 container 内部服务是正常运行的。
   ```sh
   # 进入 demo 的 container 内部
   docker exec -it DEMO_CONTAIENR_ID

   # 查看内部的进程运行情况是否正常
   ```

2. 查看 demo 的 container ip 地址和端口号。

   ```sh
   # 把 demo 和 CONTAINERID 换成对应值
   docker inspect CONTAINERID
   ```

   若发现 IP 地址为空, 说明这个 container 没有分配IP地址；可尝试执行`docker stop CONTRAINERID && sleep 5 && docker start CONTAINERID`解决。看是否能让 container 获取 IP。
   若始终没有 IP，考虑把 container remove 掉，等2分钟后 deployd 会自动启动一个新的 container，再对其进行检查。

3. 检查到 webrouter 的 container 内部，使用`curl`命令看是否可访问 demo。
   ```sh
   # 进入contaienr内部
   docker exec -it WEBROUTER_CONTAINER_ID bash

   # 尝试请求demo
   curl IP:PORT
   ```
   若不能，则是网络出了问题。请排查 [networkd](networkd.html)

4. 若2步中访问正常，查看 Tengine 的配置是否正常。
   ```sh
   # 进入 container 内部
   docker exec -it WEBROUTER_CONTAINER_ID bash

   # 查看配置
   cat /etc/nginx/upstreams/demo.upstreams
   ```
   若配置与1步中的配置不同，尝试检查 lainlet 返回的结果是否正常。
   ```sh
   curl -s lainlet.lain:9001/v2/webrouter/webprocs?appname=demo | python -m json.tool
   ```
   在结果中找到 IP 和 PORT 值，看与实际情况是否对应。若不正常，则异常出在了 [Lainlet](lainlet.html) 上。

5. 若第3步中正常，查看 webrouter watcher 是否正常。
   ```sh
   # 进入 container 内部
   docker exec -it WEBROUTER_CONTAINER_ID bash

   # 查看状态
   supervisorctl status
   # 重启watcher
   supervisorctl restart watcher
   ```


