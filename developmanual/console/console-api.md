## DEMO api

### `/api/v1/repos/`

1. `POST`
>注册一个新 App
```sh
curl /api/v1/repos/ -H "Content-Type: application/json" -d '{"appname":"hello"}'
```

2. `GET`
>获取当前所有已注册 App 信息
```sh
curl /api/v1/repos/
```

### `/api/v1/repos/:appName/`

1. `GET`
>获取已注册 App 的信息
```sh
curl /api/v1/repos/:appName/
```

### `/api/v1/repos/:appName/roles/`

1. `GET`
>根据 header 中传入的 access-token 获取 App 某个 user 的 role
```sh
curl /api/v1/repos/:appName/roles/
```

### `/api/v1/repos/:appName/roles/:username/`

1. `GET`
>获取 App 某个 user 的 role
```sh
curl /api/v1/repos/:appName/roles/:username/
```

### `/api/v1/repos/:appName/maintainers/`

1. `GET`
>获取 App 的maintainer信息以及角色
```sh
curl /api/v1/repos/:appName/maintainers/
```

2. `POST`
>增加 App 一个新的maintainer
```sh
curl -x POST /api/v1/repos/:appName/maintainers/ -H "Content-Type: application/json" -d '{"username": "username", "role": "role"}'
```

### `/api/v1/repos/:appName/maintainers/:username/`

1. `GET`
>获取 App 某个maintainer
```sh
curl /api/v1/repos/:appName/maintainers/:username/
```

2. `DELETE`
>删除 App 某个maintainer
```sh
curl -X DELETE /api/v1/repos/:appName/maintainers/:username/
```


### `/api/v1/apps/`

1. `POST`
>部署一个已注册 App，类型可以为 normal app 或者 resource 
```sh
curl /api/v1/apps/ -H "Content-Type: application/json" -d '{"appname":"hello"}'
```

2. `GET`
>获取当前所有已部署 App 信息以及状态，后可接参数 ?apptype=:apptype，apptype包括‘app, service, resource, resource-instance’    
```sh
curl /api/v1/apps/
```

### `/api/v1/apps/:appName/`

1. `GET`
>获取已部署 App 的信息以及状态
```sh
curl /api/v1/apps/:appName/
```

2. `PUT`
>更新 App 触发新版本部署
```sh
curl -X PUT /api/v1/apps/:appName/ -H "Content-Type: application/json" -d '{"meta_version": "1435893486-ead17ccee6d904f8e01f339e9e96c00812c70756"}'
```

3. `DELETE`
>下线 App 解除相关部署但保留注册信息, apptype='resource'则不允许下线
```sh
curl -X DELETE /api/v1/apps/:appName/
```

### `/api/v1/apps/:appName/procs/`

1. `GET`
>获取 App 所有 Procs 信息以及状态
```sh
curl /api/v1/apps/:appName/procs/
```

2. `POST`
>部署 App 中未成功部署的 proc
```sh
curl -X POST /api/v1/apps/:appName/procs/ -H "Content-Type: application/json" -d '{"procname": "client"}'
```

### `/api/v1/apps/:appName/procs/:procName/`

1. `GET`
>获取 proc 的信息以及状态
```sh
curl /api/v1/apps/:appName/procs/:procName
```

2. `PATCH`
>scale proc，或者修改占用 cpu, memory 数，
```sh
curl -X PATCH /api/v1/apps/:appName/procs/:procName/ -H "Content-Type: application/json" -d '{"num_instances":2, "cpu":1, "memory":"64M"}'
```

3. `DELETE`
>解除 proc 相关部署
```sh
curl -X DELETE /api/v1/apps/:appName/procs/:procName/
```



### `/api/v1/resources/:resourceName/instances/`

1. `GET`
>获取 Resource 所有 Instances 信息以及状态
```sh
curl /api/v1/resources/:resourceName/instances/
```
