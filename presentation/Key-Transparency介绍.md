## 先说说PKI

PKI全称公钥基础设施（Public Key Infrastructure），是一个管理数字证书的中央目录服务。

PKI包括以下几种角色：

* CA：可信权威机构，主要任务是签发及存储X.509数字证书
* RA：注册机构，主要任务是鉴别用户身份并通过CA分发数字证书
* VA：验证机构，主要任务是校验用户domain及数字证书的合法性

PKI一般有以下几种实现方式：

* 单一认证机构：只存在单一CA被每个人完全信任，任何人都必须得到一个合法的CA公钥PK<sub>CA</sub>，实践中PK<sub>CA</sub>是与其他软件捆绑在一起发行的，比如Web浏览器。
* 多个认证机构：单一CA存在单点故障问题，而且并不是所有人都信任单一CA，因此多个CA可以让用户选择自己信任的CA来签发数字证书，比如Web浏览器允许用户手工设置自己信任的CA证书。
* 证书链：假设Charlie充当一个CA，并签发了一个Bob的数字证书，证书链允许Bob签发一个Alice的数字证书，使得当Dave验证Alice的数字证书时，如果Dave完全信任Charlie，则Dave也能够信任Bob签发的Alice数字证书是合法的。
* 信任Web模型：任何人都可以发布证书给其他任何人，并且每个用户都必须自己决定其他用户发布的证书是否可信。假设用户Alice已经拥有某些用户A, B, C的公钥PK<sub>A</sub>, PK<sub>B</sub>, PK<sub>C</sub>，而用户Bob分别拥有用户A, C和D签发的证书Cert<sub>A</sub>, Cert<sub>C</sub>和Cert<sub>D</sub>，当Bob将这些证书发送给Alice时，Alice无法验证Cert<sub>D</sub>（因为她没有D的公钥），但是她可以验证其它的两个证书，现在Alice必须决定对A和C的信任程度（例如，Alice可能想到A或者C被敌手攻破，但她考虑不太可能两者都被攻破）。
* 区块链模型：随着区块链及P2P技术的发展，基于区块链的分布式PKI也逐渐流行起来。该模型通过区块链技术解决传统CA的信任问题，并通过分布式账本记录PKI的所有事件，达到监控和审计的目的。

## 再谈谈Certificate Transparency

Certificate Transparency (CT) 是对PKI的扩展，目的是对TLS证书的实时审计和监控，以防止CA的欺诈行为。CT主要提供以下功能：

* 未获得domain所有者的允许，CA无法签发该domain的TLS证书
* 提供开放的审计和监控系统，使得domain所有者或CA能够验证TLS证书的合法性
* 尽可能的保护用户不被恶意的TLS证书欺骗

CT通过部署独立于CA的日志服务器、监控服务器和审计服务器，实时发现恶意证书并避免恶意行为。

* 日志服务器：记录所有TLS证书的操作记录，包括签发、更新、撤销等
* 监控服务器：监控TLS证书的操作记录，实时发现恶意证书并告警
* 审计服务器：审计日志记录的正确性和一致性，保证每个TLS证书的操作记录都是完整的

## 最后看看Key Transparency

Key Transparency (KT) 的提出主要是受到CT和CONIKS的启发，是一个通用透明的目录服务，提供独立的数据防篡改审计功能，能够应用在各种需要对数据进行加密或认证的场景，并满足各种安全特征。

KT同时可以作为一个分布式PKI系统，提供对用户公钥的检索、验证、跟踪、审计等功能。

### 安全特性及设计

#### 一致性

在同一时刻，任何用户从同一发送方获取公钥，得到的结果都是相同的。即接收方可以确信，发送方发送的都是同一份数据，任何中间人都无法篡改或伪造数据。

为了实现这一特性，KT计算所有公钥的哈希值，对其进行签名得到Signed Map Head (SMH)，并通过Gossip协议在所有用户之间共享SMH。如果不同用户收到了不同的数据，则这些用户之间通过对比SMH就会立刻发现不一致。

#### 存在性

任何用户都可以直接判断KT服务器是否拥有某个用户的公钥。KT通过Merkel Tree实现这一特性，如下所示，所有公钥都作为Merkel Tree的叶子节点，并一层一层计算哈希值，最终得到Merkel Tree Root，对其进行签名可以得到SMH。

![](https://github.com/google/keytransparency/raw/master/docs/images/Sd4HCvEtUSL.png)

为了证明Merkel Tree中存在某个公钥，验证方得到公钥k及其Merkel Tree中的相应相邻节点n，计算从k到Merkel Tree Root路径上的所有哈希值x，如果x与Gossip协议得到的Merkel Tree相同，则证明k一定存在于KT服务器中。

```
           x
      /         \
     n           x
   /   \       /   \
  o     o     x     n
 / \   / \   / \   / \
o   o o   o n   k o   o
```

#### 唯一性

对于同一个用户，所有KT服务器只存在唯一的同一份公钥。否则，恶意的KT服务器可以构造不同的公钥，并且针对不同用户返回不同的公钥。

为了证明用户Alice只存在一份公钥，KT服务器通过VRF函数计算出Alice的邮箱地址和appId的哈希值比特字符串，比如```H(alice@gmail.com, pgp) = 010```。假如我们把Merkel Tree当作一棵二叉搜索树，则计算得到的比特字符串可以唯一定位到Merkel Tree的某个叶子节点（从树的根节点开始，每个比特代表该节点的左子树或右子树，且比特字符串的长度等于树的深度）。因此，每个用户只能唯一找到同一个公钥。

#### 隐私性

传统的PKI允许任何人得到所有用户的一些个人信息，比如邮箱地址。KT只允许通过邮箱地址（或其他个人唯一标识符）找到对应用户的公钥，这通过一个VRF函数计算得到该用户公钥在Merkel Tree中的位置```H(user_id+app_id)```，而不公开用户的任何个人信息。因此，攻击者只能通过在线穷举的方式才能得到KT服务器的用户个人信息。

#### 可审计性

KT以日志追加的方式保存了所有的操作历史，任何的密钥更新、密钥检索、账户重置等事件都会被记录下来。一旦某个用户绑定了新的公钥，所有与之通信的用户都会得到通知，并且决定是否信任该公钥。

另外，当有用户的数据更新时，KT会定时对整个数据库生成快照和Merkel Tree Root，任何用户都可以随时检查所有的快照和Root。为了保证快照的真实性，每个快照的Merkel Tree Root都会存储在另外的Merkel Tree，并通过Gossip协议进行共享。

![](https://github.com/google/keytransparency/raw/master/docs/images/SL84rktNJb4.png)

#### 可监控性

任何用户在任何时候都可以向KT服务器请求得到自己的公钥，如果用户发现KT服务器返回的公钥已经被修改了（与自己本地存储的不一致或者不是在自己的授权下修改的），可以向其他用户或KT服务器广播这次恶意行为。

另外，任何用户都可以分别从不同的KT服务器得到公钥和SMH，并校验返回的结果是否一致，若不一致则可以通报恶意的KT服务器。

### 参考资料

* [Key Transparency Design](https://github.com/google/keytransparency/blob/master/docs/design.md)
* [CONIKS](https://eprint.iacr.org/2014/1004.pdf)