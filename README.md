# ASDDEP

基于 Azure 与 Databricks 的实时数仓平台。

## 项目简介

本项目围绕租车/网约车场景，设计并实现了一套端到端的数据处理链路，用于承接 Web 端产生的实时订单数据与批量基础数据，并逐步沉淀为可分析的数据模型。整体架构遵循 Medallion 分层思想，覆盖数据接入、流式处理、维度建模与数仓落地。

## 核心能力

- 基于 FastAPI 提供简易 Web 入口，模拟用户下单并生成订单事件。
- 基于 Azure Event Hub 实现生产者-消费者模式的数据流式接入，支持低延迟实时写入。
- 基于 Databricks Streaming / Spark Structured Streaming 处理实时订单数据，并与批量基础映射数据汇合。
- 基于 Bronze -> Silver -> Gold 的分层思路完成数据清洗、标准化与主题建模。
- 基于 Jinja2 生成可复用 SQL 模板，减少多表拼接与 OBT 构建中的重复代码。
- 基于 Star Schema 构建事实表与维度表，并在地理维度上实现 SCD Type 2 历史追踪。

## 技术栈

- 云平台：Azure
- 数据平台：Databricks、Spark Structured Streaming
- 数据接入：Azure Event Hub、Azure Data Factory、Azure Data Lake Storage
- 应用层：FastAPI、Jinja2
- 开发语言：Python、SQL

## 架构概览

1. Web 端通过 `FastAPI` 触发模拟下单，请求会生成一条 ride confirmation 事件。
2. 实时订单事件写入 `Azure Event Hub`，作为流式数据入口。
3. 批量基础数据与历史订单样本落入 `ADLS`，作为批处理输入。
4. Databricks 在 Bronze 层接入流式与批量数据，形成统一的 `stg_rides`。
5. Silver 层将订单主数据与车辆、支付、城市、取消原因等映射表关联，生成 `silver_obt`。
6. Gold 层构建星型模型，输出事实表与维度表，支持历史分析与指标查询。

## 数据模型

项目当前沉淀的核心数仓对象包括：

- 事实表：`fact`
- 维度表：`dim_passenger`、`dim_driver`、`dim_vehicle`、`dim_payment`、`dim_booking`、`dim_location`

其中 `silver_obt` 作为宽表中间层，承接 Bronze 明细与各类映射表的统一整合，在此基础上继续拆分为星型模型。

### 事实表说明

#### `fact`

`fact` 以订单/行程为核心分析粒度，用于沉淀可聚合的业务指标，并通过外键关联各维度表，支撑收入、运力、用户行为等主题分析。

- 业务粒度：单笔 ride
- 关键关联键：`ride_id`、`pickup_city_id`、`payment_method_id`、`driver_id`、`passenger_id`、`vehicle_id`
- 主要度量字段：
  - `distance_miles`：行程里程
  - `duration_minutes`：行程时长
  - `base_fare`：基础费用
  - `distance_fare`：里程费用
  - `time_fare`：时长费用
  - `surge_multiplier`：动态加价系数
  - `total_fare`：订单总金额
  - `tip_amount`：小费金额
  - `rating`：乘客评分
  - `base_rate`、`per_mile`、`per_minute`：车辆类型对应的计价规则
- 典型分析场景：
  - 按城市、车型、支付方式统计订单量与收入
  - 分析司机/乘客维度下的里程、时长、评分表现
  - 跟踪高峰溢价对订单金额的影响

### 维度表说明

#### `dim_passenger`

乘客维度，记录乘客主数据，用于分析用户画像与乘车行为。

- 主键：`passenger_id`
- 主要字段：`passenger_name`、`passenger_email`、`passenger_phone`
- 用途：与事实表关联后，可按乘客统计订单次数、消费金额、评分等指标

#### `dim_driver`

司机维度，记录司机基础信息与服务属性，用于分析司机表现与运力情况。

- 主键：`driver_id`
- 主要字段：`driver_name`、`driver_rating`、`driver_phone`、`driver_license`
- 用途：支持司机服务质量、接单表现、评分水平等分析

#### `dim_vehicle`

车辆维度，记录车辆及车型相关属性，用于分析不同车型、品牌和车辆配置对业务表现的影响。

- 主键：`vehicle_id`
- 主要字段：`vehicle_make_id`、`vehicle_type_id`、`vehicle_model`、`vehicle_color`、`license_plate`、`vehicle_make`、`vehicle_type`
- 用途：支持按品牌、车型统计收入、单量、平均客单价等指标

#### `dim_payment`

支付维度，记录支付方式相关属性，用于分析用户支付偏好与支付结构。

- 主键：`payment_method_id`
- 主要字段：`payment_method`、`is_card`、`requires_auth`
- 用途：支持按支付方式分析交易分布、认证要求和支付习惯

#### `dim_booking`

订单维度，记录与一次订单直接相关但不适合作为度量的描述性信息，覆盖订单编号、状态、上下车地点与时间等内容。

- 主键：`ride_id`
- 主要字段：
  - `confirmation_number`
  - `dropoff_location_id`、`pickup_location_id`
  - `ride_status_id`
  - `dropoff_city_id`
  - `cancellation_reason_id`
  - `pickup_address`、`pickup_latitude`、`pickup_longitude`
  - `dropoff_address`、`dropoff_latitude`、`dropoff_longitude`
  - `booking_timestamp`、`dropoff_timestamp`
- 用途：支持订单状态追踪、取消原因分析、上下车位置分析与时序查询

#### `dim_location`

地理位置维度，记录城市及区域属性，用于支持地理层级分析。

- 主键：`pickup_city_id`
- 主要字段：`pickup_city`、`state`、`region`、`city_updated_at`
- 用途：支持按城市、州、区域进行订单与收入汇总分析
- 变更策略：使用 `SCD Type 2` 保留维度历史版本，适合追踪地理属性变更

### 建模补充说明

- `dim_location` 使用 `SCD Type 2` 保留维度历史版本。
- 其余核心维度及事实表通过流式 CDC 方式持续更新。
- 整体模型遵循 Star Schema 设计，便于面向分析场景进行高效关联与聚合。

## 项目结构

```text
ASDDEP/
├── api.py                          # FastAPI 入口
├── connection.py                   # Event Hub 连接与消息发送
├── data.py                         # 订单模拟与样本数据生成
├── templates/                      # Web 页面模板
├── Data/                           # 批量订单与映射数据样本
├── Databricks/
│   ├── bronze_adls.ipynb           # Bronze 层批量入湖与表加载
│   ├── silver_obt.ipynb            # Silver 宽表构建与验证
│   └── rides_ingest/
│       └── rides_ingest/transformations/
│           ├── ingest.py           # Event Hub 流式摄取
│           ├── silver.py           # Bronze 统一接入到 stg_rides
│           ├── silver_obt.sql      # OBT 宽表生成 SQL
│           └── model.py            # Gold 星型模型构建
└── requirements.txt
```

## 运行说明

### 1. 安装依赖

```bash
pip install -r requirements.txt
```

### 2. 配置环境变量

在项目根目录创建 `.env`，至少包含以下配置：

```env
CONNECTION_STRING=your_event_hub_connection_string
EVENT_HUBNAME=your_event_hub_name
```

### 3. 启动 Web 服务

```bash
uvicorn api:app --reload
```

启动后访问 `http://127.0.0.1:8000`，点击页面按钮即可模拟下单并向 Event Hub 发送实时订单消息。