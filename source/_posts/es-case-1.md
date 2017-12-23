---
title: 不规范ES前缀查询导致ES集群挂掉
date: 2017-12-24 00:47:53
tags:
- ELK
- casestudy
categories: ELK
---
线上订单ES集群平常好好的，今天莫名奇妙所有节点全挂，导致历史订单搜索服务不可用，还好数据量不大，及时拉起来，几分钟恢复了，查看原因。

ES各个节点挂掉的日志如下，发现Java 栈溢出，疑似一个search操作导致

    [2017-12-19T15:46:57,063][ERROR][o.e.b.ElasticsearchUncaughtExceptionHandler] [es-node1] fatal error in thread [elasticsearch[es-node1][search][T#47]], exiting
    java.lang.StackOverflowError: null
    at org.apache.lucene.util.automaton.Operations.isFinite(Operations.java:1049) ~[lucene-core-6.3.0.jar:6.3.0 a66a44513ee8191e25b477372094bfa846450316 - shalin - 2016-11-02 19:47:11]
    at org.apache.lucene.util.automaton.Operations.isFinite(Operations.java:1053) ~[lucene-core-6.3.0.jar:6.3.0 a66a44513ee8191e25b477372094bfa846450316 - shalin - 2016-11-02 19:47:11]

联系业务对时间点的日志，发现有个query使用了Prefix前缀匹配，手机号号码拼切了几万个9，而且使用多个Prefix，用or连接，显然这个操作是不规范的，是不是元凶呢？

    ((Prefix{field='main.orderNumStr', prefix='138117061949999999999999999999999999999999999999999999999999999999999999999999999999999999999999999999999(后面有几千个9)', valueType=string})
    
后去官网issue上得到了验证,具体issues:

https://github.com/elastic/elasticsearch/issues/24553

```
Prefix/Regex一类的Query，如果查询字符串过长，或者pattern本身匹配规则过于复杂，ES在解析的时候构造出来的状态机过多，之后调用一个isFinite的Lucene方法可能产生堆栈溢出。
org.apache.lucene.util.automaton.Operations.isFinite代码如下：

可以看到这段代码里用了递归，递归的深度取决于状态转移的数量。
根据注释的说明，这是一段待完善的代码，因为使用了递归，可能导致堆栈溢出:
// TODO: not great that this is recursive... in theory a
// large automata could exceed java's stack
private static boolean isFinite(Transition scratch, Automaton a, int state, BitSet path, BitSet visited) {
path.set(state);
int numTransitions = a.initTransition(state, scratch);
for(int t=0;t<numTransitions;t++) {
a.getTransition(state, t, scratch);
if (path.get(scratch.dest) || (!visited.get(scratch.dest) && !isFinite(scratch, a, scratch.dest, path, visited))) {
return false;
}
}
path.clear(state);
visited.set(state);
return true;
}
```

官方承诺在ES6.0中修复,紧急处理在业务层面限制关于模糊查询、前缀查询、正则匹配查询的字符串的长度。
