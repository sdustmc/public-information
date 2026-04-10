# Minecraft 服务器部署标准作业程序 (SOP)

本手册旨在规范圣悦灵社服务器的部署流程，确保各环境的一致性、安全性和可维护性。

---

## 🛠️ 1. 环境准备

### 1.1 操作系统与环境

- **OS**： Debian GNU/Linux 13 (trixie)
- **内核**： 6.12.37
- **Java 版本**： Java 21 (Temurin-21.0.9+10)
- **Python**： 3.13.5 (用于 MCDR)

### 1.2 用户管理

- 生产环境严禁使用 `root` 运行服务器。
- 核心用户： `mcserver` (UID： 1001，GID： 1001), 无 `sudo` 权限。

### 1.3 文件系统布局

所有服务应遵循以下结构：

- `/home/mcserver/server/`： Minecraft 服务器实例目录
- `/home/mcserver/minecraft/`： 核心数据目录
  - `./backups/`： 定期备份存放处
  - `./lib/`： Java、Python 等多版本运行时依赖
  - `./mcenv/`： MCDR 的 Python 虚拟环境
- `/home/mcserver/mcsm/`： MCSM 面板安装目录
- `/home/mcserver/rent/`： 租户隔离数据目录

---

## 🚀 2. 标准服务器部署流程 (Step-by-Step)

本流程适用于在现有基础设施下新增一个常规 Minecraft 实例。

### 第一阶段：资源规划

1. **确定端口**： 根据端口分配情况，选择对应的的端口。
2. **分配内存**： 根据服务端类型分配核心与内存

### 第二阶段：实例初始化 (MCSM)

**创建实例**：

- 登录 MCSM 面板。
- 点击“新建应用” -> “直接创建”，实例名称应遵循`<端口><名称><英文简称>`之规范，例如`25560常驻perm`指端口25560的常驻服，服务器地址为`perm.sdustmc.cn`。
- 实例目录统一设置在 `/home/mcserver/server/<英文简称>`。

### 第三阶段：管理增强 (MCDR) - *可选但推荐*

若长期服需要管理插件或脚本控制：

1. **初始化 MCDR**：SSH 登录服务器，此时应该是`mcserver`用户：

    ```bash
    cd /home/mcserver/server/<英文简称>
    source /home/mcserver/minecraft/mcenv/bin/activate
    mcdreforged init
    mcdreforged
    ```

    以上命令分别是进入实例目录、激活 Python 虚拟环境、初始化 MCDR、启动 MCDR。不出意外应该会看到 MCDR 启动，输出测试消息`[Server] Hello world from MCDReforged`后关闭，这表明 MCDR 已成功安装并能正常运行。

2. **配置参数**：回到 MCDR 控制台，在文件管理中打开`config.yml`，修改如下参数：
   - language: zh_cn  
    语言设置为中文

   - start_command:bash ./start.sh  
    启动命令改为启动 MC 服务器的启动脚本

   - handler: vanilla_handler  
    根据上面注释，不同服务端类型选择对应的消息格式

3. **配置服务端**：将 MC 服务端放入`server`文件夹下，并以第四阶段所述方法进行配置。

4. **设置启动脚本**：在实例根目录创建`mcdr.sh`脚本，内容如下：

    ```bash
    #!/usr/bin/env bash
    source /home/mcserver/minecraft/mcenv/bin/activate
    mcdreforged
    ```

    将应用实例设置中启动命令改为`bash ./mcdr.sh`，保存后即可通过 MCSM 面板启动 MCDR，从而让 MCDR 启动 Minecraft 服务器。

### 第四阶段：配置服务端

将 Minecraft 服务端 jar 包上传至实例目录,更改参数如下：

1. **server.properties**：
   - `server-port=<之前分配的端口>`
   - `online-mode=true`（如使用皮肤站验证则开启）
2. **JVM 参数**：

   ```bash
   #!/usr/bin/env bash
   taskset -c 0,1,2,16,17,18 java \
    -Xms12G \
    -Xmx12G \
	-XX:MaxDirectMemorySize=4G \
	-XX:+UseG1GC \
	-XX:+ParallelRefProcEnabled \
	-XX:MaxGCPauseMillis=200 \
	-XX:+UnlockExperimentalVMOptions \
	-XX:+DisableExplicitGC \
	-XX:+AlwaysPreTouch \
	-XX:G1NewSizePercent=40 \
	-XX:G1MaxNewSizePercent=50 \
	-XX:G1HeapRegionSize=16M \
	-XX:G1ReservePercent=15 \
	-XX:G1HeapWastePercent=5 \
	-XX:G1MixedGCCountTarget=4 \
	-XX:InitiatingHeapOccupancyPercent=20 \
	-XX:G1MixedGCLiveThresholdPercent=90 \
	-XX:G1RSetUpdatingPauseTimePercent=5 \
	-XX:SurvivorRatio=32 \
	-XX:+PerfDisableSharedMem \
	-XX:MaxTenuringThreshold=1 \
	-Dusing.aikars.flags=https://mcflags.emc.gs \
	-Daikars.new.flags=true \
    -javaagent:authlib-injector-1.2.7.jar=https://skin.sdustmc.cn/api/union/yggdrasil \
    -jar server.jar \
    nogui
   ```

   - #!/usr/bin/env bash：指定脚本解释器
   - taskset -c 0,1,2,16,17,18：绑定 CPU 核心
   - java： 调用默认 Java 运行，可指定 Java 版本
   - -javaagent:authlib-injector-1.2.7.jar=...：使用皮肤站的 Authlib Injector
   - -Xms12G -Xmx12G：设置初始和最大堆内存为 12GB
   - -XX:+UseG1GC...：示例优化参数
   - -jar server.jar 启动服务端 jar 包
   - nogui：禁用图形界面
启动后在 **eula.txt**： 将 `eula=false` 改为 `eula=true`。
---
