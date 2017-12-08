# [Reusing Reducer Logic](https://redux.js.org/docs/recipes/reducers/ReusingReducerLogic.html)

As an application grows, common patterns in reducer logic will start to emerge.  You may find several parts of your reducer logic doing the same kinds of work for different types of data, and want to reduce duplication by reusing the same common logic for each data type.  Or, you may want to have multiple "instances" of a certain type of data being handled in the store.  However, the global structure of a Redux store comes with some trade-offs: it makes it easy to track the overall state of an application, but can also make it harder to "target" actions that need to update a specific piece of state, particularly if you are using `combineReducers`.

アプリケーションが大きくなるにつれて、 reducer logic に common patternsが生まれてきます。 reducer logic のいくつかの部分が、異なる種類のデータに対して同じ種類の作業を行い、各データ型に同じ共通ロジックを使って複製を減らしたい場合があります。 または、ある種のデータの複数の「インスタンス」をストアで処理したい場合があります。 しかし、Reduxストアのグローバルな構造にはいくつかのトレードオフがあります。アプリケーションの全体の見通しは良くなりますが、反応すべき ActionType まで共通化されてしまうからです。 特にcombineReducersを使用している場合は特にそうです。

As an example, let's say that we want to track multiple counters in our application, named A, B, and C.  We define our initial `counter` reducer, and we use `combineReducers` to set up our state:

例として、A、B、Cという名前のアプリケーションで複数のカウンタをトラッキングするとします。initial `counter` reducerを定義し、combineReducersを使用して状態を設定します。

```js
function counter(state = 0, action) {
    switch (action.type) {
        case 'INCREMENT':
            return state + 1;
        case 'DECREMENT':
            return state - 1;
        default:
            return state;
    }
}

const rootReducer = combineReducers({
    counterA : counter,
    counterB : counter,
    counterC : counter
});
```

Unfortunately, this setup has a problem.  Because `combineReducers` will call each slice reducer with the same action, dispatching `{type : 'INCREMENT'}` will actually cause _all three_ counter values to be incremented, not just one of them.  We need some way to wrap the `counter` logic so that we can ensure that only the counter we care about is updated.

残念ながら、この実装には問題があります。 `combineReducers` は同じアクションで各 slice reducer を呼び出すので、`{type： 'INCREMENT'}` を送出すると、実際に3つすべてのcounterの値がインクリメントされます。 私たちが気にするカウンタだけが更新されるようにするためには、 `counter`ロジックをラップする方法が必要です。


## Customizing Behavior with Higher-Order Reducers

As defined in [Splitting Reducer Logic](SplittingReducerLogic.md), a _higher-order reducer_ is a function that takes a reducer function as an argument, and/or returns a new reducer function as a result.  It can also be viewed as a "reducer factory".  `combineReducers` is one example of a higher-order reducer.  We can use this pattern to create specialized versions of our own reducer functions, with each version only responding to specific actions.

[Splitting Reducer Logic](SplittingReducerLogic.md) で定義されているように、_higher-order reducer_ は、引数として reducer function を取り、結果として新しい reducer function を返す関数です。 また、 "reducer factory" と見なすこともできます。 `combineReducers` は、 higher-order reducer の一例です。 このパターンを使用して、独自の reducer functions の特殊バージョンを作成できます。各バージョンは特定のアクションにのみ応答します。

The two most common ways to specialize a reducer are to generate new action constants with a given prefix or suffix, or to attach additional info inside the action object.  Here's what those might look like:

reducer を特化する最も一般的な2つの方法は、与えられた接頭辞または接尾辞を使用して新しいアクション定数を生成すること、またはアクションオブジェクト内に追加情報を付加することです。 それらがどのように見えるかは次のとおりです。

```js
function createCounterWithNamedType(counterName = '') {
    return function counter(state = 0, action) {
        switch (action.type) {
            case `INCREMENT_${counterName}`:
                return state + 1;
            case `DECREMENT_${counterName}`:
                return state - 1;
            default:
                return state;
        }
    }
}

function createCounterWithNameData(counterName = '') {
    return function counter(state = 0, action) {
        const {name} = action;
        if(name !== counterName) return state;

        switch (action.type) {
            case `INCREMENT`:
                return state + 1;
            case `DECREMENT`:
                return state - 1;
            default:
                return state;
        }
    }
}
```

We should now be able to use either of these to generate our specialized counter reducers, and then dispatch actions that will affect the portion of the state that we care about:

こうすることで、固有の counter reducers を生成し、それぞれ別々に影響を与えるactionをdispatchすることができます。

```js
const rootReducer = combineReducers({
    counterA : createCounterWithNamedType('A'),
    counterB : createCounterWithNamedType('B'),
    counterC : createCounterWithNamedType('C'),
});

store.dispatch({type : 'INCREMENT_B'});
console.log(store.getState());
// {counterA : 0, counterB : 1, counterC : 0}
```


We could also vary the approach somewhat, and create a more generic higher-order reducer that accepts both a given reducer function and a name or identifier:

また、アプローチを多少変更して、特定の reducer function と名前または識別子の両方を受け入れる、より汎用的な higher-order reducer を作成することもできます。

```js
function counter(state = 0, action) {
    switch (action.type) {
        case 'INCREMENT':
            return state + 1;
        case 'DECREMENT':
            return state - 1;
        default:
            return state;
    }
}

function createNamedWrapperReducer(reducerFunction, reducerName) {
    return (state, action) => {
        const {name} = action;
        const isInitializationCall = state === undefined;
        if(name !== reducerName && !isInitializationCall) return state;

        return reducerFunction(state, action);
    }
}

const rootReducer = combineReducers({
    counterA : createNamedWrapperReducer(counter, 'A'),
    counterB : createNamedWrapperReducer(counter, 'B'),
    counterC : createNamedWrapperReducer(counter, 'C'),
});
```

You could even go as far as to make a generic filtering higher-order reducer:

あなたは、一般的な filtering higher-order reducer を作ってもよい：

```js
function createFilteredReducer(reducerFunction, reducerPredicate) {
    return (state, action) => {
        const isInitializationCall = state === undefined;
        const shouldRunWrappedReducer = reducerPredicate(action) || isInitializationCall;
        return shouldRunWrappedReducer ? reducerFunction(state, action) : state;
    }
}

const rootReducer = combineReducers({
    // check for suffixed strings
    counterA : createFilteredReducer(counter, action => action.type.endsWith('_A')),
    // check for extra data in the action
    counterB : createFilteredReducer(counter, action => action.name === 'B'),
    // respond to all 'INCREMENT' actions, but never 'DECREMENT'
    counterC : createFilteredReducer(counter, action => action.type === 'INCREMENT')
};
```


These basic patterns allow you to do things like having multiple instances of a smart connected component within the UI, or reuse common logic for generic capabilities such as pagination or sorting.

これらの基本パターンを使用すると、UI内でスマートに接続されたコンポーネントの複数のインスタンスを持つことや、ページネーションや並べ替えなどの一般的な機能に共通のロジックを再利用することができます。

In addition to generating reducers this way, you might also want to generate action creators using the same approach, and could generate them both at the same time with helper functions. See [Action/Reducer Generators](https://github.com/markerikson/redux-ecosystem-links/blob/master/action-reducer-generators.md) and [Reducers](https://github.com/markerikson/redux-ecosystem-links/blob/master/reducers.md) libraries for action/reducer utilities.

このように reducer を生成するだけでなく、同じアプローチを action creater に使用することもできます。 action や reducer の utilities を知りたくば [Action/Reducer Generators](https://github.com/markerikson/redux-ecosystem-links/blob/master/action-reducer-generators.md) や [Reducers](https://github.com/markerikson/redux-ecosystem-links/blob/master/reducers.md) を参照してください。
