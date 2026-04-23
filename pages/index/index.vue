<template>
  <view class="container">
    <!-- 首页 -->
    <view v-if="view === 'home'" class="home-view">
      <view class="header">
        <text class="game-font title">💞 恋爱飞行棋</text>
        <text class="subtitle">ROMANTIC GAME V3.0</text>
      </view>
      
      <view class="mode-list">
        <view 
          v-for="(mode, key) in modes" 
          :key="key"
          class="mode-item"
          @click="initGame(key)"
        >
          <view class="mode-content">
            <text class="mode-icon">{{ mode.icon }}</text>
            <text class="mode-name">{{ mode.name }}</text>
          </view>
          <text class="arrow">❯</text>
        </view>
      </view>
    </view>

    <!-- 游戏页面 -->
    <view v-if="view === 'game'" class="game-view">
      <!-- 头部 -->
      <view class="game-header">
        <view class="back-btn" @click="goHome">🏠</view>
        <view class="mode-badge">
          <text class="game-font">{{ modes[currentMode].name }}</text>
        </view>
        <view v-if="currentMode === 'custom'" class="edit-btn" :class="{ active: isEditing }" @click="toggleEditMode">
          {{ isEditing ? '完成' : '编辑' }}
        </view>
        <view v-else class="placeholder"></view>
      </view>

      <!-- 棋盘 -->
      <view class="board-container">
        <view 
          v-for="(cell, index) in board" 
          :key="index"
          class="cell"
          :class="getCellBg(index)"
          :style="{ left: cell.x + 'rpx', top: cell.y + 'rpx' }"
          @click="handleCellClick(index)"
        >
          <text v-if="index === 0" class="go-text">GO</text>
          <text v-else-if="index === 39" class="flag-text">🏁</text>
          <template v-else>
            <text 
              class="cell-number"
              :class="cell.task || currentMode !== 'custom' ? 'text-white' : 'text-rose'"
            >{{ index }}</text>
            <text v-if="currentMode === 'custom' && cell.task" class="star-text">★</text>
          </template>

          <template v-if="!isEditing">
            <view v-if="playerPos[0] === index" class="piece piece-girl">👧</view>
            <view v-if="playerPos[1] === index" class="piece piece-boy">👦</view>
          </template>
        </view>
      </view>

      <!-- 控制区 -->
      <view v-show="!isEditing" class="control-panel">
        <view class="players-info">
          <view class="player girl" :class="{ active: currentPlayer === 0 }">
            <view class="player-avatar" :class="{ active: currentPlayer === 0 }">👧</view>
            <text class="player-label" :class="{ active: currentPlayer === 0 }">女生回合</text>
          </view>

          <view class="dice-area">
            <text class="dice-value" :class="{ rolling: isRolling }">
              {{ isRolling ? '🎲' : diceIcons[diceValue - 1] }}
            </text>
          </view>

          <view class="player boy" :class="{ active: currentPlayer === 1 }">
            <view class="player-avatar" :class="{ active: currentPlayer === 1 }">👦</view>
            <text class="player-label" :class="{ active: currentPlayer === 1 }">男生回合</text>
          </view>
        </view>

        <button class="roll-btn" @click="roll" :disabled="isRolling">
          {{ isRolling ? '正在掷骰子...' : '掷骰子' }}
        </button>
      </view>

      <!-- 编辑提示 -->
      <view v-if="isEditing" class="edit-hint">
        <text class="hint-text">🛠️ 点击棋盘格设置专属任务</text>
        <text class="clear-btn" @click="clearCustomBoard">清空数据</text>
      </view>
    </view>

    <!-- 任务弹窗 -->
    <view v-if="modal.show" class="modal-overlay" @click="modal.show = false">
      <view class="modal-content" @click.stop>
        <text class="modal-icon">{{ modal.icon }}</text>
        <text class="modal-title">{{ modal.title }}</text>
        <scroll-view class="modal-scroll" scroll-y>
          <text class="modal-text">{{ modal.content }}</text>
        </scroll-view>
        <button class="modal-btn" @click="modal.show = false">任务完成</button>
      </view>
    </view>

    <!-- 自定义输入弹窗 -->
    <view v-if="customInput.show" class="modal-overlay dark" @click="customInput.show = false">
      <view class="input-content" @click.stop>
        <text class="input-title">第 {{ customInput.index }} 格任务</text>
        <textarea 
          class="input-area" 
          v-model="customInput.text"
          placeholder="在此输入你的创意任务..."
          maxlength="200"
        ></textarea>
        <view class="input-buttons">
          <button class="cancel-btn" @click="customInput.show = false">取消</button>
          <button class="save-btn" @click="saveCustom">保存</button>
        </view>
      </view>
    </view>
  </view>
</template>

<script>
export default {
  data() {
    return {
      view: 'home',
      currentMode: '',
      playerPos: [0, 0],
      currentPlayer: 0,
      diceValue: 1,
      isRolling: false,
      isEditing: false,
      diceIcons: ['⚀', '⚁', '⚂', '⚃', '⚄', '⚅'],
      modal: { show: false, title: '', content: '', icon: '' },
      customInput: { show: false, index: null, text: '' },
      board: [],
      modes: {
        t1: { name: '甜蜜互动（心动瞬间）', icon: '🍭' },
        t2: { name: '甜蜜互动（浓情蜜意）', icon: '🎨' },
        t3: { name: '甜蜜互动（深情默契）', icon: '🎈' },
        naughty: { name: '羞羞飞行棋', icon: '🔞' },
        custom: { name: '自定义飞行棋', icon: '⚙️' }
      },
      libraries: {
        t1: {
          task: ['互相深情对视 15 秒，先笑的人亲对方一下', '给对方剥一个水果', '为对方捏捏肩膀', '夸对方三个优点'],
          test: ['说出对方最喜欢的口红色号/游戏', '对方最想去旅游的地方是哪里？'],
          truth: ['你最喜欢对方身上的什么味道？', '你第一次对对方心动是什么时候？', '你最想和对方一起做的一件事是什么？', '对方做过最让你感动的事是什么？'],
          wish: ['满足对方一个微小愿望', '倒一杯温水给对方'],
          icon: '🍭'
        },
        t2: {
          task: ['给对方一个长达 30 秒的拥抱', '十指紧扣对视', '帮对方整理头发', '背着对方走一圈'],
          test: ['我们第一次约会是在哪里？', '对方的生日是哪一天？'],
          truth: ['你有没有偷偷看过对方的手机？', '你最嫉妒对方身边的谁？', '你觉得对方最可爱的习惯是什么？', '你有什么话一直想说但没说出口？'],
          wish: ['输家今天洗碗', '答应对方一个合理的小要求'],
          icon: '🎨'
        },
        t3: {
          task: ['说出你最喜欢对方的三个瞬间', '公主抱对方', '合拍一张搞怪的照片', '深情告白 30 秒'],
          test: ['对方最讨厌吃的蔬菜是什么？', '说出对方手机尾号'],
          truth: ['你觉得我们之间最大的默契是什么？', '如果重新选择，你还会选我吗？', '你最想改变对方的一个习惯是什么？', '你最珍惜我们在一起的哪个瞬间？'],
          wish: ['发一条关于对方的动态', '满足对方一个长久以来的小心愿'],
          icon: '🎈'
        },
        naughty: {
          task: ['脱一件衣服/裤子（没有就不用啦 [偷笑]）', '种一个超过 10 秒的"草莓"（吻痕）', '亲吻对方脸颊 10 秒', '在对方耳边哈气并表白', '亲吻对方的手背', '亲吻对方的锁骨', '咬一下对方的肩膀', '轻咬对方耳朵', '用舌尖绕着对方的耳朵画圈', '解开对方的一颗扣子', '让对方闭上眼，你选择一个部位亲吻三下', '互相交换一件贴身物品佩戴', '说出一句最想对对方说的调情话', '坐在对方的大腿上，贴着脸表白', '从背后抱住对方，并把手伸进对方的衣服里（停留 10 秒）', '亲吻对方的足尖', '让对方感受到你的心跳', '对着对方展示你最性感的眼神', '轻轻撕咬对方的嘴唇', '帮对方脱掉袜子'],
          test: ['说出你觉得对方最性感的瞬间', '对方最敏感的部位是哪里？'],
          truth: ['说出最敏感的部位在哪里？', '说出最喜欢哪种接吻方式？', '说出对方做过最让你脸红的事是什么？', '说出你有什么秘密的幻想没告诉过对方？', '说出最喜欢的内衣/内裤颜色？', '描述一下你最害羞的一个幻想', '说出觉得对方什么时候最迷人？', '说出最希望对方怎么称呼你（私下里）？', '说出最喜欢的姿势/场景（如果有的话）？', '说出对方身上哪种味道让你最着迷？', '说出一个你对对方最"色"的想法。'],
          wish: ['赢家指定今晚的一个特殊流程', '输家要穿上赢家指定的"特殊"服装', '赢家可以要求对方在一个特殊地点亲吻', '输家要对赢家做一件"不可描述"的事情', '赢家可以解锁一个新的互动动作', '输家跳个性感的舞蹈'],
          icon: '🔞'
        }
      }
    }
  },
  methods: {
    initBoard() {
      const path = []
      const startX = 230, startY = 8, gap = 37
      
      // 螺旋路径
      for (let i = 0; i < 7; i++) path.push({ x: startX - i * gap, y: startY })
      for (let i = 1; i < 9; i++) path.push({ x: startX - 6 * gap, y: startY + i * gap })
      for (let i = 1; i < 7; i++) path.push({ x: startX - 6 * gap + i * gap, y: startY + 8 * gap })
      for (let i = 1; i < 7; i++) path.push({ x: startX, y: startY + 8 * gap - i * gap })
      for (let i = 1; i < 5; i++) path.push({ x: startX - i * gap, y: startY + 2 * gap })
      for (let i = 1; i < 5; i++) path.push({ x: startX - 4 * gap, y: startY + 2 * gap + i * gap })
      for (let i = 1; i < 3; i++) path.push({ x: startX - 4 * gap + i * gap, y: startY + 6 * gap })
      for (let i = 1; i < 3; i++) path.push({ x: startX - 2 * gap, y: startY + 6 * gap - i * gap })
      path.push({ x: startX - 3 * gap, y: startY + 4 * gap })

      const types = [
        ...Array(23).fill('task'),
        ...Array(4).fill('test'),
        ...Array(8).fill('truth'),
        ...Array(3).fill('wish')
      ]
      for (let i = types.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1))
        ;[types[i], types[j]] = [types[j], types[i]]
      }

      this.board = path.map((c, i) => ({
        ...c,
        type: i === 0 ? 'start' : i === 39 ? 'end' : types.pop(),
        task: ''
      }))
    },
    initGame(mode) {
      this.currentMode = mode
      this.view = 'game'
      this.playerPos = [0, 0]
      this.currentPlayer = 0
      this.isEditing = false
      this.initBoard()

      if (mode === 'custom') {
        const saved = uni.getStorageSync('couple_board_v3')
        if (saved) {
          const data = JSON.parse(saved)
          data.forEach((t, i) => {
            if (t && this.board[i]) this.board[i].task = t
          })
        }
        if (!this.board.some(c => c.task)) this.isEditing = true
      }
    },
    getCellBg(i) {
      if (i === 0) return 'cell-start'
      if (i === 39) return 'cell-end'
      if (this.currentMode === 'custom') {
        return this.board[i].task ? 'cell-custom-task' : 'cell-custom-empty'
      }
      const colors = {
        task: 'cell-task',
        test: 'cell-test',
        truth: 'cell-truth',
        wish: 'cell-wish'
      }
      return colors[this.board[i].type] || ''
    },
    roll() {
      if (this.isRolling) return
      this.isRolling = true
      let count = 0
      const timer = setInterval(() => {
        this.diceValue = Math.floor(Math.random() * 6) + 1
        if (++count > 12) {
          clearInterval(timer)
          this.isRolling = false
          this.move()
        }
      }, 70)
    },
    async move() {
      const pIdx = this.currentPlayer
      const steps = this.diceValue

      for (let i = 0; i < steps; i++) {
        if (this.playerPos[pIdx] >= 39) break
        this.playerPos[pIdx]++
        await new Promise(resolve => setTimeout(resolve, 300))
      }

      setTimeout(() => this.triggerEvent(this.playerPos[pIdx]), 200)
    },
    triggerEvent(pos) {
      const cell = this.board[pos]
      const activePlayer = this.currentPlayer === 0 ? '女生' : '男生'

      if (pos === 39) {
        this.modal = {
          show: true,
          title: '🎉 到达终点！',
          content: `恭喜 ${activePlayer} 获得最终胜利！\n对方要接受一个终极愿望惩罚哦~`,
          icon: '🏆'
        }
        return
      }

      if (this.currentMode === 'custom') {
        if (cell.task) {
          this.modal = {
            show: true,
            title: `${activePlayer}的任务`,
            content: cell.task,
            icon: '📝'
          }
          this.currentPlayer = this.currentPlayer === 0 ? 1 : 0
        } else {
          this.currentPlayer = this.currentPlayer === 0 ? 1 : 0
        }
      } else {
        const lib = this.libraries[this.currentMode]
        const list = lib[cell.type] || lib.task
        const task = list[Math.floor(Math.random() * list.length)]
        const icons = { task: '💜', test: '💚', truth: '💗', wish: '💛' }
        this.modal = {
          show: true,
          title: `${activePlayer}的任务`,
          content: task,
          icon: icons[cell.type] || '✨'
        }
        this.currentPlayer = this.currentPlayer === 0 ? 1 : 0
      }
    },
    handleCellClick(i) {
      if (this.currentMode === 'custom' && this.isEditing && i !== 0 && i !== 39) {
        this.customInput = {
          show: true,
          index: i,
          text: this.board[i].task
        }
      }
    },
    saveCustom() {
      this.board[this.customInput.index].task = this.customInput.text
      this.customInput.show = false
      uni.setStorageSync('couple_board_v3', JSON.stringify(this.board.map(c => c.task)))
    },
    clearCustomBoard() {
      uni.showModal({
        title: '提示',
        content: '确定清空所有自定义内容吗？',
        success: (res) => {
          if (res.confirm) {
            this.board.forEach(c => c.task = '')
            uni.removeStorageSync('couple_board_v3')
          }
        }
      })
    },
    toggleEditMode() {
      this.isEditing = !this.isEditing
      if (!this.isEditing) {
        this.playerPos = [0, 0]
        this.currentPlayer = 0
      }
    },
    goHome() {
      if (this.isEditing) {
        uni.showModal({
          title: '提示',
          content: '内容未保存，确定退出？',
          success: (res) => {
            if (res.confirm) {
              this.view = 'home'
            }
          }
        })
      } else {
        this.view = 'home'
      }
    }
  }
}
</script>

<style>
.container {
  min-height: 100vh;
  background: #fff1f2;
}

/* 首页样式 */
.home-view {
  padding: 40rpx 32rpx;
}

.header {
  text-align: center;
  margin-bottom: 48rpx;
}

.title {
  font-size: 56rpx;
  color: #f43f5e;
  display: block;
  font-weight: bold;
  font-style: italic;
}

.subtitle {
  font-size: 24rpx;
  color: #fda4af;
  letter-spacing: 4rpx;
  font-weight: bold;
  margin-top: 16rpx;
  display: block;
}

.mode-list {
  display: flex;
  flex-direction: column;
  gap: 24rpx;
}

.mode-item {
  background: #fff;
  border-radius: 64rpx;
  padding: 32rpx;
  box-shadow: 0 8rpx 16rpx rgba(0,0,0,0.1);
  border-bottom: 8rpx solid #ffe4e6;
  display: flex;
  justify-content: space-between;
  align-items: center;
  transition: all 0.2s;
}

.mode-item:active {
  transform: translateY(4rpx);
  border-bottom-width: 0;
}

.mode-content {
  display: flex;
  align-items: center;
  gap: 24rpx;
}

.mode-icon {
  font-size: 48rpx;
}

.mode-name {
  font-size: 32rpx;
  color: #e11d48;
  font-weight: bold;
}

.arrow {
  color: #fecdd3;
  font-size: 32rpx;
}

/* 游戏页面样式 */
.game-view {
  padding: 20rpx;
}

.game-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24rpx;
}

.back-btn {
  width: 64rpx;
  height: 64rpx;
  background: #fff;
  border-radius: 32rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 36rpx;
  box-shadow: 0 4rpx 8rpx rgba(0,0,0,0.1);
}

.mode-badge {
  background: rgba(255,255,255,0.8);
  padding: 12rpx 32rpx;
  border-radius: 32rpx;
  backdrop-filter: blur(8rpx);
}

.mode-badge text {
  font-size: 24rpx;
  color: #f43f5e;
}

.edit-btn {
  padding: 16rpx 32rpx;
  border-radius: 24rpx;
  font-size: 24rpx;
  font-weight: bold;
  background: #f43f5e;
  color: #fff;
  box-shadow: 0 4rpx 8rpx rgba(0,0,0,0.1);
}

.edit-btn.active {
  background: #10b981;
}

.placeholder {
  width: 64rpx;
}

/* 棋盘样式 */
.board-container {
  position: relative;
  width: 680rpx;
  height: 960rpx;
  margin: 0 auto 32rpx;
  background: rgba(255,255,255,0.4);
  border-radius: 96rpx;
  padding: 32rpx;
  box-shadow: inset 0 4rpx 8rpx rgba(0,0,0,0.1);
  border: 2rpx solid rgba(255,255,255,0.5);
}

.cell {
  position: absolute;
  width: 80rpx;
  height: 80rpx;
  border-radius: 24rpx;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2rpx 4rpx rgba(0,0,0,0.1);
  transition: transform 0.1s;
}

.cell:active {
  transform: scale(0.9);
}

.cell-start {
  background: #f43f5e;
}

.cell-end {
  background: #e11d48;
}

.cell-task {
  background: #a78bfa;
}

.cell-test {
  background: #34d399;
}

.cell-truth {
  background: #f472b6;
}

.cell-wish {
  background: #fbbf24;
}

.cell-custom-task {
  background: #fb7185;
}

.cell-custom-empty {
  background: #fff;
  border: 4rpx solid #fecdd3;
}

.go-text {
  font-size: 20rpx;
  font-weight: bold;
  color: #fff;
}

.flag-text {
  font-size: 48rpx;
}

.cell-number {
  font-size: 20rpx;
  font-weight: bold;
}

.text-white {
  color: rgba(255,255,255,0.6);
}

.text-rose {
  color: #fecdd3;
}

.star-text {
  font-size: 16rpx;
  color: #fff;
}

.piece {
  position: absolute;
  font-size: 48rpx;
  z-index: 100;
  filter: drop-shadow(0 4rpx 6rpx rgba(0,0,0,0.1));
}

.piece-girl {
  top: -32rpx;
  left: -16rpx;
}

.piece-boy {
  bottom: -32rpx;
  right: -16rpx;
}

/* 控制区样式 */
.control-panel {
  background: #fff;
  border-radius: 80rpx;
  padding: 48rpx;
  box-shadow: 0 8rpx 16rpx rgba(0,0,0,0.1);
  border-top: 8rpx solid #ffe4e6;
}

.players-info {
  display: flex;
  justify-content: space-around;
  align-items: center;
  margin-bottom: 48rpx;
}

.player {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16rpx;
}

.player-avatar {
  width: 112rpx;
  height: 112rpx;
  border-radius: 32rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 48rpx;
  background: #f3f4f6;
  color: #9ca3af;
  box-shadow: 0 4rpx 8rpx rgba(0,0,0,0.1);
  transition: all 0.3s;
}

.player-avatar.active {
  background: #f43f5e;
  color: #fff;
  animation: breathe 1.5s infinite ease-in-out;
}

@keyframes breathe {
  0%, 100% {
    transform: scale(1.1);
    box-shadow: 0 0 24rpx rgba(244,63,94,0.4);
  }
  50% {
    transform: scale(1.05);
    box-shadow: 0 0 8rpx rgba(244,63,94,0.1);
  }
}

.player-label {
  font-size: 20rpx;
  font-weight: bold;
  color: #9ca3af;
}

.player-label.active {
  color: #e11d48;
}

.dice-area {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.dice-value {
  font-size: 80rpx;
  margin-bottom: 8rpx;
}

.dice-value.rolling {
  animation: bounce 0.5s infinite;
}

@keyframes bounce {
  0%, 100% { transform: translateY(0); }
  50% { transform: translateY(-16rpx); }
}

.roll-btn {
  width: 100%;
  background: #f43f5e;
  color: #fff;
  padding: 40rpx;
  border-radius: 48rpx;
  font-size: 40rpx;
  font-weight: bold;
  box-shadow: 0 8rpx 16rpx rgba(0,0,0,0.1);
  border-bottom: 8rpx solid #be123c;
  transition: all 0.2s;
}

.roll-btn:active {
  transform: scale(0.95);
}

.roll-btn[disabled] {
  opacity: 0.5;
}

/* 编辑提示 */
.edit-hint {
  text-align: center;
  padding: 32rpx;
  background: rgba(255,255,255,0.8);
  border-radius: 32rpx;
  border: 4rpx dashed #fecdd3;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

.hint-text {
  display: block;
  font-size: 28rpx;
  color: #f43f5e;
  font-weight: bold;
}

.clear-btn {
  display: block;
  margin-top: 16rpx;
  font-size: 24rpx;
  color: #9ca3af;
  text-decoration: underline;
}

/* 弹窗样式 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.2);
  backdrop-filter: blur(16rpx);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 999;
  padding: 48rpx;
}

.modal-overlay.dark {
  background: rgba(0,0,0,0.6);
}

.modal-content {
  background: #fff;
  border-radius: 80rpx;
  padding: 64rpx;
  width: 100%;
  max-width: 640rpx;
  text-align: center;
  box-shadow: 0 16rpx 32rpx rgba(0,0,0,0.2);
  border: 8rpx solid #ffe4e6;
  animation: scaleUp 0.3s ease-out;
}

@keyframes scaleUp {
  from {
    transform: scale(0.8);
    opacity: 0;
  }
  to {
    transform: scale(1);
    opacity: 1;
  }
}

.input-content {
  background: #fff;
  border-radius: 64rpx;
  padding: 48rpx;
  width: 100%;
  max-width: 640rpx;
  box-shadow: 0 16rpx 32rpx rgba(0,0,0,0.2);
  animation: scaleUp 0.3s ease-out;
}

.modal-icon {
  font-size: 96rpx;
  display: block;
  margin-bottom: 32rpx;
}

.modal-title {
  font-size: 36rpx;
  font-weight: bold;
  color: #1f2937;
  display: block;
  margin-bottom: 32rpx;
}

.modal-scroll {
  max-height: 384rpx;
  margin-bottom: 32rpx;
}

.modal-text {
  font-size: 32rpx;
  color: #4b5563;
  line-height: 1.6;
}

.modal-btn {
  width: 100%;
  background: #f43f5e;
  color: #fff;
  padding: 32rpx;
  border-radius: 32rpx;
  font-size: 32rpx;
  font-weight: bold;
  box-shadow: 0 4rpx 8rpx rgba(0,0,0,0.1);
}

.input-title {
  font-size: 32rpx;
  font-weight: bold;
  color: #f43f5e;
  display: block;
  text-align: center;
  margin-bottom: 32rpx;
}

.input-area {
  width: 100%;
  height: 256rpx;
  background: rgba(244,63,94,0.05);
  border: 4rpx solid #ffe4e6;
  border-radius: 32rpx;
  padding: 32rpx;
  font-size: 28rpx;
  margin-bottom: 32rpx;
}

.input-buttons {
  display: flex;
  gap: 16rpx;
}

.cancel-btn, .save-btn {
  flex: 1;
  padding: 24rpx;
  border-radius: 24rpx;
  font-size: 28rpx;
  font-weight: bold;
}

.cancel-btn {
  background: #f3f4f6;
  color: #9ca3af;
}

.save-btn {
  background: #f43f5e;
  color: #fff;
  box-shadow: 0 4rpx 8rpx rgba(0,0,0,0.1);
}
</style>
