https://ts.xcatliu.com/  TypeScript 入门
https://zhongsp.github.io/TypeScript/ TypeScript 使用指南

TypeScript = JavaScript + 静态类型系统 + 编译期检查，在开发编译期间会进行类型校验，但是实际编译后类型都会被去除，TypeScript 的类型系统主要存在于开发/编译阶段，它最大的价值是：
**把很多原本运行时才可能暴露的问题，提前到开发/编译阶段发现。**


| Java              | TypeScript                | 说明                  |
| ----------------- | ------------------------- | ------------------- |
| `String`          | `string`                  |                     |
| `int/long/double` | `number`                  |                     |
| `boolean`         | `boolean`                 |                     |
| `List<T>`         | `T[]` / `Array<T>`        |                     |
| `Map<K,V>`        | `Map<K,V>`                |                     |
| `class`           | `class`                   |                     |
| `interface`       | `interface`               | 用于声明对象结构，不会产生真正的对象。 |
| 泛型 `<T>`          | 泛型 `<T>`                  |                     |
| `enum`            | `enum` / union type       |                     |
| `null`            | `null`                    |                     |
| `Optional<T>`     | `T \| undefined`          |                     |
| `void`            | `void`                    |                     |
| 异步 Future         | `Promise<T>`              |                     |
| package           | module                    |                     |
| Maven/Gradle      | npm/pnpm/yarn             |                     |
| `pom.xml`         | `package.json`            |                     |
| `javac`           | `tsc`                     |                     |
| JVM               | Node.js / Browser runtime |                     |
**非常关键的区别**：
> **Java 是运行时强类型语言；TypeScript 的类型系统主要是编译期的，最终运行的是 JavaScript。**


TypeScript 项目的核心文件：`tsconfig.json`
一个典型项目：
```
my-project/
├── src/
│   ├── index.ts

│   ├── agent.ts
│   └── message.ts
├── package.json
├── tsconfig.json
└── node_modules/
```
`tsconfig.json`：
```json
{
    "compilerOptions": {
        "target": "ES2022",
        "module": "NodeNext",
        "moduleResolution": "NodeNext",
        "strict": true,
        "outDir": "dist",
        "sourceMap": true
    },
    "include": ["src/**/*.ts"]
}
```