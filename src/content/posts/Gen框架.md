---
title: Gen框架
description: 本文介绍了基于GORM的代码生成框架Gen的使用方法，包括最佳实践目录推荐、需要编写的部分以及框架自动生成的功能。
published: 2025-01-26
updated: 2025-04-13
tags:
  - 大三
  - 开发
draft: false
pin: 99
toc: true
lang: zh
abbrlink: gen-framework
---



# :books: Gen

# 前言

> 前世今生

* 在[go ORM](../技术/go%20ORM.md)基础上进一步封装的orm框架

## 最佳实践目录推荐

```shell
demo
├── cmd
│   └── generate
│       └── generate.go # 包含main函数，执行其即可完成生成代码步骤
├── dal
│   ├── dal.go # 实现具体的数据库连接等操作，如初始化db
│   └── model
│   │   ├── method.go # 指定所有自定义查询方法
│   │   └── model.go  # 描述与数据库表对应的数据结构(体)
│   └── query  # 【生成的代码存放目录】 在执行代码生成操作后自动创建
│       └── gen.go # 生成的通用查询代码
│       └── tablename.gen.go # 生成的单个表字段和相关的查询代码
├── config
│   └── config.go # 存储相关的数据库DSN
├── generate.sh # 调用generate中main函数生成代码的脚本（推荐使用）
├── go.mod
├── go.sum
└── main.go
```

# 分工

> 下载`go get gorm.io/gen`

## :one: 需要自己编写的部分

* 前置步骤

  * 创建生成器

    ```go
    // 创建一个新的生成器，配置输出路径和模式
     	g := gen.NewGenerator(gen.Config{
     		OutPath: "../../dal/query",
            Mode:    gen.WithoutContext | gen.WithDefaultQuery, // 结合，选哪种都可以，更灵活
     	})
    ```

    * 对于生成器模式

      ## 方法对比
      
      | 特点       | `query.Q.Alphabet.Create(Alphabets...)`        | `query.Alphabet.Save(Alphabets...)`        |
      | ---------- | ---------------------------------------------- | ------------------------------------------ |
      | 简洁性     | 更加简洁，不需要传递上下文                     | 需要通过 `query` 对象进行操作              |
      | 灵活性     | 适合不需要上下文管理的简单项目                 | 更加灵活，可以选择带或不带上下文进行操作   |
      | 上下文管理 | 不支持上下文管理                               | 支持上下文管理，可以选择带上下文进行操作   |
      | 生成模式   | 需要使用 `gen.WithDefaultQuery`                | 可以使用 `gen.WithoutContext` 或默认模式   |
      | 代码示例   | `err := query.Q.Alphabet.Create(Alphabets...)` | `err := query.Alphabet.Save(Alphabets...)` |
      
      ## 生成模式对比
      
      | 模式                   | 特点                                                         | 使用场景                                           | 举例-如何调用                         |
      | ---------------------- | ------------------------------------------------------------ | -------------------------------------------------- | ------------------------------------- |
      | `gen.WithDefaultQuery` | 生成全局变量作为 DAO 接口，简洁的查询方式，不需要传递上下文。 | 简化代码查询，适合小型项目或不需要上下文管理的情况 | `query.Q.User.First()`                |
      | `gen.WithoutContext`   | 既可选带上下文，也可选不带上下文。<br>**带上下文的优势是使用可以取消信号和超时控制，提供更强的控制力。** | 提供灵活的查询方式，适合需要可选上下文管理的情况   | `query.User.WithContext(ctx).First()` |

  * 导入连接好的数据库

* 情况1：已建好数据库

  >  自动从数据库中读取表——转化model结构体。公司开发中常见。

  1. 生成模板 `GenerateModel`

  2. 应用模型 `ApplyBasic`

  3. 执行 `g.Execute()`

     ---

  > 或者直接嵌套调用

  1. 应用模型 `g.ApplyBasic(g.GenerateAllTable()...)`
  2. 执行 `g.Execute()`

* 情况2：导入手动定义的模型

  1. 定义模型

     * 定义模型 

     * 定义接口 `ApplyInterface`

       > 目的是对于基本的 CRUD 操作的拓展——（例如复杂的查询、聚合操作等）添加到生成的代码中。

       * 如果你需要自定义一些查询方法，这时才需要定义接口并通过 `ApplyInterface` 方法应用这些接口。

  2. 应用模型 `ApplyBasic`

  3. 执行

  ---

  * 函数阐述

    | 函数             | 名称     | 参数名称及说明                                             |
    | ---------------- | -------- | ---------------------------------------------------------- |
    | `ApplyInterface` | 接口应用 | 1. 第一个参数是接口<br />2. 后面的参数是需要绑定的模型     |
    | `ApplyBasic`     | 应用模板 | 基本模型结构体或模板。                                     |
    | `GenerateModel`  | 生成模型 | 1. 数据库中的表名。<br />2. 生成与该表名对应的模型结构体。 |
    
    * 举例
    
      ```go
      func main() {
          	// 创建一个新的生成器，配置输出路径和模式
          	g := gen.NewGenerator(gen.Config{
          		OutPath: "../../dal/query",
          	})
          
          	// 使用已经初始化的数据库
          	g.UseDB(mysql.DB)
          
          	// 这个是从数据库表生成模型的模板
          	//userTpl := g.GenerateModel("inv_lists")
          
          	// 这个是应用自己写的模型
          	g.ApplyBasic(model.Alphabet{}, model.LexicalAnalysis{}, model.Delimiter{}, model.Keywords{}, model.Words{})
          	//g.ApplyBasic(userTpl)
          
          	// 执行生成操作
          	g.Execute()
          }
      ```


## :two: 框架可以自动生成的部分  

:a: `gen`会自动生成CRUD等操作 。

> 下面是一个具体的例子

| 函数                                                         | 说明                                                         | 示例                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `Create(values ...model.Alphabet) error`                     | 创建新的 Alphabet 记录。你可以传入一个或多个 model.Alphabet 类型的指针。 | ```alphabet.Create(&model.Alphabet{Key: 1, Value: "A"})```   |
| `CreateInBatches(values []model.Alphabet, batchSize int) error` | 批量创建新的 Alphabet 记录。你需要传入一个 model.Alphabet 类型的指针切片和一个批处理大小。 | ```alphabet.CreateInBatches([]*model.Alphabet{{Key: 1, Value: "A"}, {Key: 2, Value: "B"}}, 2)``` |
| `Save(values ...model.Alphabet) error`                       | 更新 Alphabet 记录。你可以传入一个或多个 model.Alphabet 类型的指针。 | ```alphabet.Save(&model.Alphabet{Key: 1, Value: "B"})```     |
| `First() (model.Alphabet, error)`                            | 获取第一个 Alphabet 记录。                                   | ```alphabet.First()```                                       |
| `Take() (model.Alphabet, error)`                             | 获取一个 Alphabet 记录。                                     | ```alphabet.Take()```                                        |
| `Last() (model.Alphabet, error)`                             | 获取最后一个 Alphabet 记录。                                 | ```alphabet.Last()```                                        |
| `Find() ([]model.Alphabet, error)`                           | 获取所有 Alphabet 记录。                                     | ```alphabet.Find()```                                        |
| `FindInBatch(batchSize int, fc func(tx gen.Dao, batch int) error) (results []model.Alphabet, err error)` | 批量获取 Alphabet 记录。你需要传入一个批处理大小和一个回调函数。 | ```alphabet.FindInBatch(2, func(tx gen.Dao, batch int) error { return nil })``` |
| `FindInBatches(result []model.Alphabet, batchSize int, fc func(tx gen.Dao, batch int) error) error` | 批量获取 Alphabet 记录并将结果存储在传入的切片中。你需要传入一个 model.Alphabet 类型的指针切片，一个批处理大小和一个回调函数。 | ```alphabet.FindInBatches(&[]*model.Alphabet{}, 2, func(tx gen.Dao, batch int) error { return nil })``` |
| `FirstOrInit() (model.Alphabet, error)`                      | 获取第一个 Alphabet 记录，如果没有找到，则初始化一个。       | ```alphabet.FirstOrInit()```                                 |
| `FirstOrCreate() (model.Alphabet, error)`                    | 获取第一个 Alphabet 记录，如果没有找到，则创建一个。         | ```alphabet.FirstOrCreate()```                               |
| `FindByPage(offset int, limit int) (result []model.Alphabet, count int64, err error)` | 分页获取 Alphabet 记录。你需要传入一个偏移量和一个限制数。   | ```alphabet.FindByPage(0, 10)```                             |
| `ScanByPage(result interface{}, offset int, limit int) (count int64, err error)` | 分页获取 Alphabet 记录并将结果存储在传入的接口中。你需要传入一个接口，一个偏移量和一个限制数。 | ```alphabet.ScanByPage(&[]*model.Alphabet{}, 0, 10)```       |
| `Scan(result interface{}) (err error)`                       | 获取所有 Alphabet 记录并将结果存储在传入的接口中。           | ```alphabet.Scan(&[]*model.Alphabet{})```                    |
| `Delete(models ...model.Alphabet) (result gen.ResultInfo, err error)` | 删除 Alphabet 记录。你可以传入一个或多个 model.Alphabet 类型的指针。 | ```alphabet.Delete(&model.Alphabet{Key: 1})```               |

---

:b:对于定义好的接口，`gen`还会生成自定义查询方法。

## :three: 如何使用

> 由于gen会自动生成curd，所以我们通过特定的步骤直接调用。

* 在generate.go文件

  ```go
   func main() {
   	// 创建一个新的生成器，配置输出路径和模式
   	g := gen.NewGenerator(gen.Config{
   		OutPath: "../../dal/query",
   	})
   
   	// 使用已经初始化的数据库
   	g.UseDB(mysql.DB)
   
   	// 这个是从数据库表生成模型的模板
   	//userTpl := g.GenerateModel("inv_lists")
   
   	// 这个是应用自己写的模型
   	g.ApplyBasic(model.Alphabet{}, model.LexicalAnalysis{}, model.Delimiter{}, model.Keywords{}, model.Words{})
   	//g.ApplyBasic(userTpl)
   
   	// 执行生成操作
   	g.Execute()
   }
  ```

  * 生成curd

* 在**main**函数中

  * 初始化数据库连接


  * 将数据库连接设置为默认


* 调用

  * 在biz层直接调用即可
  
  ```go
  package main
  
  import (
      "context"
      "gendemo/dal"
      "gendemo/dal/query"
      "log"
  )
  
  func main() {
  	// ......
      // 导入db，将数据库连接设置为默认
  	query.SetDefault(mysql.Db)
  }
  
  func UpdateKeywords(u []*model.Keywords) error {
  	err := query.Q.Keywords.Save(u...)
  	if err != nil {
  		return err
  	}
  	return nil
  }
  ```
  

# 参考资料

* [项目小demo](C:\Users\hello\myproject\Go\ORM\gendemo)
* [go-gorm/gen: Gen: Friendly & Safer GORM powered by Code Generation (github.com)](https://github.com/go-gorm/gen)
* [字节跳动安全中心 (bytedance.com)](https://security.bytedance.com/techs/wuheng-lab-better-orm-gen#:~:text=GEN是一个基于GORM,来最佳用户体验。)