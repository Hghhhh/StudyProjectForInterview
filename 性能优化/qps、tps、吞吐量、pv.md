qps：每秒查询率(Query Per Second) ,每秒的响应请求数，也即是最大吞吐能力。

tps：每秒处理的事务数，TPS包括一条消息入和一条消息出，加上一次用户数据库访问。

吞吐量：吞吐量是指系统在单位时间内处理请求的数量。对于无并发的应用系统而言，吞吐量与响应时间成严格的反比关系，实际上此时**吞吐量就是响应时间的倒数**。前面已经说过，对于单用户的系统，响应时间（或者系统响应时间和应用延迟时间）可以很好地度量系统的性能，但对于并发系统，通常需要用吞吐量作为性能指标。 

PV:访问量即Page View, 即页面浏览量或点击量，用户每次刷新即被计算一次