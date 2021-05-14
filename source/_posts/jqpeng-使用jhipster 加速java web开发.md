---
title: 使用jhipster 加速java web开发
tags: ["jqpeng"]
categories: ["博客","jqpeng"]
date: 2021-03-15 16:14
---
文章作者:jqpeng
原文链接: [使用jhipster 加速java web开发](https://www.cnblogs.com/xiaoqi/p/jhipster.html)

jhipster，中文释义： Java 热爱者!


> JHipster is a development platform to quickly generate, develop, & deploy modern web applications & microservice architectures.


JHipster 可以通过代码生成，让你快速开发web应用和微服务。

## 安装

1. 安装[Java](https://adoptopenjdk.net/),[Git](https://git-scm.com/) [Node.js](https://nodejs.org/)
2. 安装 JHipster `npm install -g generator-jhipster`
    - 建议安装最新的[7.0版本](https://github.com/jhipster/generator-jhipster/releases/),
3. 创建应用目录 `mkdir myApp && cd myApp`
4. 运行`jhipster`命令，根据提示设置应用
5. 可以通过[JDL Studio](https://start.jhipster.tech/jdl-studio/) 来生成`jhipster-jdl.jh`文件
6. 然后通过`jhipster jdl jhipster-jdl.jh`来生成代码，`JDL` 后续会重点介绍


## JDL 入门

JDL 是jhipster的数据模型定义文件，通过这个文件我们可以定义数据结构，然后jhipster基于这个JDL，就可以生成实体类、服务类以及前端页面。

例如，我们要开发投诉建议，假如设计的数据表如下：


| 字段 | comment | 类型 | 备注 |
| --- | --- | --- | --- |
| record\_id | 主键 | Bigint | 自增 |
| feedback\_type | 反馈类型 | unsigned tinyint | 枚举值：[1:意见与建议;5:投诉] |
| title | 标题 | varchar(64) |   |
| content | 问题描述 | varchar(512) |   |
| feedback\_status | 反馈状态 | unsigned tinyint | 枚举值：[1:待提交;5:待回复;10:待确认;15:已解决;] |
| last\_reply\_time | 最后回复时间 | timestamp | 与feedback\_status联合使用，当状态为2的时候，更新此时间，用于超时判断 |
| close\_type | 关闭类型 | unsigned tinyint | 枚举值：[1:正常关闭;5:超时关闭;] |
| created\_date | 创建时间 | timestamp |   |
| created\_by | 创建者 | char(32) |


使用`jhipster`,我们可以用`jdl`来定义：


    /**
     * 反馈记录表
     */
    entity FeedbackRecord {
        /** 反馈类型*/
        feedbackType FeedbackType,
        /** 问题描述 */
        title String,
        /** 反馈状态     */
        feedbackStatus FeedbackStatus,
         /** 是否已完成 */
        lastReplyTime Integer,
         /** 关闭类型     */
        closeType FeedbackCloseType,
         /** 创建时间 */
        createdDate Instant,
        /**     创建者 */
        createdBy String
    }
    /** 反馈类型 */
    enum FeedbackType {
        ADVICE,
        COMPLAINTS
    }
    /** 反馈状态 */
    enum FeedbackStatus {
        TO_BE_SUBMIT, TO_BE_REPLY, TO_BE_CONFIRMED
    }
    /** 关闭类型 */
    enum FeedbackCloseType {
        NORMALLY, TIMEOUT
    }
    
    dto * with mapstruct
    service all with serviceImpl
    paginate all with pagination


详细讲解：

### 实体和字段

entity 表示一个实体，可以增加字段，注意，不用增加id

语法是：


    [<entity javadoc>]
    [<entity annotation>*]
    entity <entity name> [(<table name>)] {
      [<field javadoc>]
      [<field annotation>*]
      <field name> <field type> [<validation>*]
    }


例如：


    entity A {
      name String required
      age Integer min(42) max(42)
    }


可以增加`required`、`min`、`max`等验证

字段的注释:


    /**
     * This is a comment
     * about a class
     * @author Someone
     */
    entity A {
      /** 名称 */
       name String
       age Integer // this is yet another comment
    }


JHipster支持许多字段类型。这种支持取决于您的数据库后端，因此我们使用Java类型来描述它们：Java`String`将以不同的方式存储在Oracle或Cassandra中，这是JHipster的优势之一，可以为您生成正确的数据库访问代码。

- `String`: Java字符串。它的默认大小取决于基础后端（如果使用JPA，默认情况下为255），但是您可以使用校验规则进行更改（例如，修改 `max`大小为1024）。
- `Integer`: Java整数。
- `Long`: Java长整数。
- `Float`: Java浮点数.
- `Double`: Java双精度浮点数.
- `BigDecimal`: [java.math.BigDecimal](https://docs.oracle.com/javase/8/docs/api/java/math/BigDecimal.html)对象, 当您需要精确的数学计算时使用（通常用于财务操作）。
- `LocalDate`: [java.time.LocalDate](https://docs.oracle.com/javase/8/docs/api/java/time/LocalDate.html)对象, 用于正确管理Java中的日期。
- `Instant`: [java.time.Instant](https://docs.oracle.com/javase/8/docs/api/java/time/Instant.html)对象, 用于表示时间戳，即时间线上的瞬时点。
- `ZonedDateTime`: [java.time.ZonedDateTime](https://docs.oracle.com/javase/8/docs/api/java/time/ZonedDateTime.html)对象, 用于表示给定时区（通常是日历中会议、约定）中的本地日期时间。请注意，REST和持久层都不支持时区，因此您很可能应该使用`Instant`。
- `Duration`: [java.time.Duration](https://docs.oracle.com/javase/8/docs/api/java/time/Duration.html)对象, 用于表示时间量。
- `UUID`: [java.util.UUID](https://docs.oracle.com/javase/8/docs/api/java/util/UUID.html)对象.
- `Boolean`: Java布尔型.
- `Enumeration`:Java枚举对象。选择此类型后，子生成器将询问您要在枚举中使用哪些值，并将创建一个特定的`enum`类来存储它们。
- `Blob`: Blob对象，用于存储一些二进制数据。选择此类型时，子生成器将询问您是否要存储通用二进制数据，图像对象或CLOB（长文本）。图像将专门在Angular侧进行优化处理，因此可以将其正常显示给最终用户。


字段的数据类型及数据库支持：

![数据类型](http://ainotedoc.iflyresearch.com/images/b8149b95c85096326712cb9228e7245ad86e778ae01e53f8cf59c3541aac229f.png)

### 枚举

对于可枚举的状态，建议采用枚举值：


    enum [<enum name>] {
      <ENUM KEY> ([<enum value>])
    }


例如：


    /** 反馈类型 */
    enum FeedbackType {
        ADVICE,
        COMPLAINTS
    }


### 关系

SQL数据库支持表和表的关联：

- `OneToOne`
- `OneToMany`
- `ManyToOne`
- `ManyToMany`


如何定义关系呢？


    relationship (OneToMany | ManyToOne | OneToOne | ManyToMany) {
      <from entity>[{<relationship name>[(<display field>)]}] to <to entity>[{<relationship name>[(<display field>)]}]+
    }


例如, 下面的例子里，我们定义两个对象，`File`和`Chunk`，1个`Chunk`属于一个`File`：


    /**
     * 文件
     */
    entity File {
        /** 文件名 */
        name String,
        /** 文件大小 */
        size Long,
        /** 文件路径 */
        path String,
        /** 分片数 */
        chunks Integer,
         /** 是否已完成 */
        complete Integer
    }
    
    /**
     * 文件分片
     */
    entity Chunk {
        /** md5值 */
        md5 String,
        /** 分片序号 */
        number Integer,
        /** 分片名称 */
        name String
    }
    
    relationship ManyToOne {
        /** 所属文件 */
        Chunk{file} to File
    }


对应的关系图：

![关系图](http://ainotedoc.iflyresearch.com/images/5678aa9b0f1462f73e224f3a075fc3ebf7ebcb54f39f2fab44498e4cba7f0330.png)

### 生成代码配置

JHipster提供了丰富的配置，可以用来指定生成代码时的策略，例如是否要生成`DTO对象`，是否需要支持分页，是否需要生成service类，如果生成service，是使用`serviceClass`还是`serviceImpl`。

示例如下：


    entity A {
      name String required
    }
    entity B
    entity C
    
    // 筛选实体
    filter *
    
    // 生成dto
    dto A, B with mapstruct
    
    // 分页
    paginate A with infinite-scroll
    paginate B with pagination
    paginate C with pager  // pager is only available in AngularJS
    
    // 生成service
    service A with serviceClass
    service C with serviceImpl


## 生成代码

首先定义`jdl`文件：


    /**
     * 反馈记录表
     */
    entity FeedbackRecord {
        /** 反馈类型*/
        feedbackType FeedbackType,
        /** 问题描述 */
        title String,
        /** 反馈状态     */
        feedbackStatus FeedbackStatus,
         /** 是否已完成 */
        lastReplyTime Integer,
         /** 关闭类型     */
        closeType FeedbackCloseType,
         /** 创建时间 */
        createdDate Instant,
        /**     创建者 */
        createdBy String
    }
    /** 反馈类型 */
    enum FeedbackType {
        ADVICE,
        COMPLAINTS
    }
    /** 反馈状态 */
    enum FeedbackStatus {
        TO_BE_SUBMIT, TO_BE_REPLY, TO_BE_CONFIRMED
    }
    /** 关闭类型 */
    enum FeedbackCloseType {
        NORMALLY, TIMEOUT
    }
    // 筛选实体
    filter *
    // 生成DTO
    dto * with mapstruct
    // 生成带接口和实现的service
    service all with serviceImpl
    // 支持分页
    paginate all with pagination


然后生成代码：


    jhipster jdl feedback.jh --force


可以看到类似下面的输出


    D:\Project\jhipster-7>jhipster jdl feedback.jh --force
    INFO! Using JHipster version installed locally in current project's node_modules
    INFO! Executing import-jdl feedback.jh
    INFO! The JDL is being parsed.
    info: The dto option is set for FeedbackRecord, the 'serviceClass' value for the 'service' is gonna be set for this entity if no other value has been set.
    INFO! Found entities: FeedbackRecord.
    INFO! The JDL has been successfully parsed
    INFO! Generating 0 applications.
    INFO! Generating 1 entity.
    INFO! Generating entities for application undefined in a new parallel process
    
    Found the D:\Project\jhipster-7\.jhipster\File.json configuration file, entity can be automatically generated!
    
    
    Found the D:\Project\jhipster-7\.jhipster\Chunk.json configuration file, entity can be automatically generated!
    
    
    Found the D:\Project\jhipster-7\.jhipster\FeedbackRecord.json configuration file, entity can be automatically generated!
    
         info Creating changelog for entities File,Chunk,FeedbackRecord
        force .yo-rc.json
        force .jhipster\FeedbackRecord.json
        force .jhipster\File.json
        force .jhipster\Chunk.json
        force src\main\java\com\company\datahub\domain\File.java
        force src\main\java\com\company\datahub\web\rest\FileResource.java
        force src\main\java\com\company\datahub\repository\FileRepository.java
        force src\main\java\com\company\datahub\service\FileService.java
        force src\main\java\com\company\datahub\service\impl\FileServiceImpl.java
        force src\main\java\com\company\datahub\service\dto\FileDTO.java
        force src\main\java\com\company\datahub\service\mapper\EntityMapper.java
        force src\main\java\com\company\datahub\service\mapper\FileMapper.java
        force src\test\java\com\company\datahub\web\rest\FileResourceIT.java
        force src\test\java\com\company\datahub\domain\FileTest.java
        force src\test\java\com\company\datahub\service\dto\FileDTOTest.java
        force src\test\java\com\company\datahub\service\mapper\FileMapperTest.java
        force src\main\webapp\app\shared\model\file.model.ts
        force src\main\webapp\app\entities\file\file-details.vue
        force src\main\webapp\app\entities\file\file-details.component.ts
        force src\main\webapp\app\entities\file\file.vue
        force src\main\webapp\app\entities\file\file.component.ts
        force src\main\webapp\app\entities\file\file.service.ts
        force src\main\webapp\app\entities\file\file-update.vue
        force src\main\webapp\app\entities\file\file-update.component.ts
        force src\test\javascript\spec\app\entities\file\file.component.spec.ts
        force src\test\javascript\spec\app\entities\file\file-details.component.spec.ts
        force src\test\javascript\spec\app\entities\file\file.service.spec.ts
        force src\test\javascript\spec\app\entities\file\file-update.component.spec.ts
        force src\main\webapp\app\router\entities.ts
        force src\main\webapp\app\main.ts
        force src\main\webapp\app\core\jhi-navbar\jhi-navbar.vue
        force src\main\webapp\i18n\zh-cn\file.json
        force src\main\webapp\i18n\zh-cn\global.json
        force src\main\webapp\i18n\en\file.json
        force src\main\webapp\i18n\en\global.json
        force src\main\java\com\company\datahub\domain\Chunk.java
        force src\main\java\com\company\datahub\web\rest\ChunkResource.java
        force src\main\java\com\company\datahub\repository\ChunkRepository.java
        force src\main\java\com\company\datahub\service\ChunkService.java
        force src\main\java\com\company\datahub\service\impl\ChunkServiceImpl.java
        force src\main\java\com\company\datahub\service\dto\ChunkDTO.java
        force src\main\java\com\company\datahub\service\mapper\ChunkMapper.java
        force src\test\java\com\company\datahub\web\rest\ChunkResourceIT.java
        force src\test\java\com\company\datahub\domain\ChunkTest.java
        force src\test\java\com\company\datahub\service\dto\ChunkDTOTest.java
        force src\test\java\com\company\datahub\service\mapper\ChunkMapperTest.java
        force src\main\webapp\app\shared\model\chunk.model.ts
        force src\main\webapp\app\entities\chunk\chunk-details.vue
        force src\main\webapp\app\entities\chunk\chunk-details.component.ts
        force src\main\webapp\app\entities\chunk\chunk.vue
        force src\main\webapp\app\entities\chunk\chunk.component.ts
        force src\main\webapp\app\entities\chunk\chunk.service.ts
        force src\main\webapp\app\entities\chunk\chunk-update.vue
        force src\main\webapp\app\entities\chunk\chunk-update.component.ts
        force src\test\javascript\spec\app\entities\chunk\chunk.component.spec.ts
        force src\test\javascript\spec\app\entities\chunk\chunk-details.component.spec.ts
        force src\test\javascript\spec\app\entities\chunk\chunk.service.spec.ts
        force src\test\javascript\spec\app\entities\chunk\chunk-update.component.spec.ts
        force src\main\webapp\i18n\zh-cn\chunk.json
        force src\main\webapp\i18n\en\chunk.json
       create src\main\java\com\company\datahub\domain\FeedbackRecord.java
       create src\main\java\com\company\datahub\web\rest\FeedbackRecordResource.java
       create src\main\java\com\company\datahub\repository\FeedbackRecordRepository.java
       create src\main\java\com\company\datahub\service\FeedbackRecordService.java
       create src\main\java\com\company\datahub\service\impl\FeedbackRecordServiceImpl.java
       create src\main\java\com\company\datahub\service\dto\FeedbackRecordDTO.java
       create src\main\java\com\company\datahub\service\mapper\FeedbackRecordMapper.java
       create src\test\java\com\company\datahub\web\rest\FeedbackRecordResourceIT.java
       create src\test\java\com\company\datahub\domain\FeedbackRecordTest.java
       create src\test\java\com\company\datahub\service\dto\FeedbackRecordDTOTest.java
       create src\test\java\com\company\datahub\service\mapper\FeedbackRecordMapperTest.java
       create src\main\java\com\company\datahub\domain\enumeration\FeedbackType.java
       create src\main\java\com\company\datahub\domain\enumeration\FeedbackStatus.java
       create src\main\java\com\company\datahub\domain\enumeration\FeedbackCloseType.java
       create src\main\webapp\app\shared\model\feedback-record.model.ts
       create src\main\webapp\app\entities\feedback-record\feedback-record-details.vue
       create src\main\webapp\app\entities\feedback-record\feedback-record-details.component.ts
       create src\main\webapp\app\entities\feedback-record\feedback-record.vue
       create src\main\webapp\app\entities\feedback-record\feedback-record.component.ts
       create src\main\webapp\app\entities\feedback-record\feedback-record.service.ts
        force src\main\resources\config\liquibase\changelog\20210312045459_added_entity_File.xml
        force src\main\resources\config\liquibase\fake-data\file.csv
       create src\main\webapp\app\entities\feedback-record\feedback-record-update.vue
        force src\main\resources\config\liquibase\master.xml
        force src\main\resources\config\liquibase\changelog\20210312045500_added_entity_Chunk.xml
        force src\main\resources\config\liquibase\changelog\20210312045500_added_entity_constraints_Chunk.xml
        force src\main\resources\config\liquibase\fake-data\chunk.csv
       create src\main\resources\config\liquibase\changelog\20210312072243_added_entity_FeedbackRecord.xml
       create src\main\resources\config\liquibase\fake-data\feedback_record.csv
       create src\main\webapp\app\entities\feedback-record\feedback-record-update.component.ts
       create src\test\javascript\spec\app\entities\feedback-record\feedback-record.component.spec.ts
       create src\test\javascript\spec\app\entities\feedback-record\feedback-record-details.component.spec.ts
       create src\test\javascript\spec\app\entities\feedback-record\feedback-record.service.spec.ts
       create src\test\javascript\spec\app\entities\feedback-record\feedback-record-update.component.spec.ts
       create src\main\webapp\app\shared\model\enumerations\feedback-type.model.ts
       create src\main\webapp\app\shared\model\enumerations\feedback-status.model.ts
       create src\main\webapp\app\shared\model\enumerations\feedback-close-type.model.ts
       create src\main\webapp\i18n\zh-cn\feedbackType.json
       create src\main\webapp\i18n\en\feedbackType.json
       create src\main\webapp\i18n\zh-cn\feedbackStatus.json
       create src\main\webapp\i18n\en\feedbackStatus.json
       create src\main\webapp\i18n\zh-cn\feedbackCloseType.json
       create src\main\webapp\i18n\en\feedbackCloseType.json
       create src\main\webapp\i18n\zh-cn\feedbackRecord.json
       create src\main\webapp\i18n\en\feedbackRecord.json
    Entity File generated successfully.
    Entity Chunk generated successfully.
    Entity FeedbackRecord generated successfully.
    
    Running `webapp:build` to update client app


包含domain、service、controller等都有生成：

## 测试生成程序

运行程序，

列表页面：

![列表页面](http://ainotedoc.iflyresearch.com/images/dfb9592f5a57bfd645ff822113534f78480cb8d52cd59b915f6dd8d017f22a46.png)

编辑页面：

![编辑页面](http://ainotedoc.iflyresearch.com/images/0dc5aa933e3186e0b3daf6c44637601204cecd894a66acfc38142fdda7713243.png)

## 生成代码介绍

### Domain


    
    package com.company.datahub.domain;
    
    import com.company.datahub.domain.enumeration.FeedbackCloseType;
    import com.company.datahub.domain.enumeration.FeedbackStatus;
    import com.company.datahub.domain.enumeration.FeedbackType;
    import java.io.Serializable;
    import java.time.Instant;
    import javax.persistence.*;
    
    /**
     * 反馈记录表
     */
    @Entity
    @Table(name = "feedback_record")
    public class FeedbackRecord implements Serializable {
    
        private static final long serialVersionUID = 1L;
    
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;
    
        /**
         * 反馈类型
         */
        @Enumerated(EnumType.STRING)
        @Column(name = "feedback_type")
        private FeedbackType feedbackType;
    
        /**
         * 问题描述
         */
        @Column(name = "title")
        private String title;
    
        /**
         * 反馈状态
         */
        @Enumerated(EnumType.STRING)
        @Column(name = "feedback_status")
        private FeedbackStatus feedbackStatus;
    
        /**
         * 是否已完成
         */
        @Column(name = "last_reply_time")
        private Integer lastReplyTime;
    
        /**
         * 关闭类型
         */
        @Enumerated(EnumType.STRING)
        @Column(name = "close_type")
        private FeedbackCloseType closeType;
    
        /**
         * 创建时间
         */
        @Column(name = "created_date")
        private Instant createdDate;
    
        /**
         * 创建者
         */
        @Column(name = "created_by")
        private String createdBy;
    
        // jhipster-needle-entity-add-field - JHipster will add fields here
        public Long getId() {
            return id;
        }
    
        public void setId(Long id) {
            this.id = id;
        }
    
        public FeedbackRecord id(Long id) {
            this.id = id;
            return this;
        }
    
        public FeedbackType getFeedbackType() {
            return this.feedbackType;
        }
    
        public FeedbackRecord feedbackType(FeedbackType feedbackType) {
            this.feedbackType = feedbackType;
            return this;
        }
    
        public void setFeedbackType(FeedbackType feedbackType) {
            this.feedbackType = feedbackType;
        }
    
        public String getTitle() {
            return this.title;
        }
    
        public FeedbackRecord title(String title) {
            this.title = title;
            return this;
        }
    
        public void setTitle(String title) {
            this.title = title;
        }
    
        public FeedbackStatus getFeedbackStatus() {
            return this.feedbackStatus;
        }
    
        public FeedbackRecord feedbackStatus(FeedbackStatus feedbackStatus) {
            this.feedbackStatus = feedbackStatus;
            return this;
        }
    
        public void setFeedbackStatus(FeedbackStatus feedbackStatus) {
            this.feedbackStatus = feedbackStatus;
        }
    
        public Integer getLastReplyTime() {
            return this.lastReplyTime;
        }
    
        public FeedbackRecord lastReplyTime(Integer lastReplyTime) {
            this.lastReplyTime = lastReplyTime;
            return this;
        }
    
        public void setLastReplyTime(Integer lastReplyTime) {
            this.lastReplyTime = lastReplyTime;
        }
    
        public FeedbackCloseType getCloseType() {
            return this.closeType;
        }
    
        public FeedbackRecord closeType(FeedbackCloseType closeType) {
            this.closeType = closeType;
            return this;
        }
    
        public void setCloseType(FeedbackCloseType closeType) {
            this.closeType = closeType;
        }
    
        public Instant getCreatedDate() {
            return this.createdDate;
        }
    
        public FeedbackRecord createdDate(Instant createdDate) {
            this.createdDate = createdDate;
            return this;
        }
    
        public void setCreatedDate(Instant createdDate) {
            this.createdDate = createdDate;
        }
    
        public String getCreatedBy() {
            return this.createdBy;
        }
    
        public FeedbackRecord createdBy(String createdBy) {
            this.createdBy = createdBy;
            return this;
        }
    
        public void setCreatedBy(String createdBy) {
            this.createdBy = createdBy;
        }
    
        // jhipster-needle-entity-add-getters-setters - JHipster will add getters and setters here
    
        @Override
        public boolean equals(Object o) {
            if (this == o) {
                return true;
            }
            if (!(o instanceof FeedbackRecord)) {
                return false;
            }
            return id != null && id.equals(((FeedbackRecord) o).id);
        }
    
        @Override
        public int hashCode() {
            // see https://vladmihalcea.com/how-to-implement-equals-and-hashcode-using-the-jpa-entity-identifier/
            return getClass().hashCode();
        }
    
        // prettier-ignore
        @Override
        public String toString() {
            return "FeedbackRecord{" +
                "id=" + getId() +
                ", feedbackType='" + getFeedbackType() + "'" +
                ", title='" + getTitle() + "'" +
                ", feedbackStatus='" + getFeedbackStatus() + "'" +
                ", lastReplyTime=" + getLastReplyTime() +
                ", closeType='" + getCloseType() + "'" +
                ", createdDate='" + getCreatedDate() + "'" +
                ", createdBy='" + getCreatedBy() + "'" +
                "}";
        }
    }


### Repository


    @SuppressWarnings("unused")
    @Repository
    public interface FeedbackRecordRepository extends JpaRepository<FeedbackRecord, Long>, JpaSpecificationExecutor<FeedbackRecord> {}


### Service


    
    /**
     * Service Interface for managing {@link com.company.datahub.domain.FeedbackRecord}.
     */
    public interface FeedbackRecordService {
        /**
         * Save a feedbackRecord.
         *
         * @param feedbackRecordDTO the entity to save.
         * @return the persisted entity.
         */
        FeedbackRecordDTO save(FeedbackRecordDTO feedbackRecordDTO);
    
        /**
         * Partially updates a feedbackRecord.
         *
         * @param feedbackRecordDTO the entity to update partially.
         * @return the persisted entity.
         */
        Optional<FeedbackRecordDTO> partialUpdate(FeedbackRecordDTO feedbackRecordDTO);
    
        /**
         * Get all the feedbackRecords.
         *
         * @param pageable the pagination information.
         * @return the list of entities.
         */
        Page<FeedbackRecordDTO> findAll(Pageable pageable);
    
        /**
         * Get the "id" feedbackRecord.
         *
         * @param id the id of the entity.
         * @return the entity.
         */
        Optional<FeedbackRecordDTO> findOne(Long id);
    
        /**
         * Delete the "id" feedbackRecord.
         *
         * @param id the id of the entity.
         */
        void delete(Long id);
    }


### Controller


    /**
     * REST controller for managing {@link com.company.datahub.domain.FeedbackRecord}.
     */
    @RestController
    @RequestMapping("/api")
    public class FeedbackRecordResource {
    
        private final Logger log = LoggerFactory.getLogger(FeedbackRecordResource.class);
    
        private static final String ENTITY_NAME = "feedbackRecord";
    
        @Value("${jhipster.clientApp.name}")
        private String applicationName;
    
        private final FeedbackRecordService feedbackRecordService;
    
        private final FeedbackRecordQueryService feedbackRecordQueryService;
    
        public FeedbackRecordResource(FeedbackRecordService feedbackRecordService, FeedbackRecordQueryService feedbackRecordQueryService) {
            this.feedbackRecordService = feedbackRecordService;
            this.feedbackRecordQueryService = feedbackRecordQueryService;
        }
    
        /**
         * {@code POST  /feedback-records} : Create a new feedbackRecord.
         *
         * @param feedbackRecordDTO the feedbackRecordDTO to create.
         * @return the {@link ResponseEntity} with status {@code 201 (Created)} and with body the new feedbackRecordDTO, or with status {@code 400 (Bad Request)} if the feedbackRecord has already an ID.
         * @throws URISyntaxException if the Location URI syntax is incorrect.
         */
        @PostMapping("/feedback-records")
        public ResponseEntity<FeedbackRecordDTO> createFeedbackRecord(@RequestBody FeedbackRecordDTO feedbackRecordDTO)
            throws URISyntaxException {
            log.debug("REST request to save FeedbackRecord : {}", feedbackRecordDTO);
            if (feedbackRecordDTO.getId() != null) {
                throw new BadRequestAlertException("A new feedbackRecord cannot already have an ID", ENTITY_NAME, "idexists");
            }
            FeedbackRecordDTO result = feedbackRecordService.save(feedbackRecordDTO);
            return ResponseEntity
                .created(new URI("/api/feedback-records/" + result.getId()))
                .headers(HeaderUtil.createEntityCreationAlert(applicationName, true, ENTITY_NAME, result.getId().toString()))
                .body(result);
        }
    
        /**
         * {@code PUT  /feedback-records} : Updates an existing feedbackRecord.
         *
         * @param feedbackRecordDTO the feedbackRecordDTO to update.
         * @return the {@link ResponseEntity} with status {@code 200 (OK)} and with body the updated feedbackRecordDTO,
         * or with status {@code 400 (Bad Request)} if the feedbackRecordDTO is not valid,
         * or with status {@code 500 (Internal Server Error)} if the feedbackRecordDTO couldn't be updated.
         * @throws URISyntaxException if the Location URI syntax is incorrect.
         */
        @PutMapping("/feedback-records")
        public ResponseEntity<FeedbackRecordDTO> updateFeedbackRecord(@RequestBody FeedbackRecordDTO feedbackRecordDTO)
            throws URISyntaxException {
            log.debug("REST request to update FeedbackRecord : {}", feedbackRecordDTO);
            if (feedbackRecordDTO.getId() == null) {
                throw new BadRequestAlertException("Invalid id", ENTITY_NAME, "idnull");
            }
            FeedbackRecordDTO result = feedbackRecordService.save(feedbackRecordDTO);
            return ResponseEntity
                .ok()
                .headers(HeaderUtil.createEntityUpdateAlert(applicationName, true, ENTITY_NAME, feedbackRecordDTO.getId().toString()))
                .body(result);
        }
    
        /**
         * {@code PATCH  /feedback-records} : Updates given fields of an existing feedbackRecord.
         *
         * @param feedbackRecordDTO the feedbackRecordDTO to update.
         * @return the {@link ResponseEntity} with status {@code 200 (OK)} and with body the updated feedbackRecordDTO,
         * or with status {@code 400 (Bad Request)} if the feedbackRecordDTO is not valid,
         * or with status {@code 404 (Not Found)} if the feedbackRecordDTO is not found,
         * or with status {@code 500 (Internal Server Error)} if the feedbackRecordDTO couldn't be updated.
         * @throws URISyntaxException if the Location URI syntax is incorrect.
         */
        @PatchMapping(value = "/feedback-records", consumes = "application/merge-patch+json")
        public ResponseEntity<FeedbackRecordDTO> partialUpdateFeedbackRecord(@RequestBody FeedbackRecordDTO feedbackRecordDTO)
            throws URISyntaxException {
            log.debug("REST request to update FeedbackRecord partially : {}", feedbackRecordDTO);
            if (feedbackRecordDTO.getId() == null) {
                throw new BadRequestAlertException("Invalid id", ENTITY_NAME, "idnull");
            }
    
            Optional<FeedbackRecordDTO> result = feedbackRecordService.partialUpdate(feedbackRecordDTO);
    
            return ResponseUtil.wrapOrNotFound(
                result,
                HeaderUtil.createEntityUpdateAlert(applicationName, true, ENTITY_NAME, feedbackRecordDTO.getId().toString())
            );
        }
    
        /**
         * {@code GET  /feedback-records} : get all the feedbackRecords.
         *
         * @param pageable the pagination information.
         * @param criteria the criteria which the requested entities should match.
         * @return the {@link ResponseEntity} with status {@code 200 (OK)} and the list of feedbackRecords in body.
         */
        @GetMapping("/feedback-records")
        public ResponseEntity<List<FeedbackRecordDTO>> getAllFeedbackRecords(FeedbackRecordCriteria criteria, Pageable pageable) {
            log.debug("REST request to get FeedbackRecords by criteria: {}", criteria);
            Page<FeedbackRecordDTO> page = feedbackRecordQueryService.findByCriteria(criteria, pageable);
            HttpHeaders headers = PaginationUtil.generatePaginationHttpHeaders(ServletUriComponentsBuilder.fromCurrentRequest(), page);
            return ResponseEntity.ok().headers(headers).body(page.getContent());
        }
    
        /**
         * {@code GET  /feedback-records/count} : count all the feedbackRecords.
         *
         * @param criteria the criteria which the requested entities should match.
         * @return the {@link ResponseEntity} with status {@code 200 (OK)} and the count in body.
         */
        @GetMapping("/feedback-records/count")
        public ResponseEntity<Long> countFeedbackRecords(FeedbackRecordCriteria criteria) {
            log.debug("REST request to count FeedbackRecords by criteria: {}", criteria);
            return ResponseEntity.ok().body(feedbackRecordQueryService.countByCriteria(criteria));
        }
    
        /**
         * {@code GET  /feedback-records/:id} : get the "id" feedbackRecord.
         *
         * @param id the id of the feedbackRecordDTO to retrieve.
         * @return the {@link ResponseEntity} with status {@code 200 (OK)} and with body the feedbackRecordDTO, or with status {@code 404 (Not Found)}.
         */
        @GetMapping("/feedback-records/{id}")
        public ResponseEntity<FeedbackRecordDTO> getFeedbackRecord(@PathVariable Long id) {
            log.debug("REST request to get FeedbackRecord : {}", id);
            Optional<FeedbackRecordDTO> feedbackRecordDTO = feedbackRecordService.findOne(id);
            return ResponseUtil.wrapOrNotFound(feedbackRecordDTO);
        }
    
        /**
         * {@code DELETE  /feedback-records/:id} : delete the "id" feedbackRecord.
         *
         * @param id the id of the feedbackRecordDTO to delete.
         * @return the {@link ResponseEntity} with status {@code 204 (NO_CONTENT)}.
         */
        @DeleteMapping("/feedback-records/{id}")
        public ResponseEntity<Void> deleteFeedbackRecord(@PathVariable Long id) {
            log.debug("REST request to delete FeedbackRecord : {}", id);
            feedbackRecordService.delete(id);
            return ResponseEntity
                .noContent()
                .headers(HeaderUtil.createEntityDeletionAlert(applicationName, true, ENTITY_NAME, id.toString()))
                .build();
        }
    }


### 测试接口

获取数据


        @GetMapping("/feedback-records")
        public ResponseEntity<List<FeedbackRecordDTO>> getAllFeedbackRecords(FeedbackRecordCriteria criteria, Pageable pageable) {
            log.debug("REST request to get FeedbackRecords by criteria: {}", criteria);
            Page<FeedbackRecordDTO> page = feedbackRecordQueryService.findByCriteria(criteria, pageable);
            HttpHeaders headers = PaginationUtil.generatePaginationHttpHeaders(ServletUriComponentsBuilder.fromCurrentRequest(), page);
            return ResponseEntity.ok().headers(headers).body(page.getContent());
        }


这里有意思的是`FeedbackRecordCriteria`，可以针对实体中的每个字段进行过滤，不用单独写业务代码去过滤：

比如`feedbackStatus`是一个枚举，那么可以使用`equals`，`in` 等过过滤器。

![feedbackStatus](http://ainotedoc.iflyresearch.com/images/1d9808d0eda2d85271774a65f23b896fccad85c64dd2809615ced2118f3cad4b.png)

关于过滤器：

![过滤器](http://ainotedoc.iflyresearch.com/images/d8bb64214258bdc59187a085495f9a07e5b94ae8de47490d23c8969a4a3fb43f.png)

我们测试下，比如查询`feedbackStatus` 为 `TO_BE_REPLY`的，那么可以使用`feedbackStatus.equals=TO_BE_REPLY`


    GET http://localhost:8080/api/feedback-records?sort=id,asc&page=0&size=20&feedbackStatus.equals=TO_BE_REPLY



    
    [
        {
            "id": 1,
            "feedbackType": "COMPLAINTS",
            "title": "SMTP lavender Table",
            "feedbackStatus": "TO_BE_REPLY",
            "lastReplyTime": 9391,
            "closeType": "NORMALLY",
            "createdDate": "2021-03-11T21:38:31Z",
            "createdBy": "新疆 Central Soft"
        },
        {
            "id": 2,
            "feedbackType": "ADVICE",
            "title": "上海市 haptic",
            "feedbackStatus": "TO_BE_REPLY",
            "lastReplyTime": 53521,
            "closeType": "NORMALLY",
            "createdDate": "2021-03-11T18:04:14Z",
            "createdBy": "Rubber connect 桥"
        },
        {
            "id": 4,
            "feedbackType": "ADVICE",
            "title": "Senior index",
            "feedbackStatus": "TO_BE_REPLY",
            "lastReplyTime": 67874,
            "closeType": "TIMEOUT",
            "createdDate": "2021-03-11T14:53:15Z",
            "createdBy": "Uganda"
        },
        {
            "id": 6,
            "feedbackType": "ADVICE",
            "title": "Expanded Sports compelling",
            "feedbackStatus": "TO_BE_REPLY",
            "lastReplyTime": 8032,
            "closeType": "TIMEOUT",
            "createdDate": "2021-03-12T03:53:46Z",
            "createdBy": "deposit Chicken mesh"
        },
        {
            "id": 7,
            "feedbackType": "ADVICE",
            "title": "Division overriding",
            "feedbackStatus": "TO_BE_REPLY",
            "lastReplyTime": 38000,
            "closeType": "NORMALLY",
            "createdDate": "2021-03-11T07:57:51Z",
            "createdBy": "Account stable"
        },
        {
            "id": 9,
            "feedbackType": "ADVICE",
            "title": "Loan",
            "feedbackStatus": "TO_BE_REPLY",
            "lastReplyTime": 99908,
            "closeType": "TIMEOUT",
            "createdDate": "2021-03-11T09:47:44Z",
            "createdBy": "Re-engineered"
        }
    ]


## JHipster 管理界面介绍

JHipster 自动生成的前端代码里，包含了一些管理界面：

![管理界面](http://ainotedoc.iflyresearch.com/images/1ae0597aba92bc958635fce65666676237a007614ad17227a6bb84a46af2f3ef.png)

资源监控：

![资源监控](http://ainotedoc.iflyresearch.com/images/cd050dc19d39a0a3171ea4e8d883e921a76b0bb768cd51c2535f0124dcbcff26.png)

## 开发实践

### 更新实体增加字段

在项目的工程目录下，有一个`.jhipster`文件夹，里面包含了已有的实体。

![已有实体](http://ainotedoc.iflyresearch.com/images/ad9cdc11084c56f5f78ff6aed3647038a94199516d87c59bf5aac780d928f1cd.png)

要为实体增加字段，可以打开json文件，在fields里新增即可，比如我们增加一个content字段。


    {
      "name": "FeedbackRecord",
      "fields": [
        {
          "fieldName": "feedbackType",
          "fieldType": "FeedbackType",
          "javadoc": "反馈类型",
          "fieldValues": "ADVICE,COMPLAINTS"
        },
        {
          "fieldName": "title",
          "fieldType": "String",
          "javadoc": "问题描述"
        },
        {
          "fieldName": "content",
          "fieldType": "String",
          "javadoc": "问题详情"
        },
        {
          "fieldName": "feedbackStatus",
          "fieldType": "FeedbackStatus",
          "javadoc": "反馈状态",
          "fieldValues": "TO_BE_SUBMIT,TO_BE_REPLY,TO_BE_CONFIRMED"
        },
        {
          "fieldName": "lastReplyTime",
          "fieldType": "Integer",
          "javadoc": "是否已完成"
        },
        {
          "fieldName": "closeType",
          "fieldType": "FeedbackCloseType",
          "javadoc": "关闭类型",
          "fieldValues": "NORMALLY,TIMEOUT"
        },
        {
          "fieldName": "createdDate",
          "fieldType": "Instant",
          "javadoc": "创建时间"
        },
        {
          "fieldName": "createdBy",
          "fieldType": "String",
          "javadoc": "创建者"
        }
      ],
      "relationships": [],
      "javadoc": "反馈记录表",
      "entityTableName": "feedback_record",
      "dto": "mapstruct",
      "pagination": "pagination",
      "service": "serviceImpl",
      "jpaMetamodelFiltering": true,
      "fluentMethods": true,
      "readOnly": false,
      "embedded": false,
      "applications": "*",
      "changelogDate": "20210312072243"
    }


再次运行实体生成器：


    jhipster entity FeedbackRecord


当您为现有实体运行实体子生成器时，系统会询问您“Do you want to update the entity? This will replace the existing files for this entity, all your custom code will be overwritten”(您确定需要更新实体吗？这将替换该实体的现有文件，所有自定义代码将被覆盖)，并具有以下选项：

- `Yes, re generate the entity` - 这将重新生成您的实体。提示：这可以通过在运行子生成器时传递`--regenerate`标志来强制执行
- `Yes, add more fields and relationships` - 这将需要您回答一些问题，以添加更多字段和关系
- `Yes, remove fields and relationships` - 这将需要您回答一些问题，以便从实体中删除现有字段和关系
- `No, exit` - 这将存在子生成器而无需更改任何内容


您可能由于以下原因而要更新您的实体：

提示：要立即重新生成所有实体，可以使用以下命令（不提供`--force`标识会在文件更改时询问覆盖选项）。

- Linux & Mac: `for f in`ls .jhipster`; do jhipster entity ${f%.*} --force ; done`
- Windows: `for %f in (.jhipster/*) do jhipster entity %~nf --force`


### 代码示例

