# Backup

*注1: lain 的备份功能是选择性开启的特性，若未开启，volume 备份策略会被忽视。*

*注2: portal 类型的 container 是不支持 volume 备份的，所以其备份策略会被忽视。*

我们可在lain.yaml中配置一些目录持久化。对于持久化的目录，用户对这些目录定制备份策略。lain会根据用户策略执行定期备份

## lain.yaml配置

```
  volumes:  # 与 persistent_dirs 等效
    - /var/log # 默认普通的volume
    - /etc/nginx: # 特殊配置
        backup_full: # 全量备份
          schedule: "* * * * 0" # 定时策略
          expire: "30d" # 过期时间,数字+单位, 如10d表示10天, 10h表示10小时, 3m表示3分钟
          pre_run: "backup.sh" # pre hook, 备份执行前调用
          post_run: "end-backup.py" # post hook, 备份结束后调用
        backup_increment: # 增量备份
          schedule: "0 23 * * *" # 定时策略
          expire: "10d" # 过期时间,数字+单位, 如10d表示10天, 10h表示10小时, 3m表示3分钟

```

## Best Practice

lain在执行备份任务时, 会对指定的目录进行压缩打包。这打包过程中, 如果目录中的文件发生了写操作，可能会导致打包失败，进而导致备份失败。

所以在备份的时间段里，要确保被备份的目录中的文件不可以有任何写操作。

我们可以为备份专门指定一个 volume。如:

```
  volumes:
    - /etc/nginx
    - /etc/nginx-backup:
        backup:
          schedule: "* 23 * * *" # 备份策略
          expire: "10d" # 过期时间,数字+单位, 如10d表示10天, 10h表示10小时, 3m表示3分钟
          pre_run: "backup.sh" # pre hook, 备份执行前调用
          post_run: "end-backup.py" # post hook, 备份结束后调用
```

上例中的 `/etc/nginx-backup` 是专门为备份准备的。利用 pre_run 选项，在执行备份前，`backup.sh` 程序将 `/etc/nginx` 中需要备份的内容拷贝到 `/etc/nginx-backup` 目录下；然后 lain 对 `/etc/nginx-backup` 进行备份。

这样就避免了备份时可能出现读写冲突的隐患。

在执行恢复时，lain恢复的是 `/etc/nginx-backup` 目录，用户需自行将 `/etc/nginx-backup` 下的文件挪到 `/etc/nginx` 使用。

以 MySQL 备份为例, 备份及恢复的大体流程如下:

**备份:**

1. pre_run 脚本执行 mysqldump，将数据备份到 `/backup` 下.
2. lain 对 `/backup` 进行备份.
3. post_run 脚本将 `/backup` 目录清空.

**恢复:**

1. 使用 `lain backup recover...` 执行恢复操作.
2. 等恢复完毕, 历史备份数据会恢复到 `/backup` 下.
3. `lain enter` 进入到 container 内部手动使用备份恢复数据


## `lain backup`命令手册

laincli 中提供了备份相关的一些指令, 我们已下面这个备份配置为基础，举例说明各个指令的用法:

```yaml
proc.ipaddr:
  cmd: python service.py
  port: 10000
  volumes:
    - /logs:
        backup_full:
          schedule: "*/5 * * * *"
          expire: 1h
    - /incremental:
        backup_increment:
          schedule: "* * * * *"
          expire: 30m
```

### `lain backup jobs PHASE`

列出 local 区所有的备份任务。如:

```sh
lain backup jobs local
```

### `lain backup get PHASE PROC VOLUME`

- `PROC`: proc 名字
- `VOLUME`: volume 路径，即在 `lain.yaml` 中定义的 volume 的路径。

获取某 proc 下的某个 volume 的备份文件列表。

如:

```sh
lain backup get local ipaddr /logs
```

### `lain backup run PHASE ID`

- `ID`: 任务 ID，`lain get` 可看到各个任务的 ID

即刻执行一个备份任务。

如:

```sh
lain backup run local c0a84d1527dc8c51c0f4e61c349f1eb3ad869ed9
# 输出 146250824364e9cd3ead0bbbe70d7d00269ec4551d, 这个输出值是任务记录的id, 可通过lain backup records local ID 来查询

```

### `lain backup records [-n NUM] PHASE [RECORD_ID]`

- `-n/--num`: 返回 records 的总个数。默认为10，只有当 RECORD_ID 没有指定时候才可用。
- `RECORD_ID`: 记录 ID，执行 `lain run` 返回的 ID 号。

获取最新的备份执行记录。 例如，我们可以使用任务 id 号获取刚刚手动执行的备份任务:

```sh

lain backup records local 146250824364e9cd3ead0bbbe70d7d00269ec4551d
```

### `lain backup delete PHASE PROC FILE [FILE ...]`

- `PROC`: proc 名字
- `FILE`: 备份文件名

删除一个 proc 的一个或多个备份。备份文件可以 通过`lain backup get ...` 获取。

如:

```sh
lain backup delete local ipaddr ipaddr-service-ipaddr-service.worker.ipaddr-1-logs-1462508400.tar.gz
```

### `lain backup recover PHASE PROC BACKUP ...`

- `PROC`: proc 名字
- `BACKUP`: 备份文件名。

备份恢复。备份会覆盖它所属的目录。

如，使用备份 `ipaddr-service-ipaddr-service.worker.ipaddr-1-logs-1462505700.tar.gz` 恢复 proc `ipaddr` 下的 volume。


```sh
lain backup recover local ipaddr ipaddr-service-ipaddr-service.worker.ipaddr-1-logs-1462505700.tar.gz
# 注: 恢复会清空当前volume里的所有文件
```

当 `BACKUP` 是一个增量备份时，命令最后可以追加增量备份内部的文件。表示只恢复部分文件。增量备份内部文件可使用 `lain backup list` 查看。
如:

```sh
lain backup recover local ipaddr ipaddr-service-ipaddr-service.worker.ipaddr-1-incremental time.log

# ipaddr-service-ipaddr-service.worker.ipaddr-1-incremental 是增量备份名字, 可由lain backup get 查看到
# time.log 是增量备份中的某个子文件, 可通过lain backup list ipaddr-service-ipaddr-service.worker.ipaddr-1-incremental 查看到
```

只要在 recover 指令最后追加了 `1+` 个文件参数，备份文件就会被当做增量备份处理，否则当做全量备份处理。


### `lain backup migrate -v VOLUME -t TO PHASE PROC BACKUP ...`

如果一个 proc 有多个 instance，它们之间的备份都是独立的。备份恢复只针对备份所属的 instance。若想让一个备份给其他的 instance 用，需要使用 migrate。

如，我们把 local 区 proc `ipaddr` 的一个1号 instance 的一个备份恢复到2号 instance 的 volume 下:

```sh
lain backup migrate -v /logs -t 2 local ipaddr ipaddr-service-ipaddr-service.worker.ipaddr-1-logs-1462515600.tar.gz
```

### `lain backup list PHASE PROC PATH`

- `PROC`: proc 名字
- `PATH`: 增量备份名，也可在其后设定子路径。如我们假设一个增量备份名为 `incremental-backup`，查看里面的子目录 `sub`，可设置 PATH 为 `incremental-backup/sub`

如果使用增量备份，我们可通过 `lain` 命令查看增量备份内部的文件内容。`lain backup get` 获取的备份列表中，可以看到非 `.tar.gz` 结尾的是增量备份.

```sh
lain backup list local ipaddr ipaddr-service-ipaddr-service.worker.ipaddr-1-incremental

# SIZE      FILENAME
# 4300      sub/
# 43        time.log
```
