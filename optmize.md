# 苹果CMS 性能优化

## 环境准备
### 硬件需求
+ CPU: 最低 4 核心 / 推荐 16 核心
+ 内存：最低 16 G / 推荐 64 G
+ 存储：最低 40 G SSD / 推荐 100 G SSD
### 安装 Docker

```bash
yum -y install yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo  
yum makecache fast  
yum -y install docker-ce
```

### 安装 Docker Compose
```bash
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.2.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

### 启动 Docker
```bash
systemctl start docker
```

### 安装 ES
1. 创建目录 /home/elk/
2. 将下面内容写入文件 /home/elk/docker-compose.yaml

    ```yaml
    version: "3.4"

    services:
        elasticsearch:
            container_name: es
            image: "docker.elastic.co/elasticsearch/elasticsearch:8.4.1"
            environment:
                - discovery.type=single-node
                - "ES_JAVA_OPTS=-Xms4096m -Xmx8192m"
            volumes:
                - /etc/localtime:/etc/localtime
                - /home/elk/data:/usr/share/elasticsearch/data
                - /home/elk:/usr/local/elk
            network_mode: host

        logstash:
            container_name: log
            depends_on:
                - elasticsearch
            image: "docker.elastic.co/logstash/logstash:8.4.1"
            volumes:
                - /home/elk/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
                - /home/elk:/usr/local/elk
            network_mode: host
            links:
                - elasticsearch

        kibana:
            container_name: kibana
            depends_on:
                - elasticsearch
            image: "docker.elastic.co/kibana/kibana:8.4.1"
            environment:
                - ELASTICSEARCH_URL=http://es:9200
            volumes:
                - /etc/localtime:/etc/localtime
                - /home/elk:/usr/local/elk
            network_mode: host
            links:
                - elasticsearch
    ```
3. 将所需的 jar 文件放入 /home/elk/ 目录内（ 如果 MySQL版本 不同，需要下载对应的 Jar 包，当前版本为 MySQL 5.1.22 ）
   1. mysql-connector-java-5.1.22-bin.jar
4. 将下面内容写入 /home/elk/logstash.conf
   1. 请替换 maccms 为你的数据库名称
   2. 请替换 root 为你的数据库用户名
   3. 请替换 password 为你的数据库密码
   4. 请替换 mysql-connector-java-8.0.24.jar 为匹配的版本文件（不要修改路径地址）
   5. 请留意 espassword 稍后需要替换为生成的 elastic 密码
        ```ini
        input {
            stdin{
            }
            jdbc {
                jdbc_connection_string => "jdbc:mysql://localhost:3306/maccms"
                jdbc_user => "root"
                jdbc_password => "password"
                jdbc_driver_library => "/usr/local/elk/mysql-connector-java-5.1.22-bin.jar"
                jdbc_driver_class => "com.mysql.jdbc.Driver"
                codec => plain {charset => "UTF-8"}
                use_column_value => true
                tracking_column => vod_id
                record_last_run => true
                last_run_metadata_path => "/usr/local/elk/station_parameter.txt"
                statement => "select * from mac_vod where vod_id > :sql_last_value"
                jdbc_paging_enabled => true
                jdbc_page_size => 500
                schedule => "* * * * *"
            }
        }
        filter{
            mutate { remove_field => ["@timestamp","@version"] }
        }
        output {
            elasticsearch {
                hosts => ["localhost:9200"]
                index => "mac_vod"
                document_id => "%{vod_id}"
                user => "elastic"
                password => "espassword"
            }
            stdout {
                codec => json_lines
            }
        }
        ```
5. 启动DockerCompose
    ```bash
    cd /home/elk/
    docker-compose up -d
    ```
6. 检查 ES 启动状态

    ```bash
    docker logs -f -n 100 es
    ```
    如果ES 启动失败，请检查 /home/elk 的权限，所用镜像需要用户组和用户权限为 1000
    ```bash
    chown 1000.1000 -R /home/elk/
    # 然后重启ES
    docker restart es
    # 再次检查ES 启动状态，如果有其它错误，请谷歌搜索解决方案
    docker logs -f -n 100 es 
    # 看到 [WARN] received plaintext http traffic on an https channel 就基本属于 ES 启动完成了
    # 但是由于官方镜像需要启用 SSL，所以我们需要对容器重建去除 SSL
    ```

7. 重新创建容器

    ```bash
    docker-compose down
    docker-compose up -d
    ```

8. 检查 ES 启动状态

    ```bash
    docker logs -f -n 100 es 
    # 看到 "INFO", "message":"Authentication of [elastic] was terminated by realm 就说明 ES 启动完成了
    ```

9. 初始化 ES 账户信息
   1.  进入容器内部 ``` docker exec -it es bash ```
   2.  重置 elastic 密码 ```elasticsearch-reset-password -u elastic``` , 出现询问的时候输入 y 继续
   3.  将显示的信息保存下来 ```
Password for the [elastic] user successfully reset.
New value: 2gzDJ-faafC1aq+iDWOh```
   4.  重置 kibana_system 密码 ```elasticsearch-reset-password -u kibana_system``` , 出现询问的时候输入 y 继续
   5.  将显示的信息保存下来 ```
Password for the [kibana_system] user successfully reset.
New value: ESHRzEeQUKqP3sbVzwoX```
10. 配置 Kibana
    1.  打开 IP:5601 页面（如果无法访问，请放通防火墙规则 ）
    2.  点击 Configure manually
    3.  修改地址为 http://localhost:9200
    4.  点击 Check address
    5.  输入 kibana_system 的密码
    6.  点击 Configure Elastic
    7.  执行 ``` docker logs kibana ``` 查看验证码
    8.  输入验证码
    9.  点击 Verify
    10. 账号为 elastic ，密码为之前重置的密码，登录
    11. 点击 Explore on my own
    12. 点击左上角 展开菜单，拉到最底下，选择 Dev Tools
    13. 粘贴下方代码
    ```json
    PUT mac_vod
    {
    "settings": {
        "number_of_shards": 16,
        "number_of_replicas": 1
    },
    "mappings": {
        "properties": {
        "group_id": {
            "type": "integer"
        },
        "type_id": {
            "type": "integer"
        },
        "type_id_1": {
            "type": "integer"
        },
        "vod_actor": {
            "type": "text"
        },
        "vod_area": {
            "type": "text"
        },
        "vod_author": {
            "type": "text"
        },
        "vod_behind": {
            "type": "text"
        },
        "vod_blurb": {
            "type": "text"
        },
        "vod_class": {
            "type": "text"
        },
        "vod_color": {
            "type": "text"
        },
        "vod_content": {
            "type": "text"
        },
        "vod_copyright": {
            "type": "integer"
        },
        "vod_director": {
            "type": "text"
        },
        "vod_douban_id": {
            "type": "integer"
        },
        "vod_douban_score": {
            "type": "text"
        },
        "vod_down": {
            "type": "integer"
        },
        "vod_down_from": {
            "type": "text"
        },
        "vod_down_note": {
            "type": "text"
        },
        "vod_down_server": {
            "type": "text"
        },
        "vod_down_url": {
            "type": "text"
        },
        "vod_duration": {
            "type": "text"
        },
        "vod_en": {
            "type": "text"
        },
        "vod_hits": {
            "type": "integer"
        },
        "vod_hits_day": {
            "type": "integer"
        },
        "vod_hits_month": {
            "type": "integer"
        },
        "vod_hits_week": {
            "type": "integer"
        },
        "vod_id": {
            "type": "integer"
        },
        "vod_isend": {
            "type": "integer"
        },
        "vod_jumpurl": {
            "type": "text"
        },
        "vod_lang": {
            "type": "text"
        },
        "vod_letter": {
            "type": "text"
        },
        "vod_level": {
            "type": "integer"
        },
        "vod_lock": {
            "type": "integer"
        },
        "vod_name": {
            "type": "text"
        },
        "vod_pic": {
            "type": "text",
            "index": false
        },
        "vod_pic_screenshot": {
            "type": "text",
            "index": false
        },
        "vod_pic_slide": {
            "type": "text",
            "index": false
        },
        "vod_pic_thumb": {
            "type": "text",
            "index": false
        },
        "vod_play_from": {
            "type": "text"
        },
        "vod_play_note": {
            "type": "text"
        },
        "vod_play_server": {
            "type": "text"
        },
        "vod_play_url": {
            "type": "text"
        },
        "vod_plot": {
            "type": "text"
        },
        "vod_plot_detail": {
            "type": "text"
        },
        "vod_plot_name": {
            "type": "text"
        },
        "vod_points": {
            "type": "integer"
        },
        "vod_points_down": {
            "type": "integer"
        },
        "vod_points_play": {
            "type": "integer"
        },
        "vod_pubdate": {
            "type": "text"
        },
        "vod_pwd": {
            "type": "text"
        },
        "vod_pwd_down": {
            "type": "text"
        },
        "vod_pwd_down_url": {
            "type": "text"
        },
        "vod_pwd_play": {
            "type": "text"
        },
        "vod_pwd_play_url": {
            "type": "text"
        },
        "vod_pwd_url": {
            "type": "text"
        },
        "vod_rel_art": {
            "type": "text"
        },
        "vod_rel_vod": {
            "type": "text"
        },
        "vod_remarks": {
            "type": "text"
        },
        "vod_reurl": {
            "type": "text"
        },
        "vod_score": {
            "type": "text"
        },
        "vod_score_all": {
            "type": "integer"
        },
        "vod_score_num": {
            "type": "integer"
        },
        "vod_serial": {
            "type": "text"
        },
        "vod_state": {
            "type": "text"
        },
        "vod_status": {
            "type": "integer"
        },
        "vod_sub": {
            "type": "text"
        },
        "vod_tag": {
            "type": "text"
        },
        "vod_time": {
            "type": "date",
            "format": "epoch_second"
        },
        "vod_time_add": {
            "type": "date",
            "format": "epoch_second"
        },
        "vod_time_hits": {
            "type": "date",
            "format": "epoch_second"
        },
        "vod_time_make": {
            "type": "date",
            "format": "epoch_second"
        },
        "vod_total": {
            "type": "integer"
        },
        "vod_tpl": {
            "type": "text"
        },
        "vod_tpl_down": {
            "type": "text"
        },
        "vod_tpl_play": {
            "type": "text"
        },
        "vod_trysee": {
            "type": "integer"
        },
        "vod_tv": {
            "type": "text"
        },
        "vod_up": {
            "type": "integer"
        },
        "vod_version": {
            "type": "text"
        },
        "vod_weekday": {
            "type": "text"
        },
        "vod_writer": {
            "type": "text"
        },
        "vod_year": {
            "type": "text"
        }
        }
    }
    }
    ```
    14. 点击编辑框右上角的执行按钮
    15. 如果右侧响应和下方一致，则索引创建成功
    ```json
    {
        "acknowledged": true,
        "shards_acknowledged": true,
        "index": "mac_vod"
    }
    ```
    16. 在 Management > Stack Management > Index Management 中可以看到索引已经创建成功
11. 修改 logstash.conf 中的 elastic 密码
12. 重启 logstash
    ```bash
    docker restart log 
    # 查看执行日志
    docker logs -f -n 100 log
    # 如果出现 导入日志，说明索引正常工作，可以关闭窗口了
    ```
13. 在 Kibana 中查看 索引的大小是否有增加，数据可能有延迟，稍微等待几分钟，如果正常就说明 数据导入 已经配置完成了

### 修改 MacCMS 的程序
1. 修改 Vod.php 中的 elastic 密码
2. 替换程序文件
   + 替换 /application/common/model/Vod.php
   + 替换 /application/common/model/Collect.php
