<template>
  <div class="crossnote markdown-preview">
    <h1 id="贪吃蛇可视化">贪吃蛇可视化</h1>
    <p>类似贪吃蛇的游戏，但是有些修改。</p>
    <ol>
      <li>蛇出生点一定在边上</li>
      <li>自动模式下，蛇头与蛇尾均可前进</li>
      <li>左键放置饵，右键放置墙</li>
    </ol>
  </div>
  <div class="game-container">
    <div class="grid" :style="gridStyle()">
      <div v-for="(row, rowIndex) in grid" :key="rowIndex" class="row">
        <div
          v-for="(cell, colIndex) in row"
          :key="colIndex"
          :class="getCellClass(cell)"
          class="cell"
          @click="handleClickEvent(rowIndex, colIndex)"
          @contextmenu.prevent="handleContextEvent(rowIndex, colIndex)"
        ></div>
      </div>
    </div>
    <div class="buttons">
      <div class="controls">
        <div class="vertical">
          <button @click="moveSnake('up')">上</button>
          <div class="horizontal">
            <button @click="moveSnake('left')">左</button>
            <button @click="moveSnake('right')">右</button>
          </div>
          <button @click="moveSnake('down')">下</button>
        </div>
      </div>

      <button @click="autoMove">自动寻路</button>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { reactive, onMounted, type StyleValue } from "vue";

// 定义常量（枚举）
enum THING {
  Empty = 0,
  Snake = 1,
  Food = 2,
  Wall = 3,
}

type Point = [number, number];

const mapSize = 20; // 地图大小
const new_2d_arr = <T>(x: T) => Array.from({ length: mapSize }).map(() => Array(mapSize).fill(x)); // 生成二维数组
const grid = reactive<THING[][]>(new_2d_arr(THING.Empty));
let snake: Point[] = []; // 贪吃蛇的身体坐标
const walls: Point[] = []; // 墙壁位置
const maxFood = 10; // 最大食物数量
let numFood = 0; // 记录当前场上的食物数量
const numWalls = Math.floor((mapSize * mapSize) / 10); // 生成的墙壁数量
const directions = [
  [0, -1],
  [-1, 0],
  [0, 1],
  [1, 0],
]; // 四个移动方向
let current_route: Point[] | null = null; // 记录当前寻路结果的最短路径（缓存）

/**
 * @description: 计算两个坐标之间的曼哈顿距离
 * @param {number[]} pos1 第一个坐标
 * @param {number[]} pos2 第二个坐标
 * @return {number} 两个坐标之间的曼哈顿距离
 */
function distant([x1, y1]: number[], [x2, y2]: number[]): number {
  return Math.abs(x1 - x2) + Math.abs(y1 - y2);
}

/**
 * @description: 生成 css grid 布局的 style 对象（mapSize x mapSize）
 * @return {Object} css style对象
 */
function gridStyle(): StyleValue {
  return {
    display: "grid",
    gridTemplateColumns: `repeat(${mapSize}, 20px)`,
    gridTemplateRows: `repeat(${mapSize}, 20px)`,
    padding: "20px",
  };
}

onMounted(() => {
  initGame();
});

/**
 * @description: Reset the grid by setting all cells to empty
 */
const resetGrid = () => {
  for (let i = 0; i < mapSize; i++) {
    for (let j = 0; j < mapSize; j++) {
      grid[i][j] = THING.Empty; // 重新赋值
    }
  }
};

const initGame = () => {
  resetGrid();
  generateSnake();
  generateFood(maxFood);
  generateWalls();
  console.log("Game initialized.");
};

/**
 * @description: 在地图的四条边上随机生成一个点的蛇
 */
function generateSnake() {
  // 随机生成蛇的初始位置，必定在边上
  const startEdge = Math.floor(Math.random() * 4);
  let startX = 0;
  let startY = 0;

  if (startEdge === 0) {
    startX = Math.floor(Math.random() * mapSize);
    startY = 0;
  } else if (startEdge === 1) {
    startX = mapSize - 1;
    startY = Math.floor(Math.random() * mapSize);
  } else if (startEdge === 2) {
    startX = Math.floor(Math.random() * mapSize);
    startY = mapSize - 1;
  } else {
    startX = 0;
    startY = Math.floor(Math.random() * mapSize);
  }

  snake = [[startX, startY]];
  grid[startX][startY] = THING.Snake;
}

/**
 * @description: 左键点击事件，用于生成/销毁食物
 * @param {number} x x坐标
 * @param {number} y y坐标
 * @return {void}
 */
function handleClickEvent(x: number, y: number): void {
  handleEventInner(x, y, THING.Food);
  numFood++;
}

/**
 * @description: 右键点击事件，用于生成和销毁墙壁
 * @param {number} x x坐标
 * @param {number} y y坐标
 * @return {void}
 */
function handleContextEvent(x: number, y: number): void {
  handleEventInner(x, y, THING.Wall);
}

/**
 * @description: 内部事件处理函数，根据传入的 x, y, thingType
 * 1. 如果当前位置是蛇身并且与 thingType 不同，什么都不做
 * 2. 如果当前位置是 thingType，清除该位置
 * 3. 如果当前位置是 Empty，生成 thingType
 * @param {number} x x坐标
 * @param {number} y y坐标
 * @param {THING} thingType 生成食物或墙壁类型
 * @return {void}
 */
function handleEventInner(x: number, y: number, thingType: THING): void {
  if (grid[x][y] === THING.Snake && grid[x][y] !== thingType) {
    return;
  }
  if (grid[x][y] === thingType) {
    grid[x][y] = THING.Empty;
  } else if (grid[x][y] === THING.Empty) {
    grid[x][y] = thingType;
  }
}

/**
 * @description: 生成 x 个食物，通过递归保证不重复
 * @param {number} x 生成的食物数量
 * @return {void}
 */
function generateFood(x: number): void {
  if (x === 0) {
    return;
  }
  const foodX = Math.floor(Math.random() * mapSize);
  const foodY = Math.floor(Math.random() * mapSize);
  if (grid[foodX][foodY] === 0) {
    grid[foodX][foodY] = THING.Food;
    numFood++;
    generateFood(x - 1);
    return;
  }
  generateFood(x);
}

/**
 * @description: 随机生成墙壁（数量不一定为 numWalls）
 */
function generateWalls() {
  for (let i = 0; i < numWalls; i++) {
    const wallX = Math.floor(Math.random() * mapSize);
    const wallY = Math.floor(Math.random() * mapSize);

    if (grid[wallX][wallY] === 0) {
      walls.push([wallX, wallY]);
      grid[wallX][wallY] = THING.Wall;
    }
  }
}

/**
 * @description: Move the snake by given direction
 * @param {string} direction move direction, must be one of "up", "down", "left", "right"
 * @return {void}
 */
function moveSnake(direction: string): void {
  const head: Point = snake[0];
  const newHead = [...head] as Point;

  switch (direction) {
    case "up":
      newHead[1]--;
      break;
    case "down":
      newHead[1]++;
      break;
    case "left":
      newHead[0]--;
      break;
    case "right":
      newHead[0]++;
      break;
  }
  moveToNewHead(newHead);
}

/**
 * @description: 将蛇移动到指定位置
 * @param {Point} newHead 新位置，可以是蛇头的四连接或蛇尾的四连接位置。会自动判断需要移动蛇头还是蛇尾
 * @return {void}
 */
function moveToNewHead(newHead: Point): void {
  if (!newHead) {
    alert("无效的移动");
    return;
  }
  if (!isValidMove(newHead)) {
    return;
  }

  if (distant(newHead, snake[0]) <= 1) {
    snake.unshift(newHead); // 移动蛇
    if (grid[newHead[0]][newHead[1]] !== THING.Food) {
      const tail = snake.pop(); // 蛇未吃到食物，尾部减掉
      if (tail) {
        grid[tail[0]][tail[1]] = THING.Empty;
      }
    } else {
      numFood--;
    }
    grid[newHead[0]][newHead[1]] = THING.Snake; // 更新蛇头位置
  } else if (distant(newHead, snake[snake.length - 1]) <= 1) {
    snake.push(newHead); // 移动蛇
    if (grid[newHead[0]][newHead[1]] !== THING.Food) {
      const tail = snake.shift(); // 蛇未吃到食物，尾部减掉
      if (tail) {
        grid[tail[0]][tail[1]] = THING.Empty;
      }
    } else {
      numFood--;
    }
    grid[newHead[0]][newHead[1]] = THING.Snake; // 更新蛇头位置
  }
}

/**
 * @description: Check if a given position is valid to move to
 * @param {number[2]} pos 2D coordinates of the position to check
 * @return {boolean} whether the position is valid or not
 */
function isValidMove([x, y]: Point): boolean {
  if (x < 0 || x >= mapSize || y < 0 || y >= mapSize || grid[x][y] === THING.Wall || grid[x][y] === THING.Snake) {
    return false;
  }
  return true;
}

/**
 * @description: 广度优先搜索，用于找到蛇到食物的最短路径
 * @param {THING[][]} grid 2D数组，表示游戏地图
 * @param {number[]} start 蛇的起点
 * @return {Point} 路径，null if no path is found
 */
function bfs(grid: THING[][], start: number[]): Point[] | null {
  const queue = [[start, [1, 1]]];
  const visited = new_2d_arr(null); // 假设 new_2d_arr 是初始化二维数组的方法

  while (queue.length > 0) {
    // biome-ignore lint/style/noNonNullAssertion: <explanation>
    let [[x, y], last] = queue.shift()!;

    // 标记当前节点的上一个位置
    visited[x][y] = last;
    last = [x, y]; // 更新 last 为当前节点，用于下一个移动

    // 检查是否找到目标
    if (grid[x][y] === THING.Food) {
      const ans: Point[] = [];
      do {
        ans.push([x, y]);
        [x, y] = visited[x][y]; // 回溯路径
      } while (x !== start[0] || y !== start[1]);
      return ans.reverse(); // 返回反转后的路径
    }

    // 遍历四个方向
    for (const [dx, dy] of directions) {
      const newX = x + dx;
      const newY = y + dy;

      // 检查是否可以移动，并且没有访问过该位置
      if (isValidMove([newX, newY]) && visited[newX][newY] === null) {
        queue.push([[newX, newY], last]); // 直接推入坐标和上一个位置
      }
    }
  }
  return null; // 如果没有找到目标
}

/**
 * @description: 通过 bfs 自动寻找蛇到食物的最短路径，并移动一步。
 *               如果蛇头和蛇尾都可以到达食物，那么
 *               选择路径最短的那个移动
 * @return {void}
 */
function autoMove(): void {
  if (numFood === 0) {
    alert("没有食物了");
    return;
  }
  const tmp = current_route?.shift();
  if (tmp) {
    moveToNewHead(tmp);
    return;
  }
  const head = snake[0];
  const tail = snake[snake.length - 1];
  let r1 = bfs(grid, head);
  if (head !== tail) {
    const r2 = bfs(grid, tail);
    if (r1 && r2) {
      r1 = r1.length <= r2.length ? r1 : r2;
    } else if (r2) {
      r1 = r2;
    } else if (!r1 && !r2) {
      alert("没有找到合适的路径");
      return;
    }
  }
  current_route = r1;
  autoMove();
}

/**
 * @description: 通过 cell 的值返回对应的 class
 * @param {THING} cell 一个格子的值
 * @return {string} class name
 */
function getCellClass(cell: THING): string {
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
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 30px;
}

.controls .horizontal {
  display: flex;
}

.controls .vertical {
  display: flex;
  flex-direction: column;
}

button {
  margin: 5px;
  padding: 10px;
  flex: 1;
  font-size: 16px;
  height: 50px;
}

.buttons {
  display: flex;
  align-items: center;
}
</style>
