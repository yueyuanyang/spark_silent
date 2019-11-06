### spark core - proper Optimization

涉及到的内容
- spark的层次
- spark UI
- 理解partitions
- 常见的优化机会

### spark的层次
![https://github.com/yueyuanyang/spark_silent/tree/master/optimization/img/opti1.png]

- action are eager
  - trransformations产生
    - narrow(窄依赖)
    - wide(reuires shuffle)(宽依赖)
