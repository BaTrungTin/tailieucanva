# 🌐 NETWORK PROGRAMMING - THUYẾT TRÌNH NGẮN

## 📝 **SCRIPT THUYẾT TRÌNH**

**"Tôi sẽ demo 3 giao thức chính trong network programming: HTTP, WebSocket và WebRTC."**

---

## 🚀 **1. GỬI TIN NHẮN - HTTP + WEBSOCKET**

### **Frontend - HTTP Request**
```javascript
// File: frontend/src/store/useChatStore.js
sendMessage: async (messageData) => {
  const { selectedUser, selectedGroup, messages } = get();
  try {
    let res;
    if (selectedGroup) {
      // Chat nhóm - gửi đến API khác
      res = await axiosInstance.post(
        `/groups/${selectedGroup._id}/messages`,
        messageData
      );
    } else if (selectedUser) {
      // Chat cá nhân - gửi đến API khác
      res = await axiosInstance.post(
        `/messages/send/${selectedUser._id}`,
        messageData
      );
    }
    // Optimistic update - cập nhật UI ngay lập tức
    set({ messages: [...messages, res.data] });
  } catch (error) {
    toast.error("Failed to send message");
  }
}
```

**🎤 SCRIPT:**
**"Đây là hybrid architecture - tôi kết hợp HTTP và WebSocket để tận dụng ưu điểm của cả hai. Đầu tiên, frontend gửi HTTP POST request đến backend."**

**"Tôi phân biệt 2 loại chat - cá nhân và nhóm - bằng cách gửi đến endpoint khác nhau. Sau khi nhận response từ server, tôi cập nhật state để hiển thị tin nhắn mới."**

### **Backend - HTTP Handler + WebSocket Broadcast**
```javascript
// File: backend/src/controllers/message.controller.js
export const sendMessage = async (req, res) => {
  try {
    const { text, image, groupId } = req.body;
    const { id: receiverId } = req.params;
    const senderId = req.user._id;

    // Tạo tin nhắn mới
    const newMessage = new Message({
      senderId,
      receiverId: groupId ? undefined : receiverId,
      groupId: groupId || undefined,
      text,
      image: imageUrl,
    });

    // Lưu vào database - persistence
    await newMessage.save();

    // Real-time broadcasting qua WebSocket
    if (groupId) {
      // Tin nhắn nhóm - gửi đến tất cả thành viên
      io.to(`group:${groupId}`).emit("newGroupMessage", newMessage);
    } else {
      // Tin nhắn cá nhân - gửi đến người nhận cụ thể
      const receiverSocketId = getReceiverSocketId(receiverId);
      if (receiverSocketId) {
        io.to(receiverSocketId).emit("newMessage", newMessage);
      }
    }

    // Trả về tin nhắn cho frontend
    res.status(201).json(newMessage);
  } catch (error) {
    res.status(500).json({ error: "Internal server error" });
  }
};
```

**🎤 SCRIPT:**
**"Backend nhận HTTP request, lưu tin nhắn vào MongoDB để đảm bảo persistence, sau đó sử dụng WebSocket để broadcast real-time đến người nhận."**

**"Đây là điểm quan trọng - tôi lưu tin nhắn vào database trước, sau đó mới broadcast. Điều này đảm bảo không mất dữ liệu ngay cả khi WebSocket fail."**

**"Tôi phân biệt tin nhắn nhóm và cá nhân - nhóm gửi đến tất cả thành viên, cá nhân gửi đến người nhận cụ thể thông qua socket ID."**

---

## 📨 **2. NHẬN TIN NHẮN REAL-TIME**

### **Frontend - WebSocket Listener**
```javascript
// File: frontend/src/store/useChatStore.js
listenMessages: () => {
  const socket = useAuthStore.getState().socket;
  if (!socket) return;
  
  const { selectedUser, selectedGroup } = get();
  
  // Cleanup listeners cũ để tránh memory leak
  socket.off("newMessage");
  socket.off("newGroupMessage");
  
  if (selectedGroup) {
    // Lắng nghe tin nhắn nhóm
    socket.on("newGroupMessage", (newMessage) => {
      if (newMessage.groupId !== selectedGroup._id) return;
      set({ messages: [...get().messages, newMessage] });
    });
  } else if (selectedUser) {
    // Lắng nghe tin nhắn cá nhân
    socket.on("newMessage", (newMessage) => {
      if (newMessage.senderId !== selectedUser._id) return;
      set({ messages: [...get().messages, newMessage] });
    });
  }
}
```

**🎤 SCRIPT:**
**"Đây là sức mạnh của WebSocket - real-time bidirectional communication. Frontend lắng nghe events từ server và cập nhật UI ngay lập tức."**

**"Tôi cleanup listeners cũ để tránh memory leak và duplicate messages. Sau đó setup listeners mới cho loại chat hiện tại - nhóm hoặc cá nhân."**

**"Khi nhận tin nhắn, tôi kiểm tra xem có thuộc về chat hiện tại không. Điều này đảm bảo tin nhắn chỉ hiển thị đúng nơi."**

**"Đây là điểm khác biệt lớn so với HTTP polling - thay vì phải liên tục hỏi server có tin nhắn mới không, server sẽ tự động push tin nhắn đến client."**

---

## 🔌 **3. SOCKET.IO CONNECTION MANAGEMENT**

### **Backend - Connection Handler**
```javascript
// File: backend/src/lib/socket.js
const userSocketMap = {}; // userId -> socketId mapping

io.on("connection", (socket) => {
  console.log("A user connected:", socket.id);

  const userId = socket.handshake.query.userId;
  if (userId) {
    // Lưu mapping userId -> socketId
    userSocketMap[userId] = socket.id;
    console.log("User online:", userId);
    
    // Thông báo user online cho tất cả user khác
    socket.broadcast.emit("userOnline", userId);
  }

  // Gửi danh sách user online cho tất cả clients
  const onlineUserIds = Object.keys(userSocketMap);
  io.emit("getOnlineUsers", onlineUserIds);

  // Xử lý khi user disconnect
  socket.on("disconnect", () => {
    console.log("A user disconnected:", socket.id);
    if (userId) {
      delete userSocketMap[userId]; // Xóa khỏi map
      socket.broadcast.emit("userOffline", userId); // Thông báo offline
    }
  });
});

// Helper function để lấy socket ID từ user ID
export function getReceiverSocketId(userId) {
  return userSocketMap[userId];
}
```

**🎤 SCRIPT:**
**"Đây là trái tim của real-time system - quản lý WebSocket connections. Server duy trì mapping giữa userId và socketId trong userSocketMap."**

**"Khi user connect, server lưu thông tin và thông báo cho các user khác. Khi disconnect, server cleanup và thông báo user offline."**

**"userSocketMap là cầu nối quan trọng - nó cho phép server gửi tin nhắn đến đúng user thông qua socket ID. Đây là cách implement targeted messaging trong WebSocket."**

**"Tôi cũng broadcast danh sách user online để frontend có thể hiển thị online status real-time."**

---

## 📹 **4. VIDEO CALL - WEBRTC SIGNALING**

### **Backend - Call Signaling**
```javascript
// File: backend/src/lib/socket.js
const videoCallMap = {}; // Lưu trữ thông tin cuộc gọi

socket.on("initiateCall", ({ receiverId, callType = "voice" }) => {
  const callerId = userId;
  const callId = `${callerId}-${receiverId}-${Date.now()}`;
  
  // Lưu thông tin cuộc gọi
  videoCallMap[callId] = {
    callerId,
    receiverId,
    callType,
    status: "ringing",
    startTime: new Date()
  };
  
  // Gửi thông báo cuộc gọi đến người nhận
  const receiverSocketId = getReceiverSocketId(receiverId);
  if (receiverSocketId) {
    io.to(receiverSocketId).emit("incomingCall", {
      callId,
      callerId,
      callType,
      callerInfo: { userId: callerId }
    });
  }
  
  // Xác nhận cuộc gọi đã được khởi tạo
  socket.emit("callInitiated", { callId, receiverId });
});
```

### **Frontend - WebRTC Handling**
```javascript
// File: frontend/src/components/VideoCallModal.jsx
useEffect(() => {
  const { socket } = useAuthStore.getState();
  if (socket) {
    // Lắng nghe cuộc gọi đến
    socket.on("incomingCall", (callData) => {
      useVideoCallStore.getState().handleIncomingCall(callData);
    });

    // Cuộc gọi được chấp nhận
    socket.on("callAccepted", ({ callId }) => {
      useVideoCallStore.getState().initializePeerConnection();
    });

    // WebRTC signaling
    socket.on("offer", ({ callId, offer }) => {
      useVideoCallStore.getState().handleOffer(offer);
    });

    socket.on("answer", ({ callId, answer }) => {
      useVideoCallStore.getState().handleAnswer(answer);
    });

    socket.on("iceCandidate", ({ callId, candidate }) => {
      useVideoCallStore.getState().handleIceCandidate(candidate);
    });
  }
}, []);
```

**🎤 SCRIPT:**
**"Đây là WebRTC signaling - server chỉ làm nhiệm vụ kết nối 2 clients, không xử lý media data. Đây là kiến trúc hybrid: WebSocket cho signaling, WebRTC cho media streaming."**

**"Khi user A muốn gọi user B, server tạo unique call ID và lưu thông tin cuộc gọi. Sau đó gửi thông báo đến user B thông qua WebSocket."**

**"Frontend xử lý WebRTC signaling thông qua các events: offer, answer, ICE candidates. Đây là quá trình phức tạp để thiết lập kết nối P2P."**

**"Sau khi signaling hoàn tất, 2 clients thiết lập kết nối P2P trực tiếp - video và audio được truyền trực tiếp mà không qua server, giảm latency và tải server."**

---

## 🔄 **5. TÓM TẮT NETWORK FLOW**

### **📤 Gửi Tin Nhắn - Complete Flow:**
1. **Frontend** → HTTP POST request → **Backend**
2. **Backend** → Lưu vào MongoDB → **Database**
3. **Backend** → WebSocket emit → **Real-time broadcast**
4. **Frontend** → Nhận event → **UI update**

### **📹 Video Call - Complete Flow:**
1. **User A** → `initiateCall` → **Server**
2. **Server** → `incomingCall` → **User B**
3. **User B** → `acceptCall` → **Server**
4. **Server** → `callAccepted` → **User A**
5. **Both users** → WebRTC signaling → **P2P connection**

### **🔌 Connection Management:**
- **HTTP**: Stateless, request-response
- **WebSocket**: Stateful, persistent connection
- **WebRTC**: P2P, direct connection

**🎤 SCRIPT KẾT LUẬN:**

**"Tôi đã demo 3 giao thức chính trong network programming:"**

**"1. HTTP - Request-response pattern cho persistence và reliability"**
**"2. WebSocket - Real-time bidirectional communication cho instant messaging"** 
**"3. WebRTC - Peer-to-peer media streaming cho video calls"**

**"Kiến trúc hybrid này tận dụng ưu điểm của từng giao thức:"**
- **HTTP**: Đáng tin cậy, có error handling tốt
- **WebSocket**: Low latency, real-time updates
- **WebRTC**: Ultra low latency, giảm tải server

**"Đây là cách implement network programming trong thực tế - không chỉ dùng 1 giao thức mà kết hợp nhiều giao thức để tạo trải nghiệm tốt nhất cho user."**

---

## 🎯 **KEY TAKEAWAYS:**

### **Network Programming Concepts:**
- ✅ **Request-Response vs Real-time**: HTTP vs WebSocket
- ✅ **Stateful vs Stateless**: Connection management
- ✅ **Client-Server vs P2P**: Architecture patterns
- ✅ **Hybrid Architecture**: Best of all worlds
- ✅ **Connection Management**: State synchronization

### **Real-world Applications:**
- ✅ **Chat Applications**: HTTP + WebSocket
- ✅ **Video Conferencing**: WebSocket + WebRTC
- ✅ **Gaming**: WebSocket + UDP
- ✅ **IoT**: MQTT + WebSocket
- ✅ **Live Streaming**: WebRTC + CDN
