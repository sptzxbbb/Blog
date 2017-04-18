---
title: Prime Judge
date: 2015-04-28
category: Coding
tags: [Algorithm]
---

A prime number (or a prime) is a natural number greater than 1 that has no positive divisors other than 1 and itself. 

### Trial division ###

In most cases, we need to determine whether a number is a prime.

A simple algorithm to do so is <font color="Red"> trial division</font>

The idea is to test whether <u>n is divisible by any smaller number</u>. The trial factor go no further than $\sqrt{n}$ because, if n is divisible by some number p, then $n = p × q$ and if q were smaller than p, n would have earlier been detected as being divisible by q or a prime factor of q.

<!--more-->

~~~c++
bool isPrime(int n) {
    if (n < 2) return false;
    
    if (n % 2 == 0) return false;

    for (int i = 3; i * i < n; i += 2) {
	if (n % i == 0) return false;
    }
    return true;
}

~~~

---

### Sieve ###

Sometimes, we need to find the number of the primes less than n.

The sieve is a simple but effective algorithm for this task.
The idea is that <u>if a number is a multiple of a prime n, it's not a prime apparently</u>. We start from prime 2 to take out  all those that are multiple of the prime in the table.Then look for another prime factor n sequently in the table to do the job again and again.The prime factor should not go further than $\sqrt{n}$.The same rule applies here.

~~~c++
#include "cstring"

const int Max = 1500000;

class Solution {
public:
    Solution() {
	init = false;
	memset(table, 1, sizeof(table));
    }

    int countPrimes(int n) {
	if (false == init) {
	    maketable();
	    init = true;
	}
	int ans = 0;
	for (int i = 0; i < n; ++i) {
	    if (true == table[i]) {
		++ans;
	    }
	}
	return ans;
    }

    void maketable() {
	table[0] = false;
	table[1] = false;
	for (int i = 2; i <= Max; ++i) {
	    if (true == table[i] && (i * i <= Max)) {
		int temp = 2 * i;
		while (temp <= Max) {
		    table[temp] = false;
		    temp += i;
		}
	    }
	}
    }

private:
    bool init;
    bool table[Max];
};

~~~

