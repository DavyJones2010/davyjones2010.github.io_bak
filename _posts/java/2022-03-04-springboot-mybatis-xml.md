---
layout: post
title: SpringBoot+MyBatis使用XML配置方案
subtitle: SpringBoot+MyBatis使用XML配置方案
tags: [springboot, mybatis, xml]
---

# 背景
项目代码之前是基于ibatis, 而且SQL语句是基于sqlmap.xml文件的.
现在升级到mybatis 3.4.2, 但mybatis的默认方案是基于annotation的. 
个人非常不习惯annotation的方式, 每次排查问题拼接SQL, 以及后续修改SQL, 极其麻烦, 不直观.
因此想要使用xml的方式配置. 在这个过程中踩了很多坑, 现在记录下来.

# annotation sql的问题


# xml sql的优势


# xml sql的配置
## 第一步: mapper interface定义
- 注意一定要使用 `@Repository` 注释 `UserMapper`, 否则该Mapper无法被Spring容器管理
```java
package edu.xmu.mybatis.somemapper;

import org.apache.ibatis.annotations.Param;
import org.springframework.stereotype.Repository;

/**
 * SQLMap文件参见: {@linkplain UserMapper.xml}
 *
 * @author kunlun.ykl
 */
@Repository
public interface UserMapper {
    /**
     * 根据userName获取userId
     *
     * @param userName
     * @return
     */
    Integer getUserId(@Param("userName") String userName);
}
```

## 第二步: mapper xml文件
- 注意mapper文件一定要放在 `src/main/resources/edu/xmu/mybatis/somemapper/` 目录下
- 且mapper文件名一定要与interface的名称一样, 即本例中 `UserMapper.xml`

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="edu.xmu.mybatis.somemapper.UserMapper">

    <select id="getUserId" resultType="int">
        SELECT id
        FROM `user`
        WHERE user_name = #{userName}
    </select>
</mapper>
```

