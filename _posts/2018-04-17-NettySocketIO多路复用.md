## 概念
namespace 和room的概念其实用来同一个服务端socket多路复用的。namespace，room和socketio的关系如下:
![image](https://note.youdao.com/yws/api/personal/file/51F2ECF3CFCC45BE91AD5083DEEE1061?method=download&shareKey=015eb8df033e9d256b81cde054038bbf)

每一个socket（每一个客户端）会属于某一个room，如果没有指定，那么会有一个default的room。这个room又会属于某个namespace，如果没有指定，那么就是默认的namespace /。
## 广播
socketIO广播的时候是以namespace或者room为单位的。指定某个namespace为单位，那么这个namespace下的所有room中的客户端都可以接收到广播消息。指定某个room为单位，那么只有这个room中的客户端可以接收到广播消息。
## Netty-Socketio主要类和方法
##### SocketIOClient 
客户端接口，其实现类是NamespaceClient，主要方法如下
```
joinRoom() 加入到指定房间。
leaveRoom() 从指定房间离开。
getSessionId()方法，返回由UUID生成的唯一标识。
getAllRooms() 返回当前客户端所在的room名称列表。
sendEvent(eventname,data) 向当前客户端发送事件。
```
##### SocketIOServer 
服务端实例，主要方法如下：

```
getAllClients() 返回默认名称空间中的所有客户端实例。
getBroadcastOperations() 返回默认名称空间的所有实例组成的广播对象。
getRoomOperations() 返回所有命名空间中指定房间的广播对象，如果命名空间只有一个，该方法到可以大胆使用。
getClient(uid) 返回默认名称空间的指定客户端。
getNamespace() 返回指定名称的命名空间。
```
##### Namespace 
命名空间。netty-socketio忠实的重现了socketio的server–>namespace–>room三层嵌套关系。 
 
从NamespacesHub的getRoomClients方法可以知道，SocketIOServer的getRoomOperations方法返回的是所有namespace中指定room中的客户端实例。而不是指定命名空间或者默认命名空间的，使用该方法的时候要小心。
###### 如果要获取指定命名空间的指定room中的客户端，一定要先拿到指定namespace对象。 
Namespace中的主要方法如下：

```
getAllClients() 获得本namespace中的所有客户端。
getClient() 获得指定id客户端对象。
getRoomClients(room) 获得本空间中指定房间中的客户端。
getRooms() 获得本空间中的所有房间。
getRooms(client) 获得指定客户端所在的房间列表。
leave(room,uuid) 将指定客户端离开指定房间，如果房间中已无客户端，删除该房间。
getBroadcastOperations() 返回针对空间中所有客户端的广播对象。
getRoomOperations(room) 返回针对指定房间的广播对象。
```

##### BroadcastOperations 
广播操作对象，通过对Namespace的了解我们知道，BroadcastOperations都是命名空间以指定room中的clients列表为参数创建的。 
BroadcastOperations中最通用的方法便是sendEvent方法，该方法遍历clients，通过执行客户端的send方法实现广播目的。也可以设定一个排除对象，当然用于排除发送者自己了。


```
sendEvent(eventname,data) 向本广播对象中的全体客户端发送广播。
sendEvent(eventname,excludeSocketIOClient,data) 排除指定客户端广播。
```

## 操作
##### namespace静态添加

```
private final SocketIOServer server;

@Override
public void run(String... args) throws Exception {
    // 循环添加命名空间
    for(NamespaceEnum namespace : NamespaceEnum.values()){
        server.addNamespace("/" + namespace.toString());
    }
    // server启动前添加命名空间
    server.start();
}

/**
 * 命名空间枚举
 */
private enum NamespaceEnum{
    namespace1,namespace2
}
```
##### room动态添加

js代码
```
var socket = io.connect('http://192.168.20.95:8081/namespace1?roomId=room1');
```
java代码

```
@OnConnect
public void onConnect(SocketIOClient client) {
    HandshakeData handshakeData = client.getHandshakeData();
    String roomId = handshakeData.getSingleUrlParam("roomId");
    client.joinRoom(roomId);
}
```
##### 消息发布

```
public void sendMessageToNamespaceRoom(String namespace, String room, String content) {
    BroadcastOperations broadcastOperations = server.getNamespace("/" + namespace).getRoomOperations(room);
    broadcastOperations.sendEvent("advert_data", content);
}
```

