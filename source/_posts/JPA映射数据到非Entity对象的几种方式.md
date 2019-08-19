---
title: JPA映射数据到非Entity对象的几种方式
author: 鹏徙南冥
top: false
mathjax: false
categories:
  - java
tags:
  - java
  - jpa
  - 数据映射
date: 2019-08-19 13:53:29
updated: 2019-08-19 13:53:29
img:
summary:
---
## 1 说明
本文基于SpringBoot 2.1.7.RELEASE（spring-data-jpa版本：2.1.10.RELEASE）撰写，低版本jpa中的方法可能与最新版中的方法差异较大，但总体思路没有太大变化。文中观点皆为个人观点，如有错误或者更好的思路，欢迎指正。
## 2 为什么需要把数据映射到非Entity对象
我们知道在使用jpa查询数据时，大多数情况下都有一个被@Entity注解标识的类用来与数据库表做映射，使用时先查询出这个对象再做一系列转换传递到前端。在此种情况下，如果查询方法的返回结果或者参数不是一个被@Entity注解所标识的对象，那么即便是字段映射上了，jpa仍然会给出一个IllegalArgumentException: Not a managed type异常。但是有些场景下（比如查询count(id)亦或者需要将多表的字段进行组合）我们并不方便直接做一个实体类来接收，甚至根本也找不到能映射的字段，这个时候就应该考虑其他的办法了。

## 3 数据准备
数据库student表与score表，并假设其中已经存在了部分数据
```sql
CREATE TABLE student(
    id INT NOT NULL AUTO_INCREMENT  COMMENT '主键ID' ,
    name VARCHAR(128)    COMMENT '姓名' ,
    sex INT    COMMENT '性别，0：女；1：男' ,
    age INT    COMMENT '年龄' ,
    PRIMARY KEY (id)
) COMMENT = ' ';

CREATE TABLE score(
    id INT NOT NULL AUTO_INCREMENT  COMMENT '主键ID' ,
    student_id INT    COMMENT '学生ID' ,
    subject_id INT    COMMENT '科目 科目。0：语文；1：数学；2：英语' ,
    score INT    COMMENT '分数' ,
    PRIMARY KEY (id)
) COMMENT = ' ';
```
## 4 简单的聚合函数查询
比如当只想查询所有性别为男的同学的数量时，有以下几种实现方案。当然，这里只是列举了我暂时能想到的具有明显差异的方式，至于偏向于sql还是hql，返回对象还是Map，可以由读者自行发散。
### 4.1 使用countBy方法作为查询
以下便是全部的代码，再简单不过了。
```java
public interface StudentRepo extends CrudRepository<Student, Long> {
    long countBySex(int sex);
}
```

### 4.2 使用@Query注解查询
同样并不复杂，只是我个人并不推荐在已经使用了jpa的情况下继续使用sql或hql。
```java
@Query("SELECT COUNT(id) FROM Student WHERE sex = :sex")
long queryCountBySex(@Param("sex") int sex);
```

### 4.3 使用EntityManager构造SQL或HQL查询
这种方式明显比前面几种麻烦了许多，但是好处是对于动态的查询条件有了更好的支持。
```java
String sql = "SELECT COUNT(id) FROM student WHERE sex = ? ";
Query countQuery = entityManager.createNativeQuery(sql);
countQuery.setParameter(1, 1);
System.out.println("学生数量：" + countQuery.getSingleResult());
```
### 4.4 构建CriteriaQuery查询
本质上，我们可以自己构建一个查询而不必去写sql，这样做的好处是，我们“完美”的践行了OOP的思想。
```java
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<Long> query = cb.createQuery(Long.class);
Root<Student> root = query.from(Student.class);
query.select(cb.count(root.get("id")));
query.where(cb.equal(root.get("sex"), 1));
long count = entityManager.createQuery(query).getSingleResult();
System.out.println("学生数量：" + count);
```
### 4.5 使用Specification构造查询条件查询
本质上，这应该跟3.4的方法一样，但是使用Specification的好处是，我们可以将Specification作为参数传递给方法。只需要在Repository里定义一个`long count(Specification<Student> specification);`方法，jpa会自动帮我们查询。看上去，这应该是最复杂的方式了。的确，在这个条件下，使用这种方式确实有点“得不偿失”的感觉，但如果事情变得更加复杂，我个人是最推崇这种方式的。
```java
Specification<Student> studentSpecification = new Specification<Student>() {
    @Override public Predicate toPredicate(Root<Student> root, CriteriaQuery<?> criteriaQuery,
        CriteriaBuilder criteriaBuilder) {
        criteriaQuery.multiselect(criteriaBuilder.count(root.get("id")));
        return criteriaBuilder.and(criteriaBuilder.equal(root.get("sex"), 1));
    }
};

long count = studentRepo.count(studentSpecification);
System.out.println("学生数量：" + count);
```
## 5 复杂情况下的查询
现在来看一些稍微复杂点的情况。比如现在需要统计每个学生的所有科目的成绩总和，如果写sql的话，应该看上去像是这样：
```sql
SELECT st.`name`, SUM(s.score) FROM student st LEFT JOIN score s ON s.student_id = st.id GROUP BY s.student_id
```
这里举这个例子，只是为了方便说明做的一个简单举例，有时候我们确实需要做这样的统计：它们既不方便做成单独的实体，也不方便做多次查询来处理。在这个条件下，还能使用之前的那些方式么？事实上，除了3.1跟3.5中的方式（据我自己了解），其他都是可以实现的。

在这种条件下，应该有一个类用于接收查询结果（非@Entity注解的类）：
```java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class StudentScore {
    private String name;
    private Long totalScore;
}
```
接下来看看上面的方法都来如何改进。

### 5.1 使用@Query注解查询改进
从代码中可以看出，@Query能够使用构造方法对参数进行注入。这是其实现的方式之一。
```java
@Query("SELECT new com.jptangchina.jpa.StudentScore(st.name, SUM(s.score)) FROM Student st LEFT JOIN Score s ON s.studentId = st.id GROUP BY s.studentId")
List<StudentScore> queryStudentScore();
```
虽然此方式能够实现目标功能，但是对于一些复杂场景的动态SQL支持欠佳，况且我个人实在不愿看到代码里面既有sql又有方法，所以并不推荐。
### 5.2 使用EntityManager构造SQL或HQL查询方法改进
一种可能的写法是：
```java
String sql = "SELECT st.`name`, SUM(s.score) FROM student st LEFT JOIN score s ON s.student_id = st.id GROUP BY s.student_id";
NativeQuery query = entityManager.createNativeQuery(sql).unwrap(NativeQuery.class);
List<StudentScore> studentScores = query.getResultList();
```
不说了，纯sql，撸就完事儿了。
### 5.3 构建CriteriaQuery查询方法改进
由于这里用到了多表连接查询，因此需要在Student中指名与Score的映射关系：
```java
@OneToMany(fetch = FetchType.LAZY)
@JoinColumn(name = "studentId", insertable = false, updatable = false)
private List<Score> scores;
```
然后代码可以这样改进：
```java
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<StudentScore> query = cb.createQuery(StudentScore.class);
Root<Student> root = query.from(Student.class);
Join<Student, Score> scoreJoin = root.joinList("scores");
query.multiselect(root.get("name"), cb.sum(scoreJoin.get("score")));
query.groupBy(scoreJoin.get("studentId"));
List<StudentScore> studentScores = entityManager.createQuery(query).getResultList();
```

至于为什么3.5中的方法不能实现，我看了下jpa的源码，其中有这么一段代码：
```java
protected <S extends T> TypedQuery<S> getQuery(@Nullable Specification<S> spec, Class<S> domainClass, Sort sort) {
        CriteriaBuilder builder = this.em.getCriteriaBuilder();
        CriteriaQuery<S> query = builder.createQuery(domainClass);
        Root<S> root = this.applySpecificationToCriteria(spec, domainClass, query);
        query.select(root);
        if (sort.isSorted()) {
            query.orderBy(QueryUtils.toOrders(sort, root, builder));
        }

        return this.applyRepositoryMethodMetadata(this.em.createQuery(query));
    }
```
可以看到，jpa自动构建的查询中，CriteriaQuery与Root的泛型类型是一致的，而通过3.4中的代码对比来看的话，CriteriaQuery的泛型类型应该与最终返回的对象一致才对，所以如果没有其他方案的话，这个方法应该是行不通的。而实际情况中，jpa会抛出一个PropertyReferenceException异常。

## 6 总结
以上皆为个人在平时开发中遇到过的一些问题，但是这里做了简化处理。总体来讲，既然使用了jpa，我认为就尽可能的避免以sql的形式再去处理数据库操作。

jpa中数据当然也可以返回为`List<Object[]>`、`List<Map<String, Object>>`的形式，但同样的，个人认为这已经违背了OOP的思想，是一种妥协，不应积极采用的，多使用Specification才是正道哇。

再次声明，个人观点，有错误欢迎提出。