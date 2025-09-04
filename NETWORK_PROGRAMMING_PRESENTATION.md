# 🌐 NETWORK PROGRAMMING - THUYẾT TRÌNH

## 📋 **Mục Lục**
1. [Gửi Tin Nhắn Cá Nhân](#gửi-tin-nhắn-cá-nhân)
2. [Nhận Tin Nhắn Real-time](#nhận-tin-nhắn-real-time)
3. [Socket.IO Connection](#socketio-connection)
4. [Video Call - WebRTC Signaling](#video-call---webrtc-signaling)
5. [Quản Lý Tin Nhắn Chưa Đọc](#quản-lý-tin-nhắn-chưa-đọc)
6. [Flow Hoàn Chỉnh](#flow-hoàn-chỉnh)

---

## 🚀 **GỬI TIN NHẮN CÁ NHÂN**

### **1. Frontend - User Interface**

```javascript
// File: frontend/src/components/MessageInput.jsx
const handleSendMessage = async (e) => {
  e.preventDefault();
  if (!text.trim() && !imagePreview) return;

  setIsSending(true);
  try {
    await sendMessage({
      text: text.trim(),
      image: imagePreview,
    });
    setText("");
    setImagePreview(null);
  } catch (error) {
    console.error("Error sending message:", error);
  } finally {
    setIsSending(false);
  }
};
```

**Cách hoạt động:**
- `e.preventDefault()`: Ngăn browser reload trang khi submit form
- `text.trim()`: Loại bỏ khoảng trắng đầu/cuối chuỗi
- `setIsSending(true/false)`: Quản lý trạng thái loading UI
- `sendMessage()`: Gọi function từ store để xử lý gửi tin nhắn
- `setText("")`: Xóa nội dung input sau khi gửi thành công

### **2. Frontend - State Management**

```javascript
// File: frontend/src/store/useChatStore.js
sendMessage: async (messageData) => {
  const { selectedUser, selectedGroup, messages } = get();
  try {
    let res;
    if (selectedGroup) {
      res = await axiosInstance.post(
        `/groups/${selectedGroup._id}/messages`,
        messageData
      );
    } else if (selectedUser) {
      res = await axiosInstance.post(
        `/messages/send/${selectedUser._id}`,
        messageData
      );
    } else {
      toast.error("No chat selected");
      return;
    }
    set({ messages: [...messages, res.data] });
  } catch (error) {
    toast.error(error.response?.data?.message || "Failed to send message");
  }
},
```

**Cách hoạt động:**
- `get()`: Lấy state hiện tại từ Zustand store
- `selectedUser/selectedGroup`: Xác định loại chat (cá nhân/nhóm)
- `axiosInstance.post()`: Gửi HTTP POST request đến backend API
- `[...messages, res.data]`: Spread operator để thêm tin nhắn mới vào array
- `set()`: Cập nhật state trong store để UI re-render

### **3. Backend - Controller**

```javascript
// File: backend/src/controllers/message.controller.js
export const sendMessage = async (req, res) => {
  try {
    const { text, image, groupId } = req.body;
    const { id: receiverId } = req.params;
    const senderId = req.user._id;

    if (!text && !image) {
      return res.status(400).json({ error: "Tin nhắn rỗng" });
    }

    const newMessage = new Message({
      senderId,
      receiverId: groupId ? undefined : receiverId,
      groupId: groupId || undefined,
      text,
      image: imageUrl,
    });

    await newMessage.save();

    if (groupId) {
      io.to(`group:${groupId}`).emit("newGroupMessage", newMessage);
    } else {
      const receiverSocketId = getReceiverSocketId(receiverId);
      if (receiverSocketId) {
        io.to(receiverSocketId).emit("newMessage", newMessage);
      }
    }

    res.status(201).json(newMessage);
  } catch (error) {
    console.error("Error in sendMessage:", error.message);
    res.status(500).json({ error: "Internal server error" });
  }
};
```

**Cách hoạt động:**
- `req.body`: Lấy dữ liệu từ request body (text, image, groupId)
- `req.params`: Lấy tham số từ URL (receiverId)
- `req.user._id`: Lấy user ID từ JWT middleware
- `new Message()`: Tạo instance của Mongoose model
- `await newMessage.save()`: Lưu tin nhắn vào MongoDB
- `io.to(socketId).emit()`: Gửi event real-time qua Socket.IO
- `res.status(201).json()`: Trả về tin nhắn đã tạo cho frontend

---

## 📨 **NHẬN TIN NHẮN REAL-TIME**

### **Frontend - Socket Listener**

```javascript
// File: frontend/src/store/useChatStore.js
listenMessages: () => {
  const socket = useAuthStore.getState().socket;
  if (!socket) return;
  
  const { selectedUser, selectedGroup } = get();
  
  socket.off("newMessage");
  socket.off("newGroupMessage");
  
  if (selectedGroup) {
    socket.on("newGroupMessage", (newMessage) => {
      if (newMessage.groupId !== selectedGroup._id) return;
      set({ messages: [...get().messages, newMessage] });
    });
  } else if (selectedUser) {
    socket.on("newMessage", (newMessage) => {
      if (newMessage.senderId !== selectedUser._id) return;
      set({ messages: [...get().messages, newMessage] });
    });
  }
},
```

**Cách hoạt động:**
- `socket.off()`: Xóa event listener cũ để tránh memory leak
- `socket.on()`: Lắng nghe event từ server
- `newMessage.groupId !== selectedGroup._id`: Kiểm tra tin nhắn có thuộc nhóm hiện tại không
- `newMessage.senderId !== selectedUser._id`: Kiểm tra tin nhắn có từ user đang chat không
- `[...get().messages, newMessage]`: Immutable update - tạo array mới với tin nhắn mới
- `set()`: Cập nhật state để UI tự động re-render

---

## 🔌 **SOCKET.IO CONNECTION**

### **Backend - Socket Server**

```javascript
// File: backend/src/lib/socket.js
const userSocketMap = {};

io.on("connection", (socket) => {
  console.log("A user connected:", socket.id);

  const userId = socket.handshake.query.userId;
  if (userId) {
    userSocketMap[userId] = socket.id;
    console.log("User online:", userId);
    
    socket.broadcast.emit("userOnline", userId);
  }

  const onlineUserIds = Object.keys(userSocketMap);
  io.emit("getOnlineUsers", onlineUserIds);

  socket.on("disconnect", () => {
    console.log("A user disconnected:", socket.id);
    if (userId) {
      delete userSocketMap[userId];
      socket.broadcast.emit("userOffline", userId);
    }
  });
});
```

**Cách hoạt động:**
- `userSocketMap`: Object lưu trữ mapping giữa userId và socketId
- `socket.handshake.query.userId`: Lấy userId từ query string khi connect
- `userSocketMap[userId] = socket.id`: Lưu mapping để biết user nào đang online
- `socket.broadcast.emit()`: Gửi event cho tất cả client trừ client hiện tại
- `io.emit()`: Gửi event cho tất cả clients
- `delete userSocketMap[userId]`: Xóa user khỏi map khi disconnect
- `Object.keys(userSocketMap)`: Lấy danh sách tất cả user đang online

---

## 📹 **VIDEO CALL - WEBRTC SIGNALING**

### **Backend - Call Signaling**

```javascript
// File: backend/src/lib/socket.js
const videoCallMap = {};

socket.on("initiateCall", ({ receiverId, callType = "voice" }) => {
  const callerId = userId;
  const callId = `${callerId}-${receiverId}-${Date.now()}`;
  
  videoCallMap[callId] = {
    callerId,
    receiverId,
    callType,
    status: "ringing",
    startTime: new Date()
  };
  
  const receiverSocketId = getReceiverSocketId(receiverId);
  if (receiverSocketId) {
    io.to(receiverSocketId).emit("incomingCall", {
      callId,
      callerId,
      callType,
      callerInfo: { userId: callerId }
    });
  }
  
  socket.emit("callInitiated", { callId, receiverId });
});
```

**Cách hoạt động:**
- `callId`: Tạo unique identifier cho cuộc gọi (callerId-receiverId-timestamp)
- `videoCallMap`: Object lưu trữ thông tin các cuộc gọi đang diễn ra
- `status: "ringing"`: Trạng thái cuộc gọi (ringing, accepted, rejected, ended)
- `getReceiverSocketId()`: Lấy socket ID của người nhận cuộc gọi
- `io.to(receiverSocketId).emit()`: Gửi event đến socket cụ thể
- `socket.emit("callInitiated")`: Xác nhận cuộc gọi đã được khởi tạo

### **Frontend - Call Handling**

```javascript
// File: frontend/src/components/VideoCallModal.jsx
useEffect(() => {
  const { socket } = useAuthStore.getState();
  if (socket) {
    socket.on("incomingCall", (callData) => {
      console.log("Incoming call received:", callData);
      useVideoCallStore.getState().handleIncomingCall(callData);
    });

    socket.on("callAccepted", ({ callId }) => {
      console.log("Call accepted event received:", callId);
      useVideoCallStore.getState().initializePeerConnection();
    });

    socket.on("offer", ({ callId, offer }) => {
      console.log("Offer received:", callId);
      useVideoCallStore.getState().handleOffer(offer);
    });

    socket.on("answer", ({ callId, answer }) => {
      console.log("Answer received:", callId);
      useVideoCallStore.getState().handleAnswer(answer);
    });

    socket.on("iceCandidate", ({ callId, candidate }) => {
      console.log("ICE candidate received:", callId);
      useVideoCallStore.getState().handleIceCandidate(candidate);
    });
  }
}, []);
```

**Cách hoạt động:**
- `socket.on("incomingCall")`: Lắng nghe cuộc gọi đến
- `handleIncomingCall()`: Xử lý hiển thị modal cuộc gọi
- `socket.on("callAccepted")`: Lắng nghe khi cuộc gọi được chấp nhận
- `initializePeerConnection()`: Khởi tạo WebRTC peer connection
- `socket.on("offer")`: Nhận WebRTC offer từ bên kia
- `socket.on("answer")`: Nhận WebRTC answer từ bên kia
- `socket.on("iceCandidate")`: Nhận ICE candidates để thiết lập connection

---

## 📬 **QUẢN LÝ TIN NHẮN CHƯA ĐỌC**

### **Backend - Unread Count Management**

```javascript
// File: backend/src/controllers/message.controller.js
export const getMessages = async (req, res) => {
  try {
    const { id: userToChatId } = req.params;
    const myId = req.user._id;

    const messages = await Message.find({
      $or: [
        { senderId: myId, receiverId: userToChatId },
        { senderId: userToChatId, receiverId: myId },
      ],
    }).sort({ createdAt: 1 });

    const updateResult = await Message.updateMany(
      { senderId: userToChatId, receiverId: myId, isRead: false },
      { isRead: true }
    );

    const unreadCounts = await Message.aggregate([
      {
        $match: {
          receiverId: myId,
          isRead: false,
          groupId: { $exists: false }
        }
      },
      {
        $group: {
          _id: "$senderId",
          count: { $sum: 1 }
        }
      }
    ]);
    
    const unreadMap = {};
    unreadCounts.forEach(item => {
      unreadMap[item._id.toString()] = item.count;
    });

    const receiverSocketId = getReceiverSocketId(myId);
    if (receiverSocketId) {
      io.to(receiverSocketId).emit("unreadCountsUpdate", unreadMap);
    }

    res.status(200).json(messages);
  } catch (error) {
    console.error("Error in getMessages:", error.message);
    res.status(500).json({ error: "Internal server error" });
  }
};
```

**Cách hoạt động:**
- `$or`: MongoDB operator - tìm tin nhắn thỏa mãn 1 trong 2 điều kiện
- `updateMany()`: Cập nhật nhiều document cùng lúc (đánh dấu đã đọc)
- `aggregate()`: MongoDB aggregation pipeline để tính toán phức tạp
- `$match`: Lọc documents theo điều kiện (tin nhắn chưa đọc)
- `$group`: Nhóm documents theo senderId
- `$sum: 1`: Đếm số documents trong mỗi group
- `unreadMap`: Chuyển đổi kết quả thành object {userId: count}
- `io.to(socketId).emit()`: Gửi cập nhật unread counts qua socket

---

## 🔄 **FLOW HOÀN CHỈNH**

### **Gửi Tin Nhắn - Step by Step**

1. **User nhập tin nhắn** → `MessageInput.jsx`
2. **Click Send** → `handleSendMessage()`
3. **Store xử lý** → `useChatStore.sendMessage()`
4. **HTTP Request** → `axiosInstance.post()`
5. **Backend nhận** → `message.controller.sendMessage()`
6. **Lưu Database** → `newMessage.save()`
7. **Socket emit** → `io.to(socketId).emit("newMessage")`
8. **Real-time receive** → `socket.on("newMessage")`
9. **UI update** → `set({ messages: [...messages, newMessage] })`

### **Video Call - Step by Step**

1. **User A click call** → `socket.emit("initiateCall")`
2. **Backend signaling** → Tạo `callId`, lưu vào `videoCallMap`
3. **Notify User B** → `io.to(receiverSocketId).emit("incomingCall")`
4. **User B nhận call** → `socket.on("incomingCall")`
5. **User B accept** → `socket.emit("acceptCall")`
6. **Backend confirm** → `io.to(callerSocketId).emit("callAccepted")`
7. **WebRTC setup** → Exchange offer/answer/ICE candidates
8. **P2P connection** → Direct video/audio stream
9. **UI display** → Show video call interface

### **Unread Counts - Step by Step**

1. **User mở chat** → `getMessages()` API call
2. **Backend lấy tin nhắn** → `Message.find()`
3. **Đánh dấu đã đọc** → `Message.updateMany()`
4. **Tính unread counts** → `Message.aggregate()`
5. **Socket emit** → `io.to(socketId).emit("unreadCountsUpdate")`
6. **Frontend nhận** → `socket.on("unreadCountsUpdate")`
7. **UI update** → Hiển thị badge số tin nhắn chưa đọc

---

## 🎯 **TÓM TẮT NETWORK PROGRAMMING**

### **3 Giao Thức Chính:**
- **HTTP/HTTPS**: RESTful APIs cho CRUD operations
- **WebSocket**: Real-time messaging và notifications
- **WebRTC**: Peer-to-peer video/voice calls

### **Kiến Trúc:**
- **Frontend**: React + Zustand + Socket.IO Client
- **Backend**: Node.js + Express + Socket.IO Server
- **Database**: MongoDB với Mongoose ODM
- **Real-time**: Socket.IO cho messaging, WebRTC cho calls

### **Lợi Ích:**
- ✅ Real-time communication
- ✅ Low latency video calls
- ✅ Scalable architecture
- ✅ Secure data transmission
- ✅ Cross-platform compatibility
- ✅ Smart message tracking

---

*Tài liệu này được tạo để hỗ trợ thuyết trình về Network Programming trong dự án Chat Ting Ting.*
