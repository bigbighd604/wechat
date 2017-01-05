## it's hard in feature implementation
### Need different information to make assignments 
need geo, RTT, load prediction to make good traffic assignments

### Avoid proxy mode
### Need generic enough

#### out of box SDK should support a basic working solution
#### customized SDK should support more advanced solution without any server side changes.
### It's hard to test without applying real traffic
### Need support stateful service too

### Need consider hackend health checking

### Need consider automatic failover

### Need support transparent troubleshooting mechanism for clients

### scheduling algrithm
Simple algorithms include random choice or round robin. More sophisticated load balancers may take additional factors into account, such as a server's reported load, least response times, up/down status (determined by a monitoring poll of some kind), number of active connections, geographic location, capabilities, or how much traffic it has recently been assigned.


balancing shart-lived requests, vs. long-lived requests vs. bandwith
