## Vue.js computed and watch

<br>
몇일전에 코드리뷰 받다가 <strong>watch를 가급적이면 사용하지 말아달라</strong>는 피드백을 받아서...ㅎㅎ <br>
이참에 watch와 computed의 차이를 정리해보고 watch는 언제 쓰는게 맞는것인지 좀 찾아봄.

<br>
<br>

1. ChatGPT한테 computed와 watch중 성능개선 관점에서 어느거를 사용하길 권장하냐고 물어봤더니
```markdown
일반적으로 Vue.js 프로젝트의 성능을 개선하려면 computed 속성을 사용하는 것이 watch보다 더 권장됩니다. 
그 이유는 다음과 같습니다:

1. 캐싱: computed 속성은 결과를 캐시하여 필요한 경우에만 재계산합니다. 
   이는 반복적으로 같은 계산을 수행할 때 성능 향상을 가져옵니다. 
   반면에 watch는 대상 데이터가 변경될 때마다 실행되므로 캐싱되지 않습니다.

2. 의존성 추적: Vue는 computed 속성 내에서 사용되는 데이터를 추적하여 해당 데이터가 변경될 때만 다시 계산합니다. 
   이는 성능 향상에 도움이 됩니다. 
   반면에 watch는 감시 대상이 변경될 때마다 실행되므로 의존성 추적이 명시적으로 이루어지지 않습니다.

3. 가독성 및 유지보수성: computed 속성은 계산된 값을 정의하는 데 사용되므로 코드가 간결하고 읽기 쉽습니다. 
   반면에 watch는 대상 데이터의 변경을 감시하고 그에 따라 수행할 작업을 정의하는 데 사용되기 때문에 가독성이 낮아질 수 있습니다.

따라서 일반적으로 성능을 개선하고 코드를 보다 깔끔하게 유지하려면 computed 속성을 사용하는 것이 좋습니다. 
단, 경우에 따라서는 watch가 필요할 수 있으므로 프로젝트의 요구사항을 고려하여 결정하는 것이 중요합니다.
```

요약하면 watch는 반응형 객체 자체가 변동할때마다 실행되고, computed는 객체 자체가 변동하더라도 값이 동일하면 계산을 하지 않는다고 봐야할듯. <br>
computed는 이미 캐시에 같은 값이 있는 경우 그 값을 가져다 쓰는것 같구 ㅇㅇ <br>

<br>
<br>

2. computed vs method

https://ko.vuejs.org/guide/essentials/computed.html 
<br>
vue.js 공식문서에서 확인한 내용인데... computed를 똑같이 구현한 function의 경우 결과적으로 같지만 반응형이냐 아니냐의 차이가 있는듯 함. <br>
즉, 캐싱 관점에서 function은 캐싱이 안된다고 봐야할듯.

<br>
<br>

3. watch를 사용해야 하는 조건
- 상태 변경에 대한 반응으로 "사이드 이펙트"(예: DOM을 변경하거나 비동기 작업의 결과를 기반으로 다른 상태를 변경해야 할 때
- 즉, 속성 변경 후 추가적인 작업을 해야 하는 경우라면 사용을 권장함


<br>
<br>

4. watch vs watchEffect
- watch : 명시적으로 감시된 소스만 추적함 ㅇㅇ 객체가 변화했을 때 콜백이 실행됨(oldValue, newValue 값이 변했을 경우)
- watchEffect : 데이터 변화를 감지하면 자동으로 실행됨ㅇㅇ 별도 감시대상을 설정할 필요가 없고 watch 간략판이라고 볼만하려나...
- 이전값과 현재값 자체가 중요하다면 watchEffect 보단 watch 사용이 더 유용하고, 데이터 변경만 되면 상관없다면 watchEffect가 좀더 간결하다는 듯함.



<br>
<br>

Ref.
- https://stackoverflow.com/questions/63658276/why-watcher-is-better-than-computed-to-perform-asynchronous-or-expensive-operati
- https://ko.vuejs.org/guide/essentials/watchers.html

<br>

ps.<br>
이건 나중에 봐도 좋을것 같은 글이라서... <br>

https://velog.io/@kyusung/vue-rendering-optimization <br>

vue.js 2 기반으로 작성된 내용이지만 vue.js 3에도 얼추 해당되는 내용같음. 렌더링 개선 관련된 포스팅임.