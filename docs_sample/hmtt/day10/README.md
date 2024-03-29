# 01-今日学习目标 01:09

- 理解app端行为记录
- 完成关注、点赞、阅读行为
- 掌握不喜欢和收藏实现思路
- 完成app文章关系展示功能



# 02-概念介绍和微服务介绍 08:09

创建heima-leadnews-behavior服务



# 03-表关系分析 06:18

```sql
CREATE TABLE `ap_behavior_entry` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `type` tinyint(1) unsigned DEFAULT NULL COMMENT '实体类型\r\n            0终端设备\r\n            1用户',
  `entry_id` int(11) unsigned DEFAULT NULL COMMENT '实体ID',
  `created_time` datetime DEFAULT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC COMMENT='APP行为实体表,一个行为实体可能是用户或者设备，或者其它'

```

>entry_id 如果是用户，就是用户ID；如果是设备，就是设备ID



```sql
CREATE TABLE `ap_follow_behavior` (
  `id` bigint(20) unsigned NOT NULL,
  `entry_id` int(11) unsigned DEFAULT NULL COMMENT '实体ID',
  `article_id` bigint(20) unsigned DEFAULT NULL COMMENT '文章ID',
  `follow_id` int(11) unsigned DEFAULT NULL COMMENT '关注用户ID',
  `created_time` datetime DEFAULT NULL COMMENT '登录时间',
  PRIMARY KEY (`id`) USING BTREE,
  KEY `idx_entry_id` (`entry_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC COMMENT='PP关注行为表'
```

>注意：行为表中的entry_id是entry表中的id

# 04-用户端发送消息 05:28

UserRelationServiceImpl：

```java
/**
 * 处理关注的操作
 * @param apUser
 * @param followId
 * @param articleId
 * @return
 */
private ResponseResult followByUserId(ApUser apUser, Integer followId, Long articleId) {
    //判断当前关注人是否存在
    ApUser followUser = apUserMapper.selectById(followId);
    if(followUser == null){
        return ResponseResult.errorResult(AppHttpCodeEnum.DATA_NOT_EXIST,"关注用户不存在");
    }


    ApUserFollow apUserFollow = apUserFollowMapper.selectOne(Wrappers.<ApUserFollow>lambdaQuery().eq(ApUserFollow::getUserId, apUser.getId()).eq(ApUserFollow::getFollowId, followId));
    if(apUserFollow == null){
        //保存数据  ap_user_follow   ap_user_fan
        ApUserFan apUserFan = apUserFanMapper.selectOne(Wrappers.<ApUserFan>lambdaQuery().eq(ApUserFan::getUserId, followId).eq(ApUserFan::getFansId, apUser.getId()));
        //保存app用户粉丝信息
        if(apUserFan == null){
            apUserFan = new ApUserFan();
            apUserFan.setUserId(followId);
            apUserFan.setFansId(apUser.getId().longValue());
            apUserFan.setFansName(followUser.getName());
            apUserFan.setLevel((short)0);
            apUserFan.setIsDisplay(true);
            apUserFan.setIsShieldLetter(false);
            apUserFan.setIsShieldComment(false);
            apUserFan.setCreatedTime(new Date());
            apUserFanMapper.insert(apUserFan);
        }
        //保存app用户关注信息
        apUserFollow = new ApUserFollow();
        apUserFollow.setUserId(apUser.getId());
        apUserFollow.setFollowId(followId);
        apUserFollow.setCreatedTime(new Date());
        apUserFollow.setIsNotice(true);
        apUserFollow.setLevel((short)1);
        apUserFollowMapper.insert(apUserFollow);
        //TODO 记录关注文章的行为
        FollowBehaviorDto dto = new FollowBehaviorDto();
        dto.setFollowId(followId);
        dto.setArticleId(articleId);
        dto.setUserId(apUser.getId());
        kafkaTemplate.send(FollowBehaviorConstants.FOLLOW_BEHAVIOR_TOPIC, JSON.toJSONString(dto));
        return ResponseResult.okResult(AppHttpCodeEnum.SUCCESS);
    }else{
        //已关注
        return ResponseResult.errorResult(AppHttpCodeEnum.DATA_EXIST,"已关注");
    }
}
```



# 05-保存关注行为 15:05

1.提供接口，根据用户ID或者设备ID，查询entityId

2.保存行为数据

```java
package com.heima.behavior.service.impl;

import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.heima.behavior.mapper.ApFollowBehaviorMapper;
import com.heima.behavior.service.ApBehaviorEntryService;
import com.heima.behavior.service.ApFollowBehaviorService;
import com.heima.model.behavior.dtos.FollowBehaviorDto;
import com.heima.model.behavior.pojos.ApBehaviorEntry;
import com.heima.model.behavior.pojos.ApFollowBehavior;
import com.heima.model.common.dtos.ResponseResult;
import com.heima.model.common.enums.AppHttpCodeEnum;
import lombok.extern.log4j.Log4j2;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Date;

@Service
@Log4j2
public class ApFollowBehaviorServiceImpl extends ServiceImpl<ApFollowBehaviorMapper, ApFollowBehavior> implements ApFollowBehaviorService {

    @Autowired
    private ApBehaviorEntryService apBehaviorEntryService;

    @Override
    public ResponseResult saveFollowBehavior(FollowBehaviorDto dto) {
        //1.查询行为实体
        ApBehaviorEntry apBehaviorEntry = apBehaviorEntryService.findByUserIdOrEquipmentId(dto.getUserId(), null);
        if(apBehaviorEntry == null){
            log.error("行为实体为空");
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }
        //2.保存关注行为
        ApFollowBehavior apFollowBehavior = new ApFollowBehavior();
        //注意设置entryId指的是entry表中的id
        apFollowBehavior.setEntryId(apBehaviorEntry.getId());
        apFollowBehavior.setFollowId(dto.getFollowId());
        apFollowBehavior.setArticleId(dto.getArticleId());
        apFollowBehavior.setCreatedTime(new Date());
        save(apFollowBehavior);
        return ResponseResult.okResult(AppHttpCodeEnum.SUCCESS);
    }
}

```



# 06-接收数据并测试 08:25

```java
package com.heima.behavior.kafka.listener;

import com.alibaba.fastjson.JSON;
import com.heima.behavior.service.ApFollowBehaviorService;
import com.heima.common.constans.message.FollowBehaviorConstants;
import com.heima.model.behavior.dtos.FollowBehaviorDto;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

import java.util.Optional;

@Component
public class FollowBehaviorListener {

    @Autowired
    private ApFollowBehaviorService apFollowBehaviorService;

    @KafkaListener(topics = FollowBehaviorConstants.FOLLOW_BEHAVIOR_TOPIC)
    public void receivMessage(ConsumerRecord<?,?> record){
        //接收关注行为数据，保存
        Optional<? extends ConsumerRecord<?, ?>> optional = Optional.ofNullable(record);
        if(optional.isPresent()){
            FollowBehaviorDto dto = JSON.parseObject(record.value().toString(), FollowBehaviorDto.class);
            apFollowBehaviorService.saveFollowBehavior(dto);
        }
    }
}
```



# 07-点赞行为-需求分析 08:24

```sql
CREATE TABLE `ap_likes_behavior` (
  `id` bigint(20) unsigned NOT NULL,
  `entry_id` int(11) unsigned DEFAULT NULL COMMENT '实体ID',
  `article_id` bigint(20) unsigned DEFAULT NULL COMMENT '文章ID',
  `type` tinyint(1) unsigned DEFAULT NULL COMMENT '点赞内容类型\r\n            0文章\r\n            1动态',
  `operation` tinyint(1) unsigned DEFAULT NULL COMMENT '0 点赞\r\n            1 取消点赞',
  `created_time` datetime DEFAULT NULL COMMENT '登录时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC COMMENT='APP点赞行为表'
```

```java
package com.heima.apis.behavior;

import com.heima.model.behavior.dtos.LikesBehaviorDto;
import com.heima.model.behavior.pojos.ApLikesBehavior;
import com.heima.model.common.dtos.ResponseResult;

public interface ApLikesBehaviorControllerApi {


   /**
     * 保存点赞行为
     * @param dto
     * @return
     */
	ResponseResult like(LikesBehaviorDto dto);
}
```

```java
package com.heima.model.behavior.dtos;

import com.heima.model.common.annotation.IdEncrypt;
import lombok.Data;

@Data
public class LikesBehaviorDto {
    // 设备ID
    @IdEncrypt
    Integer equipmentId;
    // 文章、动态、评论等ID
    @IdEncrypt
    Long articleId;
    /**
     * 喜欢内容类型
     * 0文章
     * 1动态
     * 2评论
     */
    Short type;

    /**
     * 喜欢操作方式
     * 0 点赞
     * 1 取消点赞
     */
    Short operation;
}
```



# 08-功能实现及测试 14:10

```java
package com.heima.behavior.service.impl;

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.heima.behavior.mapper.ApLikesBehaviorMapper;
import com.heima.behavior.service.ApBehaviorEntryService;
import com.heima.behavior.service.ApLikesBehaviorService;
import com.heima.model.behavior.dtos.LikesBehaviorDto;
import com.heima.model.behavior.pojos.ApBehaviorEntry;
import com.heima.model.behavior.pojos.ApLikesBehavior;
import com.heima.model.common.dtos.ResponseResult;

import com.heima.model.common.enums.AppHttpCodeEnum;
import com.heima.model.user.pojos.ApUser;
import com.heima.utils.threadlocal.AppThreadLocalUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Date;


@Service
public class ApLikesBehaviorServiceImpl extends ServiceImpl<ApLikesBehaviorMapper, ApLikesBehavior> implements ApLikesBehaviorService {

    @Autowired
    private ApBehaviorEntryService apBehaviorEntryService;

    @Override
    public ResponseResult like(LikesBehaviorDto dto) {
        //1.检查参数
        if(dto == null || dto.getArticleId() == null || (dto.getType() < 0 || dto.getType() > 2) || (dto.getOperation() < 0 || dto.getOperation() > 1)){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }
        //2.查询行为实体
        //获取当前线程的用户，behavior中需要加入过滤器，过滤器将用户对象放到线程中
        ApUser user = AppThreadLocalUtils.getUser();
        ApBehaviorEntry apBehaviorEntry = apBehaviorEntryService.findByUserIdOrEquipmentId(user.getId(), dto.getEquipmentId());
        if(apBehaviorEntry == null){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }

        //3.点赞或取消点赞
        //判断数据是否存在
        ApLikesBehavior apLikesBehavior = getOne(Wrappers.<ApLikesBehavior>lambdaQuery().eq(ApLikesBehavior::getArticleId, dto.getArticleId()).eq(ApLikesBehavior::getEntryId, apBehaviorEntry.getId()));
        //避免用户未点赞，操作：取消点赞
        if(apLikesBehavior == null && dto.getOperation() == 1){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }
        
        if(apLikesBehavior == null && dto.getOperation() == 0){
            //数据不存在，添加一条点赞记录
            apLikesBehavior = new ApLikesBehavior();
            apLikesBehavior.setOperation(dto.getOperation());
            apLikesBehavior.setArticleId(dto.getArticleId());
            apLikesBehavior.setEntryId(apBehaviorEntry.getId());
            apLikesBehavior.setType(dto.getType());
            apLikesBehavior.setCreatedTime(new Date());
            save(apLikesBehavior);
            return ResponseResult.okResult(AppHttpCodeEnum.SUCCESS);
        }else{
            apLikesBehavior.setOperation(dto.getOperation());
            updateById(apLikesBehavior);
            return ResponseResult.okResult(AppHttpCodeEnum.SUCCESS);
        }
    }
}
```



修改：

```java
  if(dto == null || dto.getArticleId() == null || (dto.getType() < 0 || dto.getType() > 2) || (dto.getOperation() < 0 || dto.getOperation() > 1))
```



# 09-阅读行为-需求说明 09:08

```sql
CREATE TABLE `ap_read_behavior` (
  `id` bigint(20) unsigned NOT NULL,
  `entry_id` int(11) unsigned DEFAULT NULL COMMENT '用户ID',
  `article_id` bigint(20) unsigned DEFAULT NULL COMMENT '文章ID',
  `count` tinyint(3) unsigned DEFAULT NULL,
  `read_duration` int(11) unsigned DEFAULT NULL COMMENT '阅读时间单位秒',
  `percentage` tinyint(3) unsigned DEFAULT NULL COMMENT '阅读百分比',
  `load_duration` int(11) unsigned DEFAULT NULL COMMENT '文章加载时间',
  `created_time` datetime DEFAULT NULL COMMENT '登录时间',
  `updated_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC COMMENT='APP阅读行为表'
```

新建阅读行为的api接口

```java
package com.heima.apis.behavior;

import com.heima.model.behavior.dtos.ReadBehaviorDto;
import com.heima.model.common.dtos.ResponseResult;

public interface ApReadBehaviorControllerApi {

    /**
     * 保存或更新阅读行为
     * @return
     */
    public ResponseResult readBehavior(ReadBehaviorDto dto);
}
```

ReadBehaviorDto

```java
package com.heima.model.behavior.dtos;

import com.heima.model.common.annotation.IdEncrypt;
import lombok.Data;

@Data
public class ReadBehaviorDto {
    // 设备ID
    @IdEncrypt
    Integer equipmentId;
    // 文章、动态、评论等ID
    @IdEncrypt
    Long articleId;

    /**
     * 阅读次数
     */
    Short count;

    /**
     * 阅读时长（S)
     */
    Integer readDuration;

    /**
     * 阅读百分比
     */
    Short percentage;

    /**
     * 加载时间
     */
    Short loadDuration;

}
```



# 10-功能完成及测试 10:08

```java
package com.heima.behavior.service.impl;

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.heima.behavior.mapper.ApReadBehaviorMapper;
import com.heima.behavior.service.ApBehaviorEntryService;
import com.heima.behavior.service.ApReadBehaviorService;
import com.heima.model.behavior.dtos.ReadBehaviorDto;
import com.heima.model.behavior.pojos.ApBehaviorEntry;
import com.heima.model.behavior.pojos.ApReadBehavior;
import com.heima.model.common.dtos.ResponseResult;
import com.heima.model.common.enums.AppHttpCodeEnum;
import com.heima.model.user.pojos.ApUser;
import com.heima.utils.threadlocal.AppThreadLocalUtils;
import lombok.extern.log4j.Log4j2;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Date;

@Service
@Log4j2
public class ApReadBehaviorServiceImpl extends ServiceImpl<ApReadBehaviorMapper, ApReadBehavior> implements ApReadBehaviorService {

    @Autowired
    private ApBehaviorEntryService apBehaviorEntryService;

    @Override
    public ResponseResult readBehavior(ReadBehaviorDto dto) {
        //1.参数校验
        if(dto == null || dto.getArticleId() == null){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }
        //2.查询行为实体
        ApUser user = AppThreadLocalUtils.getUser();
        ApBehaviorEntry apBehaviorEntry = apBehaviorEntryService.findByUserIdOrEquipmentId(user.getId(), dto.getEquipmentId());
        if(apBehaviorEntry == null){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }

        //3.保存或更新阅读的行为
        ApReadBehavior apReadBehavior = getOne(Wrappers.<ApReadBehavior>lambdaQuery().eq(ApReadBehavior::getEntryId, apBehaviorEntry.getId()).eq(ApReadBehavior::getArticleId, dto.getArticleId()));
        if(apReadBehavior == null){
            apReadBehavior = new ApReadBehavior();
            apReadBehavior.setCount(dto.getCount());
            apReadBehavior.setArticleId(dto.getArticleId());
            apReadBehavior.setPercentage(dto.getPercentage());
            apReadBehavior.setEntryId(apBehaviorEntry.getId());
            apReadBehavior.setLoadDuration(dto.getLoadDuration());
            apReadBehavior.setReadDuration(dto.getReadDuration());
            apReadBehavior.setCreatedTime(new Date());
            apReadBehavior.setUpdatedTime(new Date());
            save(apReadBehavior);
        }else{
            apReadBehavior.setUpdatedTime(new Date());
            apReadBehavior.setCount((short)(apReadBehavior.getCount()+dto.getCount()));
            //最好把最后一次的数据也设置上
            apReadBehavior.setLoadDuration(dto.getLoadDuration());
            apReadBehavior.setReadDuration(dto.getReadDuration());
            apReadBehavior.setPercentage(dto.getPercentage());
            updateById(apReadBehavior);
        }
        return ResponseResult.okResult(AppHttpCodeEnum.SUCCESS);
    }
}
```



# 11-不喜欢和收藏 行为思路分析 06:46

不喜欢：

1 获取行为实体

2 查询不喜欢对象，有则修改，无则新增

3 固定api接口地址：/api/v1/unlike_behavior POST请求



收藏：

收藏表在文章库中，为什么不设计在行为库？

因为app端用户可以个人中心找到自己收藏的文章列表，这样设计更方便



# 12-文章关系 需求和思路分析 07:10

![1586265034267](data:image/jpg;base64,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)

1 用户查看文章详情，展示文章信息（功能已实现），同时需要展示当前文章的行为（点赞，收藏等）

2 根据用户id(已登录)或设备id(未登录)去查询当前实体id

3 通过实体id和前端传递过来的文章id去查询收藏表、点赞表、不喜欢表；其中点赞和不喜欢需要远程调用behavior微服务获取数据。

4 在文章详情展示是否关注此作者，需要通过当前用户和作者关系表进行查询，有数据则关注，无数据则没有关注

返回的格式如下：

```
{"isfollow":true,"islike":true,"isunlike":false,"iscollection":true}
```

![1599668118109](data:image/jpg;base64,iVBORw0KGgoAAAANSUhEUgAAAxgAAAHYCAYAAADOCk75AAAgAElEQVR4nOzdXWhba373/a+Hm2GmlBtSRYHCnpR6SyISPhD0xC8hDJsiYoXtOCeBGtNMQ7YdDBOL2QmBTqAq5G6fkDwgp9TErknJNLjgEzueJ1YQw2YwsZ2Tgg6MFJbWaHNnBwYiawJlmGw2bfUcLEmWHVnW++vvAyGSLK21JC9L10/X9b+uvkwmk0FERERERKQOvtfqAxARERERke6hgCEiIiIiInWjgCEiIiIiInWjgCEiIiIiInWjgCEiIiIiInWjgCEiIiIiInWjgCEiIiIiInWjgCEiIiIiInXzv1p9ACIi7ehe36mqH3s7866u26739lq5n0bvo5HbP27btWy/lcfdCb/XZu2jWfup1z46+bypx/Y79fk38r2kkv1Uso9yjrlQ1QHj0/XPK37Mb8Z+WddtHre9arZZzX46ZR+N3k+nbrtZ+9D2K9ueiIiIdCYNkRIRERERkbrRECkRkWNU2jXc6dtr5X4avY9O3X4jj7sbfq/N2kez9lOvfXTyeVOP7ff68691H7UM0+rLZDKZah5YOBxCQx1EpByd9L5R+MbarIaLiIhIu6jlc1BDpEREREREpG4UMEREREREpG4UMEREREREpG4UMEREREREpG4UMEREREREpG40Ta2IiIiIiBxQywyKChgi0jTtPjWtiIiI1E5DpEREREREpG6q7sHQN5EiIiIiInKYejBERERERKRuFDBERERERKRuFDBERERERKRuFDBERERERKRuNE2tiIiIiIgccK/vVP5ypWtiKGCISNN8uv55/rJmohMREelOGiIlIiIiIiJ1U3UPhr6JFBERERGRw9SDISIiIiIidaMaDBGRIiotaBMRERGLejBERERERKRuFDBERERERKRuFDBERERERKRuVIMhIiLSRH19fa0+BJGiMplMqw9B2kgttYgKGCIiIg2mUCGdoPA8VdiQWihgiIiINIiChXSq3LmroCHVUMAQkabRopzSK0oFCzXYpF0VO28VNKQafRmdMSIiH7nXdyp/WWtiSCWKNdL0USud5KiArPNYyqVZpEREROpE4UK6QSaTKXreasiflEsBQ0REpA4ON76OaqSJdIpi57BChpRDAUNERKRGxcKFSLdQyJBKqchbRESkjhQupBtlMhkFix5TSy2i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aBUpEdlEujEhIiKy8ykHQ0REREREskYjGCIii7ictzf1WHtiiIiIZE4jGCIiIiIikjUKMEREREREJGsUYIiIiIiISNYoB0NEREREROZYTy6iRjBERERERCRrFGCIiIiIiEjWKMAQEREREZGsUYAhIiIiIiJZowBDRERERESyZs2rSD2u+m026yEiIiIiIjuAlqkVkU2jGxMiIiI7nwIMEZFFrHbNbxEREbEoB0NERERERLJGAYaIiIiIiGSNAgwREREREckaBRgiIiIiIpI1eTMzMzNbXQkREREREdkZ1ryK1MHQx6nHWnpSRERERERAy9SKyCbSjQkREZGdTwGGiMgiLuftTT3WnhgiIiKZU5K3iIiIiIhkjQIMERERERHJGgUYIiIiIiKSNQowREREREQka5TkLSIiIiIic6xnsRONYIiIiIiISNYowBARERERkazJyhSp9M2zVrLS5lqrOddOL2M15a233I36fTarnTa6fTIpYy3nzna9N7q9N/u6FxERkdyjEQwREREREckaBRgiIiIiIpI1eTMzMzNbXQkRke1mPatniIiI5Lr1fA5qmVoRkRWk/5PNxEr/iFd7vs0qZ7ufb7Xnz0YZmZSzWe/nesvbqPbajPbZqDK2y3ubrXK2osyt/jtfz/k3833J5XZaC02REhERERGRrFGAISIiIiIiWaMcDBERERERyRqNYIiIiIiISNYowBARERERkaxRgCEiIiIiIlmjAENERERERLJGAYaIiIiIiGSNAgwREREREckaBRgiIiIiIpI1CjBERERERCRrFGCIiIiIiEjWKMAQEREREZGsUYAhIiIiIiJZowBDRERERESyRgGGiIiIiIhkjQIMERERERHJGgUYIiIiIiKSNQowREREREQkaxRgyM4z/YC7zx/wZKvrISIiIrILKcCQnScf7jzqxv/DDQUZIiKSG6JhzIEwia2uh0gWbP8AY/opd80bfGduxh3pp9w1v+aLH5o4GPqYg+qgZuzJ9NPVt1U23tvpB3xn3uDudPqT73H2TRh/1ck3z9d6YhERkU3kgsnmK9zytinIkJy3vQOM6Qd89/Aspyc7+XJyeBM6+/v5yPics29ueEE7yt2HH3Nk8CxHQl9zN9MXZeu9zYep6R+5Yz4AnlqBzvRTnuTX8N2vLuGe/povHn7N8WTQGGrii+dP11qaiOw00Rhm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XglD7d1E2gP0us9wq/kK93rsKV9mdPHrQ2QbypuZmZnZ6kpsmOTmeNgrF83bLG/L2StFjadyJR7wRWjx/SSyU95WtMfTtNGClZK8V+eueYPHpO3TMa/c7x4qB0NElpaIhkncvkKvnYjt6T+P1+3D4Upb/SllbrIxA34u251na0Ui5hw7m18QJuSttefxW2aXoF2cM9jF4YkrqdWILGkrGK1UN5uZTGymGE/wPF6upF7n6b+/4PjEQBvDzX28TD5RVM3xaxl2nu2k8XhZsZVbkFRWTPmlbxckKS/7O0RjmLev0uu327c+wOELjRiEMVuS7VJDef85jIoCHBm/F3PL9aQnd2NfD6nz29dDhZUAn1wdytN/H687illXy9DIvHa3633PXpLYWvnKhWP+FCe7vs76GvYAzupKHG6XncMBZsNV4hh4M217kW1oZwcYOWD+juC7c4fwNXh+g+OPfs+v32nl9BuLHfCAL37o5hHvE/xAAYaIbIDk5njYG+LN2yxvyyU7/alOdphQ3mJ7PGRHIhrbuo0BM34vwoS8V3hJNceHV9uBj80ZQVjr72q2WnktC/fGsMoYVYAhO4ACDMlBT/nOvM/B/BI+WnJfiwd88XAYNIIhIrvY/B3BtUP4Fhto43pzH55LYUoXXb1qPQGQyPahAENERERkw8UYbb2DwziKseTSu2FCDf2gEQzJcQowREREREQka7b1KlIiIiIiIpJbFGCIiIiIiEjWKMAQEREREZGsUYAhIiIiIiJZowBDRERERESyRgGGiIiIiIhkjQIMERERERHJGgUYIiIiIiKSNQowREREREQkaxRgiIiIiIhI1ijAEBERERGRrPn/k2exeYXO4XkAAAAASUVORK5CYII=)

# 13-行为微服务远程接口 16:09

```java
package com.heima.apis.behavior;

import com.heima.model.behavior.pojos.ApBehaviorEntry;

public interface ApBehaviorEntryControllerApi {

    /**
     * 查询行为实体
     * @param userId
     * @param equipmentId
     * @return
     */
    public ApBehaviorEntry findByUserIdOrEquipmentId(Integer userId, Integer equipmentId);
}
```

点赞：

```java
package com.heima.behavior.controller.v1;

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.heima.apis.behavior.ApLikesBehaviorControllerApi;
import com.heima.behavior.service.ApLikesBehaviorService;
import com.heima.model.behavior.dtos.LikesBehaviorDto;
import com.heima.model.behavior.pojos.ApLikesBehavior;
import com.heima.model.common.dtos.ResponseResult;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/likes_behavior")
public class ApLikesBehaviorController implements ApLikesBehaviorControllerApi {

    @Autowired
    private ApLikesBehaviorService apLikesBehaviorService;

    @PostMapping
    @Override
    public ResponseResult like(@RequestBody LikesBehaviorDto dto) {
        return apLikesBehaviorService.like(dto);
    }

    @GetMapping("/one")
    @Override
    public ApLikesBehavior findLikeByArticleIdAndEntryId(@RequestParam("articleId") Long articleId, @RequestParam("entryId")Integer entryId, @RequestParam("type")Short type) {
        ApLikesBehavior apLikesBehavior = apLikesBehaviorService.getOne(Wrappers.<ApLikesBehavior>lambdaQuery()
                .eq(ApLikesBehavior::getArticleId, articleId).eq(ApLikesBehavior::getEntryId, entryId)
                .eq(ApLikesBehavior::getType, type));
        return apLikesBehavior;
    }
}

```

不喜欢

```java
package com.heima.behavior.controller.v1;

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.heima.apis.behavior.ApUnlikesBehaviorControllerApi;
import com.heima.behavior.service.ApUnlikesBehaviorService;
import com.heima.model.behavior.pojos.ApUnlikesBehavior;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1/un_likes_behavior")
public class ApUnlikesBehaviorController implements ApUnlikesBehaviorControllerApi {

    @Autowired
    private ApUnlikesBehaviorService apUnlikesBehaviorService;

    @GetMapping("/one")
    @Override
    public ApUnlikesBehavior findUnLikeByArticleIdAndEntryId(@RequestParam("entryId") Integer entryId, @RequestParam("articleId") Long articleId) {
        return apUnlikesBehaviorService.getOne(Wrappers.<ApUnlikesBehavior>lambdaQuery().eq(ApUnlikesBehavior::getArticleId, articleId)
                .eq(ApUnlikesBehavior::getEntryId, entryId));
    }
}

```



# 14-行为微服务测试 04:17

启动行为微服务测试，不用经过网关



# 15-用户微服务远程接口 07:27

```java
package com.heima.user.controller.v1;

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.heima.apis.user.ApUserFollowControllerApi;
import com.heima.model.user.pojos.ApUserFollow;
import com.heima.user.service.ApUserFollowService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1/user_follow")
public class ApUserFollowController implements ApUserFollowControllerApi {

    @Autowired
    private ApUserFollowService apUserFollowService;

    @GetMapping("/one")
    @Override
    public ApUserFollow findByUserIdAndFollowId(@RequestParam("userId") Integer userId, @RequestParam("followId") Integer followId) {
        return apUserFollowService.getOne(Wrappers.<ApUserFollow>lambdaQuery()
                .eq(ApUserFollow::getUserId,userId)
                .eq(ApUserFollow::getFollowId,followId));
    }
}

```



# 16-文章微服务feign接口定义 04:56

```java
package com.heima.article.feign;

import com.heima.model.behavior.pojos.ApBehaviorEntry;
import com.heima.model.behavior.pojos.ApLikesBehavior;
import com.heima.model.behavior.pojos.ApUnlikesBehavior;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient("leadnews-behavior")
public interface BehaviorFeign {

    @GetMapping("/api/v1/behavior_entry/one")
    public ApBehaviorEntry findByUserIdOrEquipmentId(@RequestParam("userId") Integer userId, @RequestParam("equipmentId") Integer equipmentId);

    @GetMapping("/api/v1/likes_behavior/one")
    public ApLikesBehavior findLikeByArticleIdAndEntryId(@RequestParam("articleId") Long articleId, @RequestParam("entryId")Integer entryId, @RequestParam("type")Short type);

    @GetMapping("/api/v1/un_likes_behavior/one")
    public ApUnlikesBehavior findUnLikeByArticleIdAndEntryId(@RequestParam("entryId") Integer entryId, @RequestParam("articleId") Long articleId);
}
```

```java
package com.heima.article.feign;

import com.heima.model.user.pojos.ApUserFollow;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient("leadnews-user")
public interface UserFeign {

    @GetMapping("/api/v1/user_follow/one")
    public ApUserFollow findByUserIdAndFollowId(@RequestParam("userId") Integer userId, @RequestParam("followId") Integer followId);
}
```



# 17-接口定义 08:14

```java
@PostMapping("/load_article_behavior")
@Override
public ResponseResult loadArticleBehavior(@RequestBody ArticleInfoDto dto) {
    return articleInfoService.loadArticleBehavior(dto);
}
```

1.不喜欢 true,false  2.点赞  3.收藏 4.关注

```java
package com.heima.model.article.dtos;

import com.heima.model.common.annotation.IdEncrypt;
import lombok.Data;

@Data
public class ArticleInfoDto {
    // 设备ID
    @IdEncrypt
    Integer equipmentId;
    // 文章ID
    @IdEncrypt
    Long articleId;
    // 作者ID
    @IdEncrypt
    Integer authorId;


}
```



# 18-功能实现 15:18

```java
package com.heima.article.service.impl;

import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.heima.article.feign.BehaviorFeign;
import com.heima.article.feign.UserFeign;
import com.heima.article.mapper.ApArticleConfigMapper;
import com.heima.article.mapper.ApArticleContentMapper;

import com.heima.article.mapper.ApCollectionMapper;
import com.heima.article.mapper.AuthorMapper;
import com.heima.article.service.ArticleInfoService;
import com.heima.model.article.dtos.ArticleInfoDto;
import com.heima.model.article.pojos.ApArticleConfig;
import com.heima.model.article.pojos.ApArticleContent;

import com.heima.model.article.pojos.ApAuthor;
import com.heima.model.article.pojos.ApCollection;
import com.heima.model.behavior.pojos.ApBehaviorEntry;
import com.heima.model.behavior.pojos.ApLikesBehavior;
import com.heima.model.behavior.pojos.ApUnlikesBehavior;
import com.heima.model.common.dtos.ResponseResult;
import com.heima.model.common.enums.AppHttpCodeEnum;
import com.heima.model.user.pojos.ApUser;
import com.heima.model.user.pojos.ApUserFollow;
import com.heima.utils.threadlocal.AppThreadLocalUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.Map;

@Service
public class ArticleInfoServiceImpl implements ArticleInfoService {

    @Autowired
    private ApArticleConfigMapper apArticleConfigMapper;

    @Autowired
    private ApArticleContentMapper apArticleContentMapper;

    @Override
    public ResponseResult loadArticleInfo(ArticleInfoDto dto) {

        Map<String, Object> resultMap = new HashMap<>();

        //1.检查参数
        if(dto == null || dto.getArticleId() == null){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }

        //2.查询文章的配置
        ApArticleConfig apArticleConfig = apArticleConfigMapper.selectOne(Wrappers.<ApArticleConfig>lambdaQuery().eq(ApArticleConfig::getArticleId, dto.getArticleId()));
        if(apArticleConfig == null){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }

        //3.查询文章的内容
        if(!apArticleConfig.getIsDelete()&& !apArticleConfig.getIsDown()){
            ApArticleContent apArticleContent = apArticleContentMapper.selectOne(Wrappers.<ApArticleContent>lambdaQuery().eq(ApArticleContent::getArticleId, dto.getArticleId()));
            resultMap.put("content",apArticleContent);
        }
        resultMap.put("config",apArticleConfig);
        //4.结果返回
        return ResponseResult.okResult(resultMap);
    }

    @Autowired
    private BehaviorFeign behaviorFeign;

    @Autowired
    private ApCollectionMapper apCollectionMapper;

    @Autowired
    private UserFeign userFeign;

    @Autowired
    private AuthorMapper authorMapper;

    @Override
    public ResponseResult loadArticleBehavior(ArticleInfoDto dto) {
        //1.检查参数
        if(dto == null || dto.getArticleId() == null){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }
        //2.查询行为实体
        ApUser user = AppThreadLocalUtils.getUser();
        ApBehaviorEntry apBehaviorEntry = behaviorFeign.findByUserIdOrEquipmentId(user.getId(), dto.getEquipmentId());
        if(apBehaviorEntry == null){
            return ResponseResult.errorResult(AppHttpCodeEnum.PARAM_INVALID);
        }

        boolean isUnlike=false,isLike = false,isCollection = false,isFollow = false;

        //3.查询不喜欢行为
        ApUnlikesBehavior apUnlikesBehavior = behaviorFeign.findUnLikeByArticleIdAndEntryId(apBehaviorEntry.getId(), dto.getArticleId());
        if(apUnlikesBehavior != null && apUnlikesBehavior.getType() == ApUnlikesBehavior.Type.UNLIKE.getCode()){
            isUnlike = true;
        }

        //4.查询点赞行为
        ApLikesBehavior apLikesBehavior = behaviorFeign.findLikeByArticleIdAndEntryId(dto.getArticleId(), apBehaviorEntry.getId(), ApLikesBehavior.Type.ARTICLE.getCode());
        if(apLikesBehavior != null && apLikesBehavior.getOperation() == ApLikesBehavior.Operation.LIKE.getCode()){
            isLike = true;
        }

        //5.查询收藏行为
        ApCollection apCollection = apCollectionMapper.selectOne(Wrappers.<ApCollection>lambdaQuery().eq(ApCollection::getEntryId, apBehaviorEntry.getId())
                .eq(ApCollection::getArticleId, dto.getArticleId()).eq(ApCollection::getType, ApCollection.Type.ARTICLE.getCode()));
        if(apCollection != null){
            isCollection = true;
        }

        //6.查询是否关注
        ApAuthor apAuthor = authorMapper.selectById(dto.getAuthorId());
        if(apAuthor != null){
            ApUserFollow apUserFollow = userFeign.findByUserIdAndFollowId(user.getId(), apAuthor.getId());
            if(apUserFollow != null){
                isFollow = true;
            }
        }


        //7.结果返回  {"isfollow":true,"islike":true,"isunlike":false,"iscollection":true}
        Map<String,Object> resultMap = new HashMap<>();
        resultMap.put("isfollow",isFollow);
        resultMap.put("islike",isLike);
        resultMap.put("isunlike",isUnlike);
        resultMap.put("iscollection",isCollection);
        return ResponseResult.okResult(resultMap);
    }
}
```



# 19-测试 06:36