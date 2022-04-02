![Lucene文件结构（8.5）](https://raw.githubusercontent.com/ppb2/note/main/imgs/Lucene%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84%EF%BC%888.5%EF%BC%89.png)

| Name                | Extension  | Brief Description                                            |
| ------------------- | ---------- | ------------------------------------------------------------ |
| Segments File       | segments_N | 存储关于提交点的信息                                         |
| Lock File           | write.lock | 写锁防止多个indexwriter写入同一个文件。                      |
| Segment Info        | .si        | 存储Segment 的元数据                                         |
| Compound File       | .cfs, .cfe | 一个可选的“虚拟”文件，由经常耗尽文件句柄的系统的所有其他索引文件组成。 |
| Fields              | .fnm       | 存储关于字段的信息                                           |
| Field Index         | .fdx       | 包含指向字段数据的指针                                       |
| Field Data          | .fdt       | 存储的文档字段                                               |
| Term Dictionary     | .tim       | 术语字典，存储术语信息                                       |
| Term Index          | .tip       | 编入术语词典的索引                                           |
| Frequencies         | .doc       | 包含每个术语和频率的文档列表                                 |
| Positions           | .pos       | 存储关于一个术语在索引中出现位置的信息                       |
| Payloads            | .pay       | 存储额外的每位置元数据信息，如字符偏移量和用户有效负载       |
| Norms               | .nvd, .nvm | 编码长度和增强因素的文档和字段                               |
| Per-Document Values | .dvd, .dvm | 编码额外的评分因素或其他每个文档的信息。                     |
| Term Vector Index   | .tvx       | 将偏移量存储到文档数据文件中                                 |
| Term Vector Data    | .tvd       | Contains term vector data.                                   |
| Live Documents      | .liv       | 关于哪些文档是存活的信息                                     |
| Point values        | .dii, .dim | 持有索引点(如果有的话)                                       |