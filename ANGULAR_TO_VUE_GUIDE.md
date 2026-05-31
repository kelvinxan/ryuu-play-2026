# 从 Angular 迁移到 Vue：注意事项和 Generator 兼容性

## 好消息！

**Vue 完全支持 Generator 和链式调用！** 这两个都是 JavaScript 语言特性，与框架无关。

---

## 第一部分：Vue 对 Generator 和链式调用的支持

### 1. Generator 在 Vue 中完全可用

Generator 是 JavaScript 的原生特性，Vue 3（以及 Vue 2）完全支持！

#### 在 Vue 中使用 Generator 的示例：

```typescript
// Vue 3 Composition API 示例
<script setup lang="ts">
import { ref } from 'vue';

const state = ref({});

// 可以直接使用 function*
function* useAttack(next: Function, store: any, state: any, effect: any) {
  // 完全和 Angular 中一样使用 yield
  yield store.prompt(state, new ChooseCardsPrompt(...), selected => {
    // 处理用户选择
    next();
  });
  
  return state;
}

// 调用方式也一样
const handleAttack = () => {
  const generator = useAttack(() => generator.next(), store, state.value, effect);
  return generator.next().value;
};
</script>
```

### 2. 链式调用在 Vue 中也完全支持

链式调用是普通的 JavaScript 模式，任何框架都支持！

```typescript
// Vue 3 中使用链式调用
<script setup lang="ts">
import { reactive } from 'vue';

const gameState = reactive({});

// 可以像 Angular 中一样链式调用
const handleTurn = () => {
  gameState
    .reduceEffect(checkAttackCost)
    .reduceEffect(checkProvidedEnergy)
    .reduceEffect(attackEffect);
};
</script>
```

---

## 第二部分：从 Angular 迁移到 Vue 的注意事项

### 1. 项目结构对比

| Angular | Vue 3 |
|---------|-------|
| NgModule (模块系统) | 不需要，直接导入/导出 |
| `@Component()` 装饰器 | `<script setup>` 或 `defineComponent()` |
| `@Input()` / `@Output()` | `defineProps()` / `defineEmits()` |
| Services (服务) | Composables / Pinia Store |
| RxJS (响应式) | Vue 响应式系统 (ref/reactive) |
| Angular Router | Vue Router |
| Angular Material | Element Plus / Naive UI / Vuetify |

### 2. 核心概念的对应关系

#### Angular Service → Vue Composable

**Angular 中的服务：**
```typescript
// game.service.ts
@Injectable({ providedIn: 'root' })
export class GameService {
  private state = new BehaviorSubject<State>(initialState);
  
  getState() {
    return this.state.asObservable();
  }
  
  dispatch(action: Action) {
    // 处理 action
  }
}
```

**迁移到 Vue Composable：**
```typescript
// useGame.ts
import { reactive, computed } from 'vue';
import { State, Action } from '@ptcg/common';

export function useGame() {
  const state = reactive<State>(initialState);
  
  const getState = computed(() => state);
  
  function dispatch(action: Action) {
    // 处理 action（可以直接用 @ptcg/common 中的代码！）
  }
  
  return { state, getState, dispatch };
}
```

#### Angular Component → Vue Component

**Angular 组件：**
```typescript
// game-table.component.ts
@Component({
  selector: 'app-game-table',
  template: `
    <div *ngFor="let card of player.hand">
      {{ card.name }}
    </div>
  `
})
export class GameTableComponent {
  @Input() player: Player;
}
```

**迁移到 Vue 组件：**
```vue
<!-- GameTable.vue -->
<template>
  <div v-for="card in player.hand" :key="card.id">
    {{ card.name }}
  </div>
</template>

<script setup lang="ts">
import { defineProps } from 'vue';
import { Player } from '@ptcg/common';

const props = defineProps<{
  player: Player;
}>();
</script>
```

### 3. 状态管理迁移：RxJS → Vue 响应式

**Angular 中使用 RxJS：**
```typescript
// 在 Angular 服务中
this.state$ = this.state.asObservable();
this.state$.subscribe(newState => {
  // 更新视图
});
```

**Vue 中的响应式（更简单！）：**
```typescript
import { reactive, watch } from 'vue';

const state = reactive<State>(initialState);

// 自动追踪变化！不需要 subscribe
watch(state, (newState) => {
  console.log('状态更新了', newState);
}, { deep: true });
```

### 4. Socket.IO 集成（几乎不需要改！）

好消息是，Socket.IO 客户端库是框架无关的！

**Angular 中的代码：**
```typescript
// socket.service.ts
import { Injectable } from '@angular/core';
import { io } from 'socket.io-client';

@Injectable({ providedIn: 'root' })
export class SocketService {
  private socket = io('http://localhost:12021');
  
  connect() {
    this.socket.connect();
  }
  
  on(event: string, callback: Function) {
    this.socket.on(event, callback);
  }
}
```

**Vue 中的代码（几乎一样）：**
```typescript
// useSocket.ts
import { io } from 'socket.io-client';

export function useSocket() {
  const socket = io('http://localhost:12021');
  
  function connect() {
    socket.connect();
  }
  
  function on(event: string, callback: Function) {
    socket.on(event, callback);
  }
  
  return { connect, on };
}
```

### 5. 路由迁移

**Angular Router：**
```typescript
// app-routing.module.ts
const routes: Routes = [
  { path: 'login', component: LoginComponent },
  { path: 'game', component: GameComponent }
];
```

**Vue Router：**
```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router';

const routes = [
  { path: '/login', component: () => import('@/views/LoginView.vue') },
  { path: '/game', component: () => import('@/views/GameView.vue') }
];

const router = createRouter({
  history: createWebHistory(),
  routes
});
```

### 6. UI 组件库迁移

Angular Material → 可以考虑：
- **Element Plus**（最接近 Material Design）
- **Vuetify**（Vue 的 Material Design 组件库）
- **Naive UI**（现代化、轻量）
- **Ant Design Vue**（企业级）

---

## 第三部分：关键技术点详解

### 关键点 1：@ptcg/common 包可以完全复用！

这是最大的好消息！**`@ptcg/common` 是纯 TypeScript，不依赖任何框架**，可以直接在 Vue 中使用！

```typescript
// 在 Vue 中完全一样地使用
import { State, CardManager, Store } from '@ptcg/common';
import * as sets from '@ptcg/sets';

// 初始化 CardManager
const cardManager = CardManager.getInstance();
cardManager.defineFormat('Standard', [
  sets.standardSets.setDiamondAndPearl
  // ...
]);

// 使用 Generator 完全一样！
function* setupGame(next: Function, store: Store, state: State) {
  // 与 Angular 中完全相同的代码！
  yield store.prompt(state, prompt, result => {
    next();
  });
}
```

### 关键点 2：拖拽功能迁移

项目使用了 `@ng-dnd`，在 Vue 中可以用：

**选项 1：VueUse 的 useDrag 和 useDrop**
```bash
npm i @vueuse/core
```

```vue
<script setup>
import { useDrag, useDrop } from '@vueuse/core';
</script>
```

**选项 2：vuedraggable**
```bash
npm i vuedraggable
```

### 关键点 3：国际化迁移

`@ngx-translate` → `vue-i18n`

```typescript
// i18n/index.ts
import { createI18n } from 'vue-i18n';
import en from './locales/en.json';

const i18n = createI18n({
  locale: 'en',
  messages: { en }
});
```

---

## 第四部分：迁移步骤建议

### 渐进式迁移策略（推荐）

1. **第 1 步：创建新的 Vue 项目**
   ```bash
   npm create vue@latest ptcg-play-vue
   ```

2. **第 2 步：集成共享包**
   ```bash
   # 在新 Vue 项目中链接本地包
   npm link ../packages/common
   npm link ../packages/sets
   ```

3. **第 3 步：从简单组件开始迁移**
   - 先迁移纯展示组件（不涉及复杂交互）
   - 再迁移表单组件
   - 最后迁移游戏核心逻辑

4. **第 4 步：保持两个版本并行**
   - 使用同一个后端服务
   - 可以在开发时对比两个版本

### 完整迁移检查清单

- [ ] Vue 项目脚手架
- [ ] TypeScript 配置
- [ ] 集成 `@ptcg/common` 和 `@ptcg/sets`
- [ ] Socket.IO 连接
- [ ] 状态管理（Pinia 或 Composables）
- [ ] 路由配置
- [ ] UI 组件库
- [ ] 拖拽功能
- [ ] 图片缓存
- [ ] Generator 游戏逻辑验证
- [ ] 测试
- [ ] 构建配置

---

## 第五部分：完整的 Vue 组件示例

### 示例 1：使用 Generator 的卡牌组件

```vue
<!-- CardComponent.vue -->
<template>
  <div class="card" @click="handleClick">
    <img :src="cardImage" :alt="card.name" />
    <div class="card-name">{{ card.name }}</div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue';
import { PokemonCard, Store, State, AttackEffect } from '@ptcg/common';

interface Props {
  card: PokemonCard;
  store: Store;
  state: State;
}

const props = defineProps<Props>();

const cardImage = computed(() => {
  return `/assets/cards/${props.card.set}/${props.card.name}.png`;
});

const handleClick = () => {
  // 完全一样地使用 Generator！
  const attack = props.card.attacks[0];
  const effect = new AttackEffect(player, opponent, attack);
  
  // 与 Angular 中完全相同的调用方式
  function* useAttack(next: Function, store: Store, state: State, effect: AttackEffect) {
    yield store.prompt(state, prompt, result => {
      // 处理逻辑
      next();
    });
    return state;
  }
  
  const generator = useAttack(() => generator.next(), props.store, props.state, effect);
  generator.next();
};
</script>

<style scoped>
.card {
  /* 样式 */
}
</style>
```

### 示例 2：游戏状态管理的 Composable

```typescript
// useGameStore.ts
import { reactive, computed } from 'vue';
import { State, Action, Store } from '@ptcg/common';

export function useGameStore() {
  const state = reactive<State>({
    // 初始状态
  });
  
  // Computed 属性（类似 Angular 的 getters）
  const currentPlayer = computed(() => state.players[state.activePlayer]);
  const isGameOver = computed(() => state.phase === 'FINISHED');
  
  // 方法
  function dispatch(action: Action) {
    // 这里可以直接复用 @ptcg/common 中的 reducer 逻辑
    // ...
  }
  
  function prompt(prompt: any, callback: Function) {
    // 处理用户提示
  }
  
  return {
    state,
    currentPlayer,
    isGameOver,
    dispatch,
    prompt
  };
}
```

---

## 总结

### 兼容性速查表

| 功能 | Angular → Vue | 支持情况 |
|------|--------------|---------|
| **Generator** | ✅ 完全支持 | JavaScript 原生特性 |
| **链式调用** | ✅ 完全支持 | JavaScript 原生特性 |
| **@ptcg/common** | ✅ 完全复用 | 框架无关 |
| **@ptcg/sets** | ✅ 完全复用 | 框架无关 |
| **Socket.IO** | ✅ 几乎无需改动 | 库本身框架无关 |
| **TypeScript** | ✅ 完全支持 | Vue 3 完美支持 TS |

### 主要优势

1. **更好的性能** - Vue 的响应式系统通常更快
2. **更小的包体积** - Vue 运行时更小
3. **更简单的 API** - Composition API 比 RxJS 更容易理解
4. **Generator 完全可用** - 核心游戏逻辑无需重写！

### 主要挑战

1. 学习 Vue 的响应式概念
2. 重新实现 UI 组件
3. 拖拽库的迁移
4. 路由和状态管理库的更换

**但好消息是：游戏的核心逻辑（Generator、卡牌效果、状态管理）几乎不需要改变！**
