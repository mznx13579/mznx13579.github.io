---
layout: post
title: Vue2 & Composition-Api 경험 정리
---

Composition-Api 는 Vue 3에 내장 되어 있는 api 이다.
해당 api는 디스크 조각 모음과 같이, Vue 2에서 사용하는 options 방식을 사용하는 방식에서 발생하는 조각 문제를 해결할 수 있다.
즉, 관련된 options(data, method, computed etc...)가 .vue 파일에 여기저기 흩어지는 현상을 관심사 별로 분리하여 관리할 수 있다는 것이 장점이다.
또한 해당 관심사 별로 모든 LifeCycle을 Setup 안에서 관리할 수 있기 때문에
'모아 놓기만 하면 한눈에'가 가능해 질 수 있다.

그러므로, 'Vue2'에서 Composition을 적용했을 때 생겨난 경험을 정리해 보고자 한다.

## 1. Ref & Reactive

해당 변수 선언에서 가장 와닿았던 것은 Composable 하게 나누었을 때 발생했다.
먼저, Vue2에서 관련한 상황과 관심사를 분리하여 Composition 했을 때의 상황 두 상황을 비교해보자

```tsx
/*
상황 설명 : page를 이동할 때 마다, api call을 해야한다.
(두 메소드는 paginate 관심과 fetchData 관심 분리로 가정)
관심사는 다르지만, 사용할 변수가 공유되어야만 하는 경우.
*/
data() {
	return {
		page: 0 as number,
	}
},

methods: {
	chnagePage(pageNumber: number) {
	  this.page = pageNumber;
  },

  getContact() {
    const parameter = {
      page: this.page,
    }

    const data = axios.create('/dataget', { parameter }).get()...apicall...
		
		return data;
  } 
}
```

단적인 예이지만, 위 처럼 'pagination 할 때 사용할 거야!', 'fetchData 할 때 사용할 거야!' 라고 분리를 했지만, 서로 필요한 데이터를 공유할 수 있는 가능성이 생각보다 잦다.
우리가 Store를 사용하는 이유와 같이 변수는 상태를 담고 있고, 이 상태가 특정 메소드에서 사용이 필요할 때가 있다.
Vue 2의 Options에는 위 처럼 사용하던 것이, Composition을 도입함으로서 고민이 생긴다.

하나의 Setup에 구성할 때는 사실상 options와 다른 점이 없지만, Composable할 때 변수는 어떻게 다루지? 에 대한 고민이다.
Compositon의 예를 살펴보자.

```tsx
// usePagination.ts
const usePagination = () => {
	const page = ref<number>(0);

	const changePage = (pageNumber: number) => {
		page.value = pageNumber;
	}
	
	return { page, changePage }
}
```

페이지네이션에 사용되는 options를 `usePagination.ts`에 구성하고 page 변수와 changePage 함수를 return 했다.

```tsx
// useFetchData.ts
const useFetchData = (page) => {
	const parameter = { page: ??? }
	
	const getContact = () => {
		const data = axios.create('/dataget', { parameter }).get()...apicall...

		return data;
	}
	
	return { getContact }
}
```

두개의 파일로 분리하고 나니 parameter에 넣을 page 정보가 없다!
this를 쓰지 못하는 것은 둘 째 치고, 파일이 다른데 어떻게 가져오지?라는 의문이 생긴다.
'에이~ 파라미터로 가져오면 되지!'
근데 자세히 보면 page 변수는 const이다.
이것이 composition-api를 경험했을 때 당황스러웠던 부분이다.

결론부터 얘기하자면, ref화(ref를 사용 혹은 reactive를 toRef)를 해주면 된다.

```
{_isRef: true}
value: (...)
_isRef: true
get value: ƒ value()
set value: ƒ value(newVal)
__proto__: Object
```

const로 인해 오해하기 쉽지만, ref화를 하게 되면 개체 상태가 되어, 상태가 변경 되어도 그 것이 해당 개체에 반영이 된다.
따라서 위의 경우 아래와 같이 구성을 해보자

```tsx
// Home.vue (로직을 관장하는 파일)
import usePagination from './composable/usePagination';
import useFetchData from './composable/useFetchData';

export default defineCompoenet({
	setup() {
		const { page, changePage } = usePagination();
		const { getContact } = useFetchdata(page);
	}
})
```

```tsx
// usePagination.ts
const usePagination = () => {
	const page = ref<number>(0);

	const changePage = (pageNumber: number) => {
		page.value = pageNumber;
	}
	
	return { page, changePage }
}
```

```tsx
// useFetchData.ts
const useFetchData = (page: Ref<number>) => { // Ref<타입이나 인터페이스>
	const parameter = { page: page.value }
	
	const getContact = () => {
		const data = axios.create('/dataget', { parameter }).get()...apicall...

		return data;
	}
	
	return { getContact }
}
```

```tsx
useFetchData에는 상단에 /* eslint-disable no-param-reassign */ 를 적용해주자.
파라미터로 받은 것을 변경하려 해서 정적인 상황에 lint 문법에 맞아 발생하는 것이기 때문이다.
```

위 처럼 필요한 정보들만 빼와서 전달해 주면 사용할 수 있고, Ref 상태이기 때문에 어느 곳에서 변경이 일어나도 반영된다. 가령 예를 들어 useFetchData.ts에서 page.value = 1로 변경해도 page의 값은 1이 된다.
더 나아가 Ref & Reactive의 여러 경우를 다뤄 볼 수 도 있지만, 이미 정리되어 있는 정보들이 많으니 찾아보는게 더 좋을 것 같다. 특히 아래의 링크가 좋다.

[https://www.danvega.dev/blog/2020/02/12/vue3-ref-vs-reactive/](https://www.danvega.dev/blog/2020/02/12/vue3-ref-vs-reactive/)

## 2. this 사용하기

Vue 3는 this 사용을 하지 않는 방향으로 나아가고 있다. 아무래도 캡슐화의 문제가 있기 때문이라고 생각되는데, 확실히 composition을 사용함으로서 보여줄 것만 보여주는 역할은 있다고 생각한다. 하지만 우리는 Vue 2 프로젝트에서 composition을 도입했다. 관심사 별로 분리하는 기능이 너무 강력한데, this를 사용하지 못하는 이유로 사용하지 못하는 것은 아닐까? 라는 고민을 했던 것 같다. 따라서 this를 사용하는 두 가지 방법을 소개하려 한다.

1) context 사용

setup()를 사용하다 보면 두 가지 파라미터를 자주 보게 될 것이다. 바로 `props`, `context`이다.
이 context 안에 root를 접근하면 this와 같은 역할을 하게 된다.
하지만 이는 Vue 3에서 사용할 수 없게 되었다. 이유는 this를 사용하지 않는 것과 같은 결이지 않을까 싶다.
this를 사용하면서 router, store, eventbus etc... 많은 것을 Vue 2에서 사용했는데 당황하지 않을 수가 없다.
Vue 3는 router, store의 타입 정의나 사용 하는 방법이 존재 하지만 이를 위해 Vue2의 라우터를 바꿀 수는 없는 노릇이었다. (그럴바엔 차라리 Vue3로 넘어가지...)
그러나, Vue2에 composition을 적용한 만큼 deprecated(비권장) 되어 있긴 하지만, Vue3에서는 못쓰지만 Vue2에서는 이러라고 쓸 수 있게 남겨 놓은 것이라고 생각한다!
composition의 장점을 살리고 this도 기존 처럼 사용할 수 있게 하기 위한 방법이다.

```tsx
setup(props, context){
	context.root.$router.push('xxx');
  context.root.$store.dispatch...;
  context.root.$eventHub....;
	/*root에 줄이 그어지고 @deprecated — not available in Vue 3 가 생길 것이다.*/
}
```

그러나 우리가 composable 하게 파일을 생성했기 때문에 해당 파일에도 this가 필요할 수도 있다.
그럴 때는 아래와 같이 context를 전달하자.

```tsx
// Home.vue
setup(props, context) {
	const { fetchData } = useFetchData(context);
}

// useFetchData.ts
import { SetupContext } from '@vue/compositon-api';

const useFetchData = (context: SetupContext) => { // context의 타입은 SetupContext
		context.root.$router.push('xxx');
		...
}
```

2) proxy 사용

```tsx
// useFetchData.ts

import { ComponentInternalInstance, getCurrentInstance } from '@vue/composition-api';

// proxy를 사용할 건데,이 이름을 $this로 별도 지정하면, Vue2에서 this를 사용하는 느낌과 비슷해진다.

const { proxy: $this } = getCurrentInstance() as ComponentInternalInstance;

$this.$router.push('xxx');
```

## 3. class 타입 정의

class로 만든 컴포넌트가 있다면, Ref 설정 때문에 해당 클래스도 타입 설정을 해줘야 한다.

아래의 예를 살펴보자

```tsx
import PaginateClass from '@해당 PaginateClass파일';

const state = ref({
	keyword: '' as string,
	paginateClass: null as PaginateClass | null
})
```

이 때 paginateClass를 타입정의 할 때는 해당 클래스 파일을 사용하면 되는데, state라는 ref로 묶이게 된다.
이 상태에서 1) 때와 같이 state 라는 ref를 전달하면 타입정의 문제게 봉착하게 된다.
state라는 ref를 받는 composable 파일은 이 state의 타입을 정의를 요구한다.

```tsx
import { State } from './types';

const useStateRef = (state: Ref<State>) => {
	...
}
```

그럼 types 파일에 인터페이스를 정의하는데, 이 때는 클래스의 모든 메소드들을 각각 정의를 요구하는데, 당황하거나 번거로울 수 있으니 주의를 요한다.

```tsx
// type.ts

export interface State {
	keyword: string;
	paginateClass: PaginateClass | null;
}

export interface PaginateClass {
	getPage: () => number;
	getLimit: () => number;
	...
	clickPage: (pageNumber: number) => void;
	...
}
```