# RyuuPlay 中的 Generator、Prefab 和 Call-Chain 概念详解

## 前言

作为初学者，这篇文档将帮你理解 RyuuPlay 项目中提到的三个核心概念：Generator（生成器）、Prefab（预制件）和 Call-Chain（调用链）。这些都是项目中用于处理游戏流程的巧妙设计模式。

## 核心概念概览

1. **Generator（生成器/发电机）**：使用 JavaScript/TypeScript 的 Generator 函数来管理异步游戏流程
2. **Prefab（预制件）**：可重用的代码组件模板
3. **Call-Chain（调用链）**：链式调用模式，使代码更简洁流畅

---

## 1. Generator（生成器/发电机）的详细解析

### 什么是 Generator？

Generator 是 JavaScript 中一种特殊的函数，它可以暂停执行并在稍后恢复。在 RyuuPlay 项目中，Generator 被用来处理**需要等待用户输入的游戏流程**。

### Generator 在项目中的应用

让我们看一个来自项目的真实例子（来自 `setup-reducer.ts`）：

```typescript
// 注意这里的 function* - 这定义了一个 Generator 函数
function* setupGame(next: Function, store: StoreLike, state: State): IterableIterator<State> {
  
  // 这里是一些准备工作...
  
  // yield 关键字让函数暂停，并等待用户操作完成
  yield store.prompt(state, new ShuffleDeckPrompt(player.id), order => {
    player.deck.applyOrder(order);
    player.deck.moveTo(player.hand, 7);
    playerHasBasic = FilterUtils.count(player.hand.cards, basicPokemon) > 0;
    next(); // 用户操作完成后调用 next() 继续执行
  });
  
  // 可以有多个 yield，每个都代表一个需要等待的步骤
  yield store.prompt(state, [
    new ChooseCardsPrompt(player.id, ...),
    new ChooseCardsPrompt(opponent.id, ...)
  ], choice => {
    putStartingPokemonsAndPrizes(player, choice[0]);
    putStartingPokemonsAndPrizes(opponent, choice[1]);
    next();
  });
  
  // 函数继续执行...
}
```

### Generator 是如何被调用的？

让我们看看 Generator 是如何启动的（同样来自 `setup-reducer.ts`）：

```typescript
const generator = setupGame(() => generator.next(), store, state);
return generator.next().value;
```

这段代码非常巧妙！让我们逐行理解：

1. **`const generator = setupGame(...)`**：创建 Generator 实例，但**还没有开始执行**
2. **`() => generator.next()`**：这是一个回调函数，稍后会传给 `setupGame` 作为 `next` 参数
3. **`return generator.next().value`**：第一次调用 `next()` 开始执行 Generator，直到遇到第一个 `yield`

### 为什么叫 "发电机"？

在 PR #4 的讨论中，可能提到了"发电机"这个比喻。这是因为：
- Generator 像发电机一样，可以"产生"值（通过 `yield`）
- 它需要"启动"（调用 `next()`）
- 每次"启动"，它会运行到下一个 `yield` 暂停
- 可以反复"启动"，直到完成

### 实际卡牌示例：Typhlosion 的攻击

在 `typhlosion.ts` 中，我们可以看到 Generator 模式的应用：

```typescript
// Typhlosion 有一个叫 "Flare Destroy" 的攻击
function* useFlareDestroy(
  next: Function,
  store: StoreLike,
  state: State,
  effect: AttackEffect
): IterableIterator<State> {
  
  // 第一步：让玩家选择丢弃自己的一个能量
  if (player.active.energies.cards.length > 0) {
    yield store.prompt(
      state,
      new ChooseCardsPrompt(...), // 弹出选择卡牌的提示
      selected => {
        // 用户选择完成后执行的回调
        const discardEnergy = new DiscardCardsEffect(effect, cards);
        store.reduceEffect(state, discardEnergy);
        next(); // 调用 next() 让 Generator 继续执行
      }
    );
  }
  
  // 第二步：让玩家选择丢弃对手的一个能量
  if (opponent.active.energies.cards.length > 0) {
    yield store.prompt(
      state,
      new ChooseCardsPrompt(...),
      selected => {
        const discardEnergy = new DiscardCardsEffect(effect, cards);
        store.reduceEffect(state, discardEnergy);
        next();
      }
    );
  }
  
  return state;
}

// 然后在卡牌的 reduceEffect 中这样调用：
if (effect instanceof AttackEffect && effect.attack === this.attacks[0]) {
  const generator = useFlareDestroy(() => generator.next(), store, state, effect);
  return generator.next().value;
}
```

---

## 2. Prefab（预制件）的概念

### 什么是 Prefab？

"Prefab" 来自游戏开发术语（Unity 等引擎），意思是**可重用的组件模板**。虽然在当前代码中可能没有明确的 "Prefab" 类，但项目中的很多设计遵循了这个思想。

### RyuuPlay 中的 Prefab 思想

以下是项目中体现 Prefab 思想的例子：

#### 例子 1：Effect（效果）系统

```typescript
// 定义一个 Effect 类
class AttackEffect extends Effect {
  player: Player;
  opponent: Player;
  attack: Attack;
  
  constructor(player: Player, opponent: Player, attack: Attack) {
    this.player = player;
    this.opponent = opponent;
    this.attack = attack;
  }
}

// 然后在不同的地方都可以重用这个预制件：
const attackEffect = new AttackEffect(player, opponent, attack);
state = store.reduceEffect(state, attackEffect);
```

#### 例子 2：Prompt（提示）系统

```typescript
// Prompt 就是一种 Prefab，可以在不同的卡牌中复用
new ChooseCardsPrompt(
  player.id,
  GameMessage.CHOOSE_ENERGIES_TO_DISCARD,
  player.active.energies,
  { superType: SuperType.ENERGY },
  { min: 1, max: 1, allowCancel: false }
)
```

---

## 3. Call-Chain（调用链）的概念

### 什么是 Call-Chain？

Call-Chain（调用链）是一种编程模式，让你可以连续调用多个方法，每个方法都返回对象本身。

虽然在当前版本的代码中可能不明显，但 PR #4 可能讨论了这种模式的应用。

### RyuuPlay 中的链式思想

让我们看看项目中类似链式调用的设计：

```typescript
// store.reduceEffect 可以链式处理多个 Effect
state = store.reduceEffect(state, checkAttackCost);
state = store.reduceEffect(state, checkProvidedEnergy);
state = store.reduceEffect(state, attackEffect);
state = store.reduceEffect(state, dealDamage);
```

### 想象中的链式 API（可能在 PR 中讨论过）

```typescript
// 这可能是 PR 中讨论的更流畅的 API 设计
state = store
  .reduceEffect(checkAttackCost)
  .reduceEffect(checkProvidedEnergy)
  .reduceEffect(attackEffect)
  .reduceEffect(dealDamage);
```

---

## 完整的工作流程示例

让我们用 Uxie 的 "Psychic Restore" 攻击来演示这三个概念如何协同工作：

### 步骤 1：定义 Generator（带 yield）

```typescript
function* usePsychicRestore(
  next: Function,
  store: StoreLike,
  state: State,
  effect: AttackEffect
): IterableIterator<State> {
  
  // 第一个 yield：询问玩家是否要使用这个效果
  let wantToUse = false;
  yield store.prompt(
    state,
    new ConfirmPrompt(effect.player.id, GameMessage.WANT_TO_SHUFFLE_POKEMON_INTO_DECK),
    result => {
      wantToUse = result;
      next(); // 用户确认后，继续执行
    }
  );
  
  // 如果玩家不想用，直接返回
  if (!wantToUse) {
    return state;
  }
  
  // 第二个 yield：让玩家选择卡牌顺序
  const cardList = new CardList();
  target.moveTo(cardList);
  
  return store.prompt(
    state,
    new OrderCardsPrompt(player.id, GameMessage.CHOOSE_CARDS_ORDER, cardList, {
      allowCancel: true,
    }),
    order => {
      if (order !== null) {
        cardList.applyOrder(order);
      }
      cardList.moveTo(player.deck);
    }
  );
}
```

### 步骤 2：使用 Prefab（Prompt 和 Effect）

```typescript
// 使用 ConfirmPrompt Prefab
new ConfirmPrompt(effect.player.id, GameMessage.WANT_TO_SHUFFLE_POKEMON_INTO_DECK)

// 使用 OrderCardsPrompt Prefab
new OrderCardsPrompt(player.id, GameMessage.CHOOSE_CARDS_ORDER, cardList, ...)
```

### 步骤 3：启动 Generator

```typescript
if (effect instanceof AttackEffect && effect.attack === this.attacks[0]) {
  // 这里的写法很巧妙！
  const generator = usePsychicRestore(() => generator.next(), store, state, effect);
  return generator.next().value;
}
```

---

## 初学者需要记住的关键点

### 1. Generator 函数的标志

看到 `function*` 就知道这是 Generator：
```typescript
function* myGenerator() { ... }
```

### 2. yield 的作用

`yield` 会让函数暂停，等待用户操作：
```typescript
yield store.prompt(state, prompt, callback);
// 函数在这里暂停...
// 用户操作完成后，callback 调用 next()，函数继续
```

### 3. next() 的作用

`next()` 让 Generator 继续执行到下一个 `yield`：
```typescript
// 回调函数中调用 next()
() => generator.next()
```

### 4. 自我引用的技巧

注意这种写法：
```typescript
const generator = setupGame(() => generator.next(), store, state);
```
- 创建 generator 变量
- 在回调中引用 generator 自己
- 这是一个常见的 JavaScript 技巧

---

## 总结

1. **Generator（生成器/发电机）**：
   - 用 `function*` 定义
   - 用 `yield` 暂停
   - 用 `next()` 继续
   - 完美处理需要等待用户输入的游戏流程

2. **Prefab（预制件）**：
   - 可重用的组件
   - 如 `Effect`、`Prompt` 等
   - 避免重复代码

3. **Call-Chain（调用链）**：
   - 连续调用的模式
   - 使代码更流畅

这三个概念共同作用，让 RyuuPlay 的游戏逻辑既强大又易于理解！
