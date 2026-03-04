在量化交易中，环境配置往往是“最难的第一步”。为了方便你后期查阅或在其他电脑上快速迁移环境，我为你整理了这份 **《PyCharm 与 Mini QMT (xtquant) 实盘环境配置指南》**。

你可以将以下内容保存为 QMT_Setup_Guide.md。

------



# PyCharm 与 Mini QMT (xtquant) 实盘环境配置指南

## 一、 核心环境要求













| 组件            | 要求                       | 备注                                        |
| --------------- | -------------------------- | ------------------------------------------- |
| **Python 版本** | **3.10 或 3.11**           | **严禁使用 3.12/3.13**（底层 C++ 库不兼容） |
| **QMT 软件**    | 极速交易终端 (国金/华泰等) | 需开通量化交易权限                          |
| **运行模式**    | **Mini QMT 模式**          | 必须切换至 Mini 模式并保持登录              |
| **操作系统**    | Windows 10/11              | 必须与 QMT 安装在同一台电脑                 |

------



## 二、 环境搭建步骤

### 1. 创建 Conda 虚拟环境

在 CMD 或 PyCharm 终端执行，确保环境纯净：

codeBash



```
# 创建环境
conda create -n qmt_trade python=3.10 -y

# 激活环境
conda activate qmt_trade

# 安装必要基础库
pip install pandas numpy
```

### 2. 关联 xtquant 库

**不要使用 pip install xtquant**。采用“路径导入法”最稳妥：

1. 
2. 找到 QMT 安装目录下的库路径：[安装目录]\bin.x64\Lib\site-packages。
3. 在 PyCharm 代码中通过 sys.path.append 动态引入（见下文代码）。

------



## 三、 PyCharm 代码标准模板

这是解决 **DLL 加载失败** 和 **模块找不到** 问题的万能模板。

codePython



```
import os
import sys
import time

# ================================================= #
# 1. 路径配置 (根据实际安装位置修改)
# ================================================= #
# QMT安装根目录
qmt_root = r'D:\国金证券QMT交易端'
# xtquant 库所在目录
xtquant_path = os.path.join(qmt_root, r'bin.x64\Lib\site-packages')
# 核心 DLL 所在目录
qmt_bin_path = os.path.join(qmt_root, r'bin.x64')

# 2. 注入系统路径
if xtquant_path not in sys.path:
    sys.path.append(xtquant_path)

# 3. 解决 Python 3.8+ 的 DLL 搜索问题 (关键)
if hasattr(os, 'add_dll_directory'):
    os.add_dll_directory(qmt_bin_path)
else:
    os.environ['PATH'] = qmt_bin_path + os.pathsep + os.environ['PATH']

# 4. 导入 xtquant
try:
    from xtquant import xtdata
    print("✅ xtquant 成功加载")
except ImportError as e:
    print(f"❌ 加载失败: {e}")
    sys.exit()

# ================================================= #
# 5. 业务逻辑示例
# ================================================= #
def main():
    symbol = '600519.SH'
    
    # 强制指定数据目录 (可选，连接失败时使用)
    # xtdata.set_data_dir(os.path.join(qmt_root, 'userdata_mini'))
    
    # 下载并获取数据
    xtdata.download_history_data(symbol, period='1d', count=10)
    time.sleep(1)
    
    data = xtdata.get_market_data_ex(
        field_list=['close'], 
        stock_list=[symbol], 
        period='1d', 
        count=6
    )
    
    if not data[symbol].empty:
        print(f"数据获取成功，最新收盘价: {data[symbol]['close'].iloc[-1]}")

if __name__ == '__main__':
    main()
```

------



## 四、 常见坑点与解决方案

### 1. 报错：ModuleNotFoundError: No module named 'xtquant.IPythonApiClient'

- 
- **原因**：Python 版本过高（如 3.13）或未正确添加 bin.x64 的 DLL 路径。
- **解决**：降级 Python 至 3.10，并确保代码中执行了 os.add_dll_directory。

### 2. 报错：Exception: 无法连接行情服务！

- 
- **原因**：QMT 没开、没登录，或者没切到 Mini 模式。
- **解决**：确认 QMT 右上角显示“Mini模式”。确认行情连接（小圆点）是绿色。尝试以**管理员身份**同时运行 QMT 和 PyCharm。

### 3. 获取到的 DataFrame 是空的

- 
- **原因**：本地没有缓存该股票的历史数据。
- **解决**：调用 xtdata.download_history_data。或者在 QMT 软件里手动点开该股票的 K 线图，强制触发软件下载数据。

### 4. 涨幅计算不准

- 
- **原因**：未处理除权除息。
- **解决**：在 get_market_data_ex 中增加参数 dividend_type='front'（前复权）。

------



## 五、 进阶操作：常用 API 简记

- 
- xtdata.get_stock_list_in_sector('沪深A股')：获取全市场代码。
- xtdata.get_full_tick(['600519.SH'])：获取实时盘口（Tick）数据。
- xtdata.subscribe_quote(symbol, callback=my_func)：订阅实时行情，触发回调函数。

------



**资深专家寄语：**
量化交易的第一步是**稳定**。建议环境调通后，不要轻易升级 Python 版本或移动 QMT 安装目录。如果未来需要多机部署，只需将此 MD 文件和对应的 Conda 环境迁移即可。