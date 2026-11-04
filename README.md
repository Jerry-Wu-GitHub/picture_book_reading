# AI驱动的绘本阅读机器人

---

[toc]

---

## 工作流程

### 启动阶段

（应用启动时执行一次）

```mermaid
sequenceDiagram

    box 开发板
    	participant bluetooth as 蓝牙模块
    	participant wlan as WiFi模块
    end

    box 云服务
        participant session as 会话管理
    end

    box 客户端
    	participant bleak as 蓝牙
    	participant http as HTTP
    end

	bluetooth ->> bleak: 设备ID
	bleak ->> bluetooth: WLAN SSID, password
	wlan ->> session: 设备ID
	Note left of session: 心跳
	http ->> session: 设备ID
	Note right of session: 订阅
```

#### 硬件、客户端

1. 开发板广播蓝牙。
2. 客户端与开发板建立蓝牙连接（由用户手动选择要连接的对象）。
3. 开发板向客户端发送设备ID。
4. 用户在客户端输入 WiFi 名、WiFi 密码（仅首次启动时获取，缓存，之后仅需在访问不了 WLAN 时再次输入），客户端把 WiFi 名、WiFi 密码传给开发板。
5. 开发板连接 WiFi。
6. 开发板通过 WiFi 与服务端通信。之后的图像数据传输走这条线。
7. 客户端凭设备 ID 与服务端建立 SSE 连接，订阅传来的音频。
8. 开发板自己触发一个“换书”事件。

#### 服务端

1. 初始化数据库。
2. 构建 cache。

### 工作阶段

#### 客户端

客户端监听 SSE 事件：

- 如果从服务端发来音频 ID 或 URL，就把它下载下来，放入音频缓冲区队列。
- 如果从服务端发来翻页事件，就清空音频缓冲区，并停止播放正在播放的音频。

客户端要监控音频缓冲区的状态：

- 如果音频缓冲区里有待播放的音频，且当前不在播放音频，则从缓冲区里取出最老的音频并播放。

```mermaid
sequenceDiagram

    box 云服务
        participant session as 会话管理
    end

    box 客户端
        participant audio_queue as 音频队列
        participant audio_player as 音频播放
    end

	loop
		Note right of session: 订阅的SSE流
        session ->> audio_queue: 翻页事件
        session ->> audio_queue: 朗读音频
        audio_queue ->> audio_player: 取出音频
    end
```

当用户点击“换书”按钮时，向硬件发送一个“换书”事件。

```mermaid
sequenceDiagram
    autonumber

    box 开发板
        participant bluetooth as 蓝牙模块
    end

    box 客户端
        participant bleak as 蓝牙
        participant ui as 前端
    end

    Note left of ui: 按钮被点击
    ui ->> bleak: 
    bleak ->> bluetooth: 换书事件
```

#### 硬件

开发板每隔 4 秒左右要向服务端发送一次心跳。

开发板每隔大约 0.5 秒（具体间隔时间取决于硬件性能）执行一次：

1. 获取摄像头输入。
2. 从照片中提取书页并展平+向量化。
3. 通过当前向量与之前的向量比较，判断是否翻了页。
4. 如果翻页了，向服务端 API 发送书页图像、设备ID。

```mermaid
sequenceDiagram

    box 开发板
        participant camera as 摄像头
        participant detect_page_turning as 翻页检测
    end

    box 云服务
    	participant submit_page_turning as 提交接口
    end

	loop 每一帧持续进行
		Note right of camera: 拍摄
        camera ->> detect_page_turning: 图片
        detect_page_turning ->> submit_page_turning: 设备ID, 书本ID, 图片
        Note left of submit_page_turning: 提交翻页事件
    end
```

当接收到换书事件时：

1. 停止拍摄循环。
2. 获取摄像头输入。
3. 向服务端查询封面所属的书。
4. 获取并记录书本ID。

```mermaid
sequenceDiagram
    autonumber

    box 客户端
    	participant bleak as 蓝牙
    end

    box 开发板
    	participant bluetooth as 蓝牙
        participant controller as 控制器
        participant camera as 摄像头
    end

    box 云服务
        participant query_book as 查询接口
    end

	bleak ->> bluetooth: 翻页事件
	bluetooth ->> controller: 停止拍摄循环
	camera ->> query_book: 设备ID, 封面
	query_book ->> controller: 书本ID
	controller ->> controller: 继续拍摄循环
```

#### 服务端

服务端收到硬件发来的书本查询请求时：

1. 在数据库里查询书本ID，返回给硬件。
2. 向订阅了该设备的客户端推送 换书时要播放的音频。

```mermaid
sequenceDiagram

	participant hardware as 硬件

    box 云服务
    	participant query_book as 查询接口
    	participant database as 数据库
        participant session as 会话管理
    end

    participant client as 客户端

	hardware ->> query_book: 设备ID, 封面
	query_book ->> database: 封面
	database ->> query_book: 书本ID
	query_book ->> session: 设备ID, 书本ID
	query_book ->> hardware: 书本ID
	session ->> database: 书本ID
	database ->> session: 音频
	session ->> client: 音频
```

服务器收到硬件发来的翻页事件后：

1. 向量化书页图像，检查 cache。
    - cache 命中 → 直接向客户端推送朗读音频。
    - cache 未命中 → 2
2. 从记忆模块取出该设备的前文梗概，把书页图像和前文梗概传给视觉大模型，描述本页概要和朗读内容。
3. 将本页概要存入记忆模块。
4. 向客户端发送翻页事件。
5. 一旦攒满一句朗读内容（以标点分割），就立刻 TTS，并把得到的音频发送给订阅该设备的客户端。直到朗读音频传输完。
6. 将本页概要、朗读内容、朗读音频存入 cache 和数据库。

```mermaid
sequenceDiagram

	participant hardware as 硬件

    box 云服务
    	participant submit_page_turning as 提交接口
    	participant narrator as Narrator
    	participant memory as 记忆模块
        participant session as 会话管理
    end

    participant client as 客户端

    hardware ->> submit_page_turning: 设备ID, 书本ID, 图片
	submit_page_turning ->> memory: 设备ID
	memory ->> narrator: 前文梗概
	submit_page_turning ->> narrator: 书本ID, 图片
	submit_page_turning ->> session: 设备ID
	session ->> client: 翻页事件
	narrator ->> memory: 场景梗概
	narrator ->> session: 朗读音频
	session ->> client: 音频
```