使用fastgpt

用到到东西有pg，fastgpt，mysql，（openwebui），（ollama），m3e，mongo，oneapi

上面这些东西用docker compose构建

![截屏2024-04-15 上午9.18.44](/Users/xcvfss/Documents/截屏2024-04-15 上午9.18.44.png)







## 下载配置文件

有两个配置文件（有链接可以直接下载）

* mkdir fastgpt（创建目录）
* cd fastgpt（进入目录）

* 在当前目录导入docker-conpose.yml配置文件

curl -O https://raw.githubusercontent.com/labring/FastGPT/main/files/deploy/fastgpt/docker-compose.yml

==这个配置文件是用于在docker搭建fastgpt的整个运行环境（就是在虚拟的环境构建模型编排，拉取pg、fastgpt、mysql、m3e、mongo、oneapi的镜像按照docker- compose.yml里面的配置要求构建容器，各个容器构建完成后，他们之间的接口会关联起来，也就是打通了）==

==可以打个比方，小强（fastgpt）邀请几个人（pg、mysql、m3e、mongo、oneapi）一起玩一款游戏，这款游戏在游戏公司的服务器（docker）运行，于是这群人就要创建自己的线上游戏角色（各自的容器），这时候将他们匹配到一起的就靠游戏公司调整（docker- compose.yml配置文件）==

* 导入config.json配置文件

curl -O https://raw.githubusercontent.com/labring/FastGPT/main/projects/app/data/config.json



---

### docker-conpose.yml

```yaml
# 数据库的默认账号和密码仅首次运行时设置有效
# 如果修改了账号密码，记得改数据库和项目连接参数，别只改一处~
# 该配置文件只是给快速启动，测试使用。正式使用，记得务必修改账号密码，以及调整合适的知识库参数，共享内存等。

version: '3.3'
services:
  pg:
    image: ankane/pgvector:v0.5.0 # git
    # image: registry.cn-hangzhou.aliyuncs.com/fastgpt/pgvector:v0.5.0 # 阿里云
    container_name: pg
    restart: always
    ports: # 生产环境建议不要暴露
      - 5432:5432
    networks:
      - fastchat
    environment:
      # 这里的配置只有首次运行生效。修改后，重启镜像是不会生效的。需要把持久化数据删除再重启，才有效果
      - POSTGRES_USER=username
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=postgres
    volumes:
      - ./pg/data:/var/lib/postgresql/data
  mongo:
    # image: registry.cn-hangzhou.aliyuncs.com/fastgpt/mongo:5.0.18
    image: mongo:5.0.18
    container_name: mongo
    restart: always
    ports:
      - 27017:27017
    networks:
      - fastchat
    command: mongod --keyFile /data/mongodb.key --replSet rs0
    environment:
      - MONGO_INITDB_ROOT_USERNAME=myusername
      - MONGO_INITDB_ROOT_PASSWORD=mypassword
    volumes:
      - ./mongo/data:/data/db
    entrypoint:
      - bash
      - -c
      - |
        openssl rand -base64 128 > /data/mongodb.key
        chmod 400 /data/mongodb.key
        chown 999:999 /data/mongodb.key
        echo 'const isInited = rs.status().ok === 1
        if(!isInited){
          rs.initiate({
              _id: "rs0",
              members: [
                  { _id: 0, host: "mongo:27017" }
              ]
          })
        }' > /data/initReplicaSet.js
        # 启动MongoDB服务
        exec docker-entrypoint.sh "$$@" &

        # 等待MongoDB服务启动
        until mongo -u myusername -p mypassword --authenticationDatabase admin --eval "print('waited for connection')" > /dev/null 2>&1; do
          echo "Waiting for MongoDB to start..."
          sleep 2
        done

        # 执行初始化副本集的脚本
        mongo -u myusername -p mypassword --authenticationDatabase admin /data/initReplicaSet.js

        # 等待docker-entrypoint.sh脚本执行的MongoDB服务进程
        wait $$!
  fastgpt:
    container_name: fastgpt
    image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt:v4.7 # git
    # image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt:v4.7 # 阿里云
    ports:
      - 3000:3000
    networks:
      - fastchat
    depends_on:
      - mongo
      - pg
    restart: always
    environment:
      # root 密码，用户名为: root。如果需要修改 root 密码，直接修改这个环境变量，并重启即可。
      - DEFAULT_ROOT_PSW=1234
      # AI模型的API地址哦。务必加 /v1。这里默认填写了OneApi的访问地址。
      # - OPENAI_BASE_URL=http://host.docker.internal:3001/v1
      - OPENAI_BASE_URL=http://oneapi:3000/v1
      # AI模型的API Key。（这里默认填写了OneAPI的快速默认key，测试通后，务必及时修改）
      - CHAT_API_KEY=sk-4XD8NTra0pOOkclF4dBfEaB9B26048349c1f7a47431cD93e
      # 数据库最大连接数
      - DB_MAX_LINK=30
      # 登录凭证密钥
      - TOKEN_KEY=any
      # root的密钥，常用于升级时候的初始化请求
      - ROOT_KEY=root_key
      # 文件阅读加密
      - FILE_TOKEN_KEY=filetoken
      # MongoDB 连接参数. 用户名myusername,密码mypassword。
      - MONGODB_URI=mongodb://myusername:mypassword@mongo:27017/fastgpt?authSource=admin
      # - MONGODB_URI=mongodb://myusername:mypassword@host.docker.internal:27017/fastgpt?authSource=admin
      # pg 连接参数
      - PG_URL=postgresql://username:password@pg:5432/postgres
      # - PG_URL=postgresql://username:password@host.docker.internal:5432/postgres
    volumes:
      - ./config.json:/app/data/config.json
      - ./fastgpt/tmp:/app/tmp
  mysql:
    image: mysql:8.0.36
    container_name: mysql
    restart: always
    ports:
      - 3306:3306
    networks:
      - fastchat
    command: --default-authentication-plugin=mysql_native_password
    environment:
      # 默认root密码，仅首次运行有效
      MYSQL_ROOT_PASSWORD: oneapimmysql
      MYSQL_DATABASE: oneapi
    volumes:
      - ./mysql/data:/var/lib/mysql
      - ./mysql/conf:/etc/mysql/conf.d
      - ./mysql/logs:/var/log/mysql
  oneapi:
    container_name: oneapi
    image: ghcr.io/songquanpeng/one-api:latest
    ports:
      - 3001:3000
    depends_on:
      - mysql
    networks:
      - fastchat
    restart: always
    environment:
      # mysql 连接参数
      - SQL_DSN=root:oneapimmysql@tcp(mysql:3306)/oneapi
      # 登录凭证加密密钥
      - SESSION_SECRET=oneapikey
      # 内存缓存
      - MEMORY_CACHE_ENABLED=true
      # 启动聚合更新，减少数据交互频率
      - BATCH_UPDATE_ENABLED=true
      # 聚合更新时长
      - BATCH_UPDATE_INTERVAL=10
      # 初始化的 root 密钥（建议部署完后更改，否则容易泄露）
      - INITIAL_ROOT_TOKEN=fastgpt
    volumes:
      - ./oneapi:/data
  m3e:
    image: registry.cn-hangzhou.aliyuncs.com/fastgpt_docker/m3e-large-api:latest
    container_name: m3e
    restart: always
    ports:
      - 6008:6008
    networks:
      - fastchat
networks:
  fastchat:  
```



---



### 重点内容



1. 当使用 Docker Compose 来管理多个容器时，Compose 会自动创建一个默认网络，使得在同一 `docker-compose.yml` 文件中定义的服务能够相互访问。这样，不同容器之间可以通过服务名称来进行通信，而无需暴露具体的 IP 地址

2. 指定各个容器共用的网络名称是fastchat

3. oneapi 是依赖mysql才能启动，所以要在environment里面设置连接mysql的连接参数

* `- SQL_DSN=root:oneapimmysql@tcp(mysql:3306)/oneapi`

  1. 这是连接 MySQL 数据库的数据源名称（DSN）。

  2. `root:oneapimmysql` 是数据库连接的用户名和密码，即 root 用户名为 `root`，密码为 `oneapimmysql`。

  3. `@tcp(mysql:3306)` 指定使用 TCP 协议连接到主机名为 `mysql`、端口为 `3306` 的 MySQL 服务器。

  4. `/oneapi` 是要连接的数据库名称。

---



4. fastgpt是依赖mango和pg数据库才能启动，所以要在environment里面设置连接mango和pg的连接参

   * PG_URL=postgresql://username:password@pg:5432/postgres

   - MONGODB_URI=mongodb://myusername:mypassword@mongo:27017/fastgpt?authSource=admin

     * `mongodb://`：MongoDB 连接的协议

     * `@mongo`：指定 MongoDB 服务器的主机名称，这里是 `mongo`(docker里创建的mongo容器)

     * `:27017`：MongoDB 默认的端口号

   

   * OPENAI_BASE_URL=http://oneapi:3000/v1        这里是指定 AI 模型的 API 请求（这个AI模型由oneapi链接，实际指定的是oneapi这个容器的本地3000端口。
     * `http://oneapi:3000`是一个oneapi容器的基本服务地址，/v1是其中的一个端点路径

---



5. 容器映射volumes:

   - ./config.json:/app/data/config.json
   - ./fastgpt/tmp:/app/tmp

     * ./ 是启动docker-conplse的当前地址

     * 冒号前面是本机的地址，冒号后面是容器的地址，`本机的地址是我，后面的地址是镜子里的我，镜子被盖住了（关闭容器）我还在不会消失`






## 程序启动步骤：

1. 命令`docker-conplse up -d`是通过docker利用安装好的镜向来创建pg、fastgpt、mysql、m3e，mongo、oneapi各自的容器，各个容器对应的接口都被映射到本地的端口，从而调取本地设备来运算。
2. `fastgpt容器`连通`pg容器和mongo容器`分别用的是`postgresql连接协议`和`MongoDB连接协议`
   * mongo容器的对外暴露的端口号是27017，（mongo容器连接的网络是fastchat）
   * pg容器对外暴露的端口号是5432，（pg容器连接的网络是fastchat）
3. `fastgpt容器`连通`oneapi容器`用的是`http连接协议`
   * oneapi容器对外暴露的端口号是3000，（onepai容器连接的网络是fastchat）
   * `OPENAI_BASE_URL=http://oneapi:3000/v1`
4. `oneapi容器`依赖`mysql容器`,然后通过tcp协议与其连接
   * `- SQL_DSN=root:oneapimmysql@tcp(mysql:3306)/oneapi`

---

<img src="/Users/xcvfss/Documents/截屏2024-04-15 晚上9.30.49.png" alt="截屏2024-04-15 晚上9.30.49" style="zoom:50%;" />

注意：我配置的m3e向量模型是安装在docker里面，而不是本地（可以迁移到本地）

所以我在oneapi上设置m3e渠道的时候，代理是设置为`http://host.docker.internal:6008`估计就是oneapi连接到本机3001端口，然后又连接到m3e容器的6008端口，最后映射到本机的6008端口跑向量模型。==注意==：鉴权密钥是：`sk-aaabbbcccdddeeefffggghhhiiijjjkkk`(默认密钥)

---

5. 配置config.josn文件
   * 要用到什么模型，复制粘贴里面现成的配置，然后改掉模型名称就好了

> ![截屏2024-04-15 晚上10.24.44](/Users/xcvfss/Documents/截屏2024-04-15 晚上10.24.44.png)
>
> 你看，gpt3.5turbo和我自己本地下载的llama2:7b只是模型名称上有区别，其他参数请自行研究

6. 输入命令跑起来，`docker-compose up -d` 
7. 浏览器输入`localhost:3001`进入onegpt的webUI

> ![截屏2024-04-15 晚上10.29.11](/Users/xcvfss/Documents/截屏2024-04-15 晚上10.29.11.png)
>
> 用户名：root	密码一般默认是123456
>
> <img src="/Users/xcvfss/Documents/截屏2024-04-15 晚上10.33.28.png" alt="截屏2024-04-15 晚上10.33.28" style="zoom:50%;" />
>
> 先创建令牌——>添加新临牌
>
> <img src="/Users/xcvfss/Documents/截屏2024-04-15 晚上10.35.11.png" alt="截屏2024-04-15 晚上10.35.11" style="zoom:50%;" />
>
> 添加一个名称就可以了，其他留空或者无限
>
> <img src="/Users/xcvfss/Documents/截屏2024-04-15 晚上10.39.26.png" alt="截屏2024-04-15 晚上10.39.26" style="zoom:50%;" />
>
> 返回令牌页面，点击这个复制⬇️
>
> <img src="/Users/xcvfss/Documents/截屏2024-04-15 晚上10.42.28.png" alt="截屏2024-04-15 晚上10.42.28" style="zoom:50%;" />
>
> 然后打开最开始下载的`docker-conpose.yml`配置文件，将刚刚复制的密钥粘贴到`CHAT_API_KEY=`后面，然后保存文件，终端用`docker-compose down`命令杀掉所有容器，再输入命令`docker-compose up -d`重新全部启动，回到⬆️这个页面，继续
>
> （以我浅薄水准的理解大致解析一下，注意有可能是错的：通过oneapi的webui界面的上述操作，生成了一个可供外部连接的API密钥，把这个密钥导入到fastgpt的配置文件中，fastgpt就可以连接到oneapi调取服务，而oneapi则通过渠道调取外部或者本地的大模型给fastgpt提供服务）
>
> 于是我们下面继续配置==渠道==栏
>
> 
>
> 
>
> ![截屏2024-04-15 晚上10.45.44](/Users/xcvfss/Documents/截屏2024-04-15 晚上10.45.44.png)
>
> 进入渠道页面——>添加新渠道
>
> 我这边添加了3个渠道，一个是连接chatgpt的API，一个是ollama部署的本地大语言模型（llama2和qwen），一个是向量模型（m3e）
>
> ![截屏2024-04-15 晚上10.58.33](/Users/xcvfss/Documents/截屏2024-04-15 晚上10.58.33.png)
>
> 好，首先配置本地的（说实话，实际体验下来，因模型参数太少，电脑算力太低导致非常辣鸡），照着下面图片的信息填写，最后`密钥`那里的空白处填写`sk-key`
>
> <img src="/Users/xcvfss/Documents/截屏2024-04-15 晚上11.07.06.png" alt="截屏2024-04-15 晚上11.07.06" style="zoom:50%;" />
>
> 然后是chatgpt的渠道设置，密钥处填写你自己的API密钥（我自己的密钥就不展示了），添加模型后要在config.josn配置文件添加信息（和前面讲的添加方式一样，这个模型的配置信息好像本身就有）
>
> <img src="/Users/xcvfss/Documents/截屏2024-04-15 晚上11.08.35.png" alt="截屏2024-04-15 晚上11.08.35" style="zoom:50%;" />
>
> 最后是向量模型的渠道设置，密钥空白处处要填这个：`sk-aaabbbcccdddeeefffggghhhiiijjjkkk`
>
> <img src="/Users/xcvfss/Documents/截屏2024-04-15 晚上11.12.18.png" alt="截屏2024-04-15 晚上11.12.18" style="zoom:50%;" />
>
> 好了，oneapi配好了，然后浏览器输入`localhost：3000`进入fastgpt的webUI界面，用户名是root，密码是1234
>
> <img src="/Users/xcvfss/Documents/截屏2024-04-15 晚上11.16.09.png" alt="截屏2024-04-15 晚上11.16.09" style="zoom:50%;" />





## 配置过程中遇到的麻烦：

1、虚拟文件共享问题

> * Docker -> Settings ->  Resources ->  File sharing
> * 添加共享目录，用于挂载容器
> * 若是没有设置会报错 error：The path  is not shared from the host and is not known to Docker.
> * ![截屏2024-04-15 下午6.27.29](/Users/xcvfss/Documents/截屏2024-04-15 下午6.27.29.png)

2、mongo版本问题

> * 报错信息：response error: Operation `users.findOne()` buffering timed out after 10000ms
> * 更换版本到`mongo:5.0.18`可以用，不要用`mongo:latest`

3、m2e在oneapi上的API设置问题

> <img src="/Users/xcvfss/Documents/截屏2024-04-15 晚上9.13.00.png" alt="截屏2024-04-15 晚上9.13.00" style="zoom:40%;" />

4、docker-compose网络设置

> `networks:`
>
> ​	`fastchat`
>
> 在配置文件里面把网络名称换一下就好了，具体原因不详！！！





