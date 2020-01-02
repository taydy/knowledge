**以下内容均假设 gitlab 本地域名为 `git.demo.com`**

## 安装 docker

略

## 安装 gitlab

一般情况下，为了避免服务器端口被占用，这里会将80端口和22端口改为自定义端口，https使用的443端口暂时未尝试。

### 启动 gitlab 容器

```sh
export GITLAB_HOME=`pwd`/gitlab
sudo docker run --detach \
    --hostname git.shuoyekeji.com \
    # --publish 443:443 --publish 80:80 --publish 22:22 \
    --publish 443:443 --publish 32786:80 --publish 32790:22 \
    --name gitlab \
    --restart always \
    --volume $GITLAB_HOME/config:/etc/gitlab \
    --volume $GITLAB_HOME/logs:/var/log/gitlab \
    --volume $GITLAB_HOME/data:/var/opt/gitlab \
    gitlab/gitlab-ce:latest
```

### 修改端口配置

进入容器内，修改 `/etc/gitlab/gitlab.rb`文件：

#### 修改 gitlab url

```shell
external_url 'http://git.shuoyekeji.com'
```

`host`修改为自己的域名地址或者 ip 地址。

####修改 nginx listen port

```shell
nginx['listen_port'] = 32786
```

####修改 ssh 端口号

```shell
gitlab_rails['gitlab_shell_ssh_port'] = 32790
```

修改完成后执行命令：

```shell
gitlab-ctl reconfigure
```



## 安装 gitlab runner

```shell
sudo docker run -d --name gitlab-runner --restart always \
   -v /srv/gitlab-runner/config:/etc/gitlab-runner \
   -v /var/run/docker.sock:/var/run/docker.sock \
   gitlab/gitlab-runner:latest
```

> **Tip:** On macOS, use `/Users/Shared` instead of `/srv`.

#### 获取 runner token

![image-20190325144814013](https://tva1.sinaimg.cn/large/006tNbRwgy1gai2o2vs83j31hm0u0h4u.jpg)

#### [注册 Runner](https://docs.gitlab.com/runner/register):

1. Run the following command:

   ```
    gitlab-runner register
   ```

2. Enter your GitLab instance URL:

   ```
    Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com )
    http://git.demo.com
   ```

3. Enter the token you obtained to register the Runner:

   ```
    Please enter the gitlab-ci token for this runner
    xxxxxxxxxx
   ```

4. Enter a description for the Runner, you can change this later in GitLab’s UI:

   ```
    Please enter the gitlab-ci description for this runner
    [hostame] my-runner
   ```

5. Enter the [tags associated with the Runner](https://docs.gitlab.com/ee/ci/runners/#using-tags), you can change this later in GitLab’s UI:

   ```
    Please enter the gitlab-ci tags for this runner (comma separated):
    my-tag,another-tag
   ```

6. Enter the [Runner executor](https://docs.gitlab.com/runner/executors/README.html):

   ```
    Please enter the executor: ssh, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, virtualbox, docker-ssh, shell:
    docker
   ```

7. If you chose Docker as your executor, you’ll be asked for the default image to be used for projects that do not define one in `.gitlab-ci.yml`:

   ```
    Please enter the Docker image (eg. ruby:2.1):
    alpine:latest
   ```

注册成功后在 gitlab 可以看到该 runner:

![image-20190325144915421](https://tva1.sinaimg.cn/large/006tNbRwgy1gai2o3l2ynj310y0akwh2.jpg)





### 以下为 runner 配置

```
## /etc/gitlab-runner/config.toml

[[runners]]
  name = "shared Runner local"
  url = "http://git.shuoyekeji.com/"
  token = "X_y8V9_yiU9YBaR2tXYo"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "go-runner-tools:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock", "/home/gitlab-runner"]
    extra_hosts = ["git.shuoyekeji.com:192.168.10.234"]
    network_mode = "host"
    pull_policy = "if-not-present"
    shm_size = 0
    dns=["192.168.10.234"]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
```


