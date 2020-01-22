# Build
## 代码仓库
* https://github.com/hyperledger/avalon 拉取代码
## 构建
* 根据文档进行构建https://github.com/hyperledger/avalon/blob/master/BUILD.md
* 需要在 docker/Dockerfile.tcf-dev中替换安装源为阿里源，还是有部分需要翻墙才能解决，暂时还是需要翻墙
* sudo docker-compose up --build 构建并启动docker
# 运行测试
* 在docker启动成功后，进入容器
```
docker exec -it containerId /bin/bash
cd $TCF_HOME/tests
python3 Demo.py --input_dir ./json_requests/ \
        --connect_uri "http://localhost:1947" work_orders/output.json
```
* 结果：Decryption result at client - You have a 71% risk of heart disease
