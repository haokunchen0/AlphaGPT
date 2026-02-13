# CODEBUDDY.md This file provides guidance to CodeBuddy when working with code in this repository.

## 常用命令

### 安装依赖
```bash
pip install -r requirements.txt
```
安装核心依赖，包括PyTorch、Pandas、SQLAlchemy、aiohttp、Solana SDK和Streamlit。如需运行实验脚本（times.py和lord/experiment.py），还需安装可选依赖：
```bash
pip install -r requirements-optional.txt
```

### 运行数据管道
```bash
cd data_pipeline && python run_pipeline.py
```
异步从Birdeye和DexScreener获取加密货币市场数据，清洗后存入PostgreSQL数据库。需要先设置环境变量`BIRDEYE_API_KEY`和数据库连接信息。管道每天同步一次，也可手动运行。

### 训练AlphaGPT模型
```bash
cd model_core && python -c "from engine import AlphaEngine; eng = AlphaEngine(use_lord_regularization=True); eng.train()"
```
使用CryptoDataLoader加载数据，训练AlphaGPT模型生成交易公式。训练完成后最佳公式保存为`best_meme_strategy.json`。支持低秩衰减（LoRD）正则化以提高泛化能力。

### 启动实时交易策略
```bash
cd strategy_manager && python runner.py
```
加载训练好的公式，循环监控市场，根据信号执行买卖。策略包含止损、止盈、移动止损和AI信号退出。需要配置Solana私钥和RPC URL。策略每60秒扫描一次，最大持仓数可配置。

### 启动仪表板
```bash
streamlit run dashboard/app.py
```
启动Streamlit Web面板，显示持仓、市场机会、系统日志。支持自动刷新和紧急停止功能。仪表板从数据库读取实时数据，依赖数据管道正常运行。

### 运行中国股票回测（times.py）
```bash
python times.py
```
独立脚本，使用Tushare获取A股数据，训练AlphaGPT模型进行开-开收益回测。输出策略公式和测试期表现图表。需要设置Tushare token。

### 运行LoRD正则化实验
```bash
cd lord && python experiment.py --mode mechanism
```
研究低秩衰减正则化对Transformer泛化的影响。支持相图扫描和机制分析两种模式，生成可视化结果。

## 高级架构

AlphaGPT是一个用于加密货币（主要针对Solana链上Meme币）自动交易的端到端量化系统。系统基于强化学习训练一个Transformer模型（AlphaGPT），该模型输出波兰表示法的数学公式，这些公式将市场特征转换为交易信号。系统分为六个核心模块，通过异步数据流和共享数据库协同工作。

### 数据管道 (data_pipeline/)
负责市场数据的采集、清洗和存储。支持多数据提供商（Birdeye付费API、DexScreener免费API），按时间范围（1分钟、15分钟）获取OHLCV、流动性、市值等数据。数据管理器（DataManager）协调获取器（Fetcher）、处理器（Processor）和数据库管理器（DbManager），实现并发请求和增量更新。清洗后的数据存入PostgreSQL表`ohlcv`和`token_metadata`，供模型训练和实时推理使用。管道设计为每日自动同步，也可手动触发。

### 模型核心 (model_core/)
实现AlphaGPT模型及其训练引擎。模型架构为Transformer编码器，输入为公式token序列，输出下一个token的logits。训练采用策略梯度（REINFORCE），奖励函数为基于历史数据的Sortino比率。关键组件包括：
- **AlphaGPT**: Transformer模型定义，支持低秩衰减正则化（LoRD）。
- **CryptoDataLoader**: 从数据库加载数据并构建特征张量（收益率、成交量变化、趋势等）。
- **StackVM**: 虚拟机，执行波兰表示法公式，将token序列转换为因子值。
- **MemeBacktest**: 回测引擎，计算因子在历史数据上的收益和风险指标。
- **AlphaEngine**: 训练循环，生成公式、评估奖励、更新模型参数。

训练过程批量生成公式，用VM执行得到因子，回测计算Sortino比率作为奖励，通过策略梯度更新模型。最佳公式保存为JSON文件。

### 策略管理器 (strategy_manager/)
执行实时交易决策。策略循环（run_loop）每60秒运行一次，依次执行：同步数据管道（每15分钟）、监控持仓、扫描入场机会。持仓管理（PortfolioManager）跟踪 entry price、最高价、当前持仓量，并实现止损、止盈、移动止损逻辑。风险引擎（RiskEngine）检查流动性和市值安全性。交易信号由VM执行最佳公式得到因子值，并转换为0-1概率分数，高于阈值则触发买入。

### 执行层 (execution/)
封装Solana区块链交互。通过Jupiter API获取实时报价和路由，使用Solders库构建和发送交易。SolanaTrader处理买卖操作，支持滑点控制和优先费。RPC处理器管理节点连接。需要配置私钥和RPC端点。

### 仪表板 (dashboard/)
基于Streamlit的监控界面，展示持仓列表、市场机会散点图、PnL分布和系统日志。数据服务（DashboardService）从数据库查询实时状态。支持紧急停止信号（写入STOP_SIGNAL文件）。

### 独立研究脚本
- **times.py**: 独立的中国股票市场回测系统，使用Tushare数据，采用类似的AlphaGPT架构但针对A股特点优化。
- **lord/experiment.py**: 研究低秩衰减正则化对Transformer泛化的影响，包含相图扫描和注意力模式可视化。

### 数据流
1. 数据管道异步获取原始市场数据，清洗后存入PostgreSQL。
2. 模型训练时，CryptoDataLoader从数据库读取数据，构建特征张量。
3. AlphaEngine训练生成公式，保存最佳公式到JSON。
4. 策略管理器加载公式，循环中从数据库读取最新数据，用VM计算因子分数。
5. 当分数超过买入阈值且风险检查通过，调用执行层发起链上交易。
6. 持仓状态写入本地JSON文件，仪表板读取数据库和文件展示实时状态。

### 关键技术点
- **波兰表示法公式**: 模型输出操作符和特征token组成的序列，由VM递归求值，确保生成的公式数学上合法。
- **低秩衰减正则化**: 对Transformer的Q、K投影矩阵施加低秩约束，防止过拟合，提升泛化能力。
- **开-开收益率**: 回测和交易使用开盘价到下一个开盘价的收益率，避免盘中波动和滑点影响。
- **异步并发**: 数据获取和交易执行均采用asyncio，提高吞吐量。
- **模块化设计**: 各模块通过配置类和环境变量解耦，便于单独测试和部署。

系统设计为半自动运行，数据管道和策略管理器可部署为后台服务，仪表板提供人工监控界面。训练阶段需要GPU加速，推理阶段可在CPU运行。适用于Solana链上高波动性Meme币的短线交易。