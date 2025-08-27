# Grafana 配置和使用指南

## 访问 Grafana

1. 打开浏览器访问: http://localhost:3000
2. 默认登录凭据:
   - 用户名: `admin`
   - 密码: `admin`
3. 首次登录时会要求修改密码

## 已配置的功能

### 1. 数据源
- **Prometheus**: 已自动配置，连接到 http://prometheus:9090

### 2. 预置仪表板
已安装 **NATS Cluster Monitoring** 仪表板，包含以下监控项：

#### 实时状态
- **NATS Server Status** - 服务器在线状态
- **Active Connections** - 活跃连接数
- **Memory Usage** - 内存使用量
- **Cluster Routes** - 集群路由数

#### 性能指标
- **Message Rate** - 消息吞吐率（收/发消息数/秒）
- **Bandwidth** - 带宽使用（收/发字节数/秒）
- **Total Subscriptions** - 订阅总数
- **Slow Consumers** - 慢消费者数量
- **JetStream Memory** - JetStream 内存使用

### 3. 告警规则
已配置以下告警：

| 告警名称 | 触发条件 | 严重级别 |
|---------|---------|---------|
| NATS Server Down | 服务器宕机超过2分钟 | Critical |
| High Memory Usage | 内存使用超过1GB持续5分钟 | Warning |
| Too Many Slow Consumers | 慢消费者超过10个持续5分钟 | Warning |
| No Cluster Routes | 没有集群路由连接超过2分钟 | Critical |

## 使用步骤

### 查看仪表板
1. 登录后点击左侧菜单的 **Dashboards**
2. 选择 **NATS** 文件夹
3. 点击 **NATS Cluster Monitoring** 仪表板

### 自定义仪表板
1. 在仪表板页面点击右上角的设置图标
2. 可以：
   - 添加新的面板
   - 修改现有面板的查询
   - 调整时间范围
   - 设置自动刷新间隔

### 查看告警
1. 点击左侧菜单的 **Alerting**
2. 可以查看所有告警规则和状态
3. 点击具体告警查看详情和历史

## 常用 PromQL 查询示例

```promql
# 查看所有 NATS 服务器状态
up{job="nats"}

# 计算消息吞吐率
rate(gnatsd_varz_in_msgs[5m])

# 查看内存使用趋势
gnatsd_varz_mem

# 统计慢消费者
gnatsd_varz_slow_consumers

# 查看集群路由健康状态
gnatsd_routez_num_routes
```

## 添加自定义面板

1. 点击仪表板右上角的 **Add panel**
2. 选择 **Add a new panel**
3. 在 Query 部分输入 PromQL 查询
4. 选择可视化类型（Graph, Gauge, Stat, Table等）
5. 设置面板标题和其他选项
6. 点击 **Apply** 保存

## 配置通知渠道

如需配置告警通知（邮件、Slack、钉钉等）：

1. 进入 **Alerting** > **Contact points**
2. 点击 **New contact point**
3. 选择通知类型并配置
4. 在告警规则中关联通知渠道

## 导出/导入仪表板

### 导出
1. 打开仪表板
2. 点击设置图标 > **JSON Model**
3. 复制 JSON 内容保存

### 导入
1. 点击 **Dashboards** > **Import**
2. 粘贴 JSON 或上传文件
3. 选择数据源
4. 点击 **Import**

## 故障排查

### 如果看不到数据
1. 检查 Prometheus 是否正常：http://localhost:9090
2. 检查 NATS Exporter：http://localhost:7777/metrics
3. 在 Grafana 中测试数据源连接

### 如果告警不工作
1. 检查告警规则配置
2. 查看告警评估日志
3. 确认通知渠道配置正确

## 性能优化建议

1. **调整刷新间隔**: 对于生产环境，建议设置为30秒或更长
2. **限制时间范围**: 查询大时间范围可能影响性能
3. **使用变量**: 创建模板变量来快速切换监控目标
4. **缓存查询结果**: 启用查询缓存以提高响应速度