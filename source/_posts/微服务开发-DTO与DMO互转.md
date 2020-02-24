---
title: 微服务开发-DTO 与 DMO 互转
date: 2020-02-24 21:54:00
categories: 微服务开发
---
## 概念
* DTO 数据传输对象
* DMO 数据模型对象

大多数情况下，DTO 与 DMO 所承载的字段大致是一样的，但是有些时候也会有差别。
比如某些情况下，DTO 可能根据会实际需要裁剪或聚合添加某些字段，另外对于不同的 API 接口场景，即使使用同一个 DTO，它们的校验方式也可能不同。
因此对于同一个 DMO，我们可能会用到一个或多个 DTO，那么为了交换数据，必然会存在两者之间的转换，也就是数据字段的拷贝。
这种拷贝如果手工去做，繁琐且容易出错，好在业界有一些实践，可以帮我们自动处理绝大多数场景下的字段映射。（少数特殊场景下，有些字段可能不能自动匹配映射，只需要简单的定制一些映射逻辑即可）

## ModelMapper
### 依赖
```xml
<dependency>
    <groupId>org.modelmapper</groupId>
    <artifactId>modelmapper</artifactId>
    <version>${modelmapper.version}</version>
</dependency>
```

### 使用示例
```java
ModelMapper modelMapper = new ModelMapper();
OrderDto order = modelMapper.map(orderEntity, OrderDto.class);
```
## MapStruct
### 依赖
```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-jdk8</artifactId>
    <version>${mapstruct.version}</version>
</dependency>
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>${mapstruct.version}</version>
</dependency>
```

### 使用示例
```java
@Mapper
public interface OrderMapper {

    OrderDto toDto(OrderEntity entity);

    OrderEntity toEntity(OrderDto dto);

}

OrderMapper mapper = Mappers.getMapper(OrderMapper.class);
OrderDto order = mapper.convert(orderEntity);
```

## 对比
* ModelMapper 是在运行期使用反射，而 MapStruct 是在编译期动态生成字节码
* MapStruct 性能远高于 ModelMapper
* ModelMapper 易用性高于 MapStruct
