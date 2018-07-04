# jLRD
Postorder Traversal (LRD) Jumped (后跟跳跃遍历)

### 1、结构定义

> *后跟跳跃遍历算法，Postorder Traversal (LRD)Jumped，是学霸君APP中启动图片、运营广告位、推荐应用等功能模块所使用的一种高性能个性化展示算法，能实现不限规则的任意可配。*

各种规则、操作组合最大支持的不同配置数达：
![$$\sum_{i=1}^m C_m^i10^i$$](https://cxq9393.oss-cn-shanghai.aliyuncs.com/math.png)

其中，m为规则类型数，已知确定的规则有:uid、city、version、platform、channel、time、date，共7种(规则名不受限制，存什么规则名就是是什么规则)，10是规则支持的运算类型数
<table style="width:200px;">
    <thead >
        <tr>
            <th>运算</th>
            <th>备注</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>eq</td>
            <td>等于</td>
        </tr>
        <tr>
            <td>neq</td>
            <td>不等于</td>
        </tr>
        <tr>
            <td>gt</td>
            <td>大于</td>
        </tr>
        <tr>
            <td>egt</td>
            <td>大于等于</td>
        </tr>
        <tr>
            <td>lt</td>
            <td>小于</td>
        </tr>
        <tr>
            <td>elt</td>
            <td>小于等于</td>
        </tr>
        <tr>
            <td>btw</td>
            <td>区间内</td>
        </tr>
        <tr>
            <td>nbtw</td>
            <td>区间外</td>
        </tr>
        <tr>
            <td>lk</td>
            <td>包含</td>
        </tr>
        <tr>
            <td>nlk</td>
            <td>不包含</td>
        </tr>
    </tbody>
</table>

> 不包含nlk(lk量超过总量一半，推荐使用nlk)

规则的作用对象(以下简称对象)和它的规则按照树结构存储于同一张表中，建议表结构设计如下:
```sql
CREATE TABLE `demo` (
`id` int(11) unsigned NOT NULL AUTO_INCREMENT,
`gid` int(10) unsigned NOT NULL,
`pid` int(10) unsigned NOT NULL,
`name` varchar(255) NOT NULL DEFAULT '',//规则名OR对象名 `value` blob NOT NULL,//规则的值OR对象的值
`op` varchar(255) NOT NULL DEFAULT '',//规则运算符
`lft` int(10) unsigned NOT NULL,
`rgt` int(10) unsigned NOT NULL,
`lvl` int(10) unsigned NOT NULL,
`add_time` int(10) unsigned NOT NULL,
PRIMARY KEY (`id`)
);
```
### 2、匹配过程

以广告位为例，将树结构推倒拍扁，一次性从表中拉取，拉取结果如下:
![图片标题](https://cxq9393.oss-cn-shanghai.aliyuncs.com/WX20180704-104734%402x.png)

> *注意当op为lk时，value存储的只是redis指针，并非规则的真实值。这里也可以用mysql存储指针指向的真实值，选择redis主要是使用了redis可以设置过期时间跟活动截止时间一致，以达到过期数据的自动清理。*

拉到列表之后，最多只需遍历一次就可以算出满足规则的所有对象。遍历过程中如遇到规则不匹配，会产生跳跃，即直接忽略该对象的其他规则的匹配过程，所以速度非常快。

相同的规则可以添加多条，他们之间是或的关系，不同规则之间是与的关系。匹配时，相同规则的多条规则只要匹配到一条就会跳过同名的其他规则继续去匹配，直到所有规则全部配成功，对象有效，否则，无效。

 由于同一个广告位只能显示一个对象，在遍历匹配的过程中如果同一个广告位匹配到多个对象，后匹配到的会覆盖之前的(列表按照加入的时间升 序排列),因此，最终只有一个对象生效。

![图片标题](https://cxq9393.oss-cn-shanghai.aliyuncs.com/WechatIMG141.png)

### 3、冲突解决机制

下图A表示能看到广告A的用户集合，B表示能看到广告B的用户集合
![图片标题](https://cxq9393.oss-cn-shanghai.aliyuncs.com/1.png)
集合A包含于集合B时，相同时间段内，如果仍希望广告A和广告B都能被用户看到，这是就需要解决冲突。
![图片标题](https://cxq9393.oss-cn-shanghai.aliyuncs.com/2.png)
如上，左图中，集合B完全覆盖了集合A，导致集合A的用户看不到广告A而看到广告B，这时应将B广告先于A广告配置，这样集合A的用户能正常看到广告A，集合B中除去集合A以外的用户能看到B广告，冲突就解决了。

当A、B不是包含于的关系，而只是存在交集，配置的先后对结果是有一定影响，但不存在冲突，各发布方沟通协调决定谁先谁后。

超过两个广告的冲突解决依此类推。

充分发挥你的想象力，没有配不了的，只有你没想到。
