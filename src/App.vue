<template>
  <div class="game-container">
    <div class="grid" :style="gridStyle()">
      <div v-for="(row, rowIndex) in grid" :key="rowIndex" class="row">
        <div v-for="(cell, colIndex) in row" :key="colIndex" :class="getCellClass(cell)" class="cell"
          @click="handleClickEvent(rowIndex, colIndex)" @contextmenu.prevent="handleContextEvent(rowIndex, colIndex)">
        </div>
      </div>
    </div>
    <div class="controls">
      <button @click="moveSnake('up')">上</button>
      <button @click="moveSnake('down')">下</button>
      <button @click="moveSnake('left')">左</button>
      <button @click="moveSnake('right')">右</button>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from "vue";

// 定义常量（枚举）
const THING = {
  Empty: 0,
  Snake: 1,
  Food: 2,
  Wall: 3,
};

const mapSize = 20; // 地图大小
const grid = reactive(
  Array(mapSize)
    .fill()
    .map(() => Array(mapSize).fill(THING.Empty)),
); // 使用 THING.Empty 初始化二维数组
let snake = []; // 贪吃蛇的身体坐标
let food = []; // 食物位置
const walls = []; // 墙壁位置
let numFood = 10; // 最大食物数量
const numWalls = Math.floor((mapSize * mapSize) / 10); // 生成的墙壁数量

function gridStyle() {
  return {
    display: "grid",
    gridTemplateColumns: `repeat(${mapSize}, 20px)`,
    gridTemplateRows: `repeat(${mapSize}, 20px)`,
    padding: "10px",
  };
}

const initGame = () => {
  console.log(grid);

  // 随机生成蛇的初始位置
  const startEdge = Math.floor(Math.random() * 4);
  let startX = 0,
    startY = 0;

  if (startEdge === 0) {
    // 顶边
    startX = Math.floor(Math.random() * mapSize);
    startY = 0;
  } else if (startEdge === 1) {
    // 右边
    startX = mapSize - 1;
    startY = Math.floor(Math.random() * mapSize);
  } else if (startEdge === 2) {
    // 底边
    startX = Math.floor(Math.random() * mapSize);
    startY = mapSize - 1;
  } else {
    // 左边
    startX = 0;
    startY = Math.floor(Math.random() * mapSize);
  }

  snake = [[startX, startY]];
  grid[startY][startX] = 1; // 蛇的位置设为1

  // 生成食物
  generateFood();
  // 生成墙壁
  generateWalls();

  console.log("Game initialized.");
};

onMounted(() => {
  initGame();
});

function setThing(x, y, thingType) {
  grid[x][y] = thingType;
}

/**
 * @description: 左键点击事件，用于生成/销毁食物
 * @param {number} x x坐标
 * @param {number} y y坐标
 * @return {void}
 */
function handleClickEvent(x, y) {
  handleEventInner(x, y, THING.Food);
}

/**
 * @description: 右键点击事件，用于生成和销毁墙壁
 * @param {number} x x坐标
 * @param {number} y y坐标
 * @return {void}
 */
function handleContextEvent(x, y) {
  handleEventInner(x, y, THING.Wall);
}

function handleEventInner(x, y, thingType) {
  if (grid[x][y] === THING.Snake && grid[x][y] !== thingType) {
    return;
  } else if (grid[x][y] === thingType) {
    setThing(x, y, THING.Empty);
  } else if (grid[x][y] === THING.Empty) {
    setThing(x, y, thingType);
  }
}

// 生成食物
function generateFood() {
  if (numFood <= 0) {
    return;
  }
  const foodX = Math.floor(Math.random() * mapSize);
  const foodY = Math.floor(Math.random() * mapSize);
  if (grid[foodY][foodX] === 0) {
    food = [foodX, foodY];
    grid[foodY][foodX] = 2; // 食物的位置设为2
    numFood -= 1;
  } else {
    generateFood();
  }
}

// 生成墙壁
function generateWalls() {
  for (let i = 0; i < numWalls; i++) {
    const wallX = Math.floor(Math.random() * mapSize);
    const wallY = Math.floor(Math.random() * mapSize);

    if (grid[wallY][wallX] === 0) {
      walls.push([wallX, wallY]);
      grid[wallY][wallX] = THING.Wall;
    }
  }
}

// 移动蛇
function moveSnake(direction) {
  const head = snake[0];
  let newHead = [...head];

  switch (direction) {
    case "up":
      newHead[0]--;
      break;
    case "down":
      newHead[0]++;
      break;
    case "left":
      newHead[1]--;
      break;
    case "right":
      newHead[1]++;
      break;
  }

  if (isValidMove(newHead)) {
    snake.unshift(newHead); // 移动蛇
    if (grid[newHead[1]][newHead[0]] !== 2) {
      const tail = snake.pop(); // 蛇未吃到食物，尾部减掉
      grid[tail[1]][tail[0]] = 0;
    }
    grid[newHead[1]][newHead[0]] = 1; // 更新蛇头位置
  }
}

// 检查移动是否合法
function isValidMove([x, y]) {
  if (x < 0 || x >= mapSize || y < 0 || y >= mapSize || grid[x][y] === THING.Wall || grid[x][y] === THING.Snake) {
    return false;
  }
  return true;
}

// 返回格子对应的 CSS 类
function getCellClass(cell) {
  switch (cell) {
    case THING.Empty:
      return "cell";
    case THING.Snake:
      return "snake";
    case THING.Food:
      return "food";
    case THING.Wall:
      return "wall";
  }
  return "";
}
</script>

<style scoped>
.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 50px;
}

.cell {
  width: 20px;
  height: 20px;
  background-color: white;
  border: 1px solid #ccc;
  /* 默认格子颜色 */
}

.snake {
  background-color: green;
}

.food {
  background-color: red;
}

.wall {
  background-color: grey;
}

.controls {
  margin-top: 20px;
}

button {
  margin: 5px;
  padding: 10px;
}
</style>
