<template>
  <div id="app">
    <div class="container">
      <div class="canvas-container">
        <canvas ref="canvas" @mousedown="startDrawing" @mouseup="stopDrawing" @mousemove="draw"></canvas>
        <div class="toolbar">
          <button @click="selectTool('pen')">Pen</button>
          <button @click="selectTool('eraser')">Eraser</button>
          <input type="color" v-model="penColor" />
        </div>
      </div>
      <div class="chat-container">
        <ul class="messages">
          <li v-for="msg in messages" :key="msg">{{ msg }}</li>
        </ul>
        <input v-model="message" @keydown.enter="sendMessage" placeholder="Type a message...">
      </div>
    </div>
  </div>
</template>

<script>
import io from 'socket.io-client';
import './styles/App.css'; // CSS 파일 불러오기

export default {
  data() {
    return {
      socket: null,
      drawing: false,
      message: '',
      messages: [],
      tool: 'pen', // 현재 선택된 도구
      penColor: '#000000', // 펜 색상
    };
  },
  mounted() {
    this.socket = io('http://localhost:3000');
    
    // 메시지 수신
    this.socket.on('message', (msg) => {
      this.messages.push(msg);
    });

    // 그림 데이터 수신
    this.socket.on('draw', (data) => {
      this.drawOnCanvas(data);
    });

    // 캔버스 크기 조정
    this.resizeCanvas();
    window.addEventListener('resize', this.resizeCanvas);
  },
  beforeUnmount() {
    window.removeEventListener('resize', this.resizeCanvas);
  },
  methods: {
    sendMessage() {
      if (this.message.trim() !== '') {
        this.messages.push(this.message); // 로컬 상태 업데이트
        this.socket.emit('message', this.message); // 서버로 전송
        this.message = ''; // 입력 필드 초기화
      }
    },
    selectTool(tool) {
      this.tool = tool;
    },
    startDrawing(event) {
      this.drawing = true;
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext('2d');
      const rect = canvas.getBoundingClientRect();
      const x = event.clientX - rect.left;
      const y = event.clientY - rect.top;
      ctx.beginPath();
      ctx.moveTo(x, y);
      this.draw(event);
    },
    stopDrawing() {
      this.drawing = false;
    },
    draw(event) {
      if (!this.drawing) return;

      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext('2d');

      const rect = canvas.getBoundingClientRect();

      const x = event.clientX - rect.left;
      const y = event.clientY - rect.top;

      // 현재 도구가 지우개인 경우
      if (this.tool === 'eraser') {
        ctx.globalCompositeOperation = 'destination-out';
        ctx.lineWidth = 10; // 지우개 크기
      } else {
        ctx.globalCompositeOperation = 'source-over';
        ctx.strokeStyle = this.penColor;
        ctx.lineWidth = 2; // 펜 크기
      }

      ctx.lineTo(x, y);
      ctx.stroke();

      const data = { x, y, tool: this.tool, color: this.penColor };
      this.socket.emit('draw', data);
    },
    drawOnCanvas(data) {
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext('2d');
      ctx.beginPath();
      ctx.moveTo(data.x, data.y);

      if (data.tool === 'eraser') {
        ctx.globalCompositeOperation = 'destination-out';
        ctx.lineWidth = 10;
      } else {
        ctx.globalCompositeOperation = 'source-over';
        ctx.strokeStyle = data.color;
        ctx.lineWidth = 2;
      }

      ctx.lineTo(data.x, data.y);
      ctx.stroke();
    },
    resizeCanvas() {
      const canvas = this.$refs.canvas;
      canvas.width = canvas.parentElement.clientWidth;
      canvas.height = canvas.parentElement.clientHeight;
    }
  }
};
</script>