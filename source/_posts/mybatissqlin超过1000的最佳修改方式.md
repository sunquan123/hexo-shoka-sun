---
layout: shoka
title: Mybatis sql in参数超过1000的最佳修改方式
date: 2023-11-14 19:05:14
tags: ['Mybatis','sql in']
---

## Mybatis sql in参数超过1000的最佳修改方式

我们日常开发过程中肯定遇到过这种问题：

当Mybatis的in查询条件超过1000时会报错，需要修改in查询的部分

示例代码如下：

```xml
<if test="idList != null and idList.size() > 0">
            and id in
            <foreach collection="idList" item="item" index="index" open="(" close=")">
                <if test="index > 0">
                    <choose>
                        <when test="index % 1000 == 999"> ) or id in( </when>
                          <otherwise>,</otherwise>
                     </choose>
                </if>
                #{item}
            </foreach>
</if>
```

如果是普通查询，到这里一般就认为没啥问题了。但是，仔细研究这条sql语句的话，会找出漏洞。

### 我们先写一个基本使用案例：

```xml
select * from student where sex = #{sex}
 <if test="idList != null and idList.size() > 0">
            and id in
            <foreach collection="idList" item="item" index="index" open="(" close=")">
                <if test="index > 0">
                    <choose>
                        <when test="index % 1000 == 999"> ) or id in( </when>
                          <otherwise>,</otherwise>
                     </choose>
                </if>
                #{item}
            </foreach>
</if>
```

当参数不超过1000时，sql语句like：

select * from student where sex = #{sex} and id in ('1','2')

当超过1000时，sql语句like：

select * from student where sex = #{sex} and id in ('1',...'999') or id in ('1000','1001')

#### 注意：

这里的查询条件从查询sex和id同时满足要求变成了sex和id在999以内为一个条件，同时or上id在1000以上为另一个条件。这就导致原本的查询条件变成了两组，查询到的结果变多了。

这里如果看懂了就不需要看这个解释部分，如果没看懂，请继续查看这一部分。

### 解释部分

原本这条sql的查询要求是根据学生性别和学生id查询到对应的学生记录。

即select * from student where sex = #{sex} and id in ('1','2')这条sql满足查询要求，不会查询到多余的数据。

如果使用select * from student where sex = #{sex} and id in ('1',...'999') or id in ('1000','1001')查询，将会得到以下数据：

学生id在1000以内的，且性别为指定性别的数据+学生id在1000以上的数据。

按照查询要求，正确的数据应该是：

学生id在1000以内的，且性别为指定性别的数据+学生id在1000以上的，且性别为指定性别的数据。

这样就能明白，上面的sql改变了查询要求，返回了不应该查到的数据。

### 最佳修改方式

理论上，应该得到的sql语句应该是：

select * from student where sex = #{sex} and (id in ('1',...'999') or id in ('1000','1001'))

所以，修改的sql in形式应该如下：

```xml
select * from student where sex = #{sex}
 <if test="xxxList != null and xxxList.size() > 0">
      and (id in
      <foreach collection="idList" item="item" index="index" open="(" close=")">
           <if test="index > 0">
                <choose>
                     <when test="index % 1000 == 999"> ) or id in( </when>
                          <otherwise>,</otherwise>
                </choose>
            </if>
            #{item}
       </foreach>
       )
</if>
```

### Mybatis-Generator生成代码也需要修改

Mybatis-Generator生成的代码也有sql in查询的问题，需要开发者自己在项目中找到并修改。

```xml
<when test="criterion.listValue">
    and ${criterion.condition}
    <foreach close=")" collection="criterion.value" item="listItem" open="(" separator=",">
    #{listItem}
    </foreach>
</when>
```

修改后的代码如下：

```xml
<when test="criterion.listValue">
    and (${criterion.condition}
    <foreach collection="criterion.value" item="listItem" index="index" open="(" close=")">
          <if test="index > 0">
               <choose>
                    <when test="index % 1000 == 999"> ) or ${criterion.condition}( </when>
                    <otherwise>,</otherwise>
               </choose>
           </if>
           #{listItem}
    </foreach>
     )
</when>
```

如果公司有定制化Mybatis-Generator的能力，也可以自己修改Mybatis-Generator项目的生成代码，最后将指定版本工具推广到全公司哟~

### 结尾

我们开发者日常使用Mybatis时，最好有检查sql日志是否正确的思维，做一个细心的小机灵鬼。
