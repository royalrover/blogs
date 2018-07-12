## TypeScript是什么
- TypeScript是JavaScript的一个超集
- TypeScript需要编译为JavaScript才能运行（语法糖）
- TypeScript提供了类型系统，规范类似Java
- TypeScript提供了ES6的支持，也可以支持部分ES7草案的特性，不用担心TypeScript无法兼容目前主流的JavaScript解释器

## 环境安装
1. 安装node.js
      https://nodejs.org/en/download/
2. 安装TypeScript包，这就是TS的编译器
        npm install -g typescript
3. 检查TypeScript是否安装成功
        tsc -v

## TypeScript开发工具
TypeScript是开源的项目，由微软开发和维护，因此最初只有微软的 Visual Studio 支持。现在，出现了更多本身支持或者通过插件支持 TypeScript 语法、智能提示、纠错、甚至是内置编译器的文本编辑器和IDE。

- Visual Studio Code，微软出品，内置支持TypeScript特性
- Sublime Text的官方插件
- WebStorm的最新版本内置支持

## 类型系统
### 原始数据类型

主类型：
    string、number、boolean
特殊类型：
    null、undefined、symbol（ES6）

**基础类型声明与使用：**
```
string：
    let name: string = ‘Alice’;
    let desc: string = `my name is ${name}`;
number：
    let norm: number = 666;
    let binaryNum: number = 0b111;
    let hexNum: number = 0xfff;
    let octalNum: number = 0o17;
    let nan: number = NaN;
    let infinity: number = Infinity;
boolean:
    let yet: boolean = true;
    let flag: boolean = Boolean(0);
null:
    let n: null = null;
undefined:
    let u: undefined = undefined;
symbol:
    let s: Symbol = Symbol(2);
void:
    let v2: void = null;
    let v5: void = undefined;
```
### 任意值类型
    ```
        let name: string = ‘Tom’;
        name = 666;

        demo.ts(2,1): error TS2322: Type '666' is not assignable to type 'string'.
    ```
    使用any类型:
    ```
        let name: any = ‘Tom’;
        name = 666;
    ```
    隐式任意值类型:
        let name;
        name = ‘Tom’;
        name = 666;

    等价于：

        let name : any;    
        name = ‘Tom’;
        name = 666;

### 类型推论
TS会在没有明确指定类型的时候推测出一个类型，这就是类型推论
```
let user = ‘Tom’;
user = 666;

demo.ts(2,1): error TS2322: Type '666' is not assignable to type 'string'.
```
### 联合类型
TS中的联合类型表示取值可为多种类型中的一种:
```
let user: string | number;
user = 666;
user = ‘Tom’;
```
访问联合类型的属性或方法时，只能访问所有类型的共有方法:
```
function test(param: string|number){
return param.length;
}

demo.ts(2,18): error TS2339: Property 'length' does not exist on type 'string | number’.
```
### 类型断言
类型断言可以手动指定一个值的类型,但是**类型断言不是强制类型转换**，TypeScript编译器不支持强制类型转换。
```
function test(param: number|string){
    if((<string>param).length)
        return (<string>param).length;
    else    
        return param.toString().length    
}
```
### 对象的类型—接口
```
interface Sport {
    name: string,
    teamwork: boolean
}

let football: Sport = {
    name: 'soccer',
    teamwork: true
}
```
可选属性：
```
interface Sport {
    name: string,
    teamwork: boolean,
    needPg?: boolean
}

let football: Sport = {
    name: 'soccer',
    teamwork: true
}

```
任意属性：
```
interface Sport {
    name: string,
    teamwork: boolean,             
    needPg?: boolean,
    [other: string]: any
}

let football: Sport = {
    name: 'soccer',
    teamwork: true,
    needPg: true,
    count: 22
}
```
**一旦定义任意属性，那么确定属性和可选属性的类型必须是它的子属性**

只读属性：
```
interface Sport {
    readonly name: string,
    teamwork: boolean
 }

let football: Sport = {
    name: 'soccer',
    teamwork: true
}
```
### 函数的类型
函数声明
```
function avg(x: number,y:number):number{
return (x+y)/2;
}
```
函数表达式
```
let avg = function(x:number,y:number):number{
return (x+y)/2;
}
```
or
```
let avg: (x:number,y:number) => number = function(x:number,y:number):number{
return (x+y)/2;
}
```

函数可选参数：
```
function avg(x: number,y?:number):number{
if(y){
 return (x+y)/2;
}else{
return x;
}
}
```
**可选参数必须在必选参数的后面**

函数的可选参数与默认值:
```
function avg(y:number = 10,x: number):number{
if(y){
 return (x+y)/2;
}else{
return x;
}
}
```
**TypeScript会将添加默认值的参数识别为可选参数，此时不受“可选参数必须在必选参数的后面”的限制**

函数重载:

**TypeScript中通过为一个函数进行多次函数定义，并实现函数完成重载**

```
function reverse(x: number): number;
function reverse(x: string): string;
function reverse(x: any):any{
if(typeof x == ‘number’){
 return Number(x.toString().split(‘’).reverse().join(‘’));
}else{
return x.split(‘’).reverse().join(‘’);
}
}
```
面向对象的函数重载：
```
interface A{
    say(x:number);
    say(x:string);
}

class AA implements A{
    say (x:any){
   if(typeof x == ‘string’)
             console.log(‘string’,x);
        else
console.log(‘number’,x);
    }
}
console.log((new AA()).say(1));
console.log((new AA()).say('123'));

```

### 字符串字面量类型
**该类型约束值只能是某几个字符串的一个，这是在编译器层面做的约束，并不会改变生成的js代码**
```
type Name = 'abc' | 'def' | 'mn';
function demo(e: Name): void{
    console.log(e);
}
demo(‘abc');
```

## TypeScript与面向对象
### 类
```
class Block {
    private hash: string;
    private prevHash: string;
    private nonce: number;
    constructor (hash: string, prevHash: string, nonce = 0){
        this.hash = hash;
        this.prevHash = prevHash;
        this.nonce = nonce;
    }
    public get $hash(): string {
        return this.hash;
    }
    public set $hash(value: string) {
        this.hash = value;
    }
    public get $prevHash(): string {
        return this.prevHash;
    }
    public set $prevHash(value: string) {
        this.prevHash = value;
    }
    public get $nonce(): number {
        return this.nonce;
    }
    public set $nonce(value: number) {
        this.nonce = value;
    }
    public computeHash(){
        let sha256 = crypto.createHash('sha256');
        sha256.update(`${this.prevHash}${this.nonce.toString(16)}`,'utf8');
        let hash = sha256.digest('hex');
        return hash;
    }
}
```

### 抽象类
TypeScript中抽象类不允许被实例化
```
abstract class BtcBlock {
    public abstract computeHash(x:string):string;
}
class Block extends BtcBlock {
    public computeHash(x:string):string{
        return `btc${x}`;
    };
}
```

### 接口
上节已提到，TS中的接口就是抽象多个类的共有属性与方法，作为对象的类型而存在
```
interface Alarm {
    alert(): void;
}
interface Light {
    lightOn(): void;
    lightOff(): void;
}
class Auto implements Alarm, Light {
    alert(){
        console.log('car alart');
    }
    lightOn(){
        console.log('car light on');
    }
    lightOff(){
        console.log('car light off');
    }
}
```

### 泛型
**即在定义类、函数或接口时不指定具体类型，而在使用时指定类型的特性。**
```
function useGeneric<T>(length: number,value: T):Array<T>{
    let array: Array<T> = [];
    for(let i=0;i<length;i++){
        array.push(value);
    }
    return array;
}
useGeneric<string>(2,'hello world');
useGeneric<number>(100,1);
```

## 装饰器（注解）
装饰器是特殊类型的声明，可以被附加到类声明、方法、访问符、属性或参数上，具体文档：
https://www.tslang.cn/docs/handbook/decorators.html

装饰器并未成为ES7的规范，因此未来可能会发生改变，并不推荐大家在线上项目中使用

### 装饰器之方法装饰器
```
function enumerable(value: boolean) {
    return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
        descriptor.enumerable = value;
    };
}

class Greeter {
    greeting: string;
    constructor(message: string) {
        this.greeting = message;
    }

    @enumerable(false)
    greet() {
        return "Hello, " + this.greeting;
    }
}
```

## TypeScript与JavaScript生态
描述非TypeScript编写的类库的类型，需要声明类库所暴露出的API，类似于C的头文件，在TypeScript中文件类型则为 .d.ts

- 使用TypeScript生态提供的声明文件
    npm install @types/node
    声明文件列表：
    http://npm.vdian.net/browse/keyword/@types
- 自己编写声明文件
    declare module “name”;