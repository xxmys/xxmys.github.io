---
title: 基于docker搭建mysql集群
tags:
  - mysql
categories:
  - 数据库
permalink: /sql/751dhdmriu
date: 2025-08-14 09:16:01
---
# MySQL Group Replication 单主模式

## 一、说明

最少有三台服务器,**时间同步**，

- **服务器 A**（主 1）：`server-id=1`，
- **服务器 B**（主 2）：`server-id=2`，
- **服务器 C**（主 3）：`server-id=3`，

## 二、脚本

参考service_app/目录下集群docker_compose脚本，运行等待数据容器启动成功。[对sh执行文件进行权限授权]

```
version: "3.7"
services:
#本地镜像名称
    mysql1:
        image: mysql:8.0.42
        container_name: mysql1
        restart: always
        env_file: .env
        privileged: true
        volumes:
            - ./mysql1/data:/var/lib/mysql
            - ./mysql1/bootstrap.sh:/usr/local/bin/bootstrap.sh
        network_mode: "host"        
        stop_signal: SIGTERM
        # 等价于--stop-timeout
        stop_grace_period: 30s
        environment:
            - TZ=Asia/Shanghai
            - "MYSQL_DATABASE=iotbuilderdb"
            - "MYSQL_USER=admin"
            - "MYSQL_PASSWORD=sysware2018."
            - "MYSQL_ROOT_PASSWORD=root"
            - "NODE_IP=${local_node_ip}"
            - "SEED_NODES=${cluster_node1_ip}:33061,${cluster_node2_ip}:33061,${cluster_node3_ip}:33061"
            - "NODE_ID=1"
            - "DELAY_MIN=10"
            - "DELAY_MAX=30"
        command: ["bash", "-c", "/usr/local/bin/bootstrap.sh"]
    mysql-router:
        image: 	container-registry.oracle.com/mysql/community-router:8.4
        container_name: mysql-router
        hostname: mysql-router
        volumes:
          - ./mysql1/mysqlrouter.conf:/etc/mysqlrouter/mysqlrouter.conf
        network_mode: "host"
        restart: always
        environment:
            - MYSQL_HOST=${local_node_ip}
            - MYSQL_PORT=3306
            - MYSQL_USER=root
            - MYSQL_PASSWORD=root
```
bootstrap.sh内容参考
```
#!/bin/bash
set -eo pipefail

PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
exec > >(tee /var/log/mysql-init.log) 2>&1

# ----------------------
# 工具函数区
# ----------------------
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1"
}

# ----------------------
# 延时启动配置区（新增）
# ----------------------
DELAY_MIN=${DELAY_MIN:-5}     # 最小延时（秒），默认5秒
DELAY_MAX=${DELAY_MAX:-30}    # 最大延时（秒），默认30秒
DELAY_SECONDS=$((RANDOM % (DELAY_MAX - DELAY_MIN + 1) + DELAY_MIN))
log "容器将随机延时 ${DELAY_SECONDS} 秒后启动..."
sleep ${DELAY_SECONDS}
log "延时结束，开始执行初始化流程..."

# ----------------------
# 配置参数区
# ----------------------
NODE_ID="$NODE_ID"
NODE_IP="$NODE_IP"
SEED_NODES="$SEED_NODES"
# 新增自定义数据库配置
CUSTOM_DATABASE="$MYSQL_DATABASE"
CUSTOM_USER="$MYSQL_USER"
CUSTOM_PASSWORD="$MYSQL_PASSWORD"
CUSTOM_PRIVILEGES="ALL PRIVILEGES"

CLUSTER_NAME="a1b2c3d4-1234-5678-9101-112131415161"
MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD"
MYSQL_DATA_DIR="/var/lib/mysql"
MYSQL_CONF_FILE="/etc/mysql/conf.d/my.cnf"
SOCKET_PATH="/var/run/mysqld/mysqld.sock"

case "$NODE_ID" in
  1)
    SERVER_ID=1
    AUTO_INCREMENT_OFFSET=1
    ;;
  2)
    SERVER_ID=2
    AUTO_INCREMENT_OFFSET=2
    ;;
  3)
    SERVER_ID=3
    AUTO_INCREMENT_OFFSET=3
    ;;
  *)
    echo "未知节点ID: $NODE_ID"
    exit 1
    ;;
esac

mysql_socket() {
    local tmp_cnf=$(mktemp)
    cat <<EOF > $tmp_cnf
[client]
user=root
password=$MYSQL_ROOT_PASSWORD
socket=$SOCKET_PATH
EOF
    mysql --defaults-extra-file=$tmp_cnf "$@"
    rm -f $tmp_cnf
}

# ----------------------
# 核心逻辑区（关键优化）
# ----------------------
generate_config() {
    log "生成动态配置文件"
    
    cat <<EOF > "$MYSQL_CONF_FILE"
[mysqld]
# 新增中继日志配置
relay_log = ${HOSTNAME}-relay-bin
relay_log_index = ${HOSTNAME}-relay-bin.index
# 基础配置
pid-file        = /var/run/mysqld/mysqld.pid
socket          = $SOCKET_PATH
datadir         = $MYSQL_DATA_DIR
lower_case_table_names = 1
default_authentication_plugin = mysql_native_password
disabled_storage_engines = "MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MEMORY"

# 二进制日志配置
log_bin=mysql-bin
binlog_format=ROW

# 集群配置
server_id               = $SERVER_ID
report_host             = $NODE_IP
bind_address            = 0.0.0.0

# GTID强制配置
gtid_mode = ON
enforce_gtid_consistency = ON
binlog_checksum = NONE

# 组复制增强配置
plugin_load_add = 'group_replication.so'
transaction_write_set_extraction = XXHASH64
binlog_transaction_dependency_tracking = WRITESET
loose_group_replication_group_name          = "$CLUSTER_NAME"
loose_group_replication_start_on_boot       = OFF
loose_group_replication_local_address       = "$NODE_IP:33061"
loose_group_replication_group_seeds         = "$SEED_NODES"
loose_group_replication_bootstrap_group     = OFF
loose_group_replication_single_primary_mode = ON
#脱离集群时，停止服务
loose_group_replication_exit_state_action = READ_ONLY
loose_group_replication_recovery_get_public_key=ON
# 强制约束检查
loose_group_replication_enforce_update_everywhere_checks=OFF

# 其他配置
auto_increment_increment = 1
auto_increment_offset    = 1
disabled_storage_engines = "MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MEMORY"
EOF
}

init_datadir() {
    local init_flag="$MYSQL_DATA_DIR/.initialized"
    
    if [ ! -f "$init_flag" ]; then
        log "安全初始化数据目录..."
        rm -rf "$MYSQL_DATA_DIR"/*
        mysqld --initialize-insecure --user=mysql
        touch "$init_flag"
    fi
    chown -R mysql:mysql "$MYSQL_DATA_DIR"
}

cleanup() {
      log "执行预启动清理..."
    
    # 通过PID文件检查进程
    if [ -f "/var/run/mysqld/mysqld.pid" ]; then
        log "发现残留MySQL进程，正在终止..."
        kill -9 $(cat /var/run/mysqld/mysqld.pid) 2>/dev/null || true
        sleep 2
    fi

    # 清理文件锁
    rm -vf /var/run/mysqld/mysqld.{pid,sock,sock.lock}
    find /var/run/mysqld -maxdepth 1 -type s -exec rm -v {} \;
    
    # 重建目录结构
    rm -rf /var/run/mysqld
    mkdir -p /var/run/mysqld
    chown -R mysql:mysql /var/run/mysqld
    chmod 755 /var/run/mysqld
}

start_mysql() {
     cleanup
    
    if [ ! -f "$MYSQL_DATA_DIR/mysql-init-done" ]; then
        log "首次启动：跳过权限验证"
        
        # 启动服务并记录PID
        /usr/local/bin/docker-entrypoint.sh mysqld --skip-grant-tables &
        echo $! > /tmp/mysql-init.pid
        
        # 通过socket检测服务状态
        wait_for_mysql "file"
        init_root_privileges
        
        log "正常重启MySQL服务..."
        kill -9 $(cat /tmp/mysql-init.pid) 2>/dev/null || true
        sleep 2
        rm -f /tmp/mysql-init.pid
    fi

    log "正常启动模式"
    /usr/local/bin/docker-entrypoint.sh mysqld &
    echo $! > /tmp/mysql-normal.pid
    wait_for_mysql "file"
}
wait_for_mysql() {
    local mode=${1:-"pid"}
    local max_attempts=30
    log "等待MySQL服务就绪..."
    
    for ((i=1; i<=max_attempts; i++)); do
        case $mode in
            "pid")
                if [ -f "/var/run/mysqld/mysqld.pid" ]; then
                    local pid=$(cat /var/run/mysqld/mysqld.pid)
                    if kill -0 $pid 2>/dev/null; then
                        log "MySQL进程已启动 (PID: $pid)"
                        return 0
                    fi
                fi
                ;;
            "file")
                if [ -S "$SOCKET_PATH" ]; then
                    if mysqladmin -uroot ping --socket=$SOCKET_PATH &>/dev/null; then
                        log "MySQL服务完全就绪"
                        return 0
                    fi
                fi
                ;;
        esac
        sleep $((1 + (i/5)))
        log "等待MySQL启动 ($i/$max_attempts)..."
    done
    
    log "ERROR: 启动超时，检查错误日志："
    tail -n 50 /var/log/mysql/error.log
    exit 1
}

init_root_privileges() {
     log "阶段1：设置root本地密码"
    
    # 安全等待直到服务可连接
    for ((i=1; i<=10; i++)); do
        if mysql -S $SOCKET_PATH -uroot -e "SELECT 1" &>/dev/null; then
            break
        fi
        sleep 1
    done

    mysql -S $SOCKET_PATH -uroot <<EOSQL
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY '$MYSQL_ROOT_PASSWORD';
EOSQL

    log "阶段2：配置远程访问"
    for ((retry=1; retry<=5; retry++)); do
        if mysql_socket -p$MYSQL_ROOT_PASSWORD <<EOSQL
CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED WITH caching_sha2_password BY '$MYSQL_ROOT_PASSWORD';
GRANT ALL ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EOSQL
        then
            touch "$MYSQL_DATA_DIR/mysql-init-done"
            return 0
        else
            sleep $((retry*2))
        fi
    done
    
    log "ERROR: 权限初始化失败"
    exit 1
}

check_gtid_status() {
    log "验证GTID配置状态..."
    
    local gtid_mode=$(mysql_socket -p$MYSQL_ROOT_PASSWORD -NB -e "SELECT @@GLOBAL.GTID_MODE")
    local gtid_consistency=$(mysql_socket -p$MYSQL_ROOT_PASSWORD -NB -e "SELECT @@GLOBAL.ENFORCE_GTID_CONSISTENCY")

    if [ "$gtid_mode" != "ON" ] || [ "$gtid_consistency" != "ON" ]; then
        log "ERROR: GTID配置异常！当前状态："
        log "gtid_mode = $gtid_mode"
        log "enforce_gtid_consistency = $gtid_consistency"
        exit 1
    fi
    
    log "GTID状态验证通过"
}

bootstrap_cluster() {
   log "安全引导集群..."
    
    check_gtid_status
    
    # 步骤1：禁用只读模式（添加重试机制）
    log "禁用super_read_only模式..."
    for ((retry=1; retry<=3; retry++)); do
        mysql_socket -p$MYSQL_ROOT_PASSWORD <<EOSQL
SET GLOBAL super_read_only = 0;
SELECT @@GLOBAL.super_read_only AS read_only_state;
EOSQL
        [ $? -eq 0 ] && break
        sleep $((retry*2))
    done

    # 步骤2：创建管理用户（优化权限配置）
    log "创建集群管理用户..."
    mysql_socket -p$MYSQL_ROOT_PASSWORD <<EOSQL
/* 8.0.26兼容语法 */
GRANT REPLICATION SLAVE ON *.* TO 'root'@'%';
FLUSH PRIVILEGES;
EOSQL

    # 步骤3：兼容性组复制配置
    log "配置组复制参数..."
    mysql_socket -p$MYSQL_ROOT_PASSWORD <<EOSQL
/* 8.0.26兼容写法 */
-- 检查插件是否已安装
SET @plugin_installed = (
    SELECT COUNT(*) 
    FROM information_schema.plugins 
    WHERE plugin_name = 'group_replication'
);

-- 动态执行安装（避免IF NOT EXISTS语法问题）
SET @install_sql = IF(
    @plugin_installed = 0,
    'INSTALL PLUGIN group_replication SONAME \'group_replication.so\'',
    'SELECT "插件已存在，跳过安装" AS status'
);

PREPARE stmt FROM @install_sql;
EXECUTE stmt;
DEALLOCATE PREPARE stmt;

-- 配置恢复通道（8.0.26需要显式指定MASTER_AUTO_POSITION）
CHANGE MASTER TO 
    MASTER_USER='root', 
    MASTER_PASSWORD='$MYSQL_ROOT_PASSWORD' 
FOR CHANNEL 'group_replication_recovery';

-- 启动引导流程
SET GLOBAL group_replication_bootstrap_group = ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group = OFF;
EOSQL

    log "集群引导成功"
}

wait_for_host_online() {
    local timeout=30
    log "等待主机 $NODE_IP 上线..."
    for ((i=1; i<=$timeout; i++)); do
        local status=$(mysql_socket -p$MYSQL_ROOT_PASSWORD -NB -e """
            SELECT MEMBER_STATE 
            FROM performance_schema.replication_group_members 
            WHERE MEMBER_HOST = '$NODE_IP'  # 按 host 过滤
            LIMIT 1
        """)
        if [ "$status" = "ONLINE" ]; then  # 直接检查状态值
            log "主机 $NODE_IP 已上线"
            return 0
        fi
        sleep 1
    done
    log "ERROR: 主机 $NODE_IP 等待超时"
    exit 1
}

# ----------------------
# 核心逻辑区（新增数据库创建函数）
# ----------------------
create_custom_schema() {
    log "开始创建自定义数据库和用户..."
	wait_for_host_online
    
    # 重试机制保障关键操作
    for ((retry=1; retry<=3; retry++)); do
        mysql_socket -p$MYSQL_ROOT_PASSWORD <<EOSQL
/* 使用IF NOT EXISTS确保幂等性 */
CREATE DATABASE IF NOT EXISTS $CUSTOM_DATABASE 
    CHARACTER SET utf8mb4 
    COLLATE utf8mb4_unicode_ci;

/* 使用双重条件判断用户是否存在 */
CREATE USER IF NOT EXISTS '$CUSTOM_USER'@'%' 
    IDENTIFIED WITH mysql_native_password 
    BY '$CUSTOM_PASSWORD';

/* 动态权限分配 */
GRANT $CUSTOM_PRIVILEGES ON $CUSTOM_DATABASE.* 
    TO '$CUSTOM_USER'@'%' 
    WITH GRANT OPTION;

/* 刷新权限并验证 */
FLUSH PRIVILEGES;
SHOW GRANTS FOR '$CUSTOM_USER'@'%';
EOSQL
        
        # 检查操作是否成功
        if [ $? -eq 0 ]; then
            log "数据库架构初始化成功"
            return 0
        else
            log "第${retry}次尝试失败，等待重试..."
            sleep $((retry*2))
        fi
    done
    
    log "ERROR: 自定义数据库初始化失败"
    exit 1
}

# ----------------------
# 新增加入集群函数（带智能重试机制）
# ----------------------
join_cluster() {
 local first_host=$1  # 通过参数接收主机名
 
 
     # 强制重置本地事务
    mysql_socket -p$MYSQL_ROOT_PASSWORD <<EOSQL
STOP GROUP_REPLICATION;
RESET MASTER;
SET @@GLOBAL.GTID_PURGED='';
EOSQL

    # 获取集群GTID_EXECUTED
    local cluster_gtid=$(mysql -h "$first_host"  -uroot -p$MYSQL_ROOT_PASSWORD -NB -e "SELECT @@GLOBAL.GTID_EXECUTED")
    
    # 同步GTID状态
    mysql_socket -p$MYSQL_ROOT_PASSWORD <<EOSQL
SET @@GLOBAL.GTID_PURGED='$cluster_gtid';
EOSQL

    # 标准加入流程
    mysql_socket -p$MYSQL_ROOT_PASSWORD <<EOSQL
CHANGE MASTER TO 
    MASTER_USER='root', 
    MASTER_PASSWORD='$MYSQL_ROOT_PASSWORD' 
FOR CHANNEL 'group_replication_recovery';

START GROUP_REPLICATION;
EOSQL
}

install_group_replication_plugin() {
    if ! mysql_socket -p$MYSQL_ROOT_PASSWORD -e "SHOW PLUGINS" | grep -q group_replication; then
        log "强制安装组复制插件..."
        mysql_socket -p$MYSQL_ROOT_PASSWORD -e "INSTALL PLUGIN group_replication SONAME 'group_replication.so';"
    fi
}

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >&2
}

is_cluster_exists() {
    local timeout=10
    log "通过种子节点探测集群状态（强制使用 3306 端口）..."
    
    IFS=',' read -ra seeds <<< "$SEED_NODES"
    for seed in "${seeds[@]}"; do
        # 提取主机部分（忽略原始端口配置）
        local host=${seed%%:*}
        local port=3306  # 强制固定端口
        
        # 集群状态查询语句
        local query="SELECT COUNT(*) FROM performance_schema.replication_group_members WHERE MEMBER_STATE='ONLINE';"
        local output
        local start_time=$(date +%s)

        # 关键修复点：分离 stdout 和 stderr
        output=$(timeout $timeout mysql --host="$host" --port="$port" \
                --user=root --password="$MYSQL_ROOT_PASSWORD" \
                --protocol=TCP --connect-timeout=$timeout \
                -NB -e "$query" 2> >(grep -v 'Using a password') 1>&2)
        local exit_code=$?
        local end_time=$(date +%s)
        local duration=$((end_time - start_time))

        # 重新捕获纯净的 stdout 输出
        output=$(mysql --host="$host" --port="$port" \
                --user=root --password="$MYSQL_ROOT_PASSWORD" \
                -NB -e "$query" 2>/dev/null)

        # 日志脱敏处理
        local safe_output=$(echo "$output" | sed "s/$MYSQL_ROOT_PASSWORD/******/g")

        # 错误分类处理
        if [[ $exit_code -eq 124 ]]; then
            log "[超时] 节点 $host:3306 在 ${timeout}秒内无响应（耗时: ${duration}s）"
            continue
        elif [[ $exit_code -ne 0 ]]; then
            log "[连接失败] 节点 $host:3306 | 错误码:$exit_code | 详情: $safe_output"
            continue
        fi

        # 数据有效性验证（过滤非数字字符）
        local numeric_output=$(echo "$output" | tr -cd '0-9')
        if [[ -z "$numeric_output" ]]; then
            log "[异常] 节点 $host:3306 返回无效响应: '$output'"
            continue
        fi

        # 集群状态判断
        if (( numeric_output >= 1 )); then
            log "[成功] 节点 $host:3306 存在有效集群 (在线成员: $numeric_output)"
            echo "$host"  # 输出有效主机供后续使用
            return 0
        else
            log "[提示] 节点 $host:3306 当前无在线成员"
        fi
    done
    
    log "[结论] 所有种子节点均未检测到有效集群"
    return 1
}

initialize_cluster_metadata() {
log "初始化 InnoDB Cluster 元数据..."
    
    # 增加等待时间确保集群稳定
    sleep 15
    
    if command -v mysqlsh &>/dev/null; then
        log "使用 mysqlsh 初始化元数据 (接管模式)..."
        
        mysqlsh --user=root --password="$MYSQL_ROOT_PASSWORD" --host=127.0.0.1 --port=3306 --no-wizard --js <<EOF
// 显式连接实例
var session = shell.connect('root:$MYSQL_ROOT_PASSWORD@127.0.0.1:3306');

// 检查是否已有集群元数据
try {
    var cluster = dba.getCluster();
    print("集群元数据已存在，跳过创建");
} catch (e) {
    print("接管现有组复制集群...");
    
    // 使用 adoptFromGR 参数接管现有组复制集群
    var cluster = dba.createCluster('myCluster', {
        force: true,
        adoptFromGR: true  // 关键修复：接管现有组复制
    });
    
    print("集群接管成功");
}

// 安全获取集群状态
try {
    var cluster = dba.getCluster();
    print("最终集群状态：");
    
    if (typeof cluster.status === 'function') {
        var status = cluster.status();
        print(JSON.stringify(status, null, 2));
    } else {
        print("无法获取状态: cluster.status 不是函数");
    }
} catch (e) {
    print("无法获取集群状态: " + e.message);
}
EOF
    else
        log "mysqlsh 不可用，跳过元数据初始化"
    fi
    
    log "元数据初始化完成"
}

check_cluster_boot_status() {
    local timeout=30
    log "验证引导节点状态..."
    
    for ((i=1; i<=timeout; i++)); do
        local status=$(mysql_socket -p$MYSQL_ROOT_PASSWORD -NB -e """
            SELECT MEMBER_STATE 
            FROM performance_schema.replication_group_members 
            WHERE MEMBER_ID = @@global.server_uuid
        """)
        
        if [ "$status" = "ONLINE" ]; then
            log "节点已在线 (状态: $status)"
            return 0
        fi
        
        log "等待节点上线 ($i/$timeout)... 当前状态: ${status:-UNKNOWN}"
        sleep 1
    done
    
    log "ERROR: 节点启动超时"
    mysql_socket -p$MYSQL_ROOT_PASSWORD -e "SHOW ENGINE INNODB STATUS" >&2
    exit 1
}

# ----------------------
# 主控制流程
# ----------------------
main() {
    mkdir -p /var/run/mysqld
    chown -R mysql:mysql /var/run/mysqld
    
    generate_config
    init_datadir
    start_mysql
	
	check_gtid_status
	
	# 安装插件（所有节点必须）
    install_group_replication_plugin
	
	 # 智能判断集群状态
    if first_host=$(is_cluster_exists); then
	   log "检测到现有集群，尝试加入...$first_host"
        join_cluster "$first_host"
    else
	 log "未检测到集群，启动新集群..."
        bootstrap_cluster
#        sleep 15
#        check_cluster_boot_status
#        
#        # 安全初始化元数据
#        initialize_cluster_metadata
    fi 
    
#  	if [[ "$NODE_ID" == "1" ]] && [[ -z "$first_host" ]]; then
#		create_custom_schema
#	fi
    
    log "最终集群状态："
    mysql_socket -p$MYSQL_ROOT_PASSWORD -e """
    SELECT 
        MEMBER_ID,
        MEMBER_HOST,
        MEMBER_PORT,
        MEMBER_STATE,
        IF(MEMBER_ID = @@global.server_uuid, 'LOCAL', 'REMOTE') AS ROLE
    FROM performance_schema.replication_group_members
    """
    
    exec tail -f /dev/null
}

main
```
mysqlrouter.conf内容如下
```
[mysqlrouter]
logging_folder = /var/log/mysqlrouter
data_folder = /var/lib/mysqlrouter
runtime_folder = /var/run/mysqlrouter
log_level = INFO

[metadata_cache:group_replication_md]
cluster_type = gr
servers = 192.168.101.30:3306,192.168.101.31:3306,192.168.101.32:3306
conf_use_gr_notifications=1 
user = root
password = root
router_id=1

# 元数据刷新间隔从5秒改为3秒
ttl=3
# 连接超时降为2秒（适应网络波动）
connect_timeout=2
 # 故障节点隔离检测间隔
error_quarantine_interval=5

[routing:read_write]
bind_address = 0.0.0.0
bind_port = 6446
destinations = metadata-cache://group_replication_md/primary
routing_strategy = first-available
max_connections = 1000
connect_timeout = 3
idle_timeout = 60

```
## 三、验证

```
-- 检查组成员
SELECT * FROM performance_schema.replication_group_members;
```

逾期结果

所有节点MEMBER_STATE的值应该为ONLINE

至少有一个节点MEMBER_ROLE值PRIMARY（单主模式）

## 四、创建元数据

```
#进入容器
docker exec -it mysql1 bash

#打开mysqlsh
mysqlsh

#若返回错误 "No cluster found"，说明无元数据，可创建
dba.getCluster(); 

dba.createCluster('my_cluster')



```

## 五、执行sql

1、创建数据库、用户以及授权

2、执行iotbuilderdb.sql

```
CREATE DATABASE IF NOT EXISTS iotbuilderdb 
    CHARACTER SET utf8mb4 
    COLLATE utf8mb4_unicode_ci;

 --使用双重条件判断用户是否存在 
CREATE USER IF NOT EXISTS 'admin'@'%' 
    IDENTIFIED WITH mysql_native_password 
    BY 'sysware2018.';

 --动态权限分配 
GRANT ALL PRIVILEGES ON iotbuilderdb.* TO 'admin' @'%' WITH GRANT OPTION;

FLUSH PRIVILEGES;
```

## 六、问题

当有节点退出后，再次加入造成数据不一致，无法同步问题

```
-- 1. 检查MySQL版本（需8.0.17及以上，克隆功能更稳定）
SELECT VERSION();

-- 2. 安装克隆插件（仅需执行一次）如果不成功：SET GLOBAL super_read_only = OFF;
INSTALL PLUGIN clone SONAME 'mysql_clone.so';

-- 3. 验证插件是否启用
SELECT PLUGIN_NAME, PLUGIN_STATUS 
FROM INFORMATION_SCHEMA.PLUGINS 
WHERE PLUGIN_NAME = 'clone';
-- 若状态为ACTIVE，则插件已启用


-- 在从节点执行（先停止组复制）
STOP GROUP_REPLICATION;

-- 配置克隆源为健康节点（如192.168.101.30）
SET GLOBAL clone_valid_donor_list = '192.168.101.30:3306';

-- 执行克隆（使用之前创建的克隆用户）
CLONE INSTANCE FROM 'root'@'192.168.101.30':3306 
IDENTIFIED BY 'root';
---如果重启失败 Restart server failed (mysqld is not managed by supervisor process).
---请手动重启

```

# MysqlRouter

它是 MySQL 官方提供的轻量级中间件，用于在应用程序和 MySQL 集群之间提供连接路由、负载均衡和高可用性支持。

一、核心作用 

1. 透明路由将应用程序的数据库连接请求智能路由到 MySQL 集群的主节点（写操作）或从节点（读操作），应用无需关心集群拓扑结构。 

2. 负载均衡对读请求进行负载均衡，自动分发到多个可用的从节点，提高读取性能。
3. 高可用性监控集群节点状态，当主节点故障时，自动将写请求路由到新的主节点，减少应用停机时间。 
4. 连接池通过复用数据库连接减少开销，提高并发处理能力（配置中max_connections=1000）。

#### **mysqlrouter.conf 配置文件**

```
[mysqlrouter]
logging_folder = /var/log/mysqlrouter  # 日志目录
data_folder = /var/lib/mysqlrouter     # 数据目录
runtime_folder = /var/run/mysqlrouter  # 运行时目录
log_level = INFO                       # 日志级别

[routing:read-write]
bind_address = 0.0.0.0         # 监听地址（所有接口）
bind_port = 6446               # 读写端口（应用连接此端口）
destinations = metadata-cache://group_replication_md/primary  # 主节点路由
routing_strategy = first-available  # 路由策略：优先选择可用的第一个节点
max_connections = 1000       # 最大连接数
connect_timeout = 10         # 连接超时时间（秒）
```

#### **Docker Compose 配置**

```
mysql-router:
  image: container-registry.oracle.com/mysql/community-router:8.4
  container_name: mysql-router
  hostname: mysql-router
  volumes:
    - ./mysql1/mysqlrouter.conf:/etc/mysqlrouter/mysqlrouter.conf  # 挂载配置文件
  network_mode: "host"  # 使用主机网络模式
  restart: always
  environment:
    - MYSQL_HOST=${local_node_ip}  # MySQL主机地址
    - MYSQL_PORT=3306              # MySQL端口
    - MYSQL_USER=root              # 连接用户名
    - MYSQL_PASSWORD=root          # 连接密码
```

