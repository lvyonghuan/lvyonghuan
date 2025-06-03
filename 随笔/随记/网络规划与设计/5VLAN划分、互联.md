# Trunk

VLAN的中继端口一般被称作trunk。trunk技术用在交换机之间互联，使不同的VLAN通过共享链路与其它交换机中的相同VLAN通信。交换机之间互联的端口称为trunk端口。

trunk是基于数据链路层的技术。

两台交换机只需要一条互连线，将互连线的端口设置为trunk模式，即可实现跨交换机的VLAN通信。

交换端口有两种模式：trunk和access。连接终端使用access模式，连接交换机使用trunk模式。

## trunk配置

cisco的trunk封装协议是dot1q。

标准协议是802.1q

# VLAN划分

## 单臂路由VLAN互联

实现跨VLAN通信的方式有两种：单臂路由和三层交换。

## 三层交换VLAN互联

需要配置VTP协议。

---

HOST A和C属于D-A，B和D属于D-B。现在部门A使用VLAN 100，B使用VLAN 200.要求同一VLAN内主机互通。

```
 S1------S2
/ \      / \
A  B    C  D
```

---

hybrid端口类型：允许一个端口同时属于多个VLAN。