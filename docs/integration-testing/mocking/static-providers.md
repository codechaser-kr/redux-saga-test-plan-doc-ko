# 정적 프로바이더

정적 프로바이더를 통해 간결한 방식으로 모의 값을 제공할 수 있습니다. `provide` 메서드에
튜플 쌍의 배열(혹은 배열 쌍의 배열)을 인자로 제공합니다. 각 쌍의 첫번째 원소는 이펙트에 매칭할 매처이고 
두번째 원소는 이팩트에 제공하려는 모의값입니다. `redux-saga-test-plan/matchers`의 매처대신
`redux-saga/effects`의 이팩트들을 사용할 수 있습니다. Redux Saga Test Plan의 매처를
사용하면 부분 매칭이 가능합니다. 예를들면 `call` 이팩트에 대응되는 `call.fn`을 인자값을 
신경쓰지 않고 사용할 수 있습니다.
<!-- You can provide mock values in a terse manner via static providers. Pass in
an array of tuple pairs (array pairs) into the `provide` method. For each pair,
the first element should be a matcher for matching the effect and the second
effect should be the mock value you want to provide. You can use effect creators
from `redux-saga/effects` as matchers or import matchers from
`redux-saga-test-plan/matchers`. The benefit of using Redux Saga Test Plan's
matchers is that they also offer partial matching. For example, you can use
`call.fn` to match any calls to a function without regard to its arguments. -->

```js
import { call, put, select } from 'redux-saga/effects';
import { expectSaga } from 'redux-saga-test-plan';
import * as matchers from 'redux-saga-test-plan/matchers';
import api from 'my-api';
import * as selectors from 'my-selectors';

function* saga() {
  const id = yield select(selectors.getId);
  const user = yield call(api.fetchUser, id);

  yield put({ type: 'RECEIVE_USER', payload: user });
}

it('provides a value for the API call', () => {
  return expectSaga(saga)
    .provide([
      // Redux Saga의 `select` 이팩트 생성자를 사용
      [select(selectors.getId), 42],

      // Redux Saga Test Plan 의 `call.fn` 매처를 사용
      [matchers.call.fn(api.fetchUser), { id: 42, name: 'John Doe' }],
    ])
    .put({
      type: 'RECEIVE_USER',
      payload: { id: 42, name: 'John Doe' },
    })
    .run();
});
```

## 매처

Redux Saga에서 사용되는 대부분의 이팩트 생성자들의 대한 매처들이 `redux-saga-test-plan/matchers`
모듈에 있습니다. 이팩트 생성자들에 대해서는 Redux Saga [문서](http://redux-saga.github.io/redux-saga/docs/api/index.html#effect-creators)를 확인하세요.
<!-- Inside the `redux-saga-test-plan/matchers` module, there are matchers for most
of the effect creators available in Redux Saga. You can reference effect
creators in Redux Saga's docs
[here](http://redux-saga.github.io/redux-saga/docs/api/index.html#effect-creators). -->

- `actionChannel(pattern, [buffer])`
- `apply(context, fn, args)`
- `call([context, fn], ...args)`
- `call(fn, ...args)`
- `cancel(task)`
- `cancelled()`
- `cps([context, fn], ...args)`
- `cps(fn, ...args)`
- `flush(channel)`
- `fork([context, fn], ...args)`
- `fork(fn, ...args)`
- `getContext(prop)`
- `join(task)`
- `put(action)`
- `putResolve(action)`
- `race(effects)`
- `select(selector, ...args)`
- `setContext(props)`
- `spawn([context, fn], ...args)`
- `spawn(fn, ...args)`
- `take(pattern)`
- `takeMaybe(pattern)`

## Partial Matchers

Sometimes you're not interested in matching a `call` effect with exact arguments
or a `put` effect with a particular action payload.
Instead you only want to match a `call` to a particular function or match a
`put` with a particular action type. You can handle these situations with
partial matchers.

The following assertions have a `like` method along with convenient helper
methods for partially matching assertions:

- `actionChannel`
- `apply`
- `call`
- `cps`
- `fork`
- `put`
- `put.resolve`
- `select`
- `spawn`

**NOTE:** the `like` method requires knowledge of the properties on effects such
as the `fn` property of `call` and the `action` property of `put`. Essentially,
`like` allows you to match effects with certain properties without worrying
about the other properties. Therefore, you can match a `call` by `fn` without
worrying about the `args` property.

In addition to `like`, there are other some common helper methods like `fn` and
`actionType` available, appropriate to the kind of effect:

| Method | Description |
| ------ | ----------- |
| `actionChannel.pattern` | Match `actionChannel` by `pattern`. Useful if you use custom `buffers` with `actionChannel`. |
| `apply.fn` | Match `apply` by `fn`. |
| `call.fn` | Match `call` by `fn`. |
| `cps.fn` | Match `cps` by `fn`. |
| `fork.fn` | Match `fork` by `fn`. |
| `put.actionType` | Match `put` by `action.type`. |
| `put.resolve.actionType` | Match `put.resolve` by `action.type`. |
| `select.selector` | Match `select` by `selector` function. |
| `spawn.fn` | Match `spawn` by `fn`. |

## Ordering

Providers are checked from left to right (or top to down depending on how you
look at it). The first provider to match an effect is used, skipping subsequent
providers. If no providers match, then the effect is handled by Redux Saga as
normal.

```js
import { call, put, select } from 'redux-saga/effects';
import * as matchers from 'redux-saga-test-plan/matchers';
import api from 'my-api';
import * as selectors from 'my-selectors';

function* saga() {
  const user = yield call(api.findUser, 1);
  const dog = yield call(api.findDog);
  const greeting = yield call(api.findGreeting);
  const otherData = yield select(selectors.getOtherData);

  yield put({
    type: 'DONE',
    payload: { user, dog, greeting, otherData },
  });
}

const fakeUser = { name: 'John Doe' };
const fakeDog = { name: 'Tucker' };
const fakeOtherData = { foo: 'bar' };

it('takes multiple providers and composes them', () => {
  return expectSaga(saga)
    .provide([
      [matchers.call.fn(api.findUser), fakeUser],
      [matchers.call.fn(api.findDog), fakeDog],
      [select(selectors.getOtherData), fakeOtherData],
    ])
    .put({
      type: 'DONE',
      payload: {
        user: fakeUser,
        dog: fakeDog,
        greeting: 'hello',
        otherData: fakeOtherData,
      },
    })
    .run();
});
```

## Throw Errors

You can simulate errors with static providers via the `throwError` function from
the `redux-saga-test-plan/providers` module. When providing an error, wrap it
with a call to `throwError` to let Redux Saga Test Plan know that you want to
simulate a thrown error.

```js
import { call, put } from 'redux-saga/effects';
import * as matchers from 'redux-saga-test-plan/matchers';
import { throwError } from 'redux-saga-test-plan/providers';
import api from 'my-api';

function* userSaga(id) {
  try {
    const user = yield call(api.fetchUser, id);
    yield put({ type: 'RECEIVE_USER', payload: user });
  } catch (e) {
    yield put({ type: 'FAIL_USER', error: e });
  }
}

it('handles errors', () => {
  const error = new Error('error');

  return expectSaga(userSaga)
    .provide([
      [matchers.call.fn(api.fetchUser), throwError(error)],
    ])
    .put({ type: 'FAIL_USER', error })
    .run();
});
```

## Static Providers with Dynamic Values

Static providers can provide dynamic values too. Instead of supplying a static
value, you can supply a function that produces the value. This function takes as
arguments the matched effect as well as a `next` function. Additionally, you
must wrap the function with a call to the `dynamic` function from the
`redux-saga-test-plan/providers` module. Inside the provider function, you can
inspect the effect further and return a mock value or return a call to the
`next` function. Returning a call to the `next` function will tell Redux Saga
Test Plan to try the next provider, similar to a middleware stack. If there are
no more providers, then Redux Saga Test Plan will let Redux Saga handle the
effect.

```js
import { call, put } from 'redux-saga/effects';
import * as matchers from 'redux-saga-test-plan/matchers';
import { dynamic } from 'redux-saga-test-plan/providers';

const add2 = a => a + 2;

function* someSaga() {
  const x = yield call(add2, 4);
  const y = yield call(add2, 6);
  const z = yield call(add2, 8);

  yield put({ type: 'DONE', payload: x + y + z });
}

const provideDoubleIf6 = ({ args: [a] }, next) => (
  // Check if the first argument is 6
  a === 6 ? a * 2 : next()
);

const provideTripleIfGt4 = ({ args: [a] }, next) => (
  // Check if the first argument is greater than 4
  a > 4 ? a * 3 : next()
);

it('works with dynamic static providers', () => {
  return expectSaga(someSaga)
    .provide([
      [matchers.call.fn(add2), dynamic(provideDoubleIf6)],
      [matchers.call.fn(add2), dynamic(provideTripleIfGt4)],
    ])
    .put({ type: 'DONE', payload: 42 })
    .run();
});
```

## Other Examples

### Parallel Effects via `all`

Providers work on effects yielded inside an `all` effect:

```js
import { put, select } from 'redux-saga/effects';
import { expectSaga } from 'redux-saga-test-plan';
import { selectors } from 'my-selectors';

function* saga() {
  const [name, age] = yield all([
    select(selectors.getName),
    select(selectors.getAge),
  ]);

  yield put({ type: 'USER', payload: { name, age } });
}

it('provides values for effects inside arrays', () => {
  return expectSaga(saga)
    .provide([
      [select(selectors.getName), 'Tucker'],
      [select(selectors.getAge), 11],
    ])
    .put({
      type: 'USER',
      payload: { name: 'Tucker', age: 11 },
    })
    .run();
});
```

### Providing in Forked/Spawned Sagas

Providers work for effects in forked/spawned sagas too.

```js
import { call, fork, put } from 'redux-saga/effects';
import { expectSaga } from 'redux-saga-test-plan';
import * as matchers from 'redux-saga-test-plan/matchers';
import api from 'my-api';

function* fetchUserSaga() {
  const user = yield call(api.fetchUser);
  yield put({ type: 'RECEIVE_USER', payload: user });
}

function* forkingSaga() {
  yield fork(fetchUserSaga);
}

function* spawningSaga() {
  yield spawn(fetchUserSaga);
}

it('provides values in forked sagas', () => {
  return expectSaga(forkingSaga)
    .provide([
      [matchers.call.fn(api.fetchUser), fakeUser],
    ])
    .put({ type: 'RECEIVE_USER', payload: fakeUser })
    .run();
});

it('provides values in spawned sagas', () => {
  return expectSaga(spawningSaga)
    .provide([
      [matchers.call.fn(api.fetchUser), fakeUser],
    ])
    .put({ type: 'RECEIVE_USER', payload: fakeUser })
    .run();
});
```

## More Examples

For some more contrived examples of providers, look in the
[repo tests](https://github.com/jfairbank/redux-saga-test-plan/tree/master/__tests__/expectSaga/providers).

## Caveats

For providers to work, `expectSaga` will necessarily wrap forked/spawned sagas
with an intermediary generator called `sagaWrapper` in order to intercept
effects. To ensure that your saga receives back a task object with a correct
`meta.name` property, Redux Saga Test Plan will attempt to rename the `sagaWrapper`
function to the name of a forked saga. This works in almost all JavaScript
environments but will fail in PhantomJS. Therefore, you **can't** depend on the
task `meta.name` property being correct in PhantomJS.
