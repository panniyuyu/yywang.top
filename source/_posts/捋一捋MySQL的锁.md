title: 捋一捋MySQL的锁
author: YyWang
tags: MySQL
categories: MySQL
date: 2020-11-16 15:04:15
---

## 引言

本文主要梳理MySQL的锁机制，主要是针对于Innodb引擎，目前网络上查的文章基本上都差不多，实际上是忽略了一些细节的，这些细节可能会成为今后搬砖过程中的恶魔，比如说插入意向锁，行锁之间的兼容关系这些，本文通过查阅资料加锁MySQL官网的说明再结合自己的理解梳理了一下MySQL的锁机制~

## 锁的划分
这里要有一个前提，就是MySQL对锁的划分是两种不同的维度，按照加锁的粒度和锁的类型，并不是固定的什么锁什么锁，就比如表锁和行锁里都有共享锁或者排他锁，同样，共享锁或排他锁中也都有行锁和表锁，是你中有我我中有你的关系，按照不同维度划分的结果，下面就来一一列举这些锁

### 锁粒度
以加锁的粒度为准可以分为，全局锁，表锁和行锁；

* 全局锁；对整个数据库加锁，让整个数据库在只读状态，在数据库备份时使用（主库上备份，所有写操作将不能进行影响业务；从库上备份，备份期间不能有写操作，不能执行binlog，主从延迟增大）
* 表锁，加锁的粒度为数据表
	* 自增锁 AUTO-INC Locks 是一种特殊的表锁，可以保证一个事务插入数据的id连续
* 行锁，加锁的粒度为数据行
	* 记录锁 Record Locks；锁定当前数据行
	* 间隙锁 Gap Locks；锁定数据行的前后间隙
		* 插入意向锁 Insert Intention Locks；在插入操作之前会把插入的区域加入插入意向锁，不同区域的的锁互相兼容
	* 临键锁 Next-Key Locks；锁定数据行+前后的间隙

### 锁类型

* 共享锁（S）和 排他锁（X）；相当于，读锁和写锁，读可以共享锁，写只能独占资源；粒度可以有行锁和表锁，比如行级或者表级的S锁（或X锁）
* 意向锁(意向共享锁IS，意向排他锁IX)，一个事务在给数据行加锁时会在数据所在的表加相同类型的意向锁(比如对数据行加X锁就会在该表加表级别的X意向锁)，表示该表有事务对数据行加了锁，意向锁直接是相互兼容的，但是与具体的表锁或者行锁有着互斥关系的，具体关系见下面分析

## 锁之间的兼容关系

||S|X|IS|IX|
|---|---|---|---|---|
|S|✅|❎|✅|❎|
|X|❎|❎|❎|❎|
|IS|✅|❎|✅|✅|
|IX|❎|❎|✅|✅|

**注:** 意向锁是表级别的锁，上面表格中与意向锁兼容和互斥关系指的是与表级别的S锁或者X锁，意向锁和行级别的锁是不冲突的；主要是为了防止一个事务在插入或者修改数据的时候另一个事务修改了表结构之间会冲突；插入或者修改数据是一般会加行锁或者间隙锁，同时在表上加IX锁(意向排他锁)，另一个事务要修改表结构是要给表加X锁，这时会和IX锁冲突等待IX锁释放

这个只是我们熟知的锁之间的兼容关系，除此之外呢，MySQL中还有更加精确的锁之间的兼容关系，也就是在所有类型的行锁之间的兼容关系，([见参考文章3](https://www.iteye.com/blog/narcissusoyf-1637309))；这个关系是在X锁与X锁或者S锁与X锁不兼容的情况下再进行比对

||G|I|R|N|
|---|---|---|---|---|
|G|✅|✅|✅|✅|
|I|❎|✅|✅|❎|
|R|✅|✅|❎|❎|
|N|✅|✅|❎|❎|

**注:**  G=Gap锁，I=Insert Intention锁，R=Record锁，N=Next-Key锁；

上表中的行代表当前已经存在的锁，理解一下这张表就假设两个X锁排斥的前提下：

* 第一列
	* 已经存在G锁，不允许再加I锁（加了间隙锁就不允许在间隙中插入操作了）
	* 已经存在G锁，还可以再加G锁、R锁和N锁（也就是说G锁之间是相互兼容的，R锁和G锁本身就不冲突当然兼容，N锁实质上就是G锁+R锁，G锁和R锁都兼容那么N锁一定兼容）
        * 间隙锁的作用是防止被锁的间隙增删数据，所以间隙锁直接是不互斥的
* 第二列，已经存在I锁，剩下的所有类型锁都可以再加
	* 这里看到再加G锁也是兼容的即使加锁的间隙是一样的
	* 根据官方文档两个I锁如果插入不是同一个位置是相互兼容的，这样可以提高并发
	* 兼容R锁也很好理解，I锁是间隙，R锁是记录本身就不冲突
	* G锁和R锁都兼容了那么N锁一定兼容
* 第三列，已经存在R锁，是可以加间隙锁的(G锁和I锁)，但如果包含记录的锁就不兼容(R锁和N锁)
* 第四列，已经存在N锁，首先包含记录的锁(R锁和N锁)是不兼容的，I锁表明要插入数据也是不兼容的，G锁是兼容的

总结：G锁与其他锁之间是相互兼容的，无论间隙是否相同，也无论当前是什么类型的锁，再加G锁也是兼容的；I锁是G锁的一种，是在插入之前表明插入操作的意向，如果当前存在G锁或者N锁，也就是加锁的区域相同就不能再加I锁，需要等待，其他情况与G锁相同都是兼容的；R锁和N锁就看加锁的数据是否冲突来判断锁是否兼容

### 参考

[秒懂InnoDB的锁](https://i6448038.github.io/2019/02/23/mysql-lock/)

[InnoDB Locking](https://dev.mysql.com/doc/refman/5.6/en/innodb-locking.html)

[从一个死锁看mysql innodb的锁机制](https://www.iteye.com/blog/narcissusoyf-1637309)
