---
title: MySQL事务原理
date: 2021-08-25 09:12:24
tags: mysql
categories: 知识
---
## 介绍

什么是事务？！

一句话来说，事务是一次操作的`逻辑单位`。要不全部成功，要不全部失败。

它主要是用来保证`数据一致性`的问题。

<!--more-->

比如去银行转账的操作，原来账户的扣款，和目标账户的加款。
这两个操作要放在一个事务里面，要不一起成功，要不一起失败。

如果在事务中间，发生问题，需要把已经执行了的操作`回滚`，以保证数据准确。

## 事务的常见命令

- START TRANSACTION：开始一个事务
- COMMIT：事务顺利完成时，提交事务
- ROLLBACK：事务发生了异常，回滚

比如我们有一张customer表，执行一下语句：

```sql
start transaction;
insert into customer set name = 'zhangsan';
insert into customer set name = 'lisi';
commit ;
```

再查询表，会得到：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/Prom8Z.png)
表示一个事务完成、提交。

我们再执行下面操作：

```sql
start transaction;
insert into customer set name = 'wangwu';
rollback ;
```

再查询customer表，会发现wangwu并没有被添加到表中，表中还只有两条记录。

- 建立保存点：SAVEPOINT 保存点名称
- 删除保存点：RELEASE SAVEPOINT 保存点名称
- 回滚到特定保存点：ROLLBACK TO SAVEPOINT 保存点名称

我们接着再执行下面点语句：

```sql
start transaction;
insert into customer set name = 'zhao liu';
savepoint my_point;
insert into customer set name = 'qian qi';
rollback to savepoint my_point;
```

我们查询到到结果是：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/gIvWTx.png)
`zhao liu` 成功入库了，而 `qian qi` 被回滚掉了。

`savepoint` 保存点，可以让我们实现部分回滚。

## Transaction 四大特性： ACID

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/NfuAJQ.png)

一个事务有下列四个重要到特性，简称ACID

- A：Atomicity 原子性。 事务里面到操作，不可分割，要不一起成功，要不一起失败。
- C：Consistency 一致性。在事务的前后，系统的整体数据保持一致。比如说银行转账。
- I：Isolation 隔离性。 事务之间，相互隔离，互不干扰。
- D：Durability 永久性。 一旦数据修改完成，永久有效。

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/9qUodw.png)
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/xJ1VeA.png)
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/jez6I1.png)
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/vXbP4p.png)

## 事务隔离级别

介绍完了ACID，我们再来重点说说ACID其中的I(Isolation)。

我们都知道，事务之间是隔离的
但是如何处理多个事务之间操作和读取同一个数据的结果，
我们需要仔细考虑。

事务隔离级别是在多个事务同时进行更改和执行查询时，
调整`性能`和结果的`可靠性`、`一致性`和`可再现性`之间的平衡的设置。

MySQL（InnoDB）提供四种事务隔离级别：

- read uncommitted 读未提交
- read committed   读已提交
- repeatable read  可重复读（MySQL默认隔离级别）
- serializable     串行

我们可以参考[官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)

### 事务隔离级别的设置

```sql
# 设置全局的事务隔离级别
SET GLOBAL TRANSACTION LEVEL [REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED | SERIALIZABLE];
# 设置当前Session的隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL [REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED | SERIALIZABLE];
# 查看当前事务隔离级别
SELECT @@transaction_ISOLATION;
```

我们先做一下前置的准备，
开启两个Session，
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/Lf8Nzf.png)

并查看一下当前的事务隔离级别
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/3zaxpJ.png)

接下来，我们一个一个看：

### 未提交读（READ UNCOMMITTED）

未提交读，指的就是一个事务读到了另外一个事务未提交的数据。

这样会导致脏读。

我们来实验一下：

我们先设置两个Session的隔离级别：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

查看下：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/WNJBVz.png)

我们现在数据库当中有两条数据

```sql
select * from customer;
```

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/trxTdG.png)

我们在Session A中开启一个事务，并修改一条记录，但是不提交。

再在Session B 中查询，看会得到怎样的结果。

我们现在Session A 中执行以下操作：

```sql
# session A
# 1、开启事务
start transaction ;

# 2、修改数据
update customer set name = 'zhangsan2' where id = 1;
```

我们在Session B 中看下结果：

```sql
# session B
# 3、查看数据
select * from customer where id = 1;
```

此时Session A中的事务还没有提交，也就是说有回滚的可能。
但在Session B中查看数据， 客户名称已经从 `zhangsan` 变成了 `zhangsan2`。

如果Session A再进行了回滚，rollback。 这样Session B就读出了一个不存在的数据，就是脏读。

这是非常不安全的一种隔离级别。

### 已提交读（READ COMMITTED）

已提交读，就是读到的数据都是另外的事务已经提交过了的数据。
可以解决脏读的问题。

我们先修改下事务隔离级别：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT @@transaction_ISOLATION;
```

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/5fAmh8.png)

我们在SessionA中修改数据：

```sql
# session A
# 1、开启事务
start transaction ;

# 2、修改数据
update customer set name = '张飞' where id = 1;
```

在Session B 中查看数据：

```sql
# session B
# 3、查看数据
select * from customer where id = 1;
```

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/YMiybO.png)
我们可以看到，客户名称还是`zhangsan`，没有变成`张飞`。

我们这个时候再把Session A中的事务提交

```sql
# session A
# 1、开启事务
# start transaction ;

# 2、修改数据
# update customer set name = '张飞' where id = 1;

# 4、提交事务
commit ;
```

再次在Session B 中查看数据：

```sql
# session B
# 3、查看数据
select * from customer where id = 1;

# 5、再次查看数据
select * from customer where id = 1;
```

![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/wY9JUG.png)
我们可以看到，客户名称已经变成了 `张飞`。

如果session A的事务回滚的话，我们在Session B中看到的结果还会是 `zhangsan`
这样，我们就成功解决了脏读的问题了。

但是如果我们在同一个事务中，读取数据，可能会查询到不同的结果。
我们按照数字的顺序，执行以下SQL：

```sql
# session B

# 1、开启事务，并查看数据
start transaction ;
select * from customer where id = 1; -- 张飞

# 3、再次查看数据
select * from customer where id = 1; -- 关羽

# 5、再次查看数据
select * from customer where id = 1; -- 刘备
```

```sql
# session A

# 2、修改数据（自动提交事务）
update customer set name = '关羽' where id = 1;

# 4、修改数据（自动提交事务）
update customer set name = '刘备' where id = 1;
```

我们可以看到，Session B 在同一个事务中，同样的查询语句，得出的结果是不一样的。
这种现象我们称之为`不可重复读`。

### 可重复读（REPEATABLE READ）

有些场景，我们需要在一个事务中，查询到到数据保持一致，不管外部事务如何改变。

这个时候，就需要我们满足`可重复读`。

我们直接看效果：

首先修改事务隔离级别：

```sql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT @@transaction_ISOLATION;
```

再按照下列SQL的顺序，执行：

```sql
# session B

# 1、开启事务，并查看数据
start transaction ;
select * from customer where id = 1; -- 刘备

# 3、再次查看数据
select * from customer where id = 1; -- 刘备

# 5、再次查看数据
select * from customer where id = 1; -- 刘备

# 6、提交当前事务，在查看数据
commit ;
select * from customer where id = 1; -- 张飞
```

```sql
# session A

# 2、修改数据（自动提交事务）
update customer set name = '关羽' where id = 1;

# 4、修改数据（自动提交事务）
update customer set name = '张飞' where id = 1;
```

我们可以看到，在Session B 中，每一次查询的结果都是一样的，
不管实际的数据怎样的变化，即使数据已经被SessionA所改变，且已提交。

但是这种隔离级别同样有这自己缺陷，它会发生幻读。

我们看下面的场景：

```sql
# 1、开启事务，并查看数据
# session B

# 1、开启事务，并查看数据
start transaction ;
select * from customer where id = 10; -- null

# 3、插入指定ID的数据
select * from customer where id = 10; -- null
insert into customer values (10, '吕布'); -- 失败

commit ;
```

```sql
# session A

# 2、插入数据
insert into customer values (10, '小吕布'); -- 成功
```

我们可以看到，SessionB在一个事务中，想要插入一条ID为10的用户。
重要的是，要第一步查询，已经确定了ID为10的用户并不存在。

但是在事务的过程中，Session A 插入了一条ID为10的客户，并成功提交事务。

等再回到Session B， 再想插入一条ID为10的客户，就报错了`Duplicate entry '10' for key 'customer.PRIMARY'`。

其实，在Session B的操作中，逻辑都是没问题的。但是还是发生了幻读，导致了系统异常。

解决办法是，在Session B中，我们这样写select语句：

```sql
select * from customer where id = 10 for update ;
```

在 select 语句后面， 添加一个`for update`，可以把`ID=10`给加上一个锁，
SessionA， 再想干扰，添加数据的时候，就会拿不到锁，而只能等待SessionB的事务完成，并释放锁。

所以保证了SessionB的原子性，杜绝的幻读。

### 串行化（SERIALIZABLE）

串行化，这种隔离级别，可以杜绝所有的干扰，包括（脏读、不可重复读、幻读）。

在此级别下，我们便不需要对 SELECT 操作显式加锁，InnoDB会自动加锁，事务安全，但性能很低。

非常不推荐使用。

## InnoDB的事务实现（MVCC）

MVCC，多版本并发控制。
顾名思义，他会记录数据变更的版本（像git一样），形成一个版本链。

在通过对这个版本链中不同版本的处理，来实现事务，和事务隔离级别。

### 版本链

在数据库行数据的结构中，除了我们存储的字段外，还有两个必不可少的隐藏字段：

当前事务ID(`trx_id)` 和 上个版本数据的引用(`roll_pointer`)。

直接上代码，我们用Java来模拟MVCC。

完整代码，请查看[这里](https://github.com/CPyeah/java-projets/tree/master/java-mvcc/src/main/java)

```java
import java.util.concurrent.locks.ReentrantLock;
import lombok.Data;

@Data
public class Row<T> {

	// 当前操作的事务id
	private Integer trx_id;

	// 历史记录的地址
	private Row<T> roll_pointer;

	// 主键ID
	private Integer primaryId;

	// 其他数据
	private T otherData;

	// 行锁
	private ReentrantLock rowLock;

}

```

我们可以通过roll_pointer来找到所有的历史版本，可以实现回滚操作，也可以选择指定的版本来展示。

我们再看一下事务操作的代码：

```java
import java.util.HashMap;
import java.util.LinkedList;
import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 事务控制
 */
public class TransactionController {

	// 当前活跃的事务列表
	public static List<Integer> m_ids = new LinkedList<>();

	// 全局事务ID
	private static final AtomicInteger globeTransactionID = new AtomicInteger(100);

	// 获取事务ID
	public static Integer getNextTransactionId() {
		return globeTransactionID.getAndAdd(100);
	}

	// 更新数据，加行锁
	public static <T> T update(Row<T> newRow, Transaction transaction,
			HashMap<Integer, Row<T>> tableData) {
		Integer primaryId = newRow.getPrimaryId();
		Row<T> oldRow = tableData.get(primaryId);
		if (oldRow == null) {
			throw new IllegalArgumentException("Data does not exist");
		} else {
			newRow.setTrx_id(transaction.getTransactionId());
			newRow.setRoll_pointer(oldRow); // 更新版本链
			tableData.put(primaryId, newRow);
		}
		return newRow.getOtherData();
	}

	// 开启一个事务
	public static Transaction start() {
		Integer transactionId = getNextTransactionId();

		Transaction transaction = new Transaction();
		transaction.setTransactionId(transactionId);

		// 把当前的事务ID，存放到活跃事务列表中
		m_ids.add(transactionId);

		return transaction;
	}

	// 事务提交
	public static void commit(Transaction transaction) {
		if (transaction.getLock() != null) {
			transaction.getLock().unlock();
		}

		// 在活跃事务列表中，移除事务ID
		m_ids.remove(transaction.getTransactionId());
	}

}

```

其中几个关键点：

- 全局事务ID ： 全局唯一，递增
- 当前活跃的事务列表： 这里维护所有的未提交的事务的ID，
- 开启一个事务： 生成一个事务ID，并把这个事务，维护到活跃事务列表中
- 更新数据： 新数据赋予事务ID，新数据可以引用到上个版本的老数据，加行锁
- 提交事务： 解锁，把活跃事务列表中的删除当前事务

我们接下来看下事务是怎么运转的：

我们设计一个这样的测试：

完整代码请看[这里](https://github.com/CPyeah/java-projets/tree/master/java-mvcc/src/test/java)

```java
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

class TransactionControllerTest {

	@BeforeAll
	static void setup() {

		Row<Customer> row = getCustomerRow(1, "刘备");

		Customer.tableData.put(row.getPrimaryId(), row);
	}

	@Test
	public void test1() {

		// 开启事务A
		Transaction transaction_A = TransactionController.start();

		// 开启事务B
		Transaction transaction_B = TransactionController.start();

		// 更新数据，同时会加上行锁
		Row<Customer> 关羽 = getCustomerRow(1, "关羽");
		TransactionController.update(关羽, transaction_A, Customer.tableData);

		// 更新数据，同时会加上行锁
		Row<Customer> 张飞 = getCustomerRow(1, "张飞");
		TransactionController.update(张飞, transaction_A, Customer.tableData);

		// 事务A提交，并释放锁
		TransactionController.commit(transaction_A);

		// 更新数据，同时会加上行锁
		Row<Customer> 赵云 = getCustomerRow(1, "赵云");
		TransactionController.update(赵云, transaction_B, Customer.tableData);

		// 更新数据，同时会加上行锁
		Row<Customer> 诸葛亮 = getCustomerRow(1, "诸葛亮");
		TransactionController.update(诸葛亮, transaction_B, Customer.tableData);

		//commit
		TransactionController.commit(transaction_B);

		// 打印undo日志链表
		printData(Customer.tableData.get(1));
		// 1-诸葛亮(200) -> 1-赵云(200) -> 1-张飞(100) -> 1-关羽(100) -> 1-刘备(null)

	}

	private void printData(Row<Customer> customerRow) {
		StringBuilder sb = new StringBuilder(toString(customerRow));

		Row<Customer> currentPointer = customerRow;

		while (currentPointer.getRoll_pointer() != null) {
			currentPointer = currentPointer.getRoll_pointer();
			sb.append(" -> ")
					.append(toString(currentPointer));
		}
		System.out.println(sb);

	}

	private String toString(Row<Customer> customerRow) {
		return customerRow.getPrimaryId() + "-" + customerRow.getOtherData().getName()
				+ "("
				+ customerRow.getTrx_id()
				+ ")";
	}


	private static Row<Customer> getCustomerRow(Integer id, String name) {
		Customer customer = new Customer();
		customer.setId(id);
		customer.setName(name);

		Row<Customer> row = new Row<>();
		row.setPrimaryId(customer.getId());
		row.setOtherData(customer);

		return row;
	}

}
```

其中使用`Customer.tableData`来模拟存储数据。

在`test1`中，我们开启了两个事务，并执行了`update`操作，最总形成了一个版本链：

效果如图所示：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/O2WJFy.png)

> 两个事务不能交叉修改同一个数据，因为在`Transaction_A`第一次执行`update`操作的时候，
> 就给这个数据加上了行锁，`Transaction_B`必须等到`Transaction_A`提交，释放了锁，
> 才可以进行`Transaction_B`的`update`操作。

得到的数据`版本链`效果如下：
![](https://cp-images.oss-cn-hangzhou.aliyuncs.com/cfz3Lp.png)

### 隔离级别的实现

我们知道有四大隔离级别，我们来看下它们是怎样实现的。

#### SERIALIZABLE

串行化，这种级别下，不管是更新，还是查询，它都会加锁，
所以每次查询的数据，肯定是事务已经提交后的数据，
而每次更新数据的过程中，数据不会再发生变化。
不管是查询，还是更新数据的过程中，版本链都不会更新。

#### READ UNCOMMITTED

读未提交，它的实现也很简单，每次查看数据的时候，都读取版本链中最新的数据，不管操作这个数据的事务有没有提交。

#### READ COMMITTED

读已提交，
他会在每次查询的时候，都会生成一个`ReadView`，
它会拿到版本链，会从版本链从上往下搜索，
找到已经提交了的，最新的一个版本的数据。

#### REPEATABLE READ

可重复读
它会在当前的事务中，第一次查询的时候，生成一个`ReadView`，
也是从版本链中搜索，找到最新一个已经提交了的事务，
但是不同于读已提交的是，这个ReadView只生成一次，
以后的每次查询，都会查看这同一个ReadView。
从而保证`可重复读`。

### ReadView获取
```java
	// 获取到ReadView
	public <T> Row<T> readView(Row<T> chain, Integer currentTrxId) {
		if (chain == null) {
			return null;
		}
		if (m_ids.isEmpty()) {
			return chain;
		}
		// 该值代表生成readView时m_ids中的最小值
		Integer min_trx_id = m_ids.get(0);
		for (Integer id : m_ids) {
			min_trx_id = Math.min(min_trx_id, id);
		}
		// 该值代表生成readView时系统中应该分配给下一个事务的id值
		Integer max_trx_id = getNextTransactionId();

		Row<T> pointer = chain;

		while (true) {
			if (isThisReadView(pointer, min_trx_id, max_trx_id)) {
				return pointer;
			}
			if (pointer.getRoll_pointer() != null) {
				pointer = pointer.getRoll_pointer();
			} else {
				return null;
			}
		}

	}

	// 判断是否此版本为ReadView
	private <T> boolean isThisReadView(Row<T> pointer,
			Integer min,
			Integer max) {
		// 如果被访问版本的trx_id属性值小于m_ids列表中最小的事务id，
		// 表明生成该版本的事务在生成ReadView前已经提交，
		// 所以该版本可以被当前事务访问。
		if (pointer.getTrx_id() < min) {
			return true;
		}
		// 如果被访问版本的trx_id属性值大于m_ids列表中最大的事务id，
		// 表明生成该版本的事务在生成ReadView后才生成，
		// 所以该版本不可以被当前事务访问。
		if (pointer.getTrx_id() > max) {
			return false;
		}
		// 如果被访问版本的trx_id属性值在m_ids列表中最大的事务id和最小事务id之间，
		// 那就需要判断一下trx_id属性值是不是在m_ids列表中，
		// 如果在，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问；
		// 如果不在，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。
		if (m_ids.contains(pointer.getTrx_id())) {
			return false;
		} else {
			return true;
		}
	}

```

## 总结
至此，我们已经学习了MySQL事务到相关知识。
- 我们了解到了事务到概念
- 学习了事务的常见用法
- 了解了事务的四大特性 ACID
- 并重点说了事务的隔离特定
- 实践了不同事务隔离级别下的效果
- 使用Java代码模拟了MVCC的实现
- 并通过MVCC，掌握事务隔离级别的实现原理

本篇文章，希望大家好好掌握。😊


