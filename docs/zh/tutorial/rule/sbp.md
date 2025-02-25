# 超级节点

::: tip
本文主要描述有关超级节点相关规则，并非详细的技术文档，详细技术文档会在技术黄皮书里提及。

相关词汇解释：
* **SBP**： Snapshot Block Producer，指超级节点。
* **抵押**： 指账户中的一部分vite被冻结，无法交易，无法使用。
* **一个轮次**： 指每过一段时间会重新统计票数，一轮次时间为：75s，每个轮次理论上有75个块产生。
* **一个周期**： 指1152轮次，约一天。
* **SBP注册地址**：指注册超级节点时的交易发起方，也就是抵押发起方。
* **SBP运行地址**：指运行超级节点时，服务器上配置的地址。
:::

Vite 快照链采用和DPOS共识算法，在核心逻辑上和BTS的DPOS算法一致，Vite 在这之上做了些许改进。

**相关改动简要如下（详细规则请阅读下文）**：

* **SBP注册**：注册SBP需要抵押50万的vite
* **出块权**：每一轮：随机选出前25个节点中的23个节点出块，随机选择排名26-100名中的2个节点出块
* **出块奖励**：出块奖励的50%分配给出块者，50%分配给前100名超级节点（按投票数权重来计算）

## SBP注册

传统的DPOS算法中，注册委托代理节点需要消耗支付一定量的token，但是在vite里，我们提倡通过**抵押**的方式来获取资源和资格。
在vite里可以通过抵押获取**配额**（交易配额），也可以通过抵押获得SBP的运行资格。

### 抵押规则

注册SBP需要抵押vite，可以通过取消SBP资格来取回抵押的vite，也就是说，如果想一直保持SBP资格，则必须一直抵押vite。

**抵押数量：**

*50万 vite*

**抵押时长：**

注册SBP抵押的vite不能立即取回，需要在**7776000**块（约3个月）之后通过发起一个**取消SBP资格**的交易来取回抵押的vite。

这里有设置一个*三个月*的冻结时间，是为了防止有人频繁的抵押、撤回，导致整个网络SBP变动非常剧烈。

### 注册逻辑

传统DPOS算法里，注册超级节点的地址默认就是出块地址和出块奖励获得地址，而且注册成为超级节点之后就永久成为超级节点。

在vite中，注册超级节点的地址、出块地址、出块奖励获得地址可以是三个不同的地址。注册成为超级节点之后，如果**取消质押**就默认失去超级节点资格。

注册时，由抵押地址发起一个调用**SBP注册**交易（实际上是调用内置合约），当该交易被内置合约接收，接收交易被快照链打包确认之后正式生效。

#### 参数

* **超级节点地址**：填写超级节点运行时用于出块的地址，也可以填写为发起**SBP注册**的地址。
推荐在服务器上生成一个新地址，然后将这个新地址作为超级节点地址，这样即使服务器被盗，不会影响超级节点抵押地址的安全。
超级节点未取消资格时，可以通过发起一个**修改注册信息**的交易来修改超级节点地址。

* **超级节点名称**：1-40个字符，包含大小写英文、数字、下划线、英文句号和空格。超级节点名称不能重复。用户投票时使用超级节点名称进行投票。

## 出块权

### 出块时间

Vite 快照链出块速度为1s，75s为一个轮次（每75s会重新统计票数，然后按票数排名选择出块节点），每轮到一次连续出块3个。

### 出块节点

Vite 每轮会选举出25个具有出块权的节点，25个节点选择规则：

* 从前25名节点（按投票比例排序）里随机选出23个（顺序是随机的），每轮出块概率为：23/25
* 从26名到100名节点里随机选出2个，每轮出块概率为：2/75

## 出块奖励

Vite 主网上线后，每年会增发不超过3%的vite用于超级节点奖励。目前在预主网上的奖励固定为： `0.951293759512937595 vite`

### 奖励分配

每个块的奖励会划分为两个部分：

#### 奖励出块者

每个块奖励的50%会给予该块的出块者作为奖励，我们称为：**按块奖励**。

#### 奖励前100名SBP

用于奖励当天最后一轮（75s为一轮）排名前100的超级节点，我们称为：**按票奖励**。

奖励规则如下：

* 按照当天最后一轮该超级节点获得的投票数作为权重来瓜分这一轮的奖励
* 奖励领取只能领取*48*轮次以前（大约一小时以前）的奖励，领取奖励按*1152*为一个周期（约一天）来计算
* 在一个周期（约一天）内，可以计算出每个节点在这一周期内的在线率： `实际出块数/应该出块数`，在线率越高，奖励越多

### 奖励计算公式

**一个周期**：从快照链的创世块开始，每1152轮为一个周期，约一天。

* `l`: 节点在一个周期内实际出块数目
* `m`: 节点在一个周期内应该出块数目
* `X`: 所有节点在一个周期内实际出块数目
* `W`: 一个周期最后一轮前100名节点获得的总投票数，和前100名的抵押金额之和
* `V`: 该节点在一个周期的最后一轮获得的投票数目和抵押金额之和
* `R`: 每个块的奖励，预主网固定为：`0.951293759512937595 vite`
* `Q`: 每个节点一个周期内的奖励

$$Q = \frac{l}{m}*\frac{V}{W}*X*R*0.5 + l*R*0.5$$

注意：
* 根据上面的公式，如果一个节点在一个周期的最后一轮投票结果中排名在100名以后（不包括第100名），那么这个节点在这个周期的奖励为0；
* 一个节点刚注册后的第一个周期没有奖励；
* 一个节点取消注册前的最后一个周期没有奖励；
* 如果一轮（75s为一轮）中所有的节点都没有出块，那么这一轮不影响这些节点的出块率，即这一轮的块不计入这些节点在这个周期内的应该出块数目。

#### 奖励提取

Vite 网络中的出块奖励不是立即发放到出块者地址，需要SBP的注册账户手动发起**奖励提取**交易，才能收到奖励。

**奖励提取规则：**

* 只能由注册SBP的账户发起，也就是抵押账户的地址。
* 每次提取只能提取*48*轮（约一小时）以前的出块奖励。
* 提取时不能指定时间范围，会一次性提取出所有可提取的出块奖励。可提取的出块奖励是指，从上次提取的时间到当前时间减一小时之间，所有完整的周期。
* 提取奖励时需要填写奖励提取的地址，每次提取时填写的地址可以不一样。
* 提取奖励需要消耗`68200`配额


## FAQ

* 一个账户上的vite不够抵押vite数量，那是否支持多地址联合抵押?
  
  目前不支持`多地址联合抵押`，SBP注册的逻辑是由**内置智能合约**实现，可以通过**合约调用合约**的方式来实现`多地址联合抵押`。

* 如果一个节点一个周期内（约一天）的在线率是0，那这个节点**按票奖励**为0，那本该属于他的**按票奖励**会平分给其他节点吗？

  节点的奖励的vite是在SBP提取时增发，如果一个节点在线率为0，按票奖励为0，这个节点将无法提取这段奖励，所以这段奖励就不会增发，故而这份奖励也不会平分给其他节点。
  
* 如果注册成为超级节点之后，却不跑节点，会收到奖励么？

  不会。如果没有运行超级节点，该节点**在线率**为0，奖励为0。
  
* 我已经运行了一个超级节点，但是在线率是否也可能为0？

  有可能会。如果节点处于一个分叉链，或者网络延迟比较高，都有可能会导致在线率为0。如果这个节点在一个周期内的应该出块数为0，在线率也为0。
  
* 我出了一个快照块，提取奖励时能同时拿到这个块的按块奖励和按票奖励吗？
  
  不一定。刚注册后的第一个周期和取消注册前的最后一个周期没有奖励；如果在这个周期的最后一轮时，这个节点的排名超过了前100名，那么也拿不到这个周期的奖励。
  
  但是，只要节点参与了完整的周期，并且在这个周期内的最后一轮时排名在前100名，就可以同时拿到这个块的按块奖励和投票奖励。
  
* 注册SBP时抵押的vite，在抵押之后是否还可以用于**抵押获取配额**？

  在目前的设计里（第一版测试网络），SBP注册抵押的vite在抵押期间无法使用，当然也不能**抵押获取配额**，后续考虑改进，已经在计划之中。
  
* 一个地址是否可以注册多个超级节点？

  支持。因为SBP注册地址和SBP运行地址可以为两个不同的地址。SBP注册地址（也就是抵押地址）每次抵押时都可以指定另一个地址为SBP运行地址。
  
* 如果一个节点在前半个周期内出块率为100%，后半个周期所有节点都没有出块，那么这个节点在这个周期的出块率是多少？

  这个周期的出块率为100%，因为后半个周期所有节点都没有出块，因此后半个周期不算应出块数，不会导致这个节点的出块率降低。
