---
layout: post
title: 软件工程中的一些最佳实践沉淀
tags: [java, software-engineering, best-practice]
---

# jar/maven版本号定义
一般推荐使用 [chronVer](https://chronver.org/)
## [semVer](https://semver.org/)
- 格式: MAJOR.MINOR.PATCH
- 语义: 
  - MAJOR version when you make incompatible API changes
  - MINOR version when you add functionality in a backwards compatible manner
  - PATCH version when you make backwards compatible bug fixes.
- 优缺点: 
  - 优点: 较为简单, 也是大家常用的方式.
  - 缺点: 不直观

## [chronVer](https://chronver.org/)
- 格式: YEAR.MONTH.DAY.CHANGESET_IDENTIFIER
- 语义: CHANGESET_IDENTIFIER
- 优缺点:
  - 优点: 直接用时间戳来标识, 较为直观. 适用较为频繁发版的系统. 
  - 缺点: 比较难以标识版本之间的兼容情况.





