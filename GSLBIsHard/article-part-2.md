
## it's hard to run

### Itself need reliable
要提高所服务系统的可用性，GSLB本身也必须更加稳定可靠。

#### stateful system
但是，做持久化又意义不大，因为计算量往往是基于历史的数十秒数据，持久化不仅增加系统复杂性，而且在进程重启后持久化的数据变的没有意义，因为进程重启时间往往大于数十秒。
所以，对可靠性的要求就更高了，坚决不能倒下。

#### Overload handling

#### architecture key points

### Itself need scale also

### GSLB as a self service
Not only register and use the service, but also provide tools to support shifting traffic.

### GSLB as a micro service
In order to support even larger number clients and services.

