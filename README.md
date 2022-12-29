# Without a bundler 

이 페이지에서는 번들러 없이 wasm을 빌드하고 브라우저에서 사용하는 방법을 알아보게습니다. 

> 이 페이지의 내용은 [Without a Bundler](https://rustwasm.github.io/docs/wasm-bindgen/examples/without-a-bundler.html)을 참고하세요. 

이 예제는 브라우저에서 직접 --target web 플래그를 사용하여 코드를 로드하는 방법을 보여줍니다. 이 배포 전략에는 Webpack과 같은 번들러가 필요하지 않습니다. 배포에 대한 자세한 내용은 [전용 설명서](https://rustwasm.github.io/docs/wasm-bindgen/reference/deployment.html)를 참고합니다. 

## 프로젝트 생성 

다음을 실행하여 프로젝트를 초기화 합니다. 이렇게 한 이유는 github에서 프로젝트를 생성하고 클론했기 때문입니다.
```shell
cargo init
```

Cargo.toml 파일을 열고 다음과 같이 수정합니다. 
```toml
[package]
name = "without-a-bundler"
version = "0.1.0"
edition = "2018"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2.83"
```
src/lib.rs 파일을 열고 다음과 같이 수정합니다.  
```rust
use wasm_bindgen::prelude::*;

// Called when the wasm module is instantiated
#[wasm_bindgen(start)]
pub fn main() -> Result<(), JsValue> {
    Ok(())
}

#[wasm_bindgen]
pub fn add(a: u32, b: u32) -> u32 {
    a + b
}
```

root에 index.html을 만들고 다음과 같이 입력합니다. 

```html
<html>
  <head>
    <meta content="text/html;charset=utf-8" http-equiv="Content-Type"/>
  </head>
  <body>
    <script type="module">
      import init, { add } from './pkg/without_a_bundler.js';

      async function run() {
        await init();

        const result = add(1, 2);
        console.log(`1 + 2 = ${result}`);
        if (result !== 3)
          throw new Error("wasm addition doesn't work!");
      }
      run();
    </script>
  </body>
</html>
```

다음을 실행하여 빌드합니다. 

```shell
wasm-pack build --target web 
```
index.html을 열고 콘솔을 확인합니다.  VSCode를 사용한다면 live server를 실행한다. 콘솔에 1 + 2 = 3이 출력될 것입니다. 


## 살펴보기

이제 코드를 자세히 들여다 보겠습니다. main() 함수에는  #[wasm_bindgen(start)] 속성이 있습니다. start는 wasm 모듈이 인스턴스화 될 때 한 번 만 호출됩니다. 특별한 일은 하지 않습니다. 시작 함수는 인수를 취하지 않아야 하며 () 또는 Result\<(), JsValue\> 를 반환해야 합니다.

**lib.rs** 
```rust
// Called when the wasm module is instantiated
#[wasm_bindgen(start)]
pub fn main() -> Result<(), JsValue> {
    Ok(())
}
```
브라우저의 자바스크립트가 호출할 add 함수가 선언되어 있습니다. #[wasm_bindgen] 속성은 자바스크립트에서 호출할 수 있도록 함수를 노출합니다.

```
#[wasm_bindgen]
pub fn add(a: u32, b: u32) -> u32 {
    a + b
}
```

빌드를 하고 나면 pkg 디렉토리에 without_a_bundler.js 파일이 생성됩니다. 이 파일은 자바스크립트에서 호출할 수 있는 함수를 노출합니다. 우리가 정의한 add() 함수와, main() 함수가 노출되어 있습니다. 그리고 init, initSync 함수도 노출되어 있습니다. init() 함수는 비동기적으로 wasm 모듈을 초기화합니다. initSync() 함수는 동기적으로 wasm 모듈을 초기화합니다.


```javascript
// ... 나머지는 생략 
/**
*/
export function main() {
    wasm.main();
}

/**
* @param {number} a
* @param {number} b
* @returns {number}
*/
export function add(a, b) {
    const ret = wasm.add(a, b);
    return ret >>> 0;
}

export { initSync }
export default init;

```

index.html을 보면 script를 선언했는데 type="module" 속성이 있습니다. 이 속성은 ES6 모듈을 사용한다는 것을 의미합니다.

```html
  <script type="module">
```

그 다음으로 pkg 디렉토리에 생성된 without_a_bundler.js 파일을 가져옵니다.  default인 init와 브라우저에서 호출할 add 함수를 임포트합니다. 

```jsx
 import init, { add } from './pkg/without_a_bundler.js';
```
그 아래에 run() 함수를 정의하고 run()을 호출합니다.  run() 함수 내부에 init()를 호출하고 기다립니다. 그래서 run() 함수 앞에 async 키워드를 붙였습니다. 
```jsx
   async function run() {
        await init();
        // And afterwards we can use all the functionality defined in wasm.
        const result = add(1, 2);
        console.log(`1 + 2 = ${result}`);
        if (result !== 3)
          throw new Error("wasm addition doesn't work!");
      }
      run();
```      
wasm 파일을 로드해야 합니다. 그래서 기본 내보내기를 사용하여 서버에서 wasm 파일이 어디에 있는지 알려주고, 반환된 프로미스를 기다려서 wasm이 로드될 때까지 기다립니다.  그것은 이렇게 보일 수 있습니다. `await init('./pkg/without_a_bundler_bg.wasm');`,그러나 `init` 함수 내에 편리한 기본값도 있습니다. js 파일에 상대적으로 wasm 파일을 찾기 위해 `import.meta`를 사용합니다.문자열 대신에 다음과 같은 것을 전달할 수도 있습니다.
   
* `WebAssembly.Module`
* `ArrayBuffer`
* `Response`
* `Promise` - 위의 어떠한 타입도 반환합니다. 예를들어, `fetch("./path/to/wasm")`

이것은 모듈이 로드되고 컴파일되는 방법에 대한 완전한 제어를 제공합니다. 또한 프로미스가 해결되면, wasm 모듈의 내보내기를 반환합니다. 이것은 다른 모드에서 `*_bg` 모듈을 가져오는 것과 동일합니다.


