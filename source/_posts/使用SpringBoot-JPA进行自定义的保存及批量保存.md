---
title: 使用SpringBoot JPA进行自定义的保存及批量保存
author: 鹏徙南冥
top: false
mathjax: false
categories:
  - java
tags:
  - java
  - SpringBoot
  - JPA
date: 2019-06-03 17:55:52
updated: 2019-06-03 17:55:52
---
## 说明
SpringBoot版本：2.1.4.RELEASE

java版本：1.8

文中所说JPA皆指spring-boot-starter-data-jpa
## 使用JPA保存一个Student对象
在JPA中保存一个对象，仅需要该对象，一个仓储即可。
StudentDO实体类：
```java
@Getter
@Setter
@Entity
@Table(name = "t_student")
public class StudentDO {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column
    private Long id;
    @Column
    private String seq;
    @Column
    private String name;
    @Column
    private int sex;
}
```
JPA仓储：

```java
@Repository
public interface StudentRepo extends JpaRepository<StudentDO, String> {
}
```
一般的，我们只需要调用StudentRepo.save()方法即可完成对实体对象的保存操作。

```java
    @Test
    public void testSave() {
        StudentDO student = new StudentDO();
        student.setName("张三");
        student.setSex(1);
        student.setSeq("123456");
        studentRepo.save(student);
        Assert.assertNotNull(student.getId());
    }
```
## 在插入过程中使用mysql函数
如果我们希望student的seq值由系统自动生成，且生成规则为“yyMMdd + 8位自增序列”（例如19060310000000）又该如何实现呢？

首先想到的是该如何生成这一串序列，mysql不像oracle自身支持sequence，因此在这里可以借用函数以及额外的sequence表来实现这一操作，网上有很多实现方式，这里就不再赘述。

现在已经有了函数getseq('student_seq')可以获取到该序列，该如何将其应用到保存对象的方法中？显然的一个问题是，像上面那样再直接调用save方法已经行不通了，应该得需要自定义插入的sql实现。

一个容易想到的办法是，在StudentDO类上使用注解@SQLInsert来定义insert的实现，它写起来应该会像这个样子：
```
@SQLInsert(sql = "INSERT INTO t_student(seq, name, sex) VALUES (getseq('student_seq'), ?, ?")
```
这条sql语句本身并没有什么问题，再次调用save()方法也确实能够执行。但是很可惜，它确会抛出一个sql异常：
```
java.sql.SQLException: Parameter index out of range (3 > number of parameters, which is 2).
```
显然是程序认为有多少个参数，就得有多少个“?”与之匹配，目前我并没有找到解决这个问题的方案，所以这种方法宣告失败。

既然@SQLInsert行不通，或许可以考虑使用@Query注解来自定义一个实现。我们可以在StudentRepo中定义这样一个方法：

```java
    @Transactional
    @Modifying
    @Query(value = "INSERT INTO t_student(seq, name, sex) VALUES (getseq('student_seq'), :#{#student.name}, :#{#student.sex})", nativeQuery = true)
    int insert(@Param("student") StudentDO student);
```
试着运行一下，结果很成功，对象被正常的存储到数据库中，并且seq的取值也正常。看上去我们的问题已经得到了解决，但事实真的如此么？
## 自定义的批量存储
上面的例子中，我们成功通过JPA调用了mysql函数将对象存储到数据库中。但上面的例子只提供了单个保存的方法，如果我们想批量保存呢？@Query里面的sql能够进行改造么？我并没有找到@Query中使用List<Object>作为参数的insert方法，但是这并不代表这一操作不能执行。JPA仍旧提供给了使用者原始的使用方式：利用EntityManager来构造sql并执行。

```java
    @PersistenceContext
    private EntityManager entityManager;

    private int batchInsert(List<StudentDO> students) {
        StringBuilder sb = new StringBuilder();
        sb.append("INSERT INTO t_student(seq, name, sex) VALUES ");
        for(StudentDO student : students) {
            sb.append("(getseq('student_seq'), ?, ?),");
        }
        String sql = sb.toString().substring(0, sb.length() - 1);
        Query query = entityManager.createNativeQuery(sql);
        int paramIndex = 1;
        for(StudentDO student : students) {
            query.setParameter(paramIndex++, student.getName());
            query.setParameter(paramIndex++, student.getSex());
        }
        return query.executeUpdate();
    }
```
就像MyBatis一样，使用者也可以自定义SQL来执行，试试看，同样没有问题，再多的数据也可以被保存到数据库中！批量保存的效果达到了。

再仔细想一想，通过上面的过程，还有什么问题么？对比JPA自带的save()方法，似乎我们的自定义保存返回的都是int结果，也就是操作影响的数据库行数。使用过JPA的人都应该了解，JPA的save()方法（或者其他JPA方法）返回的对象是经过持久化的，得益于这一特性，使用者可以在调用save()方法之后获取到对象的id等必须先插入到数据库之后才会有的值。显然这里的操作已经失去了这一特性，那如果我们把返回值对应的改为Object或者List<Object>可以做到么？答案是并不能，我们会得到如下异常：
```java
org.springframework.dao.InvalidDataAccessApiUsageException: Modifying queries can only use void or int/Integer as return type!; nested exception is java.lang.IllegalArgumentException: Modifying queries can only use void or int/Integer as return type!
```
insert方法必须使用@Modifying进行注解，而@Modifying注解的方法又只能返回int类型的结果。这种情况下或许只能先利用查询得到seq的值再进行操作。

## 总结
对于JPA的使用还不够了解，一些复杂的情况下没有找到最理想的实现方案。
1. @Query注解中是否能够使用List<Object>以及实现动态拼接参数的效果没有得到解决
2. 自定义的sql语句返回持久化对象的问题没有方案

在以后的使用了解中希望能够找到解决办法，将问题记录在这里，以便后续查看。