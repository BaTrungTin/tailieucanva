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
1. **`e.preventDefault()`**: Ngăn browser tự động reload trang khi submit form (hành vi mặc định của HTML form)
2. **`text.trim()`**: Loại bỏ khoảng trắng thừa ở đầu và cuối chuỗi để tránh gửi tin nhắn rỗng
3. **`setIsSending(true)`**: Hiển thị loading spinner/button disabled để user biết đang gửi
4. **`sendMessage()`**: Gọi function từ Zustand store để xử lý logic gửi tin nhắn
5. **`setText("")`**: Xóa nội dung input field sau khi gửi thành công
6. **`setIsSending(false)`**: Tắt loading state trong mọi trường hợp (thành công hoặc lỗi)

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
1. **`get()`**: Lấy toàn bộ state hiện tại từ Zustand store (messages, selectedUser, selectedGroup)
2. **Kiểm tra loại chat**: 
   - Nếu `selectedGroup` có giá trị → đang chat nhóm → gửi đến API `/groups/{id}/messages`
   - Nếu `selectedUser` có giá trị → đang chat cá nhân → gửi đến API `/messages/send/{id}`
3. **`axiosInstance.post()`**: Gửi HTTP POST request với dữ liệu tin nhắn đến backend
4. **`[...messages, res.data]`: Spread operator tạo array mới bằng cách:
   - Copy tất cả tin nhắn cũ (`...messages`)
   - Thêm tin nhắn mới vào cuối (`res.data`)
5. **`set()`**: Cập nhật state trong store → trigger re-render UI tự động
6. **Error handling**: Nếu lỗi → hiển thị toast notification cho user

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
1. **Lấy dữ liệu từ request**:
   - `req.body`: Lấy nội dung tin nhắn từ frontend (text, image, groupId)
   - `req.params`: Lấy ID người nhận từ URL (ví dụ: `/messages/send/123` → receiverId = "123")
   - `req.user._id`: Lấy ID người gửi từ JWT token (đã được xác thực bởi middleware)

2. **Validation**: Kiểm tra tin nhắn không rỗng (có text hoặc image)

3. **Tạo tin nhắn mới**:
   - `new Message()`: Tạo object tin nhắn theo schema MongoDB
   - `await newMessage.save()`: Lưu tin nhắn vào database MongoDB

4. **Real-time broadcasting**:
   - Nếu là tin nhắn nhóm: `io.to('group:groupId').emit()` → gửi đến tất cả thành viên nhóm
   - Nếu là tin nhắn cá nhân: `io.to(socketId).emit()` → gửi đến người nhận cụ thể

5. **Response**: `res.status(201).json()` → trả về tin nhắn đã tạo cho frontend

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
1. **Lấy socket connection**: `useAuthStore.getState().socket` → lấy WebSocket connection hiện tại

2. **Cleanup listeners cũ**: `socket.off()` → xóa các event listener cũ để tránh:
   - Memory leak (tích lũy listeners)
   - Duplicate messages (nhận tin nhắn nhiều lần)

3. **Lắng nghe tin nhắn mới**:
   - **Chat nhóm**: `socket.on("newGroupMessage")` → chỉ xử lý tin nhắn của nhóm đang chat
   - **Chat cá nhân**: `socket.on("newMessage")` → chỉ xử lý tin nhắn từ user đang chat

4. **Validation**: Kiểm tra tin nhắn có thuộc về chat hiện tại không:
   - `newMessage.groupId !== selectedGroup._id` → tin nhắn nhóm
   - `newMessage.senderId !== selectedUser._id` → tin nhắn cá nhân

5. **Cập nhật UI**: `[...get().messages, newMessage]` → tạo array mới (immutable) và `set()` để trigger re-render

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
1. **Khi user kết nối**:
   - `socket.handshake.query.userId`: Lấy userId từ query string (frontend gửi khi connect)
   - `userSocketMap[userId] = socket.id`: Lưu mapping userId → socketId để biết user nào đang online

2. **Thông báo user online**:
   - `socket.broadcast.emit("userOnline", userId)`: Thông báo cho TẤT CẢ user khác (trừ user vừa connect)
   - `io.emit("getOnlineUsers", onlineUserIds)`: Gửi danh sách user online cho TẤT CẢ clients

3. **Khi user disconnect**:
   - `delete userSocketMap[userId]`: Xóa user khỏi map (không còn online)
   - `socket.broadcast.emit("userOffline", userId)`: Thông báo user đã offline
   - Cập nhật lại danh sách user online cho tất cả clients

4. **Quản lý state**:
   - `userSocketMap`: Object lưu trữ {userId: socketId} để biết user nào đang online
   - `Object.keys(userSocketMap)`: Lấy danh sách tất cả userId đang online

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
1. **Tạo cuộc gọi**:
   - `callId`: Tạo unique ID = `${callerId}-${receiverId}-${timestamp}` (ví dụ: "123-456-1704067200000")
   - `videoCallMap[callId]`: Lưu thông tin cuộc gọi vào memory với status "ringing"

2. **Gửi thông báo cuộc gọi**:
   - `getReceiverSocketId(receiverId)`: Tìm socket ID của người nhận cuộc gọi
   - `io.to(receiverSocketId).emit("incomingCall")`: Gửi thông báo cuộc gọi đến đến người nhận

3. **Xác nhận khởi tạo**:
   - `socket.emit("callInitiated")`: Thông báo cho người gọi rằng cuộc gọi đã được khởi tạo thành công

4. **Quản lý trạng thái**:
   - `status: "ringing"`: Cuộc gọi đang đổ chuông
   - Các trạng thái khác: "accepted", "rejected", "ended"
   - `videoCallMap`: Lưu trữ tất cả cuộc gọi đang diễn ra trong memory

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
1. **Lắng nghe cuộc gọi đến**:
   - `socket.on("incomingCall")`: Nhận thông báo cuộc gọi từ server
   - `handleIncomingCall()`: Hiển thị modal "Incoming Call" với nút Accept/Reject

2. **Xử lý chấp nhận cuộc gọi**:
   - `socket.on("callAccepted")`: Nhận thông báo cuộc gọi được chấp nhận
   - `initializePeerConnection()`: Khởi tạo WebRTC peer connection để bắt đầu video call

3. **WebRTC Signaling**:
   - `socket.on("offer")`: Nhận WebRTC offer từ bên kia (chứa thông tin media)
   - `socket.on("answer")`: Nhận WebRTC answer từ bên kia (phản hồi offer)
   - `socket.on("iceCandidate")`: Nhận ICE candidates để thiết lập kết nối P2P

4. **Flow hoàn chỉnh**:
   - User A gọi → User B nhận → User B accept → Exchange offer/answer → P2P connection established

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
1. **Lấy tin nhắn giữa 2 user**:
   - `$or`: MongoDB operator tìm tin nhắn thỏa mãn 1 trong 2 điều kiện:
     - User A gửi cho User B
     - User B gửi cho User A

2. **Đánh dấu tin nhắn đã đọc**:
   - `updateMany()`: Cập nhật tất cả tin nhắn từ user kia gửi cho mình thành `isRead: true`

3. **Tính toán unread counts**:
   - `$match`: Lọc tin nhắn chưa đọc (`isRead: false`) và không phải tin nhắn nhóm
   - `$group`: Nhóm tin nhắn theo `senderId` (người gửi)
   - `$sum: 1`: Đếm số tin nhắn chưa đọc từ mỗi người gửi

4. **Chuyển đổi dữ liệu**:
   - `unreadMap`: Chuyển từ array thành object {userId: count} để dễ sử dụng

5. **Real-time update**:
   - `io.to(socketId).emit()`: Gửi cập nhật unread counts qua socket để UI hiển thị badge số tin nhắn chưa đọc

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
