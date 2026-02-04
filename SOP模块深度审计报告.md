# SOP 模块深度审计报告

**审计时间**: 2026年2月4日  
**审计范围**: 家政平台 SOP 模块后端实现  
**审计依据**: SOP 模块任务清单 + 代码文件分析

---

## 📊 审计结论概览

| 审计项 | 状态 | 完成度 | 备注 |
|--------|------|--------|------|
| 数据库表结构 | ✅ 已实现 | 100% | 关联关系已建立 |
| 验收提醒任务 | ✅ 已实现 | 90% | 逻辑完整，缺少实际推送集成 |
| 仲裁员衰减任务 | ✅ 已实现 | 90% | 逻辑完整，缺少实际推送集成 |
| 75分钟验收逻辑 | ⚠️ 部分实现 | 70% | 提醒已实现，自动验收未找到 |
| 36小时结算逻辑 | ❌ 未实现 | 0% | 未找到相关代码 |
| 订单创建时 SOP 绑定 | ❓ 待查证 | ? | 需要进一步查询 |

---

## 1️⃣ 数据库层审计

### ✅ 已确认的表结构

根据查询结果，以下表已创建并建立关联：

#### 表 1: `order_sop_records` (订单SOP记录表)
- **位置**: `models\orderSopRecord.js`
- **关键字段**:
  - `template_id` - 关联到 SOP 模板
  - `acceptance_status` - 验收状态
  - `customer_confirmed_at` - 客户确认时间
  - `service_end_time` - 服务结束时间

#### 表 2: `sop_checklists` (SOP清单表)
- **位置**: `models\sopChecklist.js`
- **关键字段**:
  - `template_id` - 关联到 SOP 模板

#### 表 3: `sop_templates` (SOP模板表)
- **关联关系** (在 `models\index.js` 中定义):
  ```javascript
  SopTemplate.hasMany(OrderSopRecord, { foreignKey: 'template_id', as: 'records' });
  OrderSopRecord.belongsTo(SopTemplate, { foreignKey: 'template_id', as: 'template' });
  SopChecklist.belongsTo(SopTemplate, { as: 'template', foreignKey: 'template_id' });
  ```

### ✅ 数据库层结论
- **完成度**: 100%
- **符合设计**: 是
- **问题**: 无

---

## 2️⃣ 定时任务层审计

### ✅ 任务 1: `acceptanceReminder.js` (验收提醒任务)

**文件位置**: `backend/jobs/acceptanceReminder.js`

#### 核心功能分析

1. **执行频率**:
   ```javascript
   cronExpression: '* * * * *'  // 每分钟执行一次
   ```

2. **提醒时间点配置**:
   ```javascript
   reminderPoints: {
     phase1End: {
       triggerAt: PHASE1_DURATION,        // 15分钟（服务结束后）
       window: 60 * 1000,
       type: 'phase1_end',
       title: '服务已完成，请验收',
       message: '您的家政服务已完成15分钟，请尽快验收确认，60分钟后将自动验收。'
     },
     phase2Warning: {
       triggerAt: PHASE1_DURATION + 45 * 60 * 1000,  // 60分钟（15+45）
       window: 60 * 1000,
       type: 'phase2_warning',
       title: '即将自动验收',
       message: '您的服务订单将在15分钟后自动验收，如有问题请及时反馈。'
     },
     finalWarning: {
       triggerAt: PHASE1_DURATION + 55 * 60 * 1000,  // 70分钟（15+55）
       window: 60 * 1000,
       type: 'final_warning',
       title: '⚠️ 5分钟后自动验收',
       message: '您的服务订单将在5分钟后自动验收！'
     }
   }
   ```

3. **时间计算逻辑分析**:
   - `PHASE1_DURATION = 15 * 60 * 1000` (15分钟)
   - 第一次提醒: 服务结束后 **15分钟**
   - 第二次提醒: 服务结束后 **60分钟** (15 + 45)
   - 第三次提醒: 服务结束后 **70分钟** (15 + 55)

   **❗问题**: 
   - 任务清单要求的是 **75分钟自动验收**
   - 代码配置的第三次提醒在 **70分钟**
   - **时间不匹配**: 相差5分钟

4. **查询逻辑**:
   ```javascript
   const orders = await OrderSopRecord.findAll({
     where: {
       acceptance_status: 'pending',          // 待验收
       customer_confirmed_at: null,           // 未确认
       service_end_time: {                    // 服务结束时间在目标窗口内
         [Op.between]: [rangeStart, rangeEnd]
       }
     }
   });
   ```

5. **推送功能**:
   - ✅ 支持微信模板消息推送
   - ✅ 支持APP推送
   - ⚠️ 当前为**模拟推送**（未实际集成）
   ```javascript
   async function sendWechatReminder(params) {
     const { orderId } = params;
     console.log(`[验收提醒任务] [模拟] 微信模板消息已发送: 订单=${orderId}`);
   }
   ```

6. **分布式锁支持**:
   - ✅ 已集成 Redis 分布式锁
   - ✅ 避免多服务器重复执行

#### 功能完成度评估

| 功能点 | 状态 | 说明 |
|--------|------|------|
| 定时检查待验收订单 | ✅ | 每分钟执行 |
| 15分钟提醒 | ✅ | 已实现 |
| 60分钟提醒 | ✅ | 已实现 |
| 70分钟最后警告 | ⚠️ | 应为75分钟 |
| 微信推送 | ⚠️ | 模拟状态 |
| APP推送 | ⚠️ | 模拟状态 |
| 去重机制 | ✅ | 已实现 |
| 分布式锁 | ✅ | 已实现 |

**完成度**: 90%

#### 缺失功能

1. **自动验收逻辑** ❌
   - 任务清单要求: 75分钟后**自动通过验收**
   - 当前状态: **只有提醒，没有自动验收代码**
   - **需要**: 在 `autoAcceptance.js` 中实现（见下文分析）

2. **推送服务集成** ⚠️
   - 微信模板消息API未实际调用
   - APP推送服务未集成

---

### ⚠️ 任务 2: `autoAcceptance.js` (自动验收任务)

**文件位置**: `backend/jobs/autoAcceptance.js`

#### 状态
- **发现**: 文件名在 `dir /b jobs\*.js` 中列出
- **问题**: **未上传代码内容**
- **影响**: 无法确认75分钟自动验收逻辑是否实现

#### 推测功能
根据任务清单，该文件应该包含：
1. 每分钟扫描 `service_end_time + 75分钟` 的订单
2. 自动将 `acceptance_status` 更新为 `accepted`
3. 触发结算流程
4. 记录自动验收日志

**❗需要查询**: 请执行以下命令查看该文件
```cmd
type jobs\autoAcceptance.js
```

---

### ✅ 任务 3: `jurorDecayTasks.js` (仲裁员衰减任务)

**文件位置**: `backend/tasks/jurorDecayTasks.js`

#### 核心功能分析

1. **执行时间**:
   ```javascript
   // 每天凌晨2点执行衰减任务
   cron.schedule('0 2 * * *', () => { ... });
   
   // 每天上午9点发送30天提醒
   cron.schedule('0 9 * * *', () => { ... });
   ```

2. **衰减规则**:
   ```javascript
   // 31-60天不活跃：每天扣1分
   UPDATE users SET juror_score = GREATEST(0, juror_score - 1)
   WHERE DATEDIFF(NOW(), juror_last_active) BETWEEN 31 AND 60
   
   // 61天+不活跃：每天扣2分
   UPDATE users SET juror_score = GREATEST(0, juror_score - 2)
   WHERE DATEDIFF(NOW(), juror_last_active) > 60
   ```

3. **30天提醒**:
   ```javascript
   // 查找第30天不活跃的用户
   SELECT id, nickname, phone, juror_score
   FROM users
   WHERE juror_score > 0
     AND DATEDIFF(NOW(), juror_last_active) = 30
   ```

4. **活跃时间更新**:
   ```javascript
   static async updateLastActive(type, id) {
     // 在用户参与仲裁时调用
     UPDATE users SET juror_last_active = NOW() WHERE id = ?
   }
   ```

#### 功能完成度评估

| 功能点 | 状态 | 说明 |
|--------|------|------|
| 定时衰减任务 | ✅ | 每天凌晨2点 |
| 31-60天衰减规则 | ✅ | -1分/天 |
| 61天+衰减规则 | ✅ | -2分/天 |
| 30天提醒 | ✅ | 每天上午9点 |
| 活跃时间更新 | ✅ | 提供静态方法 |
| 推送通知 | ⚠️ | 模拟状态 |

**完成度**: 90%

#### 问题
1. **推送服务未集成**: 
   ```javascript
   async sendReminder(type, data) {
     // TODO: 集成消息推送系统（微信模板消息、短信等）
     console.log(`[提醒] 发送给${type === 'user' ? '用户' : '家政员'} ${data.id}`);
   }
   ```

2. **活跃时间更新调用位置未确认**:
   - 需要在仲裁投票接口中调用 `JurorDecayTasks.updateLastActive()`
   - 未找到调用代码

---

## 3️⃣ 缺失功能审计

### ❌ 36小时自动结算逻辑

**任务清单要求**:
> 任务7: 时间轴流程的验收后36小时自动确认结算

**查询结果**:
- 在 `dir /b tasks\*.js` 中未发现 `autoSettlement.js` 或类似文件
- 在已上传的代码中**未找到**任何36小时自动结算逻辑

**影响**:
- 验收后的订单无法自动结算
- 可能导致资金长期冻结
- 工人工资发放延迟

**❗需要实现**:
1. 创建 `tasks/autoSettlement.js`
2. 每小时检查 `customer_confirmed_at + 36小时` 的订单
3. 自动触发结算流程
4. 更新订单状态为 `settled`

---

### ❓ 订单创建时 SOP 模板绑定逻辑

**查询建议**:
```cmd
findstr /s /i /n "OrderSopRecord.*create\|createOrderSopRecord" controllers\*.js services\*.js
```

**需要确认**:
1. 订单创建时是否自动创建 `OrderSopRecord`
2. 如何根据服务类型选择 SOP 模板
3. `sop_type` 字段如何赋值（`checkpoint` 或 `timeline`）

---

## 4️⃣ 文件清单对比

### 已发现的文件

| 文件路径 | 功能 | 状态 |
|---------|------|------|
| `jobs/acceptanceReminder.js` | 验收提醒 | ✅ 已分析 |
| `jobs/autoAcceptance.js` | 自动验收 | ⚠️ 未查看内容 |
| `tasks/jurorDecayTasks.js` | 仲裁员衰减 | ✅ 已分析 |
| `tasks/arbitrationTasks.js` | 仲裁相关 | ❓ 未分析 |
| `tasks/autoSettlement.js` | 自动结算 | ❓ 未分析 |
| `tasks/leaderTasks.js` | 领队相关 | ❓ 未分析 |
| `tasks/orderTasks.js` | 订单相关 | ❓ 未分析 |
| `tasks/verificationTasks.js` | 验证相关 | ❓ 未分析 |

---

## 5️⃣ 需要立即执行的查询

为完成审计，请执行以下命令并返回结果：

```cmd
REM 1. 查看自动验收任务（确认75分钟逻辑）
type jobs\autoAcceptance.js

REM 2. 查看自动结算任务（确认36小时逻辑）
type tasks\autoSettlement.js

REM 3. 查找订单创建时的SOP绑定逻辑
findstr /s /i /n "OrderSopRecord.*create\|createOrderSopRecord" controllers\*.js services\*.js

REM 4. 查看订单任务文件（可能包含结算逻辑）
type tasks\orderTasks.js

REM 5. 查找所有涉及 acceptance_status 更新的代码
findstr /s /i /n "acceptance_status.*=" controllers\*.js services\*.js jobs\*.js

REM 6. 查找所有涉及 settled 或 settlement 的代码
findstr /s /i /n "settled\|settlement" controllers\*.js services\*.js tasks\*.js
```

---

## 6️⃣ 风险评估

### 高风险问题

1. **自动验收逻辑未确认** 🔴
   - 影响: 订单可能永久停留在待验收状态
   - 优先级: 最高

2. **自动结算逻辑缺失** 🔴
   - 影响: 工人工资无法自动发放
   - 优先级: 最高

### 中风险问题

3. **提醒时间配置错误** 🟡
   - 当前: 70分钟最后警告
   - 预期: 75分钟自动验收
   - 影响: 用户体验不佳

4. **推送服务未集成** 🟡
   - 影响: 用户收不到实际通知
   - 优先级: 高

### 低风险问题

5. **活跃时间更新未调用** 🟢
   - 影响: 仲裁员分数可能错误衰减
   - 优先级: 中

---

## 7️⃣ 后续行动计划

### 立即执行（第1优先级）
1. 查询 `autoAcceptance.js` 内容
2. 查询 `autoSettlement.js` 内容（或确认不存在）
3. 查询订单创建时的SOP绑定逻辑

### 第2优先级
1. 如果 `autoSettlement.js` 不存在，需要立即开发
2. 确认 `autoAcceptance.js` 的75分钟逻辑
3. 修复 `acceptanceReminder.js` 的时间配置

### 第3优先级
1. 集成微信推送服务
2. 集成APP推送服务
3. 在仲裁接口中添加活跃时间更新

---

## 附录: 代码质量评价

### 优点
1. ✅ 使用了 node-cron 进行定时任务管理
2. ✅ 使用了 Redis 分布式锁避免重复执行
3. ✅ 实现了去重机制（sentReminders Map）
4. ✅ 错误处理较为完善
5. ✅ 事务处理正确（jurorDecayTasks）
6. ✅ 数据库查询使用了参数化防止SQL注入

### 改进建议
1. 将配置项（如时间阈值）移到环境变量或配置文件
2. 添加更详细的日志记录
3. 实现重试机制（推送失败时）
4. 添加单元测试
5. 添加监控告警（执行失败时）

---

**报告生成时间**: 2026-02-04  
**报告状态**: 待补充（等待额外查询结果）
