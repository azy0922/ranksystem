# 千军RankSystem设计文档 v1.1.0

```bash
#    _____             _     _____           _                 
#   |  __ \           | |   / ____|         | |                
#   | |__) |__ _ _ __ | | _| (___  _   _ ___| |_ ___ _ __ ___  
#   |  _  // _` | '_ \| |/ /\___ \| | | / __| __/ _ \ '_ ` _ \ 
#   | | \ \ (_| | | | |   < ____) | |_| \__ \ ||  __/ | | | | |
#   |_|  \_\__,_|_| |_|_|\_\_____/ \__, |___/\__\___|_| |_| |_|
#                                   __/ |                       
#                                  |___/                      
#     Date: 2020.03.14
#      Ver: 1.1.0
#   Author: @翔翔 @剁刀师甲
```

## 0x01 总体目标

- 建立货币及奖励系统RankSystem，发放**可自由流通积分Rank**~~RankCoin（简称Rank）~~
- 基于PoW共识算法，构建独有的区块链聊天即挖矿系统，发行**数字货币RankCoin**
- 在Rank框架之下，建立一套基础知识问答子系统FlyWind（福利问答），增进群成员技术讨论及交流，营造良好的群生态空间。
- 基于强伪随机数生成器，建立安全公平的Gaming系统
- 系统基于Python + Golang + Redis + MySQL实现



## 0x02 关于Rank & RankCoin

- Rank仅是一套可自由交换和流通的积分体系，**不设置上限**。
- RankCoin是一个通货紧缩的数字货币体系，**仅能通过挖矿获得**。
- Rank积分可通过答题来获得，或者帮助群成员解答问题，成员可以支付rank表示感谢。Rank的可通过购买服务支出，比如支付看福利pp，发视频，免踢群等。
- 个人支付购买某项服务的积分（比如看福利pp等）应**流入@bot账户**，不能凭空消失，也不能流入系统账户。
- 用户可获得Rank积分的场景
  - 答对猜词题 (+6 Rank)
  - 答对计算题 (+8 Rank)
  - 提出了有价值的建议或意见 (+1 ~ 10 Rank)
  - 发现了有价值漏洞 (+5 ~ 20 Rank)
  - 提交了原创工具、文章 (+5 ~ 20 Rank)
  - 在群中积极发言，可随机(+1 ~ 2 Rank)
  - 得到其他成员的转账
  - Gaming  
- 用户可支出Rank积分的场景
  - 支付Rank看福利图片 (-2 Rank/张)
  - 转账给其他成员
  - 购买群提供的服务
  - 兑换奖品
  - Gaming
  - 其他场景
 
## 0x03 FlyWind子系统

Flywind（福利问答）子系统是一套促进群交流的基础知识问答系统。每天上午，flywind会选择一个随机时间段发布一道猜词题，群成员采取抢答方式答题，首名答对成员可以获得Rank奖励。每天下午，flywind会选择一个随机时间段发布一道计算题，群成员采取抢答方式答题，首名答对成员可以获得Rank奖励。

**[注]：**正则类题目需要Python端配合完成，为防止其他人模仿优先提交者的答案，正则类题目由@bot记录回答正确的前5个答案（一个记2 Rank），当5个答案计满时一次性给出统计结果。



## 0x04 挖矿系统

**1.关于RankCoin**

- 共识算法为PoW（工作量证明），总量设定为840,000 枚。

- 挖出的块称为区块，区块的高度记为Height，每挖出一个区块Height+1，创世区块高度为0。每一个区块都应该打包这个区块内的所有消息（*这样的话以后可以增加一个命令/block 2，查看高度为2的区块的信息，将dump出这个区块内的所有发言*）。区块的结构应该如下所示：

  ```c++
  type Block struct {
  	Timestamp     int64
  	PrevBlockHash byte[]	//前一个块的hash 
  	Hash          byte[]	//本区块的hash
  	Height        int		//区块高度
      Message       Message*[]//消息数组，包含本区块内的所有消息
  }
  ```

- 起始区块奖励为1 RankCoin，每挖完2100块奖励增加一倍（与比特币相反，为了刺激挖矿），按照**1-2-3-6-13-25-50-100-200**的奖励数量递增，最高奖励是200 RankCoin。
- 若假定平均60分钟挖出一块，则一天大约挖出12块（按群活跃12小时计算），2100块需要挖175天 ≈ 5.8个月，**大约4.35年**可以挖完840,000个RankCoin，区块总数 = 2100 * 9 = 18900块，区块高度到18899截止。

**2.关于挖矿算法**

- 挖矿应该取所有字节数参与计算，不再限制10个字节。
- 为了达到平均60分钟出一块，假设按照一天400句发言数量计算，则sha256的触发概率应设置为 = 12 / 400 = 3%（取"08"开头及以下）；假设按照一天200句发言数量计算，则概率应设置为 = 12 / 200 = 6%（取"0F"开头及以下），触发概率应该根据当天出块情况在3%至6%之间动态调整。

- 挖矿算法是计算：Block.Hash是否在指定的区间内。**Block.Hash = sha256(Timestamp+PrevBlockHash+Height+Message)**。创世区块由于没有内容，故创世区块的Message="28/Feb/2020 Qianjun rank system will survive."

- 挖矿奖励回显消息应区别于其他消息，回显以下消息外加一张福利pp（pp的右上角加"矿工专属"水印）：

  > 矿工@xxxx挖出区块
  >
  > ===========
  >
  > 时间戳: XXX
  >
  > 前一个块哈希: 0xxxxxxx
  >
  > 块哈希: 0xxxxxxxxxxx
  >
  > 区块高度: 1
  >
  > 区块奖励: +X Rank
  >
  > 打包消息数：XX
  >
  
  

## 0x05 奖励系统

**1. 随机奖励**

- 系统按概率随机对发言者**【奖励】** 1 Rank值（采用SHA256(随机字串K+message)，其中奖励**【概率】**为5 / 256 = 1.95%，头部匹配"01"/"02"/"03"/"04"/"05"），假设一年的发言数量按365*200=73,000计算，则发放的**【全年随机奖励】**≈1426枚Rank，**【平均每天奖励】**3.9Rank

**2. 彩蛋奖励**

- 记录群中最后一次的发言时间T，某人的发言时间为Ti，若(Ti - T) >= 8且发言内容超过10个汉字，则获得【打破沉默】奖励 +5 Rank

**3. 挖积分大赛**

- 每个月的28日（千军Rank系统上线日），举办挖分大赛，公开一个明文随机字符串K，供参与者挖分。

- 挖分规则：矿工计算的SHA256值需至少以5位0开头即"00000"，且message必须为有意义的中文句子（大于等于5个汉字，难度约为1,048,576句，即平均每搜索5,242,880个汉字才能挖到一个合适的哈希），每挖出一个5位0开头的块奖励+50 Rank。发放的**【全年大赛奖励】**=600 Rank

  

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

5. **第二种情况：**猜错了，接收到用户的输入"/bet 0:3"

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

7. 若用户账户或@bet账户余额不足，则提示当前余额已不足，无法继续游戏。

   

## 0x07 支持命令列表

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
