---
title: Java List去重的两种方式
layout: post
date: 2023-08-08 21:06:47
published: true
categories:
  [ Java,List ]
tags:
  - Java
---
```java
public static void main(String[] args) {
        Account account = Account.builder()
                .name("张三")
                .phone("13100001111")
                .build();
        Account account1 = Account.builder()
                .name("张三")
                .phone("13100001111")
                .build();
        Account account2 = Account.builder()
                .name("张三1")
                .phone("13100001112")
                .build();
        Account account3 = Account.builder()
                .name("张三2")
                .phone("13100001111")
                .build();
        Account account4 = Account.builder()
                .name("张三3")
                .phone("13100001113")
                .build();
        List<Account> accountLists = Lists.newArrayList();
        accountLists.add(account);
        accountLists.add(account1);
        accountLists.add(account2);
        accountLists.add(account3);
        accountLists.add(account4);
        log.info("原始数组：{}", accountLists.size());
        // 通过map的方式去重
        Map<String, Account> map = Maps.newHashMap();
        accountLists.forEach(r -> {
            map.put(r.getName() + r.getPhone(), r);
            log.info(r.getName() + "-" + r.getPhone());
        });
        log.error("---通过map的方式去重---");
        List<Account> collect = map.values().stream().collect(Collectors.toList());
        log.info("新数组：{}", collect.size());
        collect.forEach(r -> {
            log.info(r.getName() + "-" + r.getPhone());
        });
        log.error("---通过Java8 的方式去重---");
        List<Account> accounts = accountLists.stream().collect(Collectors.collectingAndThen(Collectors.toCollection(() -> new TreeSet<>(Comparator.comparing(r -> r.getName() + "-" + r.getPhone()))), ArrayList::new
        ));
        log.info("新数组：{}", accounts.size());
        accounts.forEach(r -> log.info(r.getName() + "-" + r.getPhone()));
    }

```