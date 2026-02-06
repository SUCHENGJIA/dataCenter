# dataCenter 项目分析报告

## 一、项目概述

**项目名称**：dataCenter（气象数据中心系统）

**作者**：吴从周

**项目定位**：这是一个功能完整的专业级气象数据采集、处理和存储系统，专门为气象站点的观测数据管理和自动化处理设计。

**系统架构**：采用 C++ 开发的 Linux 服务端应用程序，使用 Oracle 数据库进行数据存储，支持多种数据格式（XML、CSV）的采集和处理。

---

## 二、项目结构分析

### 2.1 目录结构

```
dataCenter/
├── idc/              # IDC数据中心核心模块
├── public/           # 公共库模块
└── tools/            # 工具程序模块
```

### 2.2 模块划分

#### **IDC 模块** ([idc/](idc/))
- **cpp/**: 核心业务代码
  - [idcapp.cpp](idc/cpp/idcapp.cpp:1): 主应用程序，包含气象数据解析和入库逻辑
  - [crtsurfdata.cpp](idc/cpp/crtsurfdata.cpp:1): 创建表面数据
  - [obtcodetodb.cpp](idc/cpp/obtcodetodb.cpp:1): 代码转数据库
  - [obtmindtodb.cpp](idc/cpp/obtmindtodb.cpp:1): 观测数据转数据库
- **ini/**: 配置文件
  - [stcode.ini](idc/ini/stcode.ini:1): 全国800+气象站点信息（站号、站名、经纬度、海拔）
  - [xmltodb.xml](idc/ini/xmltodb.xml:1): XML文件到数据库表的映射配置
- **sql/**: 数据库脚本（表结构、数据定义）

#### **Public 模块** ([public/](public/))
- **核心库**: [_public.h](public/_public.h:1), [_public.cpp](public/_public.cpp:1)
- **FTP功能**: [_ftp.h](public/_ftp.h:1), [_ftp.cpp](public/_ftp.cpp:1)
- **Oracle数据库**: [db/oracle/_ooci.h](public/db/oracle/_ooci.h:1)
- **示例程序**: 50+ 演示程序

#### **Tools 模块** ([tools/](tools/))
- **工具库**: [_tools.h](tools/cpp/_tools.h:1), [_tools.cpp](tools/cpp/_tools.cpp:1)
- **文件工具**: deletefiles, gzipfiles, ftpgetfiles
- **网络服务**: webserver, rinetd (TCP转发), fileserver
- **数据同步**: syncinc, syncref
- **数据转换**: xmltodb, migratetable

---

## 三、核心功能分析

### 3.1 气象数据处理

#### **数据结构**
```cpp
// 从 idcapp.cpp 可以看出观测数据包含以下字段：
// - obtid: 站点代码
// - ddatetime: 观测时间
// - t: 温度 (放大10倍存储)
// - p: 气压 (放大10倍存储)
// - u: 湿度
// - wd: 风向
// - wf: 风速 (放大10倍存储)
// - r: 降雨量 (放大10倍存储)
// - vis: 能见度 (放大10倍存储)
```

#### **数据格式支持**
- **XML格式**: 通过 `getxmlbuffer()` 解析XML标签
- **CSV格式**: 通过 `ccmdstr` 类按逗号分隔解析

### 3.2 数据库设计

#### **核心表结构** ([createtable.sql](idc/sql/createtable.sql:1))

| 表名 | 用途 |
|------|------|
| T_ZHOBTCODE | 气象站点代码表 |
| T_ZHOBTMIND | 气象观测分钟数据表 |
| T_DATATYPE | 数据类型配置表 |
| T_INTERCFG | 接口配置表 |
| T_USERINFO | 用户信息表 |
| T_USERANDINTER | 用户权限表 |
| T_USERLOG | 用户调用日志表 |
| T_USERLOGSTAT | 用户日志统计表 |

### 3.3 公共库功能 ([_public.h](public/_public.h:1))

#### **字符串处理**
- `deletelchr/deleterchr`: 删除左右字符
- `toupper/tolower`: 大小写转换
- `replacestr`: 字符串替换
- `picknumber`: 提取数字

#### **命令行解析** - `ccmdstr` 类
```cpp
// 支持分隔符解析，如：
ccmdstr cmdstr;
cmdstr.splittocmd(strbuffer, ",");
```

#### **时间操作**
- `ltime()`: 获取当前时间
- `timetostr()`: 时间戳转字符串
- `strtotime()`: 字符串转时间戳
- `addtime()`: 时间偏移计算

#### **文件操作**
- `cdir`: 目录遍历类
- `cifile/cofile`: 文件读写类
- `copyfile/renamefile`: 文件操作

#### **日志系统** - `clogfile` 类
- 支持自动日志切换
- 支持多线程（自旋锁）
- 带时间戳的日志输出

#### **Socket 通信**
- `ctcpclient`: TCP 客户端类
- `ctcpserver`: TCP 服务端类

#### **进程间通信**
- `cpactive`: 进程心跳管理
- `squeue`: 循环队列模板
- `csemp`: 信号量类

### 3.4 工具模块功能 ([_tools.cpp](tools/cpp/_tools.cpp:1))

#### **数据库元数据操作** - `ctcols` 类
- `allcols()`: 获取表的所有字段信息
- `pkcols()`: 获取表的主键字段
- 自动识别数据类型（char, date, number）
- 从 Oracle 数据字典读取元数据

#### **XML 数据入库** ([xmltodb.cpp](tools/cpp/xmltodb.cpp:1))
- 监控指定目录的 XML 文件
- 根据 [xmltodb.xml](idc/ini/xmltodb.xml:1) 配置自动映射到数据库表
- 支持插入和更新操作
- 文件处理后自动归档

---

## 四、技术特点

### 4.1 架构特点

| 特点 | 说明 |
|------|------|
| **模块化设计** | 清晰的三层架构（业务/公共/工具） |
| **代码复用** | 公共功能独立成库，避免重复代码 |
| **配置驱动** | XML/INI 配置文件控制业务逻辑 |
| **常驻进程** | 支持守护进程运行，定时轮询 |
| **进程心跳** | 共享内存 + 信号量实现进程监控 |

### 4.2 数据处理特点

- **精度处理**: 温度、气压等数据放大10倍存储，保留一位小数
- **批量处理**: 支持批量 XML 文件解析和入库
- **错误处理**: 数据库操作失败时记录日志，不中断流程
- **文件归档**: 成功/失败文件分别移动到不同目录

### 4.3 数据库连接管理

- 使用 Oracle OCI 接口
- 预编译 SQL 语句，绑定变量
- 序列器自动生成主键
- 支持事务处理

---

## 五、核心类和接口

### 5.1 公共库核心类

| 类名 | 功能 | 文件位置 |
|------|------|------|
| `ccmdstr` | 命令行字符串解析 | [public/_public.h:80](public/_public.h#L80) |
| `clogfile` | 日志文件管理 | [public/_public.h:459](public/_public.h#L459) |
| `cdir` | 目录文件遍历 | [public/_public.h:298](public/_public.h#L298) |
| `cifile/cofile` | 文件读写 | [public/_public.h:400](public/_public.h#L400) |
| `ctcpclient` | TCP 客户端 | [public/_public.h:527](public/_public.h#L527) |
| `ctcpserver` | TCP 服务端 | [public/_public.h:564](public/_public.h#L564) |
| `csemp` | 信号量 | [public/_public.h:747](public/_public.h#L747) |
| `cpactive` | 进程心跳 | [public/_public.h:804](public/_public.h#L804) |
| `squeue` | 循环队列 | [public/_public.h:651](public/_public.h#L651) |
| `ctimer` | 精确计时器 | [public/_public.h:232](public/_public.h#L232) |
| `spinlock_mutex` | 自旋锁 | [public/_public.h:434](public/_public.h#L434) |

### 5.2 工具库核心类

| 类名 | 功能 | 文件位置 |
|------|------|------|
| `ctcols` | 表字段信息获取 | [tools/cpp/_tools.cpp:4](tools/cpp/_tools.cpp#L4) |
| `CZHOBTMIND` | 气象观测数据处理 | [idc/cpp/idcapp.cpp:9](idc/cpp/idcapp.cpp#L9) |

---

## 六、配置文件说明

### 6.1 站点配置 ([stcode.ini](idc/ini/stcode.ini:1))

**格式**: `省份,站号,站名,纬度,经度,海拔高度`

**示例**:
```ini
安徽,58015,砀山,34.27,116.2,44.2
北京,54511,北京,39.48,116.28,31.3
福建,58847,福州,26.05,119.17,84
```

### 6.2 XML映射配置 ([xmltodb.xml](idc/ini/xmltodb.xml:1))

```xml
<filename>ZHOBTMIND_*.XML</filename>
<tname>T_ZHOBTMIND1</tname>
<uptbz>1</uptbz>  <!-- 1-更新，2-不更新 -->
```

---

## 七、项目优势

1. **领域专精**: 专门针对气象数据处理设计，包含全国800+站点信息
2. **生产级质量**: 完善的日志、错误处理、进程监控机制
3. **高性能**: 使用预编译SQL、批量处理、共享内存等优化
4. **可维护性**: 模块化设计，代码结构清晰
5. **丰富的工具集**: 50+示例程序，20+工具程序
6. **跨平台**: 支持 Linux 系统

---

## 八、应用场景

1. **气象数据采集**: 自动采集各气象站观测数据
2. **数据入库处理**: XML/CSV 格式数据自动入库
3. **数据同步**: 数据库表之间数据同步
4. **文件传输**: FTP/TCP 文件传输服务
5. **数据服务**: 通过接口对外提供数据查询服务
6. **系统监控**: 进程监控和日志管理

---

## 九、编译和部署

项目使用 Makefile 编译，生成可执行文件位于 `bin/` 目录：

- **IDC核心程序**: [idc/bin/](idc/bin/)
- **工具程序**: [tools/bin/](tools/bin/)
- **启动脚本**: [idc/cpp/start.sh](idc/cpp/start.sh), [stop.sh](idc/cpp/stop.sh)

### 9.1 主要工具程序

| 程序名 | 功能 |
|--------|------|
| xmltodb | XML文件入库到Oracle数据库 |
| ftpgetfiles | FTP获取文件 |
| ftpputfiles | FTP上传文件 |
| tcpgetfiles | TCP获取文件 |
| tcpputfiles | TCP上传文件 |
| webserver | Web服务器 |
| rinetd | TCP端口转发 |
| deletefiles | 删除文件工具 |
| gzipfiles | GZIP压缩工具 |
| syncinc | 增量同步 |
| syncref | 引用同步 |
| migratetable | 表迁移工具 |
| procctl | 进程控制 |
| checkproc | 进程检查 |
| fileserver | 文件服务器 |

---

## 十、代码示例

### 10.1 气象数据解析 ([idcapp.cpp](idc/cpp/idcapp.cpp:9))

```cpp
// 解析XML格式的气象数据
bool CZHOBTMIND::splitbuffer(const string &strbuffer, const bool bisxml)
{
    memset(&m_zhobtmind, 0, sizeof(struct st_zhobtmind));

    if (bisxml == true) {
        getxmlbuffer(strbuffer, "obtid", m_zhobtmind.obtid, 5);
        getxmlbuffer(strbuffer, "ddatetime", m_zhobtmind.ddatetime, 14);
        char tmp[11];
        getxmlbuffer(strbuffer, "t", tmp, 10);
        if (strlen(tmp) > 0)
            snprintf(m_zhobtmind.t, 10, "%d", (int)(atof(tmp) * 10));
        // ... 其他字段
    }
    // ... CSV格式处理
    return true;
}
```

### 10.2 数据入库 ([idcapp.cpp](idc/cpp/idcapp.cpp:49))

```cpp
// 将解析后的数据插入数据库
bool CZHOBTMIND::inserttable()
{
    if (m_stmt.isopen() == false) {
        m_stmt.connect(&m_conn);
        m_stmt.prepare("insert into T_ZHOBTMIND(...) values (...)");
        // 绑定输入变量
    }

    if (m_stmt.execute() != 0) {
        if (m_stmt.rc() != 1) {
            // 记录错误日志
            m_logfile.write("strbuffer=%s\n", m_buffer.c_str());
        }
        return false;
    }
    return true;
}
```

### 10.3 获取表字段信息 ([_tools.cpp](tools/cpp/_tools.cpp:18))

```cpp
// 从Oracle数据字典获取表的所有字段
bool ctcols::allcols(connection &conn, char *tablename)
{
    stmt.prepare("select lower(column_name), lower(data_type), data_length \
                 from USER_TAB_COLUMNS \
                 where table_name=upper(:1) order by column_id", tablename);

    // 处理结果集，识别字段类型
    // char, date, number
}
```

---

## 十一、数据库表设计详解

### 11.1 T_ZHOBTMIND（气象观测分钟数据表）

| 字段名 | 类型 | 说明 |
|--------|------|------|
| obtid | varchar2(5) | 站点代码 |
| ddatetime | date | 观测时间 |
| t | number(10) | 温度（实际值×10） |
| p | number(10) | 气压（实际值×10） |
| u | number(10) | 湿度 |
| wd | number(10) | 风向 |
| wf | number(10) | 风速（实际值×10） |
| r | number(10) | 降雨量（实际值×10） |
| vis | number(10) | 能见度（实际值×10） |
| keyid | number(15) | 主键ID |

### 11.2 T_DATATYPE（数据类型配置表）

支持层级分类，通过 `ptypeid` 字段实现父子关系。

### 11.3 T_INTERCFG（接口配置表）

存储所有数据接口的配置信息，包括：
- 接口名称和描述
- SQL 查询语句
- 输出列配置
- 参数绑定配置

### 11.4 T_USERINFO（用户信息表）

客户端身份认证信息，包括：
- 用户名/密码
- 应用名称
- 绑定IP
- 联系方式

### 11.5 T_USERLOG（用户调用日志表）

记录每次接口调用信息，用于：
- 访问审计
- 流量统计
- 计费依据

---

## 十二、系统设计亮点

### 12.1 数据精度处理

温度、气压、风速、降雨量等浮点数据通过**放大10倍存储**的方式：
- 避免浮点数精度问题
- 节省存储空间
- 提高查询效率
- 显示时除以10还原

### 12.2 进程心跳机制

使用共享内存 + 信号量实现进程监控：
- 多个进程注册到共享内存
- 定期更新心跳时间
- 监控程序检测超时进程
- 自动重启异常进程

### 12.3 文件处理策略

XML文件入库处理流程：
1. 扫描输入目录
2. 根据配置文件匹配处理规则
3. 解析XML数据
4. 执行预SQL（如删除旧数据）
5. 批量插入/更新数据库
6. 移动文件到备份目录（成功）或错误目录（失败）

### 12.4 灵活的配置系统

- **站点配置**: 800+ 气象站点信息
- **映射配置**: XML文件名到数据库表的映射
- **接口配置**: 动态配置数据接口
- **用户权限**: 灵活的接口访问控制

---

## 十三、项目统计

### 13.1 文件统计

- **C++源文件**: 96个
- **C++头文件**: 7个
- **SQL文件**: 6个
- **配置文件**: 4个（INI/XML）
- **静态库**: 2个
- **共享库**: 1个

### 13.2 代码行数估算

| 模块 | 估算行数 |
|------|----------|
| IDC核心模块 | ~2000行 |
| 公共库模块 | ~3000行 |
| 工具模块 | ~8000行 |
| 示例程序 | ~5000行 |
| **总计** | **~18000行** |

---

## 十四、总结与评价

### 14.1 项目优势

✅ **专业性强**: 专注气象数据处理，包含完整的业务逻辑
✅ **架构清晰**: 三层模块化设计，职责分明
✅ **代码质量高**: 规范的命名，完善的注释
✅ **功能完整**: 覆盖数据采集、传输、存储、处理全流程
✅ **生产级**: 日志、监控、容错机制完善
✅ **可扩展**: 配置驱动，易于扩展新功能
✅ **学习价值**: 包含大量C++编程最佳实践

### 14.2 适用场景

- **数据采集系统**: 各种传感器数据的采集和处理
- **数据中心项目**: 数据的集中存储和管理
- **C++后端服务**: 高性能服务端程序开发
- **学习项目**: C++、Linux、Oracle数据库综合应用

### 14.3 技术栈总结

| 技术 | 用途 |
|------|------|
| C++11 | 核心开发语言 |
| Oracle OCI | 数据库访问接口 |
| Socket API | 网络通信 |
| POSIX API | 系统调用（文件、进程、IPC） |
| Makefile | 编译构建 |
| XML/INI | 配置文件格式 |

---

## 附录

### A. 气象站点数量统计

根据 [stcode.ini](idc/ini/stcode.ini:1) 配置文件，系统覆盖全国主要省份的气象站点，包括但不限于：

- 安徽、北京、福建、甘肃、广东、广西、贵州、海南、河北、黑龙江、河南、湖北、湖南、内蒙古、江苏、江西、吉林、辽宁、宁夏、青海、陕西、山东、上海、山西、四川、天津、云南、浙江等

### B. 主要数据流程

```
气象站点 → XML/CSV文件 → xmltodb工具 → Oracle数据库 → 接口服务 → 客户端应用
```

### C. 关键技术点

1. **Oracle OCI 接口**: 直接使用 Oracle Call Interface 进行数据库操作
2. **预编译SQL**: 使用绑定变量防止SQL注入，提高性能
3. **共享内存**: 用于进程间通信和心跳管理
4. **信号量**: 用于进程同步和互斥
5. **守护进程**: 支持后台长期运行
6. **日志轮转**: 自动切换日志文件，防止日志过大

---

**报告生成时间**: 2026-01-25
**分析工具**: Claude Code
**项目路径**: c:\Users\scj20\Desktop\dataCenter
