---
title: "Binary Exponentiation Extended"
date: 2022-06-29T14:54:23+05:30
draft: false
tags: []
math: true
---


I could not find any article or blog on this idea, and felt having one could be helpful to many. There might be unwanted errors, so please feel free to to share any suggestions, corrections or concerns. For those who are unware of Binary Exponentation, [this CP-Algorithms article](https://cp-algorithms.com/algebra/binary-exp.html) or [this video by Errichto](https://youtu.be/L-Wzglnm4dM?si=VL1jTQI537JWwJ4-) can help get familiar with it.

#### Introduction

Now that we know the idea behind binary exponentiation; let us try expanding the idea. Say, we break the problem into two parts, $L$ and $R$, such that, once we have both the results, combining them gives us our final answer as $ans = L \odot R$, where $\odot$ is an _associative binary operator._ If  we can find $R$ as a function of $L$ as in $R = f(L)$ fast enough, say $O(X)$, then our final answer can be found in  $O(X \cdot log(n))$.

The idea is simple enough, so we now have a look at some simple examples. For the sake of simplicity we assume problem size $n$ is of form $2^k$ , $ k \in \mathbb{Z} $ .

#### **Example 1:** Calculating $a^n$

We are starting from the simplest example. We know that $ans = \underbrace{a \cdot a \dots \cdot a \cdot a }_\text{ n times}$. If we take $R = f(L) = L$, where $L = a^{n/2}$; we can find $R$ in $O(1)$ and build our answer as -

$\newline$

$$ans = \underbrace{\underbrace{\dots \dots \dots}_{L} \odot \underbrace{\dots \dots \dots}_{R=f(L)}}_{L'} \odot \underbrace{\dots \dots \dots \dots \dots \dots \dots}_{R` = f(L')} \; \odot \dots \dots$$

We are able to compute $R'$ directly from $L'$, unlike other methods such as RMQ or Binary Lifting where we build our answer from already calculated $L$ and $R$ on the independent smaller ranges to later combine them to compute the the answer for the bigger range.

**Time Complexity:** Finding $R = f(L)$ is $O(1)$, so total time complexity is $O(log(n))$. <br/>

##### Implementation

The below function only works if $n$ is even. One simple way to handle the odd case is to break the expression as $a^n = a \cdot a^{n-1}$. We can get the answer for $a^{n-1}$ because it's even ans whether $n$ is even or odd, in $\leq 2$ steps $n$ definitely gets halved which ensures that the time complexity remains $O(log(n))$.

```cpp
int binexpo(int a,int n)
{
    if (n == 1) return a;
    if (n % 2) return a * binexpo(a, n - 1); // this line happen only #SetBits times as on every step n is halving
    int L = binexpo(a, n / 2);
    int R = L;
    return R * L;    
}
```

#### **Example 2:** Summation of GP Series

Considering a standard GP series, we find its summation as

$\newline$

$$ans = 1 + r + r^{2} \dots r^{n-1} = \frac{r^{n}-1}{r-1}$$

If we have to find the summation under arbitrary MOD then it's not necessary that $(r-1)^{-1}$ exists, and so we alternatively express our sum as 

$$ans = \underbrace{1 + r + r^{2} \dots r^{ \frac{n-1}{2} } }_\text{L} + \underbrace{r^{ \frac{n}{2} } + \dots + r^{n-1}}_\text{R} $$

which gives $R = f(L) = r^{\frac{n}{2}} \cdot L$.

To handle the case for odd $n$ during implementation, we can break series as 

$\newline$

$$ans = 1 + a \cdot \underbrace{(1 + a + a^2 + \dots + a^{n-2})}_\text{say sum is S}$$ 

We can find $S$ as $(n - 1)$ is even, and final answer will be $= 1 + a \cdot S$.

**Time Complexity:** Computing $R = f(L)$ is $O(log(n))$, so our total time complexity becomes $O(log(n)^{2})$. But we can calculate powers of $r$ along with our series, so we restrict the final time complexity to $O(log(n))$.

##### Implementation

Naive Approach ($O(log(n)^{2})$) 

```cpp
int binexpo(int x,int n){/* above code goes here */}

int GP(int r, int n)
{
    if (n == 1)
        return 1;

    if (n % 2)
        return 1 + r * GP(r, n - 1);

    int L = GP(r, n / 2);
    int R = binexpo(r, n / 2) * L;
    return L + R;
}
```

Now, applying our optimization, we compute { $ \{ r^n ,1 + r \dots + r^{n-1} \} $ } together for a final time complexity of $O(logn)$ 

```cpp
// {r^n, 1 + r + r^2 ... r^(n-1)}
pair<int, int> GP(int r, int n)
{
    if (n == 1)
        return {r, 1};

    if (n & 1) {
        auto [x, y] = GP(r, n - 1);
        return {r * x, 1 + r * y};
    }

    auto [x, L] = GP(r, n / 2);
    int R = x * L;
    return {x * x, R + L};
}
```

We can use a similar idea for calculating GP series of matrices $= I + A + A^2  + \dots + A^{n-1}$. Just like integers under modulo, we can't always guarantee $(A-I)^{-1}$ exists.

**Contest Example:** In [AtCoder ABC 293 Task E](https://atcoder.jp/contests/abc293/tasks/abc293_e) the task is to just compute the sum of a GP series. 

#### **Example 3:** Summation of AP-GP series

We start by considering the simple series $\displaystyle  r + 2 \cdot r^2 + 3 \cdot r^3 \dots + (n-1)\cdot r^{n-1} = \sum\limits_{i=0}^{n-1} i\cdot r^i$. For this we express the summation as below

$$ans = \underbrace{r + 2 \cdot r^2 + \dots + (\frac{n}{2}-1) \cdot r^{ \frac{n}{2}-1 }}_\text{L} + \underbrace{ \frac{n}{2} \cdot r^{\frac{n}{2}} + \dots + (n-1) \cdot r^{n-1}}_\text{R} $$

Now we solve for $R = f(L)$.  

##### Solution for R as a function of L

$$R = (\frac{n}{2} + 0) \cdot r^{\frac{n}{2}} + (\frac{n}{2} + 1) \cdot r^{\frac{n}{2}+1} + \dots + (\frac{n}{2} + i) \cdot r^{\frac{n}{2}+i} + \dots + (\frac{n}{2} + \frac{n}{2}-1) \cdot r^{\frac{n}{2} + \frac{n}{2}-1}$$

$$\implies R = \frac{n}{2} \cdot \left( r^{\frac{n}{2}} + r^{\frac{n}{2}+1} + \dots + r^{\frac{n}{2}+i} + \dots + r^{n-1} \right) + r^{\frac{n}{2}} \cdot \underbrace{\left( 0 \cdot r^{0} + 1 \cdot r^{1} + \dots + i \cdot r^{i} + \dots + \left(\frac{n}{2}-1\right) \cdot r^{\frac{n}{2}-1} \right)}_{\text{L}}$$


$$\implies R = r^{\frac{n}{2}} \cdot \left( \frac{n}{2} \cdot \underbrace{\left( 1 + r^{1} + \dots + r^{i} + \dots + r^{\frac{n}{2}-1} \right)}_{\text{GP series}} + L \right)$$

We shall call this GP series as $L_{0}$, whose sum we can already calculate efficiently. Hence,

$$R = r^{\frac{n}{2}} \cdot \left( \frac{n}{2} \cdot L_{0} + L \right)$$

$$\implies R  = f(L_{0},L) = r^{\frac{n}{2}} \cdot \left( \frac{n}{2} \cdot L_{0} + L \right)$$

To handle the odd case during implementation, we express summation as:

$\newline$

$$sum = r + 2\cdot r^2 + \dots + (n-1) \cdot r^{n-1}$$
$$= \left(r + r^2 + \dots + r^i + \dots + r^{n-1} \right) + \left( r^2 + 2\cdot r^3 + \dots + (i-1)\cdot r^{i-1} + \dots + (n-2)\cdot r^{n-1} \right)$$

$$= r \cdot \underbrace{\left(1 + r^1 + \dots + r^i + \dots + r^{n-2} \right)}_\text{X} + r \cdot \underbrace{\left( r^1 + 2\cdot r^2 + \dots + i\cdot r^{i} + \dots + (n-2)\cdot r^{n-2} \right)}_\text{Y}$$

We can get $X$ and $Y$ because they are of even length, and our final answer will be $= r \cdot X + r \cdot Y$ .

**Time Complexity:** To get $R$ we need to solve $L_{0}$ which takes $O(log(n))$ so total will be $O(log(n)^2)$, but we can calculate powers of $r$, $L_{0}$ and $L$ together which makes total time $O(log(n))$.

##### Implementation

```cpp
// {r^n , 1+r+r^2 ... + r^(n-1) , 0 + r + 2r^2 + ... + (n-1)r^(n-1) }
array<int, 3> APGP(int r, int n)
{
    if (n == 1)
        return {r, 1, 0};

    if (n & 1) {
        auto [x, L0, L] = APGP(r, n - 1);
        return {r * x, 1 + r * L0, r * L0 + r * L};
    }

    auto [x, L0, L] = APGP(r, n / 2);
    int R0 = x * L0;
    int R = x * (n / 2 * L0 + L);
    return {x * x, L0 + R0, L + R};
}
```

For any generic AP-GP series $\sum_{i=0}^{n-1} ( a + i \cdot b ) \cdot r^i = a \cdot \underbrace{\sum_{i=0}^{n-1} r^i}_{\text{L0}} + b \cdot \underbrace{\sum_{i=0}^{n-1} i \cdot r^i}_{\text{L}} = a \cdot L_{0} + b \cdot L$, where both are being calculated with single function call.

[AtCoder ABC 129 task F](https://atcoder.jp/contests/abc129/tasks/abc129_f) is a good practice example for the ideas in the blog.

#### Genralising the above series summations

Now, let us consider $\displaystyle S(n,m) = ‎‎\sum\limits_{i=0}^{n-1} i^m \cdot r^i $. Then we would have GP $= S(n,0)$ and AP-GP $= S(n,1)$. Hence,

$\newline$

$$\displaystyle S(n,m) = r + 2^m \cdot r^2 + \dots + i^m \cdot r^i + \dots + (n-1)^m \cdot r^{n-1}$$

$$\displaystyle  = \underbrace{r + 2^m \cdot r^2 + \dots + (\frac{n}{2}-1)^{m} \cdot r^{\frac{n}{2}-1}}_\text{L(m,n/2)} + \underbrace{{(\frac{n}{2}})^{m} \cdot r^{\frac{n}{2}}  + \dots + (n-1)^m \cdot r^{n-1}}_\text{R(m,n/2)}$$

Here $L_{m,\frac{n}{2}} = S(\frac{n}{2},m)$. Then, $S(n,m) = L_{m,\frac{n}{2}} + R_{m,\frac{n}{2}}$, We need to compute $R_{m,\frac{n}{2}}$.

$\newline$

$$\displaystyle R_{m,\frac{n}{2}} = r^{\frac{n}{2}} \cdot \left( (\frac{n}{2})^m + (\frac{n}{2}+1)^m \cdot r + \dots + (\frac{n}{2} + i)^m \cdot r^i + \dots (\frac{n}{2} + \frac{n}{2} - 1)^m \cdot r^{\frac{n}{2}-1} \right)$$

$$\displaystyle R_{m,\frac{n}{2}} = r^{\frac{n}{2}} \cdot \left( \sum\limits_{i=0}^{\frac{n}{2}-1} \left( \sum\limits_{j=0}^{m} \binom{m}{j} \left(\frac{n}{2}\right)^{m-j} \cdot i^j \right) \cdot r^i  \right) $$

Now, swapping the summations -

$$\displaystyle R_{m,\frac{n}{2}} = r^{\frac{n}{2}} \cdot \left( \sum\limits_{j=0}^{m} \binom{m}{j} \left( \frac{n}{2}\right) ^{m-j} \underbrace{\left( \sum\limits_{i=0}^{\frac{n}{2}-1} i^j \cdot r^i \right)}_\text{L(j,n/2)}   \right) $$

Finally,

$$\displaystyle R_{m,\frac{n}{2}} = r^{\frac{n}{2}} \cdot \left( \sum\limits_{j=0}^{m} \binom{m}{j} \left( \frac{n}{2}\right) ^{m-j} \cdot L_{j,\frac{n}{2}}   \right) = r^{\frac{n}{2}} \cdot \left( \sum\limits_{j=0}^{m} \binom{m}{j} \left( \frac{n}{2}\right) ^{m-j} \cdot S_{\frac{n}{2},j}   \right) $$

As $j \leq m$ if we calculate everything together, just like above and hence know the value of every $L_{j,\frac{n}{2}}$ without any extra time. We can validate our results for the above illustrated examples of AP and AP-GP series.

##### Validating our results against our previous examples

**Case 1:** for $m=0$ , which is simple GP series

$R_{0,\frac{n}{2}} = r^{\frac{n}{2}} \cdot \left( \sum\limits_{j=0}^{0} \binom{0}{j} \left( \frac{n}{2}\right) ^{0-j} \cdot L_{j,\frac{n}{2}}   \right) = r^{\frac{n}{2}} \cdot L_{0,\frac{n}{2}}$

**Case 2:** for $m=1$ , which is AP-GP series

$R_{1,\frac{n}{2}} = r^{\frac{n}{2}} \cdot \left( \sum\limits_{j=0}^{1} \binom{1}{j} \left( \frac{n}{2}\right) ^{1-j} \cdot L_{j,\frac{n}{2}}   \right) = r^{\frac{n}{2}} \cdot \left(  \left( \frac{n}{2}\right) ^{1} \cdot L_{0,\frac{n}{2}} +  L_{1,\frac{n}{2}} \right)$ 

We can now evaluate our original summation as $\displaystyle S_{n,m} = S_{\frac{n}{2},m} + r^{\frac{n}{2}} \cdot \left( \sum\limits_{j=0}^{m} \binom{m}{j} \left( \frac{n}{2}\right) ^{m-j} \cdot S_{\frac{n}{2},j}   \right)$.

##### Handling Odd Case

We have $\displaystyle S_{n,m} = \sum\limits_{i=0}^{n-1} i^m \cdot r^i = r \cdot \left( \sum\limits_{i=1}^{n-2} (i+1)^m \cdot r^i \right)$. However we can only write this if $m>0$ but for $m=0$ first term will not be $0$ but instead be $\displaystyle \lim_{x \to 0} x^x = 1$. So for $m=0$ we will add $1$ instead of $0$, keeping the remainder of the expression same. Thus 

$$\displaystyle S_{n,m} = r \cdot \left( \sum\limits_{i=0}^{n-2} \sum\limits_{j=0}^{m} \binom{m}{j} i^j \cdot r^i \right)$$

And swapping summations

$$\displaystyle S_{n,m} = r \cdot \left( \sum\limits_{j=0}^{m} \binom{m}{j} \underbrace{\sum\limits_{i=0}^{n-2} i^j \cdot r^i}_\text{S(n-1,j)} \right) = r \cdot \left( \sum\limits_{j=0}^{m} \binom{m}{j} S_{n-1,j} \right) \hspace{0.25cm} \text{for}  \hspace{0.25cm}m > 0$$

Hence, we have the final result as:

$$
\displaystyle
\begin{equation}
S_{n,m}=
    \begin{cases}
        1 + r \cdot S_{n-1,0}  & \text{if } m=0\\
        r \cdot \left(\displaystyle \sum\limits_{j=0}^{m} \binom{m}{j} S_{n-1,j} \right) & \text{if } m > 0
    \end{cases}
\end{equation}
$$

We can also construct a clean visualization of the even and odd cases using matrix multiplication instead of summation. These are illustrated as below.

##### Odd Case

Dimensions are: $(m+1,1) = (m+1,m+2) \cdot (m+2,1)$.


$$\begin{bmatrix} S_{n,0} \\ S_{n,1} \\ \vdots \\ S_{n,m} \end{bmatrix} = \begin{bmatrix} 1 & \binom{0}{0} & 0 & 0 & \dots & 0 \\ 0 & \binom{1}{0} & \binom{1}{1} & 0 & \dots & 0 \\ \vdots & \vdots & \vdots & \ddots & \vdots & \vdots \\ 0 & \binom{m}{0} & \binom{m}{1} & \binom{m}{2} & \dots & \binom{m}{m} \end{bmatrix} \cdot \begin{bmatrix} 1 \\ r \cdot S_{n-1,0} \\ r \cdot S_{n-1,1} \\ \vdots \\ r \cdot S_{n-1,m} \end{bmatrix}$$

We will return the array $(S_{n,0},S_{n,1}, \dots , S_{n,m}, r^n)$.

##### Even Case

Dimension are: $(m+1,1) = (m+1,m+1) \cdot (m+1,1)$.

$$
\begin{bmatrix} 
    R_{0,\frac{n}{2}}  \\ 
    R_{1,\frac{n}{2}}  \\ 
    \vdots   \\ 
    R_{m,\frac{n}{2}}  
\end{bmatrix} 
= r^{\frac{n}{2}} \cdot 
\begin{bmatrix} 
    \binom{0}{0} & 0 & 0 & \dots  & 0 \\ 
    \binom{1}{0} \cdot \frac{n}{2} & \binom{1}{1} & 0 & \dots  & 0 \\ 
    \vdots & \vdots & \vdots & \ddots & \vdots \\ 
    \binom{m}{0} \cdot \left(\frac{n}{2}\right)^m &  \binom{m}{1} \cdot \left(\frac{n}{2}\right)^{m-1} &  \binom{m}{2} \cdot \left(\frac{n}{2}\right)^{m-2} & \dots  &  \binom{m}{m}  
\end{bmatrix} 
\cdot 
\begin{bmatrix} 
    L_{0,\frac{n}{2}}  \\ 
    L_{1,\frac{n}{2}}  \\ 
    L_{2,\frac{n}{2}}  \\ 
    \vdots   \\ 
    L_{m,\frac{n}{2}}  
\end{bmatrix}
$$

We will return the array $(L_{0, n/2} + R_{0, n/2},L_{1, n/2} + R_{1, n/2}, \dots , L_{m, n/2} + R_{m, n/2}, r^n)$.

In the above cases, whenever we are computing values of $R_{j, n / 2} \hspace{0.25cm} \text{for} \hspace{0.25cm} j \in [0, m]$, they still indeed maintain our original form of $R = f(L)$ as we can express them as 

$$\vec{R_{\frac{n}{2}}} = f(\vec{L})= r^{\frac{n}{2}} \left( M(n,m) \cdot \vec{L_{\frac{n}{2}}} \right) $$

**Time Complexity:**  To find $R_{j,\frac{n}{2}}$ takes summation of $m$ terms and we do that for every $j \leq m$ which will take $O(m^2)$. So, total complexity is $O(m^2 \cdot log(n)).$ 

##### Implementation

1. Precalculation of $nCr$ takes $O(m^2)$.

2. During our computations, whenever we have even $n$ we need to compute $(\frac{n}{2})^j$ for each $j \leq m$, which takes $O(m \cdot log(n))$.

Total complexity $O(m^2 + m \cdot log(n) + m^2 \cdot log(n))$ asymptotically remains same $O(m^2 \cdot log(n))$ .

```cpp
using int64 = long long;
int MOD = 1e9+7;
const int M=1002;
void add(int&x,int y){
    x+=y;
    if(x>=MOD)x-=MOD;
}
void mul(int&x,int y){
    x = 1ll * x * y % MOD;
}

int ncr[M][M]{0};
int nCr(int nn,int rr){
 return ncr[nn][rr];   
}
// Precalculate calculate n choose r in O(M^2)
void nCrinit(){
    ncr[0][0]=1;
    for(int n=0;n<M;n++)ncr[n][n]=1,ncr[n][0]=1;
    for(int n=1;n<M;n++){
        for(int r=1;r<n;r++){
            add(ncr[n][r],ncr[n-1][r-1]);
            add(ncr[n][r],ncr[n-1][r]);
        }
    }
}

// Precalculation of powers of n in O(M * logN)
using vt = vector<int>;
map<int64,vt> ni;
int n_i(int64 n,int i){
    return ni[n][i];
}
void n_iinit(int64 n){
    while(n>0){
        if(n&1){
            n--;
            continue;
        }
        vt x(M); x[0]=1;
        for(int i=1;i<M;i++)x[i] = 1ll * x[i-1] * (n/2 % MOD) % MOD;
        ni[n/2] = x;
        n>>=1;
    }
    ni[1] = vector<int>(M,1);
}

// Function to generate the series {S(n,0),S(n,1),...,S(n,m),r^n}
vector<int> Series(int r,int64 n,int m)
{
    if(n==1){
        vector<int> a(m+2,0);
        a[0]=1;
        a[m+1]=r;
        return a;
    }
    if(n&1){
        // Recursive case for odd n
        vector<int>a = Series(r,n-1,m);
        vector<int> S(m+2,0);
        add(S[m+1],1ll * r * a[m+1]%MOD);
        add(S[0],1);
        add(S[0],1ll*r*a[0]%MOD);
        for(int i=1;i<=m;i++){
            for(int j=0;j<=i;j++){
                add(S[i], 1ll * nCr(i,j) * a[j] % MOD);
            }
            mul(S[i],r);
        }
        return S;
    }

    // Recursive case for even n
    vector<int> L = Series(r,n/2,m);
    vector<int> R(m+1,0);
    for(int i=0;i<=m;i++){
        for(int j=0;j<=i;j++){
            int h = 1;
            mul(h,nCr(i,j));
            mul(h,n_i(n/2,i-j));
            mul(h,L[m+1]);
            mul(h,L[j]);
            add(R[i],h);
        }
    }
    vector<int> S(m+2,0);
    S[m+1] = 1ll * L[m+1] * L[m+1] % MOD;
    for(int i=0;i<=m;i++){
        // S[i]=(MOD + R[i] + L[i])%MOD;
        add(S[i],R[i]);
        add(S[i],L[i]);
    }
    return S;
}
```

