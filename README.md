# 千军网安RankSystem设计文档 v1.0.2

```bash
#    _____             _     _____           _                 
#   |  __ \           | |   / ____|         | |                
#   | |__) |__ _ _ __ | | _| (___  _   _ ___| |_ ___ _ __ ___  
#   |  _  // _` | '_ \| |/ /\___ \| | | / __| __/ _ \ '_ ` _ \ 
#   | | \ \ (_| | | | |   < ____) | |_| \__ \ ||  __/ | | | | |
#   |_|  \_\__,_|_| |_|_|\_\_____/ \__, |___/\__\___|_| |_| |_|
#                                   __/ |                       
#                                  |___/                      
#     Date: 2020.03.02
#      Ver: 1.0.2
#   Author: @翔翔 @剁刀师甲
```

## 0x01 总体目标

- 建立货币及奖励系统RankSystem，发行可自由流通货币RankCoin（简称Rank）。

- 在Rank框架之下，建立一套基础知识问答子系统FlyWind（福利问答），增进群成员技术讨论及交流，营造良好的群生态空间。

- 基于强伪随机数生成器，建立安全公平的Gaming系统

- 系统基于Python + Golang + Redis + MySQL实现



## 0x02 关于RankCoin

- Rank是一个通货紧缩系统，RankCoin总量恒定=210,000 枚，起始的21w枚均属于系统。 每天发放数量18枚，一年发放6570枚，从系统中扣除，转账到个人账户。
- Rank支持群成员之间相互转账，但已完成交易均**不可回滚**。
- Rank可通过答题来获得，或者帮助群成员解答问题，成员可以支付rank表示感谢。Rank的可通过购买服务支出，比如支付看福利pp，发视频，免踢群等。
- 个人支付购买某项服务的币（比如看福利pp等）应**流入@bot账户**，不能凭空消失，也不能流入系统账户（**因为如果由系统发放的币再流回系统账户，相当于无限增发**）
- 用户可获得RankCoin的场景
  - 答对猜词题 (+8 Rank)
  - 答对计算题 (+10 Rank)
  - 提出了有价值的建议或意见 (+1 ~ 10 Rank)
  - 发现了有价值漏洞 (+5 ~ 20 Rank)
  - 提交了原创工具、文章 (+5 ~ 20 Rank)
  - 在群中积极发言，可随机(+1 ~ 2 Rank)
  - 得到其他成员的转账
  - Gaming
- 用户可支出RankCoin的场景
  - 支付Rank币看福利图片 (-2 Rank/张)
  - 转账给其他成员
  - 购买群提供的服务
  - 兑换奖品
  - Gaming
  - 其他场景



## 0x03 FlyWind子系统

Flywind（福利问答）子系统是一套促进群交流的基础知识问答系统。每天上午，flywind会选择一个随机时间段发布一道猜词题，群成员采取抢答方式答题，首名答对成员可以获得Rank奖励。每天下午，flywind会选择一个随机时间段发布一道计算题，群成员采取抢答方式答题，首名答对成员可以获得Rank奖励。



## 0x04 支持命令列表

**私聊时支持的命令：**

- [x] /rank - 获取本人rank值
- [x] /profile - 获取本人的个人资料（备用）
- [x] /history 10- 获取本人最近10条rank收入、支出记录
- [x] /history 20- 获取本人最近20条rank收入、支出记录
- [x] /top 10 - 获取rank值最高的10人
- [x] /top 20 - 获取rank值最高的20人
- [x] /follow - 订阅bot开关，出题时进行消息通知
- [x] /unfollow - 取消订阅bot开关

**群聊时支持的命令**：

- [x] @某人 send:1 - 向某人转账1 rank
- [x] @某人 bonus:1 - 向某人奖励1 rank（从系统账户中转出，**只能由管理员操作**）。此功能除了奖励外也可用来在系统无法发出奖励的时候，手动发币给用户。
- [x] /fly 4 - 请求bot发出4张福利图，扣除8 rank值
- [x] /donate 4:2 - 随机选取4名群友捐赠2 rank（撒币总值不得超过发送者的rank上限）
- [ ] ~~/video - 请求发视频，成功后将扣除-10 rank并标记该用户可发视频1次（若不申请便发视频，立即被踢出）~~ 



## 0x05 聊天奖励系统

- 系统可以按概率随机一个发言的群友bonus  +1~2 Rank值（采用sha256+随机字串的方式，其中奖励1 Rank的概率为4 / 256 = 1.6%，奖励2 Rank的概率为1 / 256 = 0.4%）
- 记录群中最后一次的发言时间T，某人的发言时间为Ti，若(Ti - T) >= 8并且此人的发言内容超过10个字符，则获得【打破沉默】奖励 +5 Rank



## 0x06 Gaming系统

在Gaming系统中，设置一个猜硬币游戏，如果猜中，则奖励用户下注金额的200%，否则将损失掉下注金额。

**系统设计原则：**

- 需要在内部建立一个bet账户，提前向其中划拨1000Rank。所有转账行为均发生在用户和bet账户之间。

- 尽可能使用强伪随机数生成器，确保硬币生成正反两面的概率无限接近50%。（python函数见下图）

  ![](C:\Users\azy\Desktop\微信图片编辑_20200229064534.jpg)

- 必须首先生成游戏局面，将局面结果的md5值发给用户，然后用户再下注（确保公平，没有猫腻）

**支持的命令：**

- /coin - 进入猜硬币游戏
  - /bet 1:3 - 下注3 Rank猜正面，猜中将获得（200% * 3 = 6 ）Rank，亦即净增加3Rank
  - /bet 0:10 - 下注10 Rank猜反面，猜错将损失10 Rank

**游戏流程及时序：**

1. 接收用户输入的/coin指令

2. 在系统内随机生成以下数据（拿json举例）：

   {result:"1", token:"Aasge2HFAHw235678",md5:"c8d46d341bea4fd5bff866a65ff8aea9"}

   其中的md5值=md5(result + ":" + token)，result和token均随机生成（为了确保安全，尽可能地使用强伪随机数生成，token的长度为16字节）。result代表硬币的正1或者反0

3. 将用户wxid、token和md5的值，以key-value形式存入redis备用

4. 客户端向用户返回md5值等信息：

   > ~欢迎来到猜硬币游戏~
   >
   > 猜硬币正反，猜对的话有200%的奖励哦~
   >
   > 为了确保游戏公平，我们会事先生成结果。
   >
   > 游戏结果已生成（c8d46d341bea4fd5bff866a65ff8aea9）
   >
   > 请下注。
   >
   > 【指令提示】
   >
   > /bet 1:2 下注2Rank猜正面
   >
   > /bet 0:2 下注2Rank猜反面

5. **第一种情况**：猜对了。接收到用户的输入"/bet 1:3"

   - 通过用户id找到关联的token值，再将用户猜测的结果和token进行md5(result + ":" + token)运算，并与redis中的md5比较，如果相同，那么说明用户猜对了。

   - 奖励用户+3 Rank

   - 重新生成新的随机数据（包括新的result和新的token），然后重新跟该用户绑定存入redis，并返回如下信息：

     > 恭喜你，猜对了！得到2倍奖励！
     >
     > 新一轮游戏结果已生成（c8d46d341bea4fd5bff866a65ff8aea9）
     >
     > 请下注。

5. **第二种情况**：猜错了，接收到用户的输入"/bet 0:3"

   - 经过计算，发现md5值不符

   - 扣除用户Rank

   - 向用户**回显出随机字串token**（便于用户自己验证结果），**证明**我们是在用户下注之前就生成结果了，而不是在下注之后做手脚

   - 重新生成新的随机数据，然后重新跟该用户绑定并存入redis，并返回如下信息：

     > 好可惜，猜错了！
     >
     > 上一轮的随机字符串为：Aasge2HFAHw235678
     >
     > 新一轮游戏结果已生成（a104b0b532da7c6cb14f7992cf463aeb）
     >
     > 请下注。

6. 若用户账户或@bet账户余额不足，则提示当前余额已不足，无法继续游戏。



## 0x07 Todo List

- [ ] 生产库与开发（测试）库的分离，代码复用与逻辑解耦。
- [ ] 进一步优化猜词题目的出题算法，尝试引入NLP和机器学习算法对抓取到的信息进行合理提取加工。最终的理想效果是归纳概括出名词的范围、内涵、外延等特性，通过逐步递进给出提示语句，缩小猜词范围，最终猜出名词。实例：
  1. 它是一个密码学算法。（通过此条猜出+10Rank）
  2. 它由X发明。（通过此条猜出+9Rank）
  3. 它主要用于XX领域。（通过此条猜出+8Rank）
- [ ] 加速扩展计算题的题库分类，扩充至30个分类是短期内的紧要目标（目前仅有6个）。
