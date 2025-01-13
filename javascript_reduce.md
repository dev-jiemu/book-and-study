### Reduce function

- 의미 : 누산기?
  - 함수를 실행하고 하나의 결과값을 반환하는 함수
  - 배열 내 빈요소 제외하고 각 요소에 대해 callback 함수를 한번씩 실행함
  
```javascript
arr.reduce(callback[, initialValue])
```
- callback
  - accumulator : 반환값 누적함
  - currentValue : 현재 처리할 요소
  - currentIndex : 현재 처리할 요소 인덱스값(Optional)
  - array : reduce를 호출한 배열(Optional)
- initialValue : 최초 호출할때 제공할 값 (제공 안하면 0번째 말고 1번째부터 시작함)

```javascript
let arr = [1, 2, 3, 4, 5]
// acc : accumulator
const result = arr.reduce((acc, cur, idx) => { return acc += cur; }, 0)
console.log(result) // 15
```

Ref.
- https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce
