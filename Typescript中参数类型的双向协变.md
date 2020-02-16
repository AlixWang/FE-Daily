# Typescript中的协变和逆变 #

## 什么是协变、逆变 ##


## Typescript中的协变和逆变有什么不同 ##


## 代码解释 ##


## 具体应用 ##

```javascript
interface Animal {}
interface Humen extends Animal {
  type: "humen";
}
interface BlackMen extends Humen {}
type Call<T, U> = (arg: T) => U;

type CallA = Call<Humen, Humen>;
type CallB = Call<Animal, BlackMen>;
type CallD = Call<BlackMen, BlackMen>;
type CallE = Call<BlackMen, Animal>;

type CallC = (arg: CallA) => string;

const a: CallA = arg => {
  console.log(arg);
  return { type: "humen" };
};

const b: CallB = arg => {
  console.log(arg);
  return { type: "humen" };
};

const d: CallD = arg => {
  console.log(arg);
  return { type: "humen" };
};

const e: CallE = arg => {
  console.log(arg);
  return {};
};

const c: CallC = arg => 1;

c(e);

```
