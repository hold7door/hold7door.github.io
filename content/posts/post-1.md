---
title: "Kickstart Round G 2020 Combination Lock"
date: 2021-01-02T02:01:58+05:30
description: "Kickstart Round G 2020 Combination Lock"
tags: [coding-problem, kickstart]
---

This is a reiteration of the official Editorial supported with code. Link to Problem - <a href="https://codingcompetitions.withgoogle.com/kickstart/round/00000000001a0069/0000000000414a24" target="_blank">Combination Lock</a>. _I would suggest you to read the original problem statement first before going through this editorial._

### Problem

Your are given an array of numbers of length **W** and an integer **N**. Each number of array, let's say **W<sub>i</sub>** is in range **1 <= W<sub>i</sub> <= N**.
In one move you can select any number from the array and increase/decrease that number by 1, _wrapping around between 1 and N_. For example, let's N=6, and we choose 6. If we increase it should wrap around to 1. Similarly, if 1 is decreased, it becomes 6.

**You need to find the minimum number of moves to make all elements of array equal.**

##### Constraints

1 <= W <= 10<sup>2</sup>  
2 <= N <= 10<sup>9</sup>

#### A Brute Force Solution

> Observation 1: The moves performed at any index/position in the array is independent of moves performed at any other position.

If we choose a target value **T** and we want each element to be equal to T, then the moves we perform at any position in array won't affect moves performed at other positions. Thus, the number of moves at every position can be calculated separately.
There are N possible values of T i.e 1 <= T <= N.

> Observation 2: There are two possible 'number of moves' for a value _X_ to reach T. If T >= X, they are: **T - X and N - (T - X)**. If X >= T then **X - T and N - (X - T)**.

Using above two observation we can construct the brute force solution as

- For each value of T, 1 <= T <= N
  - Calculate number of moves for each position separately and add all of them. Number of moves at each position should be minimum of the two possible values (Observation 2).
- Find minimum of the total moves for all values of T.

Time complexity of this approach is: **O(N \* W)**. We are performing W operations for each value of N.  
Unfortunately, this is not sufficient to pass the given constraints.  
Continue, if you have properly understood the brute force solution.

#### Improvement of the brute force solution

> Observation 3: A major observation is that we can get the minimum possible moves by bringing all the array values to one of the initial values of the array.

##### Proof

First let's sort the array. Now, consider a value **val** that is not equal to any of the initial values of array and we want _val_ to be the target value. Also, consider two numbers _from the initial array_: **X<sub>i</sub>** and **X<sub>j</sub>** such that X<sub>i</sub> is the last number smaller than _val_ and X<sub>j</sub> is the first number larger than _val_.

To optimally reach _val_ each number from initial array has to first reach X<sub>i</sub> or X<sub>j</sub>. After performing those operations, let n<sub>1</sub> numbers are at X<sub>i</sub> and n<sub>2</sub> numbers are at X<sub>j</sub>. Then the final number of moves for all numbers to reach _val_ would be equal to:  
<code>(val - X<sub>i</sub>) _ n1 + (X<sub>j</sub> - val) _ n2</code>.  
 We can show that this can always be improved by shifting _val_ to either X<sub>i</sub> or X<sub>j</sub>

1.  If n1 < n2 then we can shift val to X<sub>j</sub> and the number of operations would become <code>(val - X<sub>i</sub>) _ n1 + (X<sub>j</sub> - val) _ n1</code>. We are transforming numbers at X<sub>i</sub> to value X<sub>j</sub>. The numbers at X<sub>j</sub> are already at correct place so no operations for them. Similarly,
2.  If n2 <= n1 then we can shift val to X<sub>i</sub> and the number of operations would become <code>(val - X<sub>i</sub>) _ n2 + (X<sub>j</sub> - val) _ n2</code>

If you compare then in both cases we have lesser number operations than keeping _val_ unchanged.

#### The Optimal Solution

So, now we know that the target value is one of the intial values of the array.

_Remember that the array is sorted._

> Observation 4: Consider, one of the initial values as target value **T** and two numbers **X1** and **X2** such that X1 < X2 <= T. Then, it is never possible that X1 reaches T by performing increment operations only and X2 reaches T by performing decrement operations only. _If the optimal way for X1 to reach T is by perfoming increment operations then X2 would also prefer increasing as it closer to T than X1_.

So, from this we can also conclude that -

1.  There exists some position **i**, _0 <= i <= t_ such that for all _j_, _0 <= j < i_ values at position _j_ prefer decreasing and for all k, _i <= k <= t_ would prefer increasing.
2.  There exists some position **u**, _t <= u < W_ such that for all _v_, _t <= v <= u_ values at position _v_ prefer decreasing and for all w, _u < w < W_ would prefer increasing.

**t** is the index of _T_.
**We can binary search for _i_ and _u_ for every _t_ we choose.**  
 Let's see how to find _i_ for certain _t_. _u_ can be found similarly.

Consider the following scenerio. We know that _i_ is in range [0, t]. Consider mid point of this range let's say _x_.  
 ![Image](/static/3.png)

If, `T-arr[x] > N - (T-arr[x])` this means that value at x will increase to reach T. This also means that all positions greater than _x_ would also prefer increasing.  
 So, from this we can say that either `i = x` or `i < x` and we can omit the range [x, t]. We have omitted exactly half of the range which is the essence behind Binary Search. We proceed similarly with the other half until _i_ is found. (See code for implementation details)

Now, let's calculate total moves once we know _i_ and _u_.  
Let's also assume that we have a function **getSum(m, n)** which returns **arr[m] + arr[m+1] ... arr[n]** i.e sum of all numbers from postion _m_ to _n_ (inclusive).
Extending on the above points we have

1.  For range, 0 <= j < i, we have, (Recall Observation 2)

    <code>

        sum1 = (N - T + arr[i-1]) + (N - T + arr[i-2]) ...

        sum1 = i * (N - T) + (arr[i-1] + arr[i-2] .. + arr[0])

        sum1 = i * (N - T) + getSum(0, i-1)

    </code>

2.  For i <= k <= t,

    <code>

        sum2 = (T - arr[i]) + (T - arr[i+1]) ..

        sum2 = T * (t - i + 1) - getSum(i, t)

    </code>

Similarly, 3. For t <= v <= u, `sum3 = getSum(t, u) - (u - t + 1) * T`

4. For u < w < W, `sum4 = (N + T) * (W - u - 1) - getSum(u+1, W-1)`

Thus the total moves for target value T at position t denoted by _sum_ is,  
 `sum = sum1 + sum2 + sum3 + sum4`

Also, _getSum(m, n) = prefixSum[n] - prefixSum[m-1]_  
where, prefix[x] is the sum of elements with index less than equal to x. We can preprocess and store prefix[x] to achive a constant time operation to calculate _sum_.

So, we calculate _sum_ for all possible initial values and take minimum of them which is the answer.

Time complexity of this solution is: **O(W \* log W)**

**For implemention see below -**

```
#include <bits/stdc++.h>

using namespace std;

typedef long long ll;


int w, n;
vector<ll> prefix;

ll getSum(int l, int r){
	if (l <= r){
		if (l <= 0) return prefix[r];
		return prefix[r] - prefix[l-1];
	}
	return 0;
}

int main(){
	int t, tc = 0;
	cin >> t;
	while (t--){
		tc ++;
		cin >> w >> n;
		vector<ll> arr(w);
		for (int i=0; i<w; i++)
			cin >> arr[i];
		sort(arr.begin(), arr.end());
		prefix = vector<ll>(w, 0);
		prefix[0] = arr[0];
		for (int i=1; i<w; i++){
			prefix[i] = prefix[i-1] + arr[i];
		}

		ll res = LLONG_MAX;
		for (int i=0; i<w; i++){
			ll op1, op2;
			int p, q;

			int lo = 0, hi = i;
			while (lo <= hi){
				int mid = lo + (hi - lo) / 2;
				op1 = arr[i] - arr[mid];
				op2 = n - op1;
				if (op1 <= op2){
					p = mid;
					hi = mid - 1;
				}
				else{
					lo = mid + 1;
				}
			}

			lo = i, hi = w - 1;
			while(lo <= hi){
				int mid = lo + (hi - lo) / 2;
				op1 = arr[mid] - arr[i];
				op2 = n - op1;
				if (op1 <= op2){
					q = mid;
					lo = mid + 1;
				}
				else{
					hi = mid - 1;
				}
			}
			ll sm1 = 0, sm2 = 0, ans = 0;
			sm1 = (i - p + 1) * arr[i] - getSum(p, i);
			sm2 = p * (n - arr[i]) + getSum(0, p-1);
			ans += sm1 + sm2;

			sm1 = 0, sm2 = 0;
			sm1 = getSum(i, q) - (q - i + 1) * arr[i];
			sm2 = (n + arr[i]) * (w - q - 1) - getSum(q+1, w-1);
			ans += sm1 + sm2;
			res = min(res, ans);
		}
		cout << "Case #" << tc << ": " << res << endl;
	}
}
```
