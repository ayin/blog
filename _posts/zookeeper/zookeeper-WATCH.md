* client WATCH可以落到follower？
* 一个client只能连一个zk服务器，同一台zk服务器上事务是有序的，因此WATCH到的事件也是有序的
* 如果被连的follower挂掉，client端保存有watch信息，重新连到其他follower上时会重新注册WATCH信息
* WATCH的时候有保存节点的版本信息，如果重连的过程中WATCH的节点信息发生了变化，可以根据版本来触发Notification