# Redoodle

Redoodle 是一个增强 Redux 和 Typescript 整合的插件库。
Redoodle 包含了几个可以单独使用的插件，并且相互可以运行的很好：

Redoodle is an addon library for Redux that enhances its integration with TypeScript.

Redoodle includes a few major categories of addons that can be used individually
and play well with each other:

- [Typed Actions](#typed-actions), or "finally my reducers will stop breaking when I refactor"
- [Compound Actions](#compound-actions), or "my action creators are finally sane"
- Immutable typesafe state manipulation [functions](#utilities)
- Some [opinions on state management](#on-initial-state-management)

[![Circle CI](https://img.shields.io/circleci/project/github/palantir/redoodle/master.svg?style=flat-square)](https://circleci.com/gh/palantir/redoodle)
[![npm](https://img.shields.io/npm/v/redoodle.svg?style=flat-square)](https://www.npmjs.com/package/redoodle)


## Motivation

动机

Redux 带来了明智的 state 管理，在众多的 React 应用架构中找到了自己的定位。
Typescript 同样获得了大量的注意，和令人印象深刻的每个月的高质量版本迭代。
遗憾的是，他们两个之间有几处不同导致不能将他们很好的运行在一起：

1. Redux 的 Action 拥有魔法字符串定义，和不透明的参数定义。
  没有明显的机制来追踪两者之间的相关性。

2. Redux Reducers 没有一种简单的方法来判断不同的Action分支正确的参数类型。

3. TypeScript / JavaScript STL (标准模版库) 缺乏良好的不可变状态更新功能。
  尽管随着对象和数组的扩展以及它们的合理的类型，这一情况变得更好了，对于某些常见的Redux更新工作流来说，仍然缺乏精确的行为控制。

Redoole尝试解决这些集成上的痛苦，和开发人员在项目中使用Redux and Typescript上的一些痛点。


Redux brought sanity to state management,
and has found itself a seat in many a React application's architecture.
TypeScript is also gaining a large amount of traction,
and has impressed the community with its month-over-month high quality releases.
Unfortunately, there were a few places where default Redux and TypeScript
failed to play well with each other:

1. Redux Actions have magic string types and opaque payloads,
   with no obvious mechanism to track the correlation between the two.
2. Redux Reducers have no easy way to infer correct typings for different Action branches.
3. The TypeScript/JavaScript STL lacks good immutable state update functions.
   While this has gotten better with Object and Array spread and their reasonable typings,
   there is still a dearth of exact behavior for certain common Redux update workflows.

Redoodle attempts to solve these integration pains,
and takes a stab at a few more common points of developer frustration when working
with Redux and Typescript.


## Features

特性


### Typed Actions

Typed Actions 是解决跨越Redux-TypeScript鸿沟的问题。来关联Redux Action中魔法字符串和魔法负载参数。通过使用 Redoodle，你可以创建一个Action的定义，来替代你之前所有需要猜测，转换，和哭泣的地方。

Typed Actions do the most legwork to bridge the Redux-TypeScript divide, by correlating
Action magic type strings with Action magic payloads.
With Redoodle, you can create an Action Definition, and then use that Definition
in all places you were previously guessing, casting, or crying.

```ts
import { TypedAction, TypedReducer, setWith } from "redoodle";

// FlipTable is an action Definition for actions of type "app::flip_table",
// and associates the given payload type with the action.
const FlipTable = TypedAction.define("app::flip_table")<{
  tableId: string;
  face: "happy" | "angry";
}>();

// We can use a Definition to create a matching Action. This is all type-safe.
// At runtime, this `action` has the value
//
//    { type: "app::flip_table",
//      payload: { tableId: "2567f216-59b7-4bfe-b46f-909c6711fea4", face: "happy" }
//    }
//
const action = FlipTable.create({
  tableId: "2567f216-59b7-4bfe-b46f-909c6711fea4",
  face: "happy"
});

// We can also use Definitions to create Reducers that offer slick type inferencing.
// No more manual casts!
interface BanquetHall {
  [tableId: string]: {
    isFlipped: boolean;
  }
}

// banquetHallReducer is a standard Redux Reducer<BanquetHall>. Nothing fancy.
const banquetHallReducer = TypedReducer.builder<BanquetHall>()
  // FlipTable.TYPE is the plain string "app::flip_table", but it's branded with
  // rich compile-time information about its associated payload type {tableId, face}.
  .withHandler(FlipTable.TYPE, (state, {tableId, face}) => {
    // All of the operations here are 100% type-safe. If I misspelled `isFlipped` below
    // in either place, or fail to copy the correct subset of state when applying my
    // immutable copies, the compiler will save me!
    return setWith(state, {
      [tableId]: setWith(state[tableId], {
        isFlipped: !state[tableId].isFlipped
      })
    });
  })
  .build();
```

For more on Typed Actions, probably the most simple and significant utility
Redoodle offers, check out the Redoodle
[docs](https://palantir.github.io/redoodle/docs/actions/).


### Compound Actions

The only builtin Action Definition that Redoodle ships with is the CompoundAction,
which is a simple higher-level Action that wraps another set of Actions.
Its alternatives in stock Redux is either to

1. Create more complex DoFooAndBar actions, which have the drawback of making every Reducer that cares about
   DoFoo _or_ DoBar also care about DoFooAndBar.
1. Dispatch DoFoo and DoBar individually, which has the drawback of causing multiple Store subscription events,
   which often results in e.g. multiple React renders. It also has the drawback of publishing a possibly inconsistent
   intermediate state when DoFoo was applied but not DoBar.

```ts
import { CompoundAction } from "redoodle";

const doFoo: Action = {...};
const doBar: Action = {...};

store.dispatch(CompoundAction.create([doFoo, doBar]));
```

To use CompoundActions, your store must be configured to correctly unwrap and reduce them; thankfully Redoodle
comes with a `compoundActionsEnhancer()` (or the simpler `reduceCompoundActions` decorator) for exactly that.

For more on CompoundActions, check out the Redoodle
[docs](https://palantir.github.io/redoodle/docs/compound/).


### Utilities

Redoodle packages a number of utility functions,
some explicitly for clean TypeScript immutable state manipulation,
some for completeness in developer experience.

- [combineReducers(reducers)](https://palantir.github.io/redoodle/docs/utilities/combineReducers.html)
- [composeReducers(...reducers)](https://palantir.github.io/redoodle/docs/utilities/composeReducers.html)
- [compoundActionsEnhancer()](https://palantir.github.io/redoodle/docs/utilities/compoundActionsEnhancer.html)
- [createStore()](https://palantir.github.io/redoodle/docs/utilities/createStore.html)
- [loggingMiddleware([options])](https://palantir.github.io/redoodle/docs/utilities/loggingMiddleware.html)
- [omit(state, keys)](https://palantir.github.io/redoodle/docs/utilities/omit.html)
- [reduceCompoundActions(reducer)](https://palantir.github.io/redoodle/docs/utilities/reduceCompoundActions.html)
- [setWith(state, ...overrides)](https://palantir.github.io/redoodle/docs/utilities/setWith.html)

### On Initial State Management

Redoodle firmly believes that specific benefits the TypeScript compiler provides
should change the way we think about Redux State initialization.
This shifts the behavior of some Redoodle utilities slightly to be more ergonomic for developers,
and is discussed in greater detail [here](https://palantir.github.io/redoodle/docs/InitialState.html).

## Documentation

* [Introduction](https://palantir.github.io/redoodle/)
* [Typed Actions](https://palantir.github.io/redoodle/docs/actions/)
* [Compound Actions](https://palantir.github.io/redoodle/docs/compound/)
* [Initial State Management](https://palantir.github.io/redoodle/docs/InitialState.html)
* [Utilities](https://palantir.github.io/redoodle/docs/utilities/)
* [FAQ](https://palantir.github.io/redoodle/docs/FAQ.html)


## Getting Involved

Coming Soon!

## License

Apache 2.0
