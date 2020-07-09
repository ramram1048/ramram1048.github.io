---
layout: post
title:  "dynamic programming: N으로 표현"
categories: test-til
---

# N으로 표현
[Programmers 코딩 테스트](https://programmers.co.kr/learn/challenges): Dynamic Programming

```
문제 설명
아래와 같이 5와 사칙연산만으로 12를 표현할 수 있습니다.

12 = 5 + 5 + (5 / 5) + (5 / 5)
12 = 55 / 5 + 5 / 5
12 = (55 + 5) / 5

5를 사용한 횟수는 각각 6,5,4 입니다. 그리고 이중 가장 작은 경우는 4입니다.
이처럼 숫자 N과 number가 주어질 때, N과 사칙연산만 사용해서 표현 할 수 있는 방법 중 N 사용횟수의 최솟값을 return 하도록 solution 함수를 작성하세요.

제한사항
N은 1 이상 9 이하입니다.
number는 1 이상 32,000 이하입니다.
수식에는 괄호와 사칙연산만 가능하며 나누기 연산에서 나머지는 무시합니다.
최솟값이 8보다 크면 -1을 return 합니다.
입출력 예
N	number	return
5	12	4
2	11	3
입출력 예 설명
예제 #1
문제에 나온 예와 같습니다.

예제 #2
11 = 22 / 2와 같이 2를 3번만 사용하여 표현할 수 있습니다.

출처: https://www.oi.edu.pl/old/php/show.php?ac=e181413&module=show&file=zadania/oi6/monocyfr
```

<br />

읽어본다음 어떻게 풀어야되나? 갑갑했었는데 하나씩 직접써보니까 풀이법이 눈에보였다.

n=1일때 [5]

n=2일때 [55 (55), 5*5 (25), 5/5 (1), 5+5 (10), 5-5 (0)]

n=3일때는... 

n=1인거랑 n=2인거를 똑같이 concat, mul, div, add, sub 5개연산 돌리면 되겠다!!

<br />

그러면 n을 1씩 늘려가면서 합해봐??

지금보니까 n=2인것도 n=1이랑 n=1을 concat, mul, div, add, sub한거네.

n=3인것은 n=1 ~ n=2하고 n=2 ~ n=1인것의 **합집합**

n=4인것은 13 22 31

그럼 이거 조합을 어케하지? 뭐 어떡함 for loop 2개돌려서 노가다해야지 노가다 ㄱㄱ

<br /><br />

코드 대충 짜서 테스트돌린 결과가 나왔는데 실패함. 이유를 생각해보자...

- n=8까지만 검사하라는 문제를 안읽었음

조건문만 더 넣으면 되니까 넣어줬다.

<br />

- "나누기연산에서 나머지는 무시"하니까, 여기서 자꾸 오차가 생기는건가?

나누기했다가 곱하기하는과정이나 나누기를 여러번하는 과정에서 오차가 생기는 것 같다는 느낌을 받았다.

그런데 뭐로 테스트해야됨?? 잘 모르겠어서 일단 answer가 있는지 체크만하는 trunc가 적용된 array랑 실제 숫자계산만하는 array로 구분해서 계산해보기로함

<br />

- concat도 틀린것같다.

5 3개를 사용해서 555는 만들수있어도 (5*5)5 => 255 이런숫자를 만들수는 없음. 말이안됨

그리고 "5-5" 이런숫자도 존재하지 않음

그냥 n=1일때 N, 2일때 NN, 3일때 NNN인 경우밖에 없어. 조합은 사칙연산만 해야되고 concat은 그냥 초기화할때만 넣어주면 되겠다.

<br />

이렇게 수정해서 돌렸는데

![2020-07-09-fail.PNG](https://res.cloudinary.com/djhv99xj7/image/upload/v1594287856/jekyll/2020-07-09-fail_dcta0l.png)

으악!!

<br />

- array가 길어지는게 문제인것같은데 어떻게해야될까.

n=1에 N이 있고

n=3에서 또 N이나오면 이건 중복인건가? 중복이 아닌건가?

<br />

생각해봤는데 "중복은 아니지만, 필요없는 수"겠구나. 왜냐면 최소한의 값을 찾는건데 똑같은 값을 찾아냈으니 **최소한의 값을 찾는데는 필요없는 수**다...

그니까 중복 뿐만아니라 필요없는 수도 저장하지 않도록하자.

<br />

- 또 생각해봤는데... div를할 때 소숫점이 나오면 그것도 이미 최솟값을 구하기에는 필요없는 수다. 

소숫점의 계산결과를 계속 줄줄 들고다닐필요가 없다. 이것도 만약 div가 나누어떨어지지 않는 경우는 버리는걸로 하고

중간에 trunc값이랑 실제값을 구분해서 저장했던 것도 다시 하나로 합치자.

<br /><br />

이렇게 바꿔서 테스트 돌린 결과? 🙏🙏🙏🙏

![2020-07-09-pass.PNG](https://res.cloudinary.com/djhv99xj7/image/upload/v1594287792/jekyll/2020-07-09-pass_wlns5a.png)

성공따리~🎉

<br />

최종 코드: 

```js
function solution(N, number) {
    if(number === N) return 1
    var answer = 1;
    let dynamicArray = [[], [N,], ]
    const makeDynamicArray = (index) => {
        let set = [N.toString()]
        while(set[0].length < index){
            set[0] += N.toString()
        }
        set[0] = parseInt(set[0])
        for(let i = 1; i < index; i++){
            let j = index - i;
            dynamicArray[i].map((a) => {
                dynamicArray[j].map((b) => {
                    summantionTwoNumber(a, b).forEach((item) => {
                        // dup check
                        if(!dynamicArray.some((sets) => sets.includes(item))
                        && !set.includes(item)){
                            set.push(item) 
                        }
                    })
                })
            })
        }
        // console.log(index, set)
        return set
    }
    
    while(!dynamicArray[answer].includes(number)) {
        // console.log(dynamicArray[answer], number)
        if(answer >= 8) return -1
        answer++
        dynamicArray[answer] = makeDynamicArray(answer)
    }
    
    return answer;
}

const summantionTwoNumber = (a, b) => {
    let array = [
        a+b,
        a-b,
        a*b,
    ]
    if(b !== 0){
        if(a/b === Math.trunc(a/b)) array.push(a/b) // 소수점이면 어차피 쓸데없는값!!
    }
    return array
}
```

<br />

코드가 깔끔하지는 않지만 생각보다 금방 풀 수 있었다!