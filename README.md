# simple-drawing-backend

此项目使用 GO 语言编写了一个简单白板（涂鸦）服务端功能。 所使用的通信协议是 WebSocket，因此所有的用户都将实时看到彼此的绘制路径。其次，用户还可以为画笔设置颜色等。

首先，我们需要创建一个用于与用户交互消息的桥梁（Hub）。这个思路类似于Gorilla's 的 chat 例子。

Client struct
创建一个 client.go 文件
```
package main

import (
  "github.com/gorilla/websocket"
  uuid "github.com/satori/go.uuid"
)

type Client struct {
  id       string
  hub      *Hub
  color    string
  socket   *websocket.Conn
  outbound chan []byte
}
为client编写一个构造方法，这里使用了UUID和随机颜色库

func newClient(hub *Hub, socket *websocket.Conn) *Client {
  return &Client{
    id:       uuid.NewV4().String(),
    color:    generateColor(),
    hub:      hub,
    socket:   socket,
    outbound: make(chan []byte),
  }
}
```
新建 utilities.go 文件来编程 generateColor 方法
```
package main

import (
  "math/rand"
  "time"
  colorful "github.com/lucasb-eyer/go-colorful"
)

func init() {
  rand.Seed(time.Now().UnixNano())
}

func generateColor() string {
  c := colorful.Hsv(rand.Float64()*360.0, 0.8, 0.8)
  return c.Hex()
}
```
写一个用于到Hub读取消息 read 的方法，如果有错误发生，将会通知unregistered通道
```
func (client *Client) read() {
  defer func() {
    client.hub.unregister <- client
  }()
  for {
    _, data, err := client.socket.ReadMessage()
    if err != nil {
      break
    }
    client.hub.onMessage(data, client)
  }
}
```
write方法从outbound通道获取消息并发送给用户。这样，服务器将能够发送消息到客户端。
```
func (client *Client) write() {
  for {
    select {
    case data, ok := <-client.outbound:
      if !ok {
        client.socket.WriteMessage(websocket.CloseMessage, []byte{})
        return
      }
      client.socket.WriteMessage(websocket.TextMessage, data)
    }
  }
}
```
在 client struct中添加 启动 和 结束 进程的方法，并且在启动方法中使用 goroutine 运行 read 和 write 方法
```
func (client Client) run() {
  go client.read()
  go client.write()
}

func (client Client) close() {
  client.socket.Close()
  close(client.outbound)
}
```
#### Hub struct
新建 hub.go 文件并声明 Hub struct
```
package main

import (
  "encoding/json"
  "log"
  "net/http"

  "github.com/gorilla/websocket"
  "github.com/tidwall/gjson"
)

type Hub struct {
  clients    []*Client
  register   chan *Client
  unregister chan *Client
}
```
添加构造方法
```
func newHub() *Hub {
  return &Hub{
    clients:    make([]*Client, 0),
    register:   make(chan *Client),
    unregister: make(chan *Client),
  }
}
```
添加 run 方法
```
func (hub *Hub) run() {
  for {
    select {
    case client := <-hub.register:
      hub.onConnect(client)
    case client := <-hub.unregister:
      hub.onDisconnect(client)
    }
  }
}
```
编写一个将http升级到WebSockets请求的方法。 如果升级成功，客户端将被添加到 clients 中。
```
var upgrader = websocket.Upgrader{
  // Allow all origins
  CheckOrigin: func(r *http.Request) bool { return true },
}

func (hub *Hub) handleWebSocket(w http.ResponseWriter, r *http.Request) {
  socket, err := upgrader.Upgrade(w, r, nil)
  if err != nil {
    log.Println(err)
    http.Error(w, "could not upgrade", http.StatusInternalServerError)
    return
  }
  client := newClient(hub, socket)
  hub.clients = append(hub.clients, client)
  hub.register <- client
  client.run()
}
```
编写一个发送消息到客户端的方法
```
func (hub *Hub) send(message interface{}, client *Client) {
  data, _ := json.Marshal(message)
  client.outbound <- data
}
```
编写一个广播(broadcast)消息到所有客户端的方法（排除自己）
```
func (hub *Hub) broadcast(message interface{}, ignore *Client) {
  data, _ := json.Marshal(message)
  for _, c := range hub.clients {
    if c != ignore {
      c.outbound <- data
    }
  }
}
```
####Messages
Messages 将使用JSON作为交互格式。 每条消息将携带一个“kind”字段，以区分消息

新建messages.go文件并创建 message 包

声明所有消息“kinds”的枚举
```
package message

const (
  // KindConnected is sent when user connects
  KindConnected = iota + 1
  // KindUserJoined is sent when someone else joins
  KindUserJoined
  // KindUserLeft is sent when someone leaves
  KindUserLeft
  // KindStroke message specifies a drawn stroke by a user
  KindStroke
  // KindClear message is sent when a user clears the screen
  KindClear
)
```
声明一些简单的数据结构
```
type Point struct {
  X int `json:"x"`
  Y int `json:"y"`
}

type User struct {
  ID    string `json:"id"`
  Color string `json:"color"`
}
```
声明所有的消息类型结构体并编写 构造函数。kind 字段在构造函数中设置
```
type Connected struct {
  Kind  int    `json:"kind"`
  Color string `json:"color"`
  Users []User `json:"users"`
}

func NewConnected(color string, users []User) *Connected {
  return &Connected{
    Kind:  KindConnected,
    Color: color,
    Users: users,
  }
}

type UserJoined struct {
  Kind int  `json:"kind"`
  User User `json:"user"`
}

func NewUserJoined(userID string, color string) *UserJoined {
  return &UserJoined{
    Kind: KindUserJoined,
    User: User{ID: userID, Color: color},
  }
}

type UserLeft struct {
  Kind   int    `json:"kind"`
  UserID string `json:"userId"`
}

func NewUserLeft(userID string) *UserLeft {
  return &UserLeft{
    Kind:   KindUserLeft,
    UserID: userID,
  }
}

type Stroke struct {
  Kind   int     `json:"kind"`
  UserID string  `json:"userId"`
  Points []Point `json:"points"`
  Finish bool    `json:"finish"`
}

type Clear struct {
  Kind   int    `json:"kind"`
  UserID string `json:"userId"`
}
```
####Handling message flow
返回hub.go文件，添加所有缺少的功能。

onConnect函数表示客户端连接，在 run方法中调用。 它将用户的画笔颜色和其他用户的信息发送给客户端。 它还将当前连接的用户信息通知给其他在线用户。
```
func (hub *Hub) onConnect(client *Client) {
  log.Println("client connected: ", client.socket.RemoteAddr())
  // Make list of all users
  users := []message.User{}
  for _, c := range hub.clients {
    users = append(users, message.User{ID: c.id, Color: c.color})
  }
  // Notify user joined
  hub.send(message.NewConnected(client.color, users), client)
  hub.broadcast(message.NewUserJoined(client.id, client.color), client)
}

onDisconnect函数从 clients 中删除断开连接的客户端，并通知其他人有人离开。

func (hub *Hub) onDisconnect(client *Client) {
  log.Println("client disconnected: ", client.socket.RemoteAddr())
  client.close()
  // Find index of client
  i := -1
  for j, c := range hub.clients {
    if c.id == client.id {
      i = j
      break
    }
  }
  // Delete client from list
  copy(hub.clients[i:], hub.clients[i+1:])
  hub.clients[len(hub.clients)-1] = nil
  hub.clients = hub.clients[:len(hub.clients)-1]
  // Notify user left
  hub.broadcast(message.NewUserLeft(client.id), nil)
}

```
每当从客户端收到消息时，都会调用onMessage函数。 首先通过使用tidwall/gjson包来读取它是什么样的消息，然后分别处理每个情况。

在这个例子中，情况都是相似的。 每个消息获得用户的ID，然后转发给其他客户端。
```
func (hub *Hub) onMessage(data []byte, client *Client) {
  kind := gjson.GetBytes(data, "kind").Int()
  if kind == message.KindStroke {
    var msg message.Stroke
    if json.Unmarshal(data, &msg) != nil {
      return
    }
    msg.UserID = client.id
    hub.broadcast(msg, client)
  } else if kind == message.KindClear {
    var msg message.Clear
    if json.Unmarshal(data, &msg) != nil {
      return
    }
    msg.UserID = client.id
    hub.broadcast(msg, client)
  }
}
```
最后，编写main.go文件
```
package main

import (
  "log"
  "net/http"
)

func main() {
  hub := newHub()
  go hub.run()
  http.HandleFunc("/ws", hub.handleWebSocket)
  err := http.ListenAndServe(":3000", nil)
  if err != nil {
    log.Fatal(err)
  }
}
```
#####Front-end app
前端应用程序将用纯JavaScript编写。 在client目录创建index.html文件
```
<!DOCTYPE html>
<html>
<head>
  <title>Collaborative Drawing App</title>
  <style>
    #canvas {
      border: 1px solid #000;
    }
  </style>
</head>
<body>
  <canvas id="canvas"
          width="480"
          height="360">
  </canvas>
  <div>
    <button id="clearButton">Clear</button>
  </div>
  <script>
    MESSAGE_CONNECTED = 1;
    MESSAGE_USER_JOINED = 2;
    MESSAGE_USER_LEFT = 3;
    MESSAGE_STROKE = 4;
    MESSAGE_CLEAR = 5;

    window.onload = function () {}
  </script>
</body>
</html>
```
上面的代码会创建一个画布和一个清除按钮。 以下所有JavaScript代码都编写在window.onload事件处理程序中。

#####Drawing on canvas
声明一些变量
```
var canvas = document.getElementById('canvas');
var ctx = canvas.getContext("2d");
var isDrawing = false;
var strokeColor = '';
var strokes = [];
```
编写canvas处理事件
```
canvas.onmousedown = function (event) {
  isDrawing = true;
  addPoint(event.pageX - this.offsetLeft, event.pageY - this.offsetTop, true);
};

canvas.onmousemove = function (event) {
  if (isDrawing) {
    addPoint(event.pageX - this.offsetLeft, event.pageY - this.offsetTop);
  }
};

canvas.onmouseup = function () {
  isDrawing = false;
};

canvas.onmouseleave = function () {
  isDrawing = false;
};
```
编写addPoint方法，strokes是一个画笔数组，存储所有的点。
```
function addPoint(x, y, newStroke) {
  var p = { x: x, y: y };
  if (newStroke) {
    strokes.push([p]);
  } else {
    strokes[strokes.length - 1].push(p);
  }
  update();
}
```
update 方法重绘
```
function update() {
  ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
  ctx.lineJoin = 'round';
  ctx.lineWidth = 4;

  ctx.strokeStyle = strokeColor;
  drawStrokes(strokes);
}
```
drawStrokes 绘制多个路径
```
function drawStrokes(strokes) {
  for (var i = 0; i < strokes.length; i++) {
    ctx.beginPath();
    for (var j = 1; j < strokes[i].length; j++) {
      var prev = strokes[i][j - 1];
      var current = strokes[i][j];
      ctx.moveTo(prev.x, prev.y);
      ctx.lineTo(current.x, current.y);
    }
    ctx.closePath();
    ctx.stroke();
  }
}
```
清除 点击事件
```
document.getElementById('clearButton').onclick = function () {
  strokes = [];
  update();
};
```
#####Server communication
要与服务器通信，请首先声明一些额外的变量。
```
var socket = new WebSocket("ws://localhost:3000/ws");
var otherColors = {};
var otherStrokes = {};
```
otherColors对象将保存其他客户端的颜色，其中key将是用户ID。 otherStrokes将保存绘图数据。

在addPoint函数中增加发送消息。 对于这个例子，points数组只有一个点。 理想情况下，分数将根据一些标准分批发送。
```
function addPoint(x, y, newStroke) {
  var p = { x: x, y: y };
  if (newStroke) {
    strokes.push([p]);
  } else {
    strokes[strokes.length - 1].push(p);
  }
  socket.send(JSON.stringify({ kind: MESSAGE_STROKE, points: [p], finish: newStroke }));
  update();
}
```
处理发送 "clear" 消息
```
document.getElementById('clearButton').onclick = function () {
  strokes = [];
  socket.send(JSON.stringify({ kind: MESSAGE_CLEAR }));
  update();
};
```
onmessage 处理函数
```
socket.onmessage = function (event) {
  var messages = event.data.split('\n');
  for (var i = 0; i < messages.length; i++) {
    var message = JSON.parse(messages[i]);
    onMessage(message);
  }
};

function onMessage(message) {
  switch (message.kind) {
    case MESSAGE_CONNECTED:
      break;
    case MESSAGE_USER_JOINED:
      break;
    case MESSAGE_USER_LEFT:
      break;
    case MESSAGE_STROKE:
      break;
    case MESSAGE_CLEAR:
      break;
  }
}
```
对于 MESSAGE_CONNECTED 情况，设置用户的画笔颜色并用给定的信息填充“other”对象。
```
strokeColor = message.color;
for (var i = 0; i < message.users.length; i++) {
  var user = message.users[i];
  otherColors[user.id] = user.color;
  otherStrokes[user.id] = [];
}
```
对于MESSAGE_USER_JOINED的情况，设置用户的颜色并准备一个空的笔划数组。
```
otherColors[message.user.id] = message.user.color;
otherStrokes[message.user.id] = [];
在MESSAGE_USER_LEFT的情况下，如果有人离开，需要删除他的数据，并从画布上清除他的绘画。

delete otherColors[message.userId];
delete otherStrokes[message.userId];
update();
```
在MESSAGE_STROKE的情况下，更新用户的笔画数组。
```
if (message.finish) {
  otherStrokes[message.userId].push(message.points);
} else {
  var strokes = otherStrokes[message.userId];
  strokes[strokes.length - 1] = strokes[strokes.length - 1].concat(message.points);
}
update();
```
对于MESSAGE_CLEAR情况，只需清除用户的笔划数组。
```
otherStrokes[message.userId] = [];
update();
```
更新update方法以显示他人的图纸。

```
function update() {
  ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
  ctx.lineJoin = 'round';
  ctx.lineWidth = 4;
  // Draw mine
  ctx.strokeStyle = strokeColor;
  drawStrokes(strokes);
  // Draw others'
  var userIds = Object.keys(otherColors);
  for (var i = 0; i < userIds.length; i++) {
    var userId = userIds[i];
    ctx.strokeStyle = otherColors[userId];
    drawStrokes(otherStrokes[userId]);
  }
}
```

### Install dependeice

```
$ glide install
```

### Running

Buld and run the server.

```
$ go build -o server && ./server
```

Open client/index.html in your browser.

### Snapshot

![效果截图](snap.gif)
