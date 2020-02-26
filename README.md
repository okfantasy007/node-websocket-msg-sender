# todolist系统消息推送后台

#### 安装依赖：npm install

#### 启动服务：npm start

#### 物料：nodejs + express + redis

#### redis：用户会话管理，存储用户与websocket连接实例之间的双向映射关系

本项目redis为了清晰划分用途，开辟了两个分库，即db0和db1，其中db0存储用户的会话信息，db1存储用户id（uid）与socketId（sid）之间的双向映射关系

------------

## 提供的服务（服务端主动向客户端推送消息）

### 私发消息给某个客户端

```javascript
//服务端给指定用户发消息
ioSvc.serverToPrivateMsg = function (uid, data) {
	console.log('发送私人消息');
	console.log(data);
	var _this = this;
	redis.get(uid, function (err, sid) {
		if (err) {
			console.error(err);
		}
		console.log("uid", uid);
		console.log("sid", sid);
		if (sid && _this.io.sockets.connected[sid]) {
			//给指定的客户端发送消息
			_this.io.sockets.connected[sid].emit('message', data);
		}
	});
};
```

**基本思路**

- 首先，通过用户id（uid）查询到socketId（sid）
- 然后，通过`_this.io.sockets.connected[sid].emit('message', data);`给特定客户端用户发送消息

### 广播消息

```javascript
//服务器给所有客户端广播消息
ioSvc.serverBroadcastMsg = function (data) {
	console.log('发送广播消息');
	console.log(data);
	console.log("this.io.sockets", this.io.sockets);
	fetchUserSocketIdArr().then((arr) => {
		if (arr && arr.length > 0) {
			arr.forEach((item) => {
				this.io.sockets.connected && this.io.sockets.connected[item] && this.io.sockets.connected[item].emit('message', data);
			});
		}
	});
	// 通过以下方式广播信息，可能会出现同一个客户端，同时推送多次完全一样的信息的现象
	// this.io.sockets.emit('message', data);
};
```
此处，我对原先的广播消息的写法进行了改造。因为按照`this.io.sockets.emit('message', data);`这种原始方式会遇到一个问题，即一个客户端可能会有多个连接，广播信息的时候，一个客户端可能会受到多条重复的消息。为此，我对该方法进行了改造，改造思路如下：

- 首先，通过redis获取到所有已登录的会话信息（以value的形式存储在redis中），这步流程封装到`fetchUserSocketIdArr`中了。

```javascript
function fetchUserSocketIdArr() {
	return new Promise((resolve, reject) => {
		redisClient1.keys("*", (err, reply) => {
			if (err) {
				console.log(err);
				resolve(false);
			} else {
				if (reply && reply.length > 0) {
					var arr = redisClient1.mget(reply, (err, reply1) => {
						if (err) {
							console.log(err);
							resolve(false);
						} else {
							arr = reply1.map((item) => {
								return JSON.parse(item).login_user_info.socketId;
							});
							console.log('arr', arr);
							resolve(arr);
						}
					});
				}
			}
		});
	});
}
```
- 然后，从这些会话信息中，提取出所有的socketId，放入数组中
- 最后，遍历数组中的socketId，以私发消息的方式间接达到广播的效果

这样就有效避免了一个客户端会同时收到多条重复消息的问题。

### 通知客户端重登陆

```javascript
ioSvc.redirectToLogin = function (socketId) {
	console.log('重定向到登录页');
	if (socketId && this.io.sockets.connected[socketId]) {
		this.io.sockets.connected[socketId].emit('redirect_to_login');
	}
};
```
注意此处emit的是`redirect_to_login`指令，那么客户端就要相应地监听`redirect_to_login`指令，设置回调函数。

客户端的代码如下所示：
```javascript
window.socketConnected && window.socketConnected.on('redirect_to_login', function () {
	console.log("window.socketConnected(redirect_to_login)", window.socketConnected);
	window.socketConnected = null;
	localStorage.clear();
	_this.toPath("/login");
});
```
客户端在重登陆之前做了两件事情，即
1. 将全局的`window.socketConnected`置空
2. 将localStorage缓存清空

### 统计在线用户列表和数量

```javascript
ioSvc.updateOnlieCount = function (params) {
	if (params && params.deleteFlag && params.uid && params.userName) {
		console.log("params.userName", params.userName);
		console.log("--->", `sess:${params.userName}/*`);
		redisClient1.keys(`sess:${params.userName}/*`, (err, res) => {
			console.log("res", res);
			if (res && res.length > 0) {
				redisClient1.del(res[0], (e, r) => {
					if (e) {
						console.log(e);
						console.log("删除该用户名对应的redis key失败");
					} else {
						console.log("删除该用户名对应的redis key成功");
						// this指向ioSvc，传入
						updateOnlieCountFunc(this);
						deleteToRedirectLogin(this, params.uid)
					}
				})
			}
		})
	} else {
		// this指向ioSvc，传入
		updateOnlieCountFunc(this);
	}
};
```
该服务又被封装到`updateOnlieCountFunc`
```javascript
function updateOnlieCountFunc(self) {
	let count = 0;
	let userList = [];
	redisClient1.keys("*", (err, val) => {
		if (err) {
			console.error(err);
		}
		console.log("val", val);
		let arr = [];

		if (val && val.length > 0) {
			val.forEach((item) => {
				let un = item.split("/")[0].split("sess:")[1];
				if (un !== "undefined") {
					arr.push(item);
				}
			});
			if (arr.length > 0) {
				redisClient1.mget(arr, (e, v) => {
					if (e) {
						console.error(e);
					} else {
						if (v && v.length > 0) {
							v.forEach((item) => {
								const i = JSON.parse(item);
								if (i && i.login_user_info && i.login_user_info.id && i.login_user_info.user_name) {
									count++;
									userList.push(i.login_user_info.user_name);
								}
							})
						}
					}
					console.log("===========拉取当前在线用户信息=============");
					console.log('当前在线人数：' + count);
					self.io.sockets.emit('update_online_count', {
						online_count: count,
						user_list: userList
					});
				})
			} else {
				console.log("===========拉取当前在线用户信息=============");
				console.log('当前在线人数：' + count);
				self.io.sockets.emit('update_online_count', {
					online_count: count,
					user_list: userList
				});
			}
		} else {
			console.log("===========拉取当前在线用户信息=============");
			console.log('当前在线人数：' + count);
			self.io.sockets.emit('update_online_count', {
				online_count: count,
				user_list: userList
			});
		}
	});
}
```
**基本思路**
- 从redis db0中获取到所有的key，即所有已登录的用户会话信息的key
- 从key列表中分离出（key的格式为`sess:${用户名}/${时间戳}`）所有已登录的用户名
- 通过广播`update_online_count`指令的方式，将最新的在线用户列表和用户数量信息推送给所有已登录的客户端

#### 拉取在线用户信息的时机

1. 新用户登录
2. 已登录用户退出登录
3. 管理员账号删除普通账号（如果该普通账号已登录，我们需要让其重定向到登录页）

其中，对于第3种时机，处理情况比较特殊，需结合[myapp（todolist系统主后台）](https://gitlab.com/okfantasy007/myapp.git "myapp（todolist系统主后台）")来处理。

请看以下代码（来源于[myapp（todolist系统主后台）](https://gitlab.com/okfantasy007/myapp.git "myapp（todolist系统主后台）")）：

```javascript
//删除
router.post('/delete', function (req, res, next) {
	APP.dbpool.getConnection(function (err, conn) {
		if (err) {
			return sqlConnError(conn, "获取数据库连接失败", res);
		}

		conn.beginTransaction(function (err) {
			if (err) {
				console.log(err);
				return;
			}

			//事务逻辑开始
			const {id, userName} = req.body;

			if (id) {
				var delSql = 'DELETE FROM user WHERE id = ' + parseInt(id);
				console.log('delSql:', delSql);
				//删 delete
				conn.query(delSql, function (err, result) {
					if (err) {
						console.log('[DELETE ERROR] - ', err.message);
						APP.dbpool.releaseConnection(conn);
						return res.status(200).json({
							success: false,
							msg: err.message,
							code: '0001'
						});
					}
					console.log('-------------DELETE--------------');
					console.log('DELETE affectedRows', result.affectedRows);
					console.log('==============================');
					if (result.affectedRows !== 1) {
						conn.rollback(function () {
							console.log('删除失败,回滚!');
							//释放资源
							APP.dbpool.releaseConnection(conn);
							res.status(200).json({
								success: false,
								msg: '删除账号失败',
								code: '0001'
							});
						});
						return;
					}
					//没有错误，提交事务
					conn.commit(function (err) {
						if (err) {
							return res.status(200).json({
								success: false,
								msg: '删除账号失败',
								code: '0001'
							});
						}

						console.log('成功,提交!');
						var options = {
							url: APP.config.wsIP + '/ws/deleteAccountToUpdateOnline',
							headers: {
								'Content-Type': 'application/json;charset=UTF-8',
							},
							body: JSON.stringify(
								{
									uid: id,
									userName: userName
								}
							)
						};
						request.post(options, function (err, response, data) {
							if (err) {
								console.log(err);
								console.log("广播消息失败");
								APP.dbpool.releaseConnection(conn);
								return res.status(200).json({
									success: false,
									msg: "删除账号失败",
									code: '0001'
								});
							}
							console.log("广播消息成功");
							APP.dbpool.releaseConnection(conn);
							return res.status(200).json({
								success: true,
								msg: "删除账号成功",
								code: '0000'
							});
						});
						//释放资源
						/*APP.dbpool.releaseConnection(conn);
						res.status(200).json({
							success: true,
							msg: "删除账号成功",
							code: '0000'
						});*/
					});
				});
			} else {
				APP.dbpool.releaseConnection(conn);
				res.status(200).json({
					success: false,
					msg: '删除账号失败',
					code: '0001'
				});
			}
			//事务逻辑结束
		});
	});
});
```
其中有一段代码是，在成功删除账号之后去请求`node-websocket-msg-sender`的接口`/ws/deleteAccountToUpdateOnline`。

```javascript
router.post('/deleteAccountToUpdateOnline', function (req, res, next) {
	const {uid, userName} = req.body;
	ioSvc.updateOnlieCount({
		deleteFlag: true,
		uid,
		userName
	});
	return res.send({code: 200, msg: '请求成功'});
});
```
在`/ws/deleteAccountToUpdateOnline`的接口处理逻辑中，调用了`ioSvc.updateOnlieCount`去具体处理，并向其传递了三个参数`deleteFlag`、`uid`、`userName`，为的就是与上述另外两种时机区分开来。

我们把视线再投入到`ioSvc.updateOnlieCount`中来，发现代码中用了一个`if...else...`语句对第3种时机与第1、2种时机进行了区别处理。

第3种时机，我们在删除掉对应的redis key之后，做了两步处理，即
1. updateOnlieCountFunc
2. deleteToRedirectLogin

其中，`deleteToRedirectLogin`这一步就是在干【让被删除的已登录的普通账号对应的客户端重定向到登录页】这件事情。

```javascript
function deleteToRedirectLogin(self, uid) {
	redis.get(uid, (err, sid) => {
		if (err) {
			console.log(err);
		} else {
			self.redirectToLogin(sid);
		}
	})
}
```
## 部署
该项目结合自己搭建的jenkins实现自动化部署。

### Build阶段
```shell
node -v
npm -v
#安装依赖包
npm install
#如果node-websocket-msg-sender.tar.gz存在，则删除；如果node-websocket-msg-sender.tar.gz不存在，则不进行删除操作
rm -rf node-websocket-msg-sender.tar.gz
#将当前workspace目录下的所有文件压缩到node-websocket-msg-sender.tar.gz
tar -czf node-websocket-msg-sender.tar.gz *
```
### Post-Build阶段
```shell
#!/bin/bash
echo "=====deploy node-websocket-msg-sender start====="

#获取系统当前时间
t=$(date +%y%m%d_%H%M%S)

#打开后端文件node-websocket-msg-sender所在目录
cd /home/liuyuanbing/msg

#备份当前myapp文件夹，备份后的文件夹以"myapp_ + 当前时间戳"的形式命名
cp -r node-websocket-msg-sender node-websocket-msg-sender_$t

#清理myapp的备份文件夹，最多只保留最近的两个备份文件夹
sh /home/liuyuanbing/shell/keep_most_2_node-websocket-msg-sender_copy_by_for.sh

#清空服务器当前目录下myapp文件夹中的内容
rm -rf ./node-websocket-msg-sender/*

#将myapp.tar.gz解压到服务器当前目录下的myapp文件夹中
tar vxf node-websocket-msg-sender.tar.gz -C ./node-websocket-msg-sender

#删除服务器当前目录下的myapp.tar.gz
rm -rf node-websocket-msg-sender.tar.gz

#pm2路径变量
pm2=/root/node-v8.0.0-linux-x64/bin/pm2

#www路径变量
www=/home/liuyuanbing/msg/node-websocket-msg-sender/bin/www

#pm2版本
echo "pm2 --version："
$pm2 --version

#获取当前pm2任务列表
echo "current pm2 list："
$pm2 list

#停止名字为myapp的pm2任务
$pm2 stop msg

#删除名字为myapp的pm2任务
$pm2 delete msg

#创建新的名字为myapp的pm2任务
$pm2 start $www -i 0 --name "msg" --log-date-format="YYYY-MM-DD HH:mm Z"

#重新获取pm2任务列表
echo "updated pm2 list："
$pm2 list

echo "=====deploy node-websocket-msg-sender end====="
```
### nginx
```shell
#websocket	
location ^~ /socket.io/ {
	# 转发websocket需要的设置 start
	proxy_pass http://127.0.0.1:8082;
	proxy_set_header Host $host;
	proxy_http_version 1.1; 
	proxy_set_header Upgrade $http_upgrade;
	proxy_set_header Connection "upgrade";
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_connect_timeout 60;
	proxy_read_timeout 600;
	proxy_send_timeout 600;
	# 转发websocket需要的设置 end
}

location ^~ /ws/ {
	proxy_pass http://127.0.0.1:8082;
}
#websocket
```
此处要特别注意的是，websocket提供的服务接口是以`socket.io`为特征的，因此需要针对`socket.io`进行websocket协议升级。
