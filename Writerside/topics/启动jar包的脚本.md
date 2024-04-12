# 启动 jar 包的脚本

创建可执行文件, 将下面的脚本内容复制到文件中;
```Bash
vim rj
```

```Bash
chmod +x ./rj
```

使用示例
```Bash
./rj example.jar
./rj -e test example.jar
```


```Bash
#!/bin/bash

ENVIRONMENT="test"
JAR_FILE=""

while getopts ":e:" opt; do
  case ${opt} in
    e ) 
      ENVIRONMENT="$OPTARG"
      ;;
    \? ) 
      echo "Invalid option: $OPTARG" 1>&2
      exit 1
      ;;
    : )
      echo "Invalid option: $OPTARG requires an argument" 1>&2
      exit 1
      ;;
  esac
done
shift $((OPTIND -1))

JAR_FILE="$1"

if [ -z "$JAR_FILE" ]; then
    echo "No argument supplied. Please specify the jar file."
    exit 1
fi

# 提取 JAR 文件的基本名称作为日志文件名
JAR_NAME=$(basename "$JAR_FILE" .jar)

nohup java -Xms512m -Xmx1024m -XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=1024m -jar "$JAR_FILE" --spring.profiles.active="$ENVIRONMENT" > /dev/null 2>&1 &
```

```Bash
nohup java -Xms512m -Xmx1024m -XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=1024m -jar "$JAR_FILE" --spring.profiles.active="$ENVIRONMENT" >> "./logs/$JAR_NAME.log" &
```