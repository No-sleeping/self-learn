## CURD

```sql
CREATE TABLE `student_tbl` (
  `student_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `student_name` varchar(100) NOT NULL,
  `student_score` varchar(40) NOT NULL,
  `student_sex` varchar(40) NOT NULL,
  `student_date` date DEFAULT NULL,
  `leader` int(10) DEFAULT NULL,
  PRIMARY KEY (`student_id`)
) ENGINE=InnoDB AUTO_INCREMENT=8 DEFAULT CHARSET=utf8
```

```sql
insert into base_test.student_tbl 
(student_name,student_score,student_sex) 
values ("qwe",23,"male");
```

```sql
delete from base_test.student_tbl where student_id=7;
```

```sql
SELECT * 
FROM base_test.student_tbl 
where student_score > 
(select avg(student_score) from base_test.student_tbl);
```

```sql
mysql> UPDATE tb_courses_new
    -> SET course_name='DB',course_grade=3.5
    -> WHERE course_id=2;
```

## 一、事务

ACID

1. 原子性（Atomicity）
事务被视为不可分割的最小单元，事务的所有操作要么全部提交成功，要么全部失败回滚。

回滚可以用回滚日志（Undo Log）来实现，回滚日志记录着事务所执行的修改操作，在回滚时反向执行这些修改操作即可。

<br/>

2. 一致性（Consistency）
数据库在事务执行前后都保持一致性状态。在一致性状态下，所有事务对同一个数据的读取结果都是相同的。
   
   <br/>
3. 隔离性（Isolation）
一个事务所做的修改在最终提交以前，对其它事务是不可见的。
   
   <br/>
4. 持久性（Durability）
一旦事务提交，则其所做的修改将会永远保存到数据库中。即使系统发生崩溃，事务执行的结果也不能丢失。
   
   <br/>

系统发生崩溃可以用重做日志（Redo Log）进行恢复，从而实现持久性。与回滚日志记录数据的逻辑修改不同，重做日志记录的是数据页的物理修改。

<br/>

## 二、并发一致性问题

### 脏读

如果一个事务「读到」了另一个「未提交事务修改过的数据」，就意味着发生了「脏读」现象。

![](https://img-blog.csdnimg.cn/img_convert/10b513008ea35ee880c592a88adcb12f.png)

### 不可重复读

在一个事务内多次读取同一个数据，如果出现前后两次读到的数据不一样的情况，就意味着发生了「不可重复读」现象。

![](https://img-blog.csdnimg.cn/img_convert/f5b4f8f0c0adcf044b34c1f300a95abf.png)

### 幻读

在一个事务内多次查询某个符合查询条件的「记录数量」，如果出现前后两次查询到的记录数量不一样的情况，就意味着发生了「幻读」现象。

![](https://img-blog.csdnimg.cn/img_convert/d19a1019dc35dfe8cfe7fbff8cd97e31.png)

<br/>

## 三、隔离级别

### 读未提交（read uncommitted），指一个事务还没提交时，它做的变更就能被其他事务看到；

### 读提交（read committed），指一个事务提交之后，它做的变更才能被其他事务看到；

### 可重复读（repeatable read），指一个事务执行过程中看到的数据，一直跟这个事务启动时看到的数据是一致的，MySQL InnoDB 引擎的默认隔离级别；

### 串行化（serializable ）；会对记录加上读写锁，在多个事务对这条记录进行读写操作时，如果发生了读写冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行； 

![](https://img-blog.csdnimg.cn/img_convert/4e98ea2e60923b969790898565b4d643.png)

<br/>

![](https://img-blog.csdnimg.cn/img_convert/d5de450e901ed926d0b5278c8b65b9fe.png)

- 在「读未提交」隔离级别下，事务 B 修改余额后，虽然没有提交事务，但是此时的余额已经可以被事务 A 看见了，于是事务 A 中余额 V1 查询的值是 200 万，余额 V2、V3 自然也是 200 万了；
- 在「读提交」隔离级别下，事务 B 修改余额后，因为没有提交事务，所以事务 A 中余额 V1 的值还是 100 万，等事务 B 提交完后，最新的余额数据才能被事务 A 看见，因此额 V2、V3 都是 200 万；
- 在「可重复读」隔离级别下，事务 A 只能看见启动事务时的数据，所以余额 V1、余额 V2 的值都是 100 万，当事务 A 提交事务后，就能看见最新的余额数据了，所以余额 V3 的值是 200 万；
- 在「串行化」隔离级别下，事务 B 在执行将余额 100 万修改为 200 万时，由于此前事务 A 执行了读操作，这样就发生了读写冲突，于是就会被锁住，直到事务 A 提交后，事务 B 才可以继续执行，所以从 A 的角度看，余额 V1、V2 的值是 100 万，余额 V3 的值是 200万。
