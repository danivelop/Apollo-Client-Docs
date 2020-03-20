# Apollo Client
## Introduction
[Apollo client react공식문서](https://www.apollographql.com/docs/react)를 공부했던 내용들을 한글로 더 편하게 보고 가끔씩 잊어버린 내용들을 리마인드 하기 위해 기존의 공식문서에 부족한 설명이나 빠진내용을 추가하고 너무 자세하게 나와 오히려 빠른 학습에는 방해가 되는 내용은 삭제하며 내용을 정리하던 중, 이 내용을 공유하면 GraphQL을 공부하고 Apollo를 시작하는 분들에게 도움이 되지 않을까 해서 문서를 작성하게 되었습니다. Apollo공식문서도 언급되어 있듯이 기본적인 GraphQL 문법을 알고 React에서 `ApolloProvider`나 `ApolloClient`로 기본적인 셋팅이 되어있다고 가정하여 작성했습니다. GraphQL문법에 익숙하지 않으시다면 [GraphQL Guide](https://graphql-kr.github.io/learn/)를 먼저 학습하신 후 Apollo를 학습하시는 것을 추천드립니다.

> 아직 문서작성을 완료하지 못했습니다... 꾸준히 업데이트 하겠습니다

> 저도 아직 부족한게 많은 개발자라 내용중 간혹 틀린부분이 있을 수 있습니다. 그런 경우 알려주시면 더 공부하여 수정하도록 하겠습니다!

## Table of Contents
- [Query](#query)
	- [Code example](#code-example)
	- [Cache업데이트](#cache업데이트)
	- [요청상태 검사](#요청상태-검사)
	- [에러상태 검사](#에러상태-검사)
	- [쿼리 수동실행](#쿼리-수동실행)
	- [useQuery API](#usequery-api)

## Query
`useQuery` [React hook](https://reactjs.org/docs/hooks-intro.html)은 Apollo에서 쿼리를 실행하는 기본 API입니다. React 구성 요소 내에서 쿼리를 실행하려면 `useQuery`를 호출하고 GraphQL 쿼리 문자열을 매개변수로 전달합니다. 구성 요소가 렌더링되면 `useQuery`는 컴포넌트를 렌더링하는 데 사용할 수 있는 데이터 속성이 포함된 객체를 반환합니다.

### Code example
```javascript
import gql from 'graphql-tag';
import { useQuery } from '@apollo/react-hooks';

const GET_DOG_PHOTO = gql`
  query Dog($breed: String!) {
    dog(breed: $breed) {
      id
      displayImage
    }
  }
`;

function DogPhoto({ breed }) {
  const { loading, error, data } = useQuery(GET_DOG_PHOTO, {
    variables: { breed },
  });

  if (loading) return null;
  if (error) return `Error! ${error}`;

  return (
    <img src={data.dog.displayImage} style={{ height: 100, width: 100 }} />
  );
}
```
`useQuery`를 사용하여 GraphQL서버에 요청을 보내며 응답시 상태에 따라 컴포넌트를 자동으로 리렌더링 합니다. `loading`이 `true`이며 `error`가 없으면 쿼리요청 무사히 완료된 겁니다.
> Apollo client는 쿼리결과를 자동으로 cache하여 컴포넌트 리렌더링 등에 따라 쿼리 재요청시 서버로 요청을 보내지 않고 캐시된 데이터에서 찾아 응답합니다. `useQuery`의 옵션에 `fetchPolicy`을 주어 요청처리방법을 변경할 수 있습니다.(default값은 `cache-first`입니다.)
> |PROPERTY|TYPE|DESCRIPTION|
> |--|----|----|
> |`cache-first`|string|캐시를 먼저 확인 후 없으면 요청을 보냅니다.(default)|
> |`cache-only`|string|캐시만 확인하고 요청은 보내지 않습니다.|
> |`cache-and-network`|string|캐시를 통해 먼저 요청처리를 하고 이와는 별개로 최신데이터를 얻기위해 서버요청을 합니다.|
> |`network-only`|string|쿼리를 처리할 때 네트워크 요청만 사용합니다.|
> |`no-cache`|string|항상 네트워크 요청을 사용해 데이터를 처리하고 응답결과를 캐싱하지 않습니다.|
>
> **code example**
> ```javascript
> ...
> 
> function DogPhoto({ breed }) {
>  const { loading, error, data } = useQuery(GET_DOG_PHOTO, {
>    variables: { breed },
>    fetchPolicy: 'cache-and-network'
>  });
>  
> ...
> ```

### Cache업데이트
`useQuery`가 응답한 데이터를 자동으로 캐시에 저장하기 때문에 최신데이터 상태를 유지하지 못합니다(ex. 게시글목록을 보고 있는데 누군가 새게시글을 작성하더라도 이를 볼 수 없음).
Apollo는 데이터를 항상 최신상태로 유지하는 방법을 2가지 제공합니다.
- Polling
- Refetching

#### Polling
```javascript
...

function DogPhoto({ breed }) {
  const { loading, error, data } = useQuery(GET_DOG_PHOTO, {
    variables: { breed },
    skip: !breed,
    pollInterval:  500,
  });
  
...
```
`useQuery`의 옵션으로 `pollInterval`을 입력해주면 `0.5`초마다 서버로 요청을 보내게 됩니다. 만약, 응답받은 데이터가 기존과 다르다면 캐시를 업데이트하고 컴포넌트를 리렌더링 합니다. 만약, 데이터가 같다면 아무일도 발생하지 않습니다. 또한, `pollInterval`을 `0`으로 주게되면 polling이 일어나지 않습니다.
> `skip`이 `true`라면 쿼리요청은 일어나지 않습니다. 위의 예에서는 `breed`가 `undefined`라면 쿼리요청이 무시됩니다.

> `useQuery`로부터 반환되는 `startPolling`과 `stopPolling`을 사용하여 polling이 일어나는 시점을 동적으로 정할 수 있습니다.
> |PROPERTY|TYPE|DESCRIPTION|
> |----|----|----|
> |`startPolling`|(interval: number) => void|`ms`형태의 interval간격을 인자로 받아 polling을 발생시킵니다.|
> |`stopPolling`|() => void|polling을 중지시킵니다.|

#### Refetching
```javascript
...

function DogPhoto({ breed }) {
  const { loading, error, data, refetch } = useQuery(GET_DOG_PHOTO, {
    variables: { breed },
    skip: !breed,
  });
  
...

  return (
    <div>
      <img src={data.dog.displayImage} style={{ height: 100, width: 100 }} />
      <button onClick={() => refetch()}>Refetch!</button>
    </div>
  );
}
```
`useQuery`는 `refetch`함수를 반환하며 호출시 refetching이 발생합니다. `button`을 클릭시 refetching이 발생합니다.

### 요청상태 검사
위의 예에서는 refetching을 하게 되면 요청상태를 알 수 없습니다. `useQuery`가 반환하는 `loading`는 초기 요청상태만을 나타내기 때문입니다. 뿐만 아니라, 이 외에도 쿼리의 요청상태를 더 자세히 알아야 할 경우가 있습니다. 이와 같은 경우엔 다음과 같은 방법으로 상태를 알 수 있습니다.
```javascript
...

function DogPhoto({ breed }) {
  const { loading, error, data, refetch, networkStatus } = useQuery(
    GET_DOG_PHOTO,
    {
      variables: { breed },
      skip: !breed,
      notifyOnNetworkStatusChange: true,
    },
  );

  if (networkStatus === 4) return 'Refetching!';
  if (loading) return null;
  if (error) return `Error! ${error}`;

  return (
    <div>
      <img src={data.dog.displayImage} style={{ height: 100, width: 100 }} />
      <button onClick={() => refetch()}>Refetch!</button>
    </div>
  );
}
```
`useQuery`옵션에 `notifyOnNetworkStatusChange`의 값을 `true`로 주고 `useQuery`로부터 반환되는 `networkStatus`를 사용하여 요청의 상태를 알 수 있습니다. `networkStatus `는 1~8까지의 정수로 각각 다음과 같은 상태를 나타냅니다.
|networkStatus|DESCRIPTION|
|--|--|
|1|처음 실행되는 쿼리에 대해 현재 요청중|
|4|`refetch`가 호출되었고 현재 요청중|
|6|현재 polling중|
|7|현재 아무 요청도 진행중이 아니며 에러도 없는 경우|
|8|요청이 진행중이지 않지만 에러가 있는경우|
> 나머지 상태나 더 자세한 내용은 [source](https://github.com/apollographql/apollo-client/blob/master/src/core/networkStatus.ts)를 참고하세요.

### 에러상태 검사
`useQuery`의 옵션으로 `errorPolicy`를 설정하면 에러에 따른 응답 데이터를 조정할 수 있습니다. `errorPolicy`의 값으로는 `none`와 `all`을 줄 수 있으며 default값은 `none`입니다.

```javascript
...

function DogPhoto({ breed }) {
  const { loading, error, data } = useQuery(GET_DOG_PHOTO, {
    variables: { breed },
    errorPolicy: 'none'
  });
  console.log(data)  // undefined
  
...
```
`errorPolicy`의 값으로 'none'를 주게 되면 응답과정에서 에러가 발생 했을 경우 `data`의 값은 `undefined`가 됩니다.
```javascript
...

function DogPhoto({ breed }) {
  const { loading, error, data } = useQuery(GET_DOG_PHOTO, {
    variables: { breed },
    errorPolicy: 'all'
  });
  console.log(data)  // { dog: null }
  
...
```
하지만, `errorPolicy`의 값으로 'all'를 주게 되면 응답과정에서 에러가 발생하더라도 `data`의 값은 포맷은 일치하지만 값으로 `null`이 들어간 응답객체가 됩니다.
> 응답에서 반환되는 `error`객체는 `graphQLErrors`, `networkError`, `message`, `extraInfo`를 키값으로 가지고 있는 객체이며 에러메세지는 `error.graphQLErrors[0].message`에 들어가게 됩니다.

### 쿼리 수동실행
쿼리는 일반적으로 컴포넌트가 마운트 및 렌더링 되고 `useQuery`가 호출되면 자동으로 실행됩니다. 하지만, 쿼리를 수동으로 실행하고 싶다면 `useLazyQuery`를 사용해야 합니다.
```javascript
...

import React, { useState } from 'react';
import { useLazyQuery } from '@apollo/react-hooks';

...

function DogPhoto({ breed }) {
  const [dog, setDog] = useState(null);
  const [getDog, { loading, data }] = useLazyQuery(GET_DOG_PHOTO);

  if (loading) return <p>Loading ...</p>;

  if (data && data.dog) {
    setDog(data.dog);
  }

  return (
    <div>
      {dog && <img src={dog.displayImage} />}
      <button onClick={() => getDog({ variables: { breed: 'bulldog' } })}>
        Click me!
      </button>
    </div>
  );
}
```
컴포넌트가 렌더링 되고 `useLazyQuery`를 호출해도 쿼리가 바로 실행되지 않고 버튼 클릭시 `useLazyQuery`에서 반환된 `getDog`함수를 호출하여 쿼리를 실행합니다. `getDog`의 매개변수대신 해당값을 `useLazyQuery`의 옵션에 넣어도 결과는 동일합니다.

### useQuery API

#### Options
|OPTION|TYPE|DESCRIPTION|
|--|--|--|
|`query`|DocumentNode|graphql 쿼리|
|`variables`|{ [key: string]: any }|쿼리실행에 필요한 변수|
|`pollInterval`|number|`ms`형태의 polling interval 간격|
|`notifyOnNetworkStatusChange`|boolean|요청에 따른 network status를 업데이트할지 여부|
|`fetchPolicy`|FetchPolicy|네트워크 요청전략.|
|`errorPolicy`|ErrorPolicy|에러여부에 따른 `data`포맷 핸들링|
|`ssr`|boolean|server-side-rendering동안 쿼리 skip여부|
|`displayName`|string|React 개발툴에 나타내기 위한 컴포넌트 이름. default값은 'Query'.|
|`skip`|boolean|만약 `true`라면 쿼리가 실행되지 않습니다. `useLazyQuery`과 함께 사용할 수 없습니다.|
|`onCompleted`|(data: TData `|` {}) => void|쿼리가 성공적으로 종료될때 호출되는 콜백함수|
|`onError`|(error: ApolloError) => void|쿼리요청중 에러발생시 호출되는 콜백함수|
|`context`|Record<string, any>|컴포넌트와 network interface간에 공유되는 context|
|`partialRefetch`|boolean|...|
|`client`|ApolloClient|`ApolloClient`인스턴스|
|`returnPartialData`|boolean|...|

#### Results
|PROPERTY|TYPE|DESCRIPTION|
|--|--|--|
|`data`|TData|GraphQL쿼리의 결과를 포함하는 객체|
|`loading`|boolean|요청의 진행중 여부|
|`error`|ApolloError|`graphQLErrors`과 `networkError`의 runtime에러|
|`variables`|{ [key: string]: any }|쿼리가 실행될때 전달했던 변수를 포함하는 객체|
|`networkStatus`|NetworkStatus|network의 상태를 나타내는 값|
|`refetch`|(variables?: TVariables) => Promise`<`ApolloQueryResult`>`|쿼리를 refetch하도록 해주는 함수|
|`fetchMore`|({ query?: DocumentNode, variables?: TVariables, updateQuery: Function}) => Promise`<`ApolloQueryResult`>`|페이지네이션에 쓰이는 함수|
|`startPolling`|(interval: number) => void|주기적으로 polling이벤트를 발생시키는 함수|
|`stopPolling`|() => void|polling을 멈추는 함수|
|`subscribeToMore`|(options: { document: DocumentNode, variables?: TVariables, updateQuery?: Function, onError?: Function}) => () => void|subscription에 사용되는 함수. 반환되는 함수는 subscription취소에 사용된다.|
|`updateQuery`|(previousResult: TData, options: { variables: TVariables }) => TData|fetch, mutation, subscription외에 캐시를 업데이트 할 수 있도록 해주는 함수|
|`client`|ApolloClient|`ApolloClient`인스턴스. 주기적으로 쿼리를 없애거나 캐시에 접근할 수 있다.|
|`called`|boolean|`useLazyQuery`로부터 반환되는 속성. `useLazyQuery`가 호출되었는지 여부를 나타낸다.|
