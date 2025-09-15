---
title: Excel导出内存优化方案
date: 2025-09-05 15:52:16
tags:
---

### 问题核心分析

Excel导出导致内存飙升和Full GC的本质原因是：*
*在数据处理和生成的过程中，在内存中同时驻留了大量的对象（通常是导出数据的DTO/Entity、Excel单元格对象等），这些对象的总大小超过了JVM堆空间的容量，导致频繁GC，最终引发Stop-The-World的Full
GC，甚至OOM。**

***

### 一、 问题排查方向 (How to Diagnose)

当线上出现这个问题时，不要盲目优化，首先要定位瓶颈。我会采用以下步骤进行排查：

1. **确认数据量和对象大小**
    * 首先，确认单次导出的数据行数和列数。100万行 x 20列 和 1万行 x 100列 的处理方式完全不同。
    * 估算一个Java对象（例如`UserDTO`）的大致大小，可以使用`jol-core`工具库。然后计算所有对象的总大小，与设置的堆内存（`-Xmx`
      ）进行对比，立刻就能发现是否明显不够。

2. **检查代码结构和API使用**
    * **最可疑点：是否在使用Apache POI的`XSSFWorkbook`？** `XSSFWorkbook`是用于处理`.xlsx`
      文件的模型，它将整个工作表以对象树的形式保存在内存中（一个单元格一个对象）。数据量极大时，内存消耗会呈指数级增长，这是最常见的原因。
    * 检查代码中是否存在不必要的对象持有。例如，是否为了方便，将数据库查询出的所有`List<Entity>`全部保存在内存中，同时又在转换生成另一个
      `List<DTO>`，导致数据在内存中存在多份拷贝。

3. **利用监控和诊断工具（关键步骤）**
    * **启用GC日志**：在JVM启动参数中加入`-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log`
      。通过分析GC日志，可以清晰看到每次GC的耗时、回收前后堆空间的变化，尤其是Full GC的频率和原因。
    * **使用JVM监控工具**：`jstat -gc <pid> 1s` 实时查看堆内存各区域（Eden, S0/S1, Old）的使用情况和GC次数时间。
    * **生成和分析堆转储（Heap Dump）**：
        * 在OOM时自动生成：`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./dump.hprof`
        * 或在系统高负载时，使用`jmap -dump:live,format=b,file=dump.hprof <pid>`手动生成。
        * 使用**Eclipse MAT**或**JProfiler**等工具分析`hprof`文件。重点关注：
            * **Dominator Tree**：找到内存中占用最大的对象。
            * **Leak Suspects Report**：查看工具自动分析的疑似内存泄漏点。
            * 查看`POI`相关对象（如`XSSFCell`, `XSSFRow`）或业务数据对象的数量和大小的直方图。

***

### 二、 解决方案与优化方案 (How to Solve & Mitigate)

根据排查结果，从简单到复杂、从代码到架构逐层解决问题：

#### 1. 使用正确的POI API（立即生效）

**摒弃`XSSFWorkbook`，使用`SXSSFWorkbook`进行流式导出。**

* **`SXSSFWorkbook`** 是`XSSFWorkbook`的流式API扩展，专门用于处理大数据量。
* **原理**：它通过在内存中只保留一定数量的行（一个“滑动窗口”），将超出数量的行**flush到磁盘临时文件**中，从而极大地降低内存消耗。
* **用法示例**：
  ```java
  // 在内存中保持100行，超过100行后将最早的行写入临时文件
  SXSSFWorkbook workbook = new SXSSFWorkbook(100);
  Sheet sheet = workbook.createSheet("数据");

  // 写入表头...
  // 分页查询数据库，循环写入数据
  for (int i = 0; i < largeDataCount; i++) {
      Row row = sheet.createRow(i);
      // ...创建单元格，设置值
      // 每处理一定次数后，手动flush一行（非必须，但有时可更好控制内存）
      if (i % 100 == 0) {
          ((SXSSFSheet)sheet).flushRows(100); // 保留最后100行
      }
  }
  // 将最终文件写入HttpServletResponse的OutputStream
  workbook.write(outputStream);
  // 删除临时文件
  workbook.dispose();
  workbook.close();
  ```

#### 2. 优化数据查询过程（减少源头数据）

* **分页查询，流式处理**：
    * 绝对不要使用`JPA`的`findAll()`或MyBatis一次性查询百万数据到`List`中。
    * 使用**数据库游标**（如MyBatis的`Cursor`）或**分页多次查询**。
    * 示例（分页查询）：
      ```java
      int pageSize = 1000;
      int pageNo = 1;
      Page<User> page;
      do {
          // 使用MyBatis-Plus等分页插件或手写分页SQL
          page = userService.page(new Page<>(pageNo, pageSize));
          List<User> records = page.getRecords();
          // 将这一页数据写入SXSSFWorkbook
          writeDataToSheet(records, sheet);
          pageNo++;
          // 注意：清空分页查询的上下文，避免Hibernate/MyBatis一级缓存堆积
      } while (page.hasNext());
      ```
    * 示例（MyBatis Cursor）：
      ```java
      @Select("SELECT * FROM large_table ${ew.customSqlSegment}")
      Cursor<User> selectAll(@Param(Constants.WRAPPER) Wrapper<User> wrapper);

      try (Cursor<User> cursor = mapper.selectAll(wrapper)) {
          cursor.forEach(user -> {
              // 逐行处理，写入Excel
              writeSingleRowToSheet(user, sheet);
          });
      }
      ```

#### 3. 优化数据处理和对象创建

* **避免在循环中创建不必要的对象**：例如`SimpleDateFormat`，应该在循环外创建好。
* **重用对象**：对于可以重用的对象（如某些值对象），考虑使用对象池或线程局部变量（`ThreadLocal`），但要谨慎评估复杂性。
* **使用原始类型而非包装类**：如果模型允许，使用`long`而非`Long`，减少对象头开销。

#### 4. 架构层面的优化（根治问题）

如果上述方法后数据量依然巨大（例如数千万行），则需要从架构上重新设计。

* **异步导出 + 任务队列 + 结果下载**
    1. 用户点击导出后，后端立即返回一个`taskId`或`jobId`。
    2. 将导出任务放入**消息队列**（如RabbitMQ、RocketMQ）或**线程池**中异步处理。
    3. 后端工作者消费任务，使用上述`SXSSFWorkbook`+**分页/游标**的方式在后台慢慢生成Excel文件，并将最终文件上传到**OSS**
       或文件服务器。
    4. 前端通过`taskId`轮询任务状态。完成后，提供的是一个**下载链接**。

    * **好处**：
        * 解耦了HTTP请求和耗时操作，避免了HTTP超时。
        * 完美解决内存问题，因为是在后台可控环境下处理，甚至可以分配到独立JVM或机器上处理。
        * 用户体验好，不会阻塞界面。

* **CSV格式替代Excel**
    * 如果业务可接受，导出CSV是更优选择。CSV是纯文本格式，生成过程几乎不消耗额外内存（一行一行写入输出流即可），速度极快。

#### 5. JVM参数调优（辅助手段，不能根治问题）

在代码优化基础上，适当调整JVM参数可以增加系统的鲁棒性，作为最后一道防线。

* **增大堆空间**：`-Xmx4g -Xms4g` 避免堆自动扩展。
* **优化GC器**：对于大量创建短暂存活对象的导出任务，**G1GC**通常表现更好。可以设置：`-XX:+UseG1GC -XX:MaxGCPauseMillis=200`。
* **增大Survivor区**：由于导出任务会产生大量朝生夕死的对象，可以适当增大年轻代大小和Survivor区的比例（`-XX:SurvivorRatio`
  ），让这些对象在Young GC就被回收掉，避免过早进入老年代引发Full GC。

### 总结

“对于Excel导出导致的内存Full GC问题，我的排查和解决思路是分四步走的：

1. **首先精准排查**：通过GC日志和Heap Dump分析，确定内存被哪些对象占用了，是因为数据量本身过大还是代码中存在内存泄漏。
2. **立即代码优化**：核心是使用POI的流式API `SXSSFWorkbook` 替代 `XSSFWorkbook`，并结合数据库分页或游标查询，避免一次性加载所有数据，从根本上减少内存中的对象数量。
3. **架构异步解耦**：对于超大数据量，采用‘异步任务+消息队列+云端存储’的架构，将耗时操作与在线请求分离，这是最彻底的解决方案。
4. **辅助JVM调优**：适当增大堆内存、选择G1垃圾收集器等，为系统提供更多缓冲。

在实际项目中，我通常会优先采用`SXSSFWorkbook`结合分页查询的方案，因为它改造成本低且效果显著。如果数据量达到亿级，则会推动架构改造，采用异步导出的方案。”

