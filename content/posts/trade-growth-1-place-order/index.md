---
title: "交易的生长史（一）：一个叫“下单”的接口"
date: 2026-11-10T01:00:00+08:00
draft: false
description: "一个叫“下单”的接口，如何被一次次具体的麻烦逼成“订单”，再逼成“交易”：状态机、快照、价格明细、库存占用，以及名字背后视野的变化。（技术叙事，人物与数据均为虚构复合）"
tags: ["交易系统", "订单设计", "架构演进", "领域建模"]
categories: ["架构演进"]
---

> 这是「拾光」系列的新篇章，也是《交易的生长史》的第一篇，与[《一个互联网应用的生长史》](/posts/growth-history/)同一个世界。主角是创始团队的工程师老魏。
> 公司、人物、数字、事故均为虚构复合，但每个阶段的困境、决策和代价，来自真实交易系统里反复发生的事。
> 文中代码是**示意代码，未经完整编译**；2016–2018 年的代码按当时的 Java 8 写法书写。
> 我想讲的不是"订单系统应该怎么设计"，而是：**一个叫"下单"的接口，是怎么被一次次具体的麻烦，逼成"订单"，再逼成"交易"的。**
> 注意看它的名字——名字变了，说明它管的事变了。

---

## 0｜2025 年，周六晚上十点四十

值班室里只有两个人。

客服升级上来一张工单，沈砚打开，看了三遍。

> 用户购买"周末游套餐"，实付 1288 元。套餐含：景区门票 2 张、酒店 1 晚、餐券 2 张。
> 门票：已出票。餐券：已发放。酒店：**确认失败**（对方房态已满）。
> 用户要求：全额退款并补偿。
> 系统：主订单状态"处理中"，退款按钮灰色。

沈砚把工单转给了老魏。

他想问的不是怎么处理，处理方案有预案；他想问的是另一件事。他在工单旁边的订单详情页里，数了一下这一单上挂着的状态：主单一个，三个子单各一个，外加履约、支付、退款各自的状态——**一共十一个**。

"魏哥，"他说，"当初就没想过一次把状态想清楚吗？"

老魏盯着屏幕，没有马上回答。过了一会儿，他说：

"想不清楚的。那时候它还不叫'交易'，也不叫'订单'，就叫'下单'，是个接口。第一个状态字段，只有五个值，我们当时觉得**绰绰有余**。"

他把椅子转过来：

"你知道最麻烦的是什么吗？不是它后来变得多复杂，而是**每一次变复杂，当时都是对的**。"

"从头讲吧。"沈砚说。

"从 2016 年开始。那时候，还没人叫我老魏。"

---

## 1｜2016 年初：它叫"下单"

### 1.1 一周

2016 年 1 月，拾光上线满一年。

日访问两万多，注册用户八万。到目前为止，用户在这里看图文、关注别人、收藏商品，想买的时候，**点一下链接，跳去第三方电商下单**。

周总（创始人）在周一的会上说：

> "我们一直在给别人导流。我想做自己的商城。**一周内要能卖货。**"

团队六个人。老魏当时二十八岁，是三个创始工程师之一，那年大家还叫他小魏。需求落在他手上：用户点"购买"，能生成一个订单，能付款。

他没有设计"订单系统"。他只打算写**一个接口**。

### 1.2 第一版

整个拾光当时是一个 Java 应用，打成 war 包扔进 Tomcat。他在现有的 `shiguang-web` 里加了一个包 `shop`，包里放一个控制器：

```java
// shiguang-web / com.shiguang.shop.PlaceOrderController
@PostMapping("/shop/placeOrder")
public String placeOrder(@RequestParam long userId,
                         @RequestParam long skuId,
                         @RequestParam int quantity) {
    Sku sku = skuDao.get(skuId);
    if (sku.getStock() < quantity) {
        throw new SoldOutException(skuId);
    }
    skuDao.reduceStock(skuId, quantity);            // 扣库存
    BigDecimal amount = sku.getPrice().multiply(BigDecimal.valueOf(quantity));
    String orderNo = newOrderNo();
    orderDao.insert(orderNo, userId, skuId, quantity, amount, 0);   // 0 = 待付款
    return orderNo;
}
```

一张表：

```sql
CREATE TABLE orders (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_no    VARCHAR(32) NOT NULL,
  user_id     BIGINT NOT NULL,
  sku_id      BIGINT NOT NULL,
  quantity    INT NOT NULL,
  amount      DECIMAL(10,2) NOT NULL,    -- 元
  status      TINYINT NOT NULL,          -- 0 待付款 1 已付款
  created_at  DATETIME NOT NULL
);
```

支付走第三方渠道，渠道付款成功后会通知拾光一个地址。他又写了一个：

```java
@PostMapping("/shop/payNotify")
public String payNotify(@RequestParam String orderNo) {
    orderDao.updateStatus(orderNo, 1);              // 1 = 已付款
    return "success";
}
```

三十行，一个下午。

### 1.3 为什么它当时是对的

这件事值得多说一句，因为很多人第一反应是"这也太简陋了"。

- 日订单量，头一个月是**四十来单**。
- 团队六个人，没有人懂交易，也没有人要懂：目标是**周五能卖出第一件货**。
- 任何"订单系统""支付中心""状态机"，在这个阶段都是对**未知**的猜测。

老魏后来说过一句话：

> **"你不知道将来哪里会疼，所以先别给它装护具。先让它跑，疼的地方会告诉你。"**

### 1.4 第一个疼点：最后一件

上线第八天，运营做了一个"限量 10 件"的活动。

有两个用户在同一秒点了购买，库存只剩最后 1 件。两个请求都读到了 `stock = 1`，都通过了 `if (sku.getStock() < quantity)`，都执行了扣减。

库存变成了 **-1**。

仓库那边的同事发来一条消息："这单发不出去，你们系统卖了不存在的东西。"

这是老魏第一次真正面对**并发**。问题出在"先读、再判断、再写"这三步之间，是可以被别人插进来的。他把检查和扣减合成了一条 SQL：

```sql
UPDATE sku
   SET stock = stock - #{quantity}
 WHERE id = #{skuId}
   AND stock >= #{quantity}
```

```java
int affected = skuDao.reduceStock(skuId, quantity);
if (affected == 0) {
    throw new SoldOutException(skuId);          // 没改到任何一行，说明被别人抢先了
}
```

**判断条件放进 WHERE 里，由数据库在写的那一刻裁决。**

他把这个手法记在了笔记本上，没想到它在接下来三年里还要用无数次。

### 命名与视野

这一章里，所有东西都是**动作**：

- 方法叫 `placeOrder`（下单）——动词
- 包叫 `shop`（商城）——它只是商城页面背后的一个功能
- 异步通知叫 `payNotify`——也是动词

**这个阶段的视野是：用户点了一下按钮，系统要做完一件事。** 订单只是这件事留下的一条记录。

### 新问题

> 它开始"留下东西"了。这些东西，会有自己的故事。

---

## 2｜2016 年夏：订单有了脾气

### 2.1 一连串的怪事

接下来三个月，订单量从日均四十涨到了几百。怪事一件接一件地来。

**怪事一：同一笔钱，通知了三次。**

支付渠道的通知有一个特点：**它没收到你回的 `success`，就会重发。** 拾光的服务在高峰期偶尔会慢，渠道没等到回应，就再发一次。

老魏的 `payNotify` 每次都老老实实执行 `updateStatus(orderNo, 1)`。单看没问题，状态本来就是 1。但同事在这个接口里**顺手加了两行**：发一条站内信、给用户加 10 个积分。

一个用户收到了三条站内信，加了 30 积分。

**怪事二：用户付了钱，订单却是"已关闭"。**

运营要求：**30 分钟没付款的订单自动关闭，把库存放回去。** 老魏写了一个定时任务，每分钟扫一遍。

有个用户在第 29 分 58 秒点了付款。付款的通知，在第 30 分 02 秒到达，但**就在 2 秒之前，定时任务刚把订单关掉了。**

通知进来，执行 `updateStatus(orderNo, 1)`，订单从"已关闭"被改回"已付款"，**库存却已经放回去了，被别人买走了。**

**怪事三：库存被放了两次。**

这个时候拾光已经有两台应用机器（《生长史》里那次扩容）。定时任务在两台机器上各跑一份。**两个任务同时扫到同一个超时订单，各自关闭、各自放回库存。**

库存多出了一份。

**怪事四：对账差了 3 笔。**

财务老梁每天早上拉渠道的对账单，和拾光的订单逐笔对。那个月对出来 3 笔：渠道收了钱，拾光订单是"待付款"。

老魏一笔一笔排查，发现是通知到达时数据库正好有一次短暂的连接超时，`updateStatus` 失败了，但 `payNotify` 吞掉了异常，仍然回了 `success`。**渠道以为我们收到了，不再重发。**

四件怪事，看起来互不相关。老魏把它们写在白板上，盯了很久，然后意识到：

> **它们是同一件事。订单不再是"一条记录"，它有了"生命"：有些状态能走到另一些状态，有些不能；而改变状态的人，不止一个，还可能同时来。**

### 2.2 决策：把"状态"当成一件正经事

**选项：**

| 方案 | 做法 | 代价 |
|---|---|---|
| A. 到处加判断 | 每个会改状态的地方，先查再改 | 同一个"先读后写"的窗口，换个地方又出现（最后一件那件事已经教过一次） |
| B. **状态机 + 条件更新** | 先把"合法的流转"写成一张表；每次改状态，用 `WHERE status = 旧值` 让数据库裁决 | 要多写一张表、一个类；所有改状态的地方必须走同一个入口，不能再随手 `update` |

他选了 B。

**第一步：写下"允许怎么走"。**

```java
public enum OrderStatus {
    UNPAID, PAID, SHIPPED, COMPLETED, CLOSED;

    private static final Map<OrderStatus, Set<OrderStatus>> NEXT =
            new EnumMap<OrderStatus, Set<OrderStatus>>(OrderStatus.class);
    static {
        NEXT.put(UNPAID,    EnumSet.of(PAID, CLOSED));
        NEXT.put(PAID,      EnumSet.of(SHIPPED));
        NEXT.put(SHIPPED,   EnumSet.of(COMPLETED));
        NEXT.put(COMPLETED, EnumSet.noneOf(OrderStatus.class));
        NEXT.put(CLOSED,    EnumSet.noneOf(OrderStatus.class));
    }

    public boolean canMoveTo(OrderStatus to) {
        return NEXT.get(this).contains(to);
    }
}
```

**第二步：唯一的改状态入口。**

```java
// shiguang-order / com.shiguang.order.OrderService
@Transactional
public boolean moveTo(String orderNo, OrderStatus from, OrderStatus to) {
    if (!from.canMoveTo(to)) {
        throw new IllegalTransitionException(orderNo, from, to);
    }
    return orderDao.updateStatus(orderNo, from, to) == 1;
}
```

```sql
UPDATE orders
   SET status = #{to}, updated_at = NOW()
 WHERE order_no = #{orderNo}
   AND status   = #{from}
```

关键在返回值：**`true` 意味着"这一次流转是我赢得的"，`false` 意味着"别人先改了"。** 返回 `false` 的那一方，不再往下做任何事。

### 2.3 四件怪事，逐个收拾

**怪事一（重复通知）。** 先记录"这笔通知是否处理过"，再改状态：

```java
@Transactional
public NoticeDisposition accept(PaymentNotice notice) {
    // (channel, channel_trade_no) 上有唯一索引
    if (noticeDao.insertIfAbsent(notice) == 0) {
        return NoticeDisposition.DUPLICATE;            // 渠道重发，之前处理过
    }
    if (orderService.moveTo(notice.getOrderNo(), UNPAID, PAID)) {
        return NoticeDisposition.APPLIED;
    }
    // 没赢得流转：订单已不是"待付款"
    refundTodoDao.insert(notice);                      // 钱收了，订单却不能收，进待退款清单
    return NoticeDisposition.LATE_AFTER_CLOSE;
}
```

写通知记录和改状态**在同一个事务里**——要么都成功，要么都没发生。重复通知落在唯一索引上，直接被挡住，站内信和积分的那两行只在 `APPLIED` 时执行一次。

返回值不是 `boolean`，也不是一个叫 `Result` 的泛泛对象，而是一个**说得出名字的枚举**：`APPLIED`（生效）、`DUPLICATE`（重复）、`LATE_AFTER_CLOSE`（订单已关闭后才到）。调用方一看就知道发生了什么，日志里也能直接统计。

**怪事二（付款与关单赛跑）。** 不再靠"谁先到"，而是让数据库裁决。付款通知和关单任务都调用同一个 `moveTo(…, UNPAID, …)`，**只会有一个返回 `true`**。

输的是付款方，就是上面代码里的 `LATE_AFTER_CLOSE`：订单保持已关闭，钱**不丢，不装作没收到**，而是进"待退款清单"，由程序自动发起退款。

**怪事三（库存放两次）。** 一个原则：**只有赢得"关闭"这次流转的那个人，才有资格去放库存。**

```java
public void closeExpired() {
    List<String> candidates = orderDao.findUnpaidBefore(now().minusMinutes(30), 200);
    for (String orderNo : candidates) {
        if (orderService.moveTo(orderNo, OrderStatus.UNPAID, OrderStatus.CLOSED)) {
            stockService.release(orderNo);           // 只有赢家才释放
        }
    }
}
```

两台机器同时扫到同一单，**只有一台能返回 `true`**。另一台什么都不做。

**怪事四（对账差异）。** 两个动作：一是 `accept` 里的异常**不再吞掉**，失败就不回 `success`，让渠道重发（重复由唯一索引兜底）；二是加一个**每日对账任务**：拉渠道账单，与 `payment_notice` 逐笔比对，差异写入待处理表，由财务确认。

### 2.4 它的名字变了

重构的过程中，老魏发现自己一直在写这样的句子：

- "**订单**什么时候能关闭"
- "**订单**能不能从已付款退回待付款"
- "**订单**现在处于什么状态"

主语从"下单"变成了"订单"。这个变化是悄悄发生的，但它让代码的结构跟着变了：

| 之前（2016 年初） | 现在（2016 年夏） |
|---|---|
| 包：`shiguang-web` 里的 `shop` | 独立的 maven 模块 `shiguang-order` |
| `PlaceOrderController.placeOrder()` | `OrderService.place()`、`moveTo()`、`close()` |
| `payNotify`（一个动词） | `PaymentNotice`（一个"通知"，有自己的表、自己的唯一键） |
| 表 `orders`（一张） | `orders` + `payment_notice` + `refund_todo` |
| 状态：数字 0、1 | 枚举 `OrderStatus` + 流转表 |

**模块拆出来，不是因为"架构应该分模块"，而是因为有了一堆只和订单有关的规则，它们需要一个家。** 这个家叫"订单模块"。

注意一个细节：它仍然在同一个 war 包里，仍然共用同一个数据库，**只是"边界"在代码里出现了。**

### 2.5 怎么知道变好了

| 指标 | 之前 | 之后 |
|---|---|---|
| 重复通知造成的重复站内信 | 偶发，每周几十条 | 0 |
| "付了钱订单被关闭"的投诉 | 每周 2–3 起 | 0 起，自动进退款清单 |
| 库存对不上（超放） | 每周几次 | 一个月内 0 次 |
| 对账差异 | 每月 3 笔，人工查 | 每日自动出清单，当天处理 |

### 2.6 它的代价

- **所有人都必须走 `moveTo`。** 三个月后，还是有同事为了赶需求直接 `UPDATE orders SET status = 3`，绕过了流转表。老魏加了一条规则：代码评审里，**搜到 `UPDATE orders SET status` 直接打回**。
- **"流转表"是死的，业务是活的。** 后来运营要求"已发货也能取消"，`SHIPPED → CLOSED` 这条路得补，补了之后还要问：库存怎么办、钱怎么办。**每多一条边，都是一个要回答的业务问题。**
- **只有一个 `status` 字段，终究会不够用。** 业界对订单状态的常见建议，是"主状态 + 子状态"，配合显式的流转表，避免随手改状态产生的"脏状态"。老魏当时并不知道，他是后来才一次次撞到：**"已付款"其实包含了"待发货""部分发货""已出库"……** 这个问题，要拖到好几年之后才会真正爆发（见本系列第 4 篇）。
- **不是所有订单都需要状态机。** 如果每天只有四十单、一个人处理，这一章的代码是过度设计。它之所以值得，是因为**两台机器、渠道重发、定时任务这三件事叠在一起**，让"同时来"变成了常态。

### 新问题

> 订单现在有了生命，但它的**内容**还有一个问题——用户看到的东西，和他下单时看到的东西，不一定是一回事。

---

## 3｜2017 年：订单里，价格是谁的

### 3.1 改价的那个下午

2017 年 3 月的一个下午，客服群里炸了。

一位用户一个月前买过一件帆布包，付了 169 元。今天他打开"我的订单"，**价格显示 129 元，名字也变了**，并且发来一句话："我是不是多付了 40 块？"

调查结果很简单：订单详情页是这样查的——

```sql
SELECT o.order_no, o.quantity, s.title, s.price, s.cover_image
  FROM orders o JOIN sku s ON o.sku_id = s.id
 WHERE o.order_no = ?
```

`title`、`price`、`cover_image` 都是 **现在** 的商品数据。运营上周把这件商品降了价，改了标题。**历史订单跟着变了。**

更严重的是，`orders.amount` 里存着当时付的钱，**页面展示的价格和付的钱对不上**，用户和客服都说不清。

### 3.2 选项

订单里要展示"当时买的是什么、多少钱"，有几种做法：

| 方案 | 做法 | 优点 | 代价 |
|---|---|---|---|
| A. 继续引用商品表 | 订单只存 `sku_id`，展示时关联 | 存储省、改商品一处生效 | 历史订单会"被修改"，对账、售后、纠纷全部失真 |
| B. **订单里存一份副本** | 下单时，把需要展示的字段**复制**进订单行 | 订单是自洽的历史，不依赖商品当前状态 | 多存数据；字段选少了，以后补不回来 |
| C. 商品做版本化 | 商品每次修改生成新版本，订单引用版本号 | 不重复存储，可追溯 | 商品系统要重做，当时没有人力 |

老魏选了 B，并且给自己定了一条界线：

> **订单里存"买卖双方对这笔交易达成一致的内容"，而不是"为了展示方便的东西"。**

具体包括：商品标题、规格描述、成交单价、主图 URL、数量。**不包括：**库存、销量、当前是否上架这类"会变、也不属于这笔交易"的信息。

### 3.3 代码与表

多件商品、购物车也在这一年上线，所以"一单一商品"的结构撑不住了。订单拆成了**订单头**和**订单行**：

```sql
CREATE TABLE order_line (
  id                BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_no          VARCHAR(32) NOT NULL,
  line_no           INT NOT NULL,
  sku_id            BIGINT NOT NULL,
  -- ↓ 快照：下单那一刻，买卖双方约定的内容
  sku_title         VARCHAR(200) NOT NULL,
  sku_spec          VARCHAR(500),
  unit_price        DECIMAL(10,2) NOT NULL,     -- 成交单价（元）
  cover_image_url   VARCHAR(500),
  quantity          INT NOT NULL,
  UNIQUE KEY uk_order_line (order_no, line_no)
);
```

```java
// shiguang-order / com.shiguang.order.OrderLine
public static OrderLine snapshotOf(int lineNo, Sku sku, int quantity) {
    OrderLine line = new OrderLine();
    line.lineNo = lineNo;
    line.skuId = sku.getId();
    line.skuTitle = sku.getTitle();           // 复制，而不是引用
    line.skuSpec = sku.getSpecText();
    line.unitPrice = sku.getPrice();
    line.coverImageUrl = sku.getCoverImageUrl();
    line.quantity = quantity;
    return line;
}
```

订单详情页改成只读 `order_line`，不再关联商品表。

### 3.4 代价与边界

- **字段选少了，补不回来。** 三个月后售后要求展示"商品的材质说明"，快照里没有。老魏只能给新订单补，**旧订单永远缺这一项。** 这是副本方案的固有代价。
- **存储变大。** 每一行多存了几百字节。他认为这笔账值得，原因只有一个：**当用户和平台对价格有争议时，订单必须是唯一的事实依据。**
- **不是所有信息都该快照。** 如果快照了"库存""是否上架"，订单会出现"已经下架的商品显示可购买"这类错误。**快照的是"当时约定的"，不是"当时的世界"。**
- **什么时候不需要：** 如果商品几乎不变（比如标准化的虚拟充值），引用商品表也许就够了。

### 3.5 它的名字：订单有了"行"

这一章里，一个很小的名词变化，影响了后面所有的代码：

**订单不再是"一个人买了一件东西"，而是"一个人买了若干东西，每一项都有自己的约定"。** `Order` 里有了 `List<OrderLine>`，后面的积分、优惠、退款，都会挂在"行"上。

### 新问题

> 订单行里的 `unit_price` 是商品的单价。但用户**实际付的钱**，并不等于单价乘数量。优惠券、积分、运费，要进来了。

---

## 4｜2017 年底到 2018 年：钱被拆开了

### 4.1 一张表放不下的"价格"

2017 年 11 月，为了筹备第二年的大促，拾光把几件原本各自独立的事一起接进了下单：优惠券（2016 年其实就有了一个最简单的满减版本，当时只由运营手工核销，不参与价格计算）、积分抵扣、满减活动、包邮门槛。

在这之前，价格计算只是一行：

```java
BigDecimal amount = sku.getPrice().multiply(BigDecimal.valueOf(quantity));
```

到了十二月，同一个方法里变成了这样：

```java
BigDecimal amount = sumOfLines(lines);
amount = applyPromotion(amount, promotions);        // 满减
amount = applyCoupon(amount, couponCode);           // 券
amount = applyPoints(amount, pointsToUse);          // 积分
amount = amount.add(calcShipping(amount, address)); // 运费
```

每一步都在**覆盖** `amount`。最后存进库里的，只有最终的一个数字。

于是问题来了：

- 用户退了其中一件商品，**该退多少钱？** 订单里只存了总价，不知道这件商品分摊了多少优惠。
- 财务要对账：**这一单优惠券到底承担了多少？** 无从查起。
- 客服被问"为什么比标价便宜了 23.7 元"，**答不上来是哪个优惠**。

### 4.2 决策：价格要"可解释"

**把"算出一个数"改成"算出一份明细"。**

老魏给这份明细起了一个名字：**报价单（`Quote`）**。名字不是随便起的：它描述的就是"系统在下单前，对这一笔交易给出的一份完整的、可核对的报价"。

```java
// shiguang-order / com.shiguang.order.pricing
public enum PriceKind { ITEM, PROMOTION, COUPON, POINTS, SHIPPING }

public final class PriceLine {
    private final PriceKind kind;
    private final String sourceRef;      // 哪张券、哪个活动
    private final long amountFen;        // 带符号，优惠为负数，单位：分
    private final Integer orderLineNo;   // 分摊到哪一行；整单级别的为 null
    // 构造器与 getter 略
}

public final class Quote {
    private final List<PriceLine> lines;

    public long payableFen() {
        long sum = 0;
        for (PriceLine l : lines) sum += l.getAmountFen();
        return sum;
    }
    // 按订单行汇总：该行实际承担的金额
    public long payableOfLine(int orderLineNo) { /* 略 */ }
}
```

**这里有一个决定，影响深远：新表里的金额，统一用"分"（整数），不再用"元"（小数）。**

原因很具体：优惠分摊、折扣计算会产生小数，`BigDecimal` 的舍入规则一不留神就不一致，而"分"的整数相加永远精确。

但他没来得及把**所有**旧代码都改过来。优惠券模块是更早写的，**它里面的金额还是"元"。** 老魏在评审会上说了一句：

> "券那块先不动，边界清楚，只要在交界处转换一下就行。"

这句话，会在 2019 年双十一那天找他算账。那是第 2 篇的事。

### 4.3 分摊：难的不是算，是"尾差"

**为什么要把优惠分摊到行？** 因为退款。用户买了 A 和 B 两件商品，用了一张满减券，现在要退 A。退多少？**不能退 A 的标价，应该退 A 实际承担的金额。** 这个数字，只有在下单时就分摊好、存下来，才不会吵起来。

业界这类文章里，常见的做法是**按商品金额比例分摊优惠到各个 SKU**。老魏的实现也是这个思路，但他遇到一个小麻烦：

三件商品，价格分别 100 元、100 元、100 元，一张 10 元的券。按比例，每件分摊 3.3333 元。以"分"为单位：**333 + 333 + 333 = 999，差了 1 分。**

```java
/** 把 totalFen 按 weights 的比例分成若干份，保证各份之和恰好等于 totalFen。 */
static long[] apportion(long totalFen, long[] weights) {
    long sum = 0;
    int maxIdx = 0;
    for (int i = 0; i < weights.length; i++) {
        sum += weights[i];
        if (weights[i] > weights[maxIdx]) maxIdx = i;
    }
    if (sum == 0) throw new IllegalArgumentException("没有可分摊的金额");

    long[] parts = new long[weights.length];
    long assigned = 0;
    for (int i = 0; i < weights.length; i++) {
        parts[i] = totalFen * weights[i] / sum;      // 向下取整
        assigned += parts[i];
    }
    parts[maxIdx] += totalFen - assigned;            // 尾差归给金额最大的一行
    return parts;
}
```

两个细节：

1. **尾差归谁？** 他选"金额最大的行"。原因：最大的行，多一分少一分，占比最小，用户感知最低。如果归给最后一行，而最后一行恰好金额是 0 的赠品，就会出现"赠品被分摊了优惠"的荒唐结果。
2. **必须在下单时算好、存下来。** 不能退款时再算。因为退款时，券的规则可能已经变了。

存储上，`PriceLine` 落一张 `order_price_line` 表，每行一条，`orderLineNo` 非空的表示已分摊。

### 4.4 怎么知道变好了

| 问题 | 之前 | 之后 |
|---|---|---|
| "为什么比标价便宜 23.7 元？" | 答不上来 | 客服后台展示 `PriceLine` 列表，一行一个来源 |
| 部分退款 | 手工估算，常出错 | 直接取 `payableOfLine` |
| "优惠券承担多少"的对账 | 无法查 | 按 `kind = COUPON` 汇总 |
| 新增一种优惠（如"新人立减"） | 改计算、改取消、改退款、改对账、改展示，约 5 处 | 新增一个 `PriceKind` 和对应的计算规则，约 3 处，且**都在订单模块里** |

### 4.5 代价

- **"分"和"元"并存。** 新旧模块的边界上，金额需要转换。**两套单位在同一个系统里共存，是这一章留下的债。**
- **明细会膨胀。** 一单十几个 `PriceLine`，数据量变大，报表查询变慢。他后来给报表单独建了一张汇总表。
- **分摊规则必须稳定。** 一旦上线，规则就不能随便改，否则历史订单退款会出现新旧不一致。

### 新问题

> 价格算清楚了。但**算清楚**之后，下单还要做另一件事：**把东西"占"住**。库存、券、积分，都要在用户付款前被锁定，否则别人可以抢走。

---

## 5｜2018 年：下单变成了协调

### 5.1 下单方法的体重

2018 年春天，`OrderService.place()` 的方法体，长到了 **217 行**。

老魏数了一下，它现在要做的事：

1. 校验用户、地址、商品状态
2. 取商品最新价格
3. 计算活动、优惠券、积分、运费
4. **扣库存**
5. **锁定优惠券**
6. **冻结积分**
7. 生成订单头、订单行、价格明细
8. 任何一步失败，**把前面做的全部撤销**

最后一步最麻烦。**第 6 步失败了，要把第 4、5 步撤回；第 5 步失败了，要把第 4 步撤回。** 每次改动，都要在这个方法里检查"撤销路径"有没有漏。

拾光这时候已经三十多个人。库存是仓储组维护的，优惠券是营销组维护的，积分是用户成长组维护的。**下单这件事，不再只涉及订单模块自己的数据，而是要协调三个别人的东西。**

此时发生了一件看起来不起眼、但很关键的事：团队开会时，没有人再说"下单"和"订单"，大家开始说：

> "**交易**要先问库存，再问营销，再问积分。"

"交易"这个词，开始用来指代**一次完整的买卖过程，包括它涉及的所有参与方**，而"订单"只是这个过程留下的核心记录。

### 5.2 重新划定"它是什么"

老魏借这个机会，把模块的结构彻底重排。这一次，改名字的同时，**改的是边界**：

```
shiguang-trade/                        ← 从 shiguang-order 改名而来
 ├─ trade.order      订单本身：订单头、订单行、状态流转（原来的全部内容）
 ├─ trade.pricing    报价：Quote、PriceLine、分摊
 ├─ trade.hold       占用：交易过程中，对库存、券、积分的"占住"与"释放"
 ├─ trade.payment    支付：支付单、支付通知、对账
 └─ trade.port       对外需求：交易需要别人提供什么
```

最后一个包最值得说。`trade.port` 里没有任何实现，只有接口：

```java
// shiguang-trade / com.shiguang.trade.port
public interface StockPort {
    /** 为这笔交易占住库存。同一个 (orderNo, skuId) 重复调用，结果相同。 */
    void acquire(String orderNo, List<StockDemand> demands);
    void confirm(String orderNo);      // 付款成功后：占用转为扣减
    void release(String orderNo);      // 交易关闭后：释放占用
}

public interface CouponPort { /* acquire / confirm / release */ }
public interface PointsPort { /* acquire / confirm / release */ }
```

**接口的名字、参数、语义，都是用"交易"的话说的：** 交易需要"占住""确认""释放"三个动作，**不关心**库存那边是怎么存的。库存模块、营销模块各自提供实现。

这一刻之后，一个重要的变化发生了：**依赖的方向反转了。** 过去是订单模块去调用库存的方法，库存怎么变、订单跟着改；现在是交易定义"我需要什么"，别人来适配。

入参也换了名字：

| 之前 | 现在 |
|---|---|
| `place(userId, skuId, quantity)` 三个参数 | `submit(Submission)`：一个对象，包含谁买、买什么（多行）、用哪张券、用多少积分、收货地址、来源渠道 |

"Submission"（提交）不是"请求"也不是"参数"，它代表**用户提交的一次购买意图**，在交易的话语里，它是有含义的。

### 5.3 选项：三个占用怎么保证"要么都占住，要么都没占"

最麻烦的问题还没解决：库存、券、积分**属于不同的团队，各自有自己的存储**，没有办法放进同一个数据库事务。

更糟的是，这一年拾光做了**分库分表**（《生长史》第 3 阶段）：订单按用户 ID 哈希分片，库存是全局的，两者连"同一个库"都不是了。

| 方案 | 做法 | 代价 |
|---|---|---|
| A. 分布式事务（两阶段提交） | 三方一起提交或一起回滚 | 各方都要支持；高峰期锁住资源，吞吐明显下降；当时的团队没人愿意背这个复杂度 |
| B. **先记账，再逐个占，失败就补偿** | 先在本地事务里写下"打算占什么"，再逐个去占；任何一个失败，就把已占的释放 | 会出现"中间状态"：占了一半；必须有兜底任务清理 |

老魏选了 B。核心是一张**占用账本**：

```sql
CREATE TABLE trade_hold (
  id           BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_no     VARCHAR(32) NOT NULL,
  hold_kind    VARCHAR(16) NOT NULL,     -- STOCK / COUPON / POINTS
  hold_status  VARCHAR(16) NOT NULL,     -- PLANNED / ACQUIRED / CONFIRMED / RELEASED
  expire_at    DATETIME NOT NULL,
  UNIQUE KEY uk_hold (order_no, hold_kind)
);
```

```java
// shiguang-trade / com.shiguang.trade.TradeService
public String submit(Submission submission) {
    Quote quote = pricing.quote(submission);                  // 只读：试算，不改任何东西
    Order order = orders.createUnpaid(submission, quote);     // 本地事务：订单 + 价格明细 + 占用账本(PLANNED)

    try {
        holds.acquireAll(order);                              // 按账本逐个去占，每个都幂等
    } catch (HoldFailedException e) {
        holds.releaseAcquired(order.getOrderNo());            // 补偿：把已占住的释放
        orders.moveTo(order.getOrderNo(), OrderStatus.UNPAID, OrderStatus.CLOSED);
        throw new SubmissionRefusedException(e.getKind());    // 告诉用户：哪一项没占住
    }
    return order.getOrderNo();
}
```

**为什么先记账？** 因为如果在"占到一半"时应用崩了，账本里留着 `PLANNED` 和 `ACQUIRED`，**兜底任务能看见，能清理**。如果不记账，崩溃之后，被占住的东西没有任何痕迹，永远解不开。

**库存的"占用"怎么落到数据库：**

```sql
-- 占用：可用转为已占，条件更新保证不超卖
UPDATE stock
   SET available = available - #{n}, held = held + #{n}
 WHERE sku_id = #{skuId} AND available >= #{n};

-- 确认（付款成功）：已占变成真实扣减
UPDATE stock SET held = held - #{n} WHERE sku_id = #{skuId} AND held >= #{n};

-- 释放（关单/失败）：已占退回可用
UPDATE stock
   SET available = available + #{n}, held = held - #{n}
 WHERE sku_id = #{skuId} AND held >= #{n};
```

这和 2016 年那条"最后一件"的 SQL 是同一个手法，只是**把"库存"一个数拆成了"可用 + 已占"两个数**。

### 5.4 订单号：一个被忽略的、关乎边界的设计

这一年，订单号也被重新设计了。

最早的订单号是"时间戳 + 随机数"，**重复的概率虽小但存在，而且别人能猜到下一个订单号**（也能估算你每天的订单量）。

分库分表之后，出现了一个新问题：**支付渠道通知里只有订单号，没有用户 ID。** 系统收到通知，要知道这笔订单在**哪个分片**，否则只能挨个分片去查。

解法是**把分片信息放进订单号**：

```
订单号 = 日期(6位) + 分片编号(2位) + 序列(8位) + 校验位(1位)
         180612       07            00042816      3
```

**拿到订单号，不查任何东西，就知道它在哪个分片。** 序列由独立的号段分配器生成，不同机器不会重复。

这是一个典型的**让标识自带路由信息**的设计，代价是：**订单号一旦发出，分片规则就不能随便改**，否则旧号无法定位。

### 5.5 名字的最后一次变化

到这一章结束，老魏把团队文档里的一段话改了：

> 原：订单模块负责订单的创建、支付状态流转和关闭。
> 改：**交易域**负责一次买卖从提交到结束的全过程；订单是它的核心记录，价格、占用、支付是它协调的几个部分。

改名后，**三件事变了**：

1. **负责什么**：不是"管订单"，而是"管一次买卖的全过程"
2. **依赖谁**：从"调用别人"变成"定义需要什么，别人来适配"
3. **别人怎么看它**：营销、库存、用户成长这三个团队，开始把"交易域"当作一个**需要对接的对象**，而不是"订单代码里的某个类"

### 5.6 怎么知道变好了

| 指标 | 之前（2018 春） | 之后（2018 秋） |
|---|---|---|
| `place/submit` 方法行数 | 217 | 约 30（协调逻辑），其余分布在 pricing / hold / order |
| 新增一种"占用"（如新增"运费险额度"） | 改下单、改取消、改过期、改失败补偿 4 处 | 新增一个 `HoldKind` 和一个 `Port`，约 2 处 |
| 占用到一半崩溃后的残留 | 无法发现，靠用户投诉 | 账本中的 `PLANNED/ACQUIRED` 被兜底任务扫出，每 5 分钟清理一次 |
| 营销组改券的存储结构 | 订单代码一起改 | 只改自己的适配实现，交易域不动 |

### 5.7 它的代价

- **中间状态出现了。** "订单已创建但库存还没占住""已占住但还没确认"，这些状态必须能被看见、被清理。系统从"要么成功要么失败"，变成了"可能卡在半路"。
- **补偿不是回滚。** 释放库存的那条 SQL，在被调用两次时必须安全；兜底任务和正常流程可能同时处理同一单，**依然要靠条件更新来裁决。**
- **边界一旦划错，付出的代价更大。** 为什么优惠券的"占用"归交易域协调，但"券本身"归营销？这个划分，在后来的拆服务过程中被反复拿出来争论。
- **本地事务装不下了。** 订单在一个分片，库存在另一个库，**"一个本地事务搞定一切"的时代结束了。** 这个事实，会在一年之后的双十一被流量放大成一场事故。

---

## 6｜回头看

### 6.1 三年，三个名字

| | 下单（2016 初） | 订单模块（2016 夏起） | 交易域（2018 起） |
|---|---|---|---|
| **它是什么** | 用户点一下，系统做完一件事 | 一个有生命的实体：有状态，会被多人同时改 | 一次买卖的完整过程，协调多个参与方 |
| **视野** | 一个动作 | 一个对象的生命周期 | 一个跨团队的流程 |
| **主要的名词** | `placeOrder`、`payNotify` | `Order`、`OrderLine`、`OrderStatus`、`PaymentNotice` | `Submission`、`Quote`、`PriceLine`、`Hold`、`Port` |
| **代码结构** | `shiguang-web` 里的 `shop` 包 | 独立模块 `shiguang-order` | `shiguang-trade`：`order / pricing / hold / payment / port` |
| **表** | `orders` | `orders`、`order_line`、`payment_notice`、`refund_todo` | `trade_order`、`trade_order_line`、`order_price_line`、`trade_hold`…… |
| **谁在乎它** | 老魏一个人 | 订单相关的开发和客服 | 仓储、营销、用户成长、财务 |
| **引出的麻烦** | 并发、重复、对不上 | 内容会变、价格说不清 | 本地事务装不下、边界要争论 |

**看这张表，有一个规律：名字不是随便起的。** 每一次改名，背后是"这件东西管的事变了"：

- **动词 → 名词**：系统开始关心"它是什么、现在怎样"，而不只是"做一件事"
- **名词 → 过程**：系统开始关心"它和谁有关、怎么协作"，而不只是自己的状态

如果一个名字和它实际管的事不匹配，代码就会开始别扭：比如 2018 年如果还叫 `OrderService.place()`，那 217 行里的库存、券、积分就没有名分，只能一直堆在"订单"里。

### 6.2 每一个机制，都来自一次具体的事故

| 机制 | 逼出它的事故 |
|---|---|
| 条件更新（`WHERE` 里判断） | 最后一件被卖了两次 |
| 状态机 + 流转表 | 重复通知、关单与付款赛跑、定时任务双跑 |
| 通知唯一键 + 同事务写入 | 渠道重发造成三条站内信 |
| 待退款清单 | 付了钱订单却被关闭 |
| 订单快照 | 商品降价，历史订单显示变了 |
| 价格明细 + 分摊 | 部分退款退多少说不清 |
| 占用账本 + 补偿 | 占库存、券、积分，只占到一半 |
| 订单号带分片 | 分库分表后，通知里只有订单号 |

**没有一个是"设计出来的"，都是"疼出来的"。** 这也是老魏开头那句话的意思。

### 6.3 三条我看到的东西

**第一，一个看似简单的 SQL 思路，能守住很多地方。**

"把判断放进 `WHERE`，由数据库在写的那一刻裁决"，这一个手法，先后守住了：库存超卖、状态流转赛跑、定时任务双跑、库存占用与释放。

它有一个明确的上限：**它依赖"这些数据在同一个数据库里"。** 当数据被分到不同的库、不同的服务，这个手法就管不到了。那是下一篇的开头。

**第二，每一次"让它更清晰"，都付出了一种新的复杂度。**

状态机让流转清晰，代价是"所有人都必须走同一个入口"；快照让订单自洽，代价是"字段选少了补不回来"；价格明细让钱说得清，代价是"分和元并存"；占用账本让失败可恢复，代价是"出现了中间状态"。

**第三，名字是视野的化石。**

回头翻一个系统的包名、类名、表名，你能读出它在哪一年管的事变了。**如果名字十年没变，而它管的事早已不同，那往往意味着有一块东西一直没有名分。**

---

## 7｜2025 年，周六晚上十一点

老魏讲完，值班室里安静了一会儿。

屏幕上，那张套餐订单还在"处理中"。沈砚看着那十一个状态，忽然问：

"所以最早那五个值——`UNPAID`、`PAID`、`SHIPPED`、`COMPLETED`、`CLOSED`——现在还在吗？"

"在。"老魏说，"主单上还在用，只是意思变了。`PAID` 在 2016 年的意思是'钱到了'，今天的意思是'钱到了，但接下来要看各个子单各自的命'。"

"那……后来呢？"

老魏看了一眼窗外。

"后来，2019 年的双十一，来了。"

他笑了一下：

"那天我才知道，我 2018 年画的那条边界，有一个地方画错了。"

---

*本文为技术叙事，人物与事件均为虚构复合。文中代码为示意，未经完整编译，仅用于表达设计意图；"日订单量""217 行""每周几十条"等数字均为虚构，用于说明问题的量级与变化。*

**参考与说明：**

- 订单状态"主状态 + 子状态、配显式流转表"的建议，参见有赞的科普文章 [《电商订单中台怎么搭？从0构建完整步骤》](https://www.youzan.com/cms/article/76709.html)。该文偏科普，本文只用它佐证"通用做法"，不引用其中数字。
- "拆单规则、优惠按商品比例分摊到 SKU"的做法，参见产品经理文章 [《电商后台产品经理——订单中心（一）》](http://www.woshipm.com/pd/2417871.html)（2019）。
- 订单快照、库存"占用—确认—释放"、订单号带分片信息等设计，均为本文的**设计思路**，不对应任何公司的具体实现，也未引用外部出处。
