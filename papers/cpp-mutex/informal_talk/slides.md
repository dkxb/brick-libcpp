---
theme: default
title: 'C++ mutexes: behavior and verification'
info: |
  Expanded informal talk with selected program examples from
  A Separation Logic Formalization of the C++ Mutex Library.
  Audience: basic separation logic. No fixed time budget.
class: cover
layout: default
colorSchema: light
canvasWidth: 1280
aspectRatio: 16/9
fonts:
  sans: Arial
  mono: monospace
  provider: none
transition: fade
mdc: false
twoslash: false
wakeLock: false
drawings:
  enabled: false
exportFilename: cpp-mutex-informal-talk
---

# A Separation Logic Formalization of the C++ Mutex Library



<div class="presenter"><span class="presenter-name">Ke Du</span><time datetime="2026-10-28">10/28/2026</time></div>




---


# std::mutex for mutual exclusion

<div class="paper-code"><div class="code-pair"><div class="code-panel"><strong>mutex ensures mutual exclusion</strong><pre class="example-code"><code>unsigned int x = 0;
std::mutex m{};
<span></span>
void add_two() {
<span class="example-focus">  m.lock();</span>
<span class="example-focus">  x += 2;</span>
<span class="example-focus">  m.unlock();</span>
}</code></pre></div><div class="code-panel"><strong>Running add_two concurrently</strong><pre class="example-code"><code>int main() {
  std::thread t(add_two);
  add_two();
  t.join();
}</code></pre></div></div></div>

While one caller holds m, the other waits at <code>m.lock()</code>.


---

# Lock specifications in separation logic


$$
\{\ell \mapsto \mathsf{lock}(P)\}
\quad \texttt{lock}(\ell) \quad
\{\ell \mapsto \mathsf{lock}(P) * \mathsf{locked}(\ell) * P\}
$$

$$
\{\ell \mapsto \mathsf{lock}(P) * \mathsf{locked}(\ell) * P\}
\quad \texttt{unlock}(\ell) \quad
\{\ell \mapsto \mathsf{lock}(P)\}
$$

$\ell \mapsto \mathsf{locked}(\ell)$ says $l$ stores a lock, protecting resource $P$.



---

# Cross-thread unlock

<div class="code-pair">
  <div class="code-panel">
    <strong>Thread L</strong>
    <pre class="example-code"><code><span class="example-focus">acquire(a);</span>
*x = *x + 1;
release(b);</code></pre>
  </div>
  <div class="code-panel">
    <strong>Thread R</strong>
    <pre class="example-code"><code>acquire(b);
*y = *x + 1;
<span class="example-focus">release(a);</span></code></pre>
  </div>
</div>

Pass $\mathsf{locked(a)}$ and $x$ to thread $R$ with lock $b$.

Works in heap lang, forbidden in C++.

---

# recursive_mutex: can acquire multiple times

<div class="paper-code"><div class="code-strip"><pre class="example-code"><code>unsigned long balance_a, balance_b;
std::recursive_mutex m{};</code></pre></div>

<div class="code-pair"><div class="code-panel"><strong>Update the first balance</strong><pre class="example-code"><code>void update_a(long x) {
  m.lock();
  balance_a += x;
  m.unlock();
}</code></pre></div><div class="code-panel"><strong>Update the second balance</strong><pre class="example-code"><code>void update_b(long x) {
  m.lock();
  balance_b += x;
  m.unlock();
}</code></pre></div></div></div>

---

# Atomic Account Transfer With Recursive Mutex

<div class="paper-code"><div class="code-panel"><strong>Outer call · lock count starts at 0</strong><pre class="example-code"><code>void transfer(int x) {
  m.lock();            // 0 -> 1
  update_a(x);         // 1 -> 2 -> 1
  update_b(-(long)x);  // 1 -> 2 -> 1
  m.unlock();          // 1 -> 0
}</code></pre></div></div>

The thread owns m while its lock count is positive.


---

# shared_mutex: Reader-Writer Lock

<div class="paper-code"><div class="code-panel"><pre class="example-code"><code>bool resource = false;
std::shared_mutex m{};
<span></span>
void set_resource() {
<span class="example-focus">  m.lock();</span>
<span class="example-focus">  resource = true;</span>
<span class="example-focus">  m.unlock();</span>
}</code></pre></div></div>

While this writer holds m, readers and other writers wait.



---

# shared_mutex: Reader-Writer Lock

<div class="paper-code"><div class="code-panel"><pre class="example-code"><code>void await_resource() {
  for (;;) {
<span class="example-focus">    m.lock_shared();</span>
<span class="example-focus">    bool ret = resource;</span>
<span class="example-focus">    m.unlock_shared();</span>
    if (ret)
      break;
  }
}</code></pre></div></div>

Several <code>await_resource</code> calls may read at once.



---

# lock_guard: RAII wrapper of a BasicLockable object

<div class="paper-code"><div class="code-panel"><pre class="example-code"><code>std::mutex m;
int res;
<span></span>
void f() {
<span class="example-focus">  std::lock_guard&lt;std::mutex&gt; lock(m);</span>
<span class="example-focus"> // critical section </span>
}</code></pre></div></div>

Named requirements: informal C++ contracts.

<code>m.lock()</code>: Blocks until a lock can be acquired for the current thread.

<code>m.unlock()</code>: Unlocks the lock.


---

# Requirements revisited

<div class="rows">
  <div class="row"><strong>Unlock</strong><span>by a thread that did not acquire the mutex</span></div>
  <div class="row"><strong>Reacquire</strong><span>an ordinary mutex already owned by this thread</span></div>
  <div class="row"><strong>Destroy</strong><span>a mutex while any thread holds it</span></div>
</div>

<div class="callout">Violation: <strong>undefined behavior in C++.</strong></div>

---

# Contribution

<div class="rows">
  <div class="row"> Thread-aware specs for mutexes, RAII wrappers </div>
  <div class="row"> Formalize "BasicLockable" </div>
  <div class="row"> Reference implementations for mutexes, satisfy our specs and passed libC++ test suite  </div>
  <div class="row"> Case study: data structure predicate that uses lock under the hood is not tied to a thread and still transferrable.  </div>
</div>

---

# BRiCk: an Iris-based program logic for C++

<div class="rows">
  <div class="row"><strong>C++ support</strong><span> class, template. Translates C++ to an AST defined in Rocq. </span></div>
  <div class="row"><strong>Iris based</strong><span>Ghost state, invariants, WP. No semantics, axiomatized proof rules. </span></div>
  <div class="row"><strong>Thread-local resource</strong><span>Every predicate is implicitly indexed by a thread ID.</span></div>
</div>

$\mathsf{tid}(t)$ a special proposition that says **“the current thread is $t$.”**. Cannot transfer between threads.

Other (primitive) assertions can move between threads, like in most CSLs.

---

# std::mutex: lock and unlock

<div class="mutex-share-note">

$N$: **namespace**; $q, \pi$: fraction.

$$
m \mapsto_{\pi_1} \mathsf{mutex}_{N,\gamma}(P)
* m \mapsto_{\pi_2} \mathsf{mutex}_{N,\gamma}(P)
\dashv\vdash
m \mapsto_{\pi_1+\pi_2} \mathsf{mutex}_{N,\gamma}(P)
\qquad (\pi_1+\pi_2 \le 1)
$$

</div>

<div class="mutex-specifications mutex-operation-specifications">

$$
\begin{gathered}
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{mutex}_{N,\gamma}(P) * \mathsf{not\_locked}_{N,\gamma}(t,q)\} \\
\texttt{m.lock()} \\
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{mutex}_{N,\gamma}(P) * \mathsf{m\_locked}_{N,\gamma}(t,q) * P\}
\\[1em]
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{mutex}_{N,\gamma}(P) * \mathsf{m\_locked}_{N,\gamma}(t,q) * \triangleright P\} \\
\texttt{m.unlock()} \\
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{mutex}_{N,\gamma}(P) * \mathsf{not\_locked}_{N,\gamma}(t,q)\}
\end{gathered}
$$

</div>






---

# std::mutex: constructor and destructor

<div class="mutex-specifications">

$$
\begin{gathered}
\{\triangleright P\} \\
\texttt{mutex()} \\
\{m.\; \exists \gamma.\; m \mapsto \mathsf{mutex}_{N,\gamma}(P) * \mathsf{token}_{\gamma}(1)\}
\\[1em]
\{m \mapsto \mathsf{mutex}_{N,\gamma}(P) * \mathsf{token}_{\gamma}(1)\} \\
\texttt{m.}\!\sim\!\texttt{mutex()} \\
\{P\}
\end{gathered}
$$

</div>


---

# Registration rules

$\mathsf{mutex\_perm}_t(A)$ says thread $t$ can register for mutex that has namespace $A$. A namespace is a set.
A thread is given $\mathsf{mutex\_perm}_t(\top)$ when spawn.

$$
\mathsf{mutex\_perm}_t(A \uplus B)
\dashv\vdash
\mathsf{mutex\_perm}_t(A) * \mathsf{mutex\_perm}_t(B)
$$ 
where $A \cap B = \varnothing$.

$$
\mathsf{mutex\_perm}_t(N) * \mathsf{token}_{\gamma}(q)
\xrightleftarrows[\text{unregister}]{\text{register}}
\mathsf{not\_locked}_{N,\gamma}(t,q)
$$

---

# Specs should ensure that

<div class="rows mutex-checklist">

<div class="row">

**✓ Unlock**

Only the thread that acquired the mutex may unlock it.

</div>

<div class="row">

**✓ Reacquire**

Forbidden if this thread already owns the same ordinary mutex.

</div>

<div class="row">

**✓ Destroy**

Only when no thread holds the mutex.

</div>
</div>

$$
\mathsf{m\_locked}_{N,\gamma}(t,q)
$$



---

# Verifying add_two

<div class="annotated-program">

$$
\begin{array}{l}
\texttt{unsigned int x = 0;} \\
\texttt{std::mutex m\{\};} \\
\texttt{// } P \triangleq \exists n.\; x \mapsto n * \mathsf{even}(n)
\\[0.4em]
\textcolor{#087f8c}{\{\mathsf{tid}(t) * m \mapsto_{1/2} \mathsf{mutex}_{N,\gamma}(P) * \mathsf{token}_{\gamma}(1/2) * \mathsf{mutex\_perm}_t(N)\}} \\
\texttt{void add\_two() \{} \\
\quad \textcolor{#087f8c}{\Rightarrow \{\mathsf{tid}(t) * m \mapsto_{1/2} \mathsf{mutex}_{N,\gamma}(P) * \mathsf{not\_locked}_{N,\gamma}(t,1/2)\}} \\
\quad \texttt{m.lock();} \\
\quad \textcolor{#087f8c}{\{\ldots * P * \mathsf{m\_locked}_{N,\gamma}(t,1/2)\}} \\
\quad \texttt{x += 2;} \\
\quad \textcolor{#087f8c}{\{\ldots * P * \mathsf{m\_locked}_{N,\gamma}(t,1/2)\}} \\
\quad \texttt{m.unlock();} \\
\quad \textcolor{#087f8c}{\{\mathsf{tid}(t) * m \mapsto_{1/2} \mathsf{mutex}_{N,\gamma}(P) * \mathsf{not\_locked}_{N,\gamma}(t,1/2)\} \Rightarrow} \\
\texttt{\}} \\
\textcolor{#087f8c}{\{\mathsf{tid}(t) * m \mapsto_{1/2} \mathsf{mutex}_{N,\gamma}(P) * \mathsf{token}_{\gamma}(1/2) * \mathsf{mutex\_perm}_t(N)\}}
\end{array}
$$

</div>



---

# Verifying the two-thread client

<div class="annotated-program">

$$
\begin{array}{l}
\texttt{int main() \{} \\
\quad \textcolor{#087f8c}{\{\mathsf{tid}(t_0) * m \mapsto \mathsf{mutex}_{N,\gamma}(P) * \mathsf{token}_{\gamma}(1) * \mathsf{mutex\_perm}_{t_0}(\top)\}} \\
\quad \texttt{std::thread t(add\_two);} \\
\quad \textcolor{#087f8c}{\{\mathsf{tid}(t_0) * m \mapsto_{1/2} \mathsf{mutex}_{N,\gamma}(P) * \mathsf{token}_{\gamma}(1/2) * \mathsf{mutex\_perm}_{t_0}(\top)\}} \\
\quad \textcolor{#087f8c}{\Rightarrow \{\ldots * \mathsf{mutex\_perm}_{t_0}(N) * \mathsf{mutex\_perm}_{t_0}(\top \setminus N)\}} \\
\quad \texttt{add\_two();} \\
\quad \textcolor{#087f8c}{\{\ldots * \mathsf{mutex\_perm}_{t_0}(N) * \mathsf{mutex\_perm}_{t_0}(\top \setminus N)\}} \\
\quad \textcolor{#087f8c}{\Rightarrow \{\mathsf{tid}(t_0) * m \mapsto_{1/2} \mathsf{mutex}_{N,\gamma}(P) * \mathsf{token}_{\gamma}(1/2) * \mathsf{mutex\_perm}_{t_0}(\top)\}} \\
\quad \texttt{t.join();} \\
\quad \textcolor{#087f8c}{\{\mathsf{tid}(t_0) * m \mapsto \mathsf{mutex}_{N,\gamma}(P) * \mathsf{token}_{\gamma}(1) * \mathsf{mutex\_perm}_{t_0}(\top)\}} \\
\texttt{\}}
\end{array}
$$

</div>



---

# std::recursive_mutex

<div class="paper-code"><div class="code-panel"><pre class="example-code"><code>std::recursive_mutex rm;
<span></span>
// not locked
<span class="example-focus">rm.lock();   // acquired</span>
<span class="example-focus">rm.lock();   // same thread!</span>
...
rm.unlock();
<span class="example-focus">rm.unlock(); // really released the lock</span></code></pre></div></div>



---

# The $holds$ predicate

<div class="recursive-specifications">

An ordinary mutex uses $\mathsf{m\_locked}_{N,\gamma}(t,q) * P$.

A recursive mutex uses $\mathsf{holds}_{N,\gamma}(t,n,q,P)$; $n$ is the acquisition depth.



$$
\mathsf{holds}_{N,\gamma}(t,0,q,P)
\triangleq \mathsf{not\_locked}_{N,\gamma}(t,q)
$$

$$
\dfrac{n>0}{
\mathsf{holds}_{N,\gamma}(t,n,q,P)
\vdash P * \left(P \mathbin{-\!*}
\mathsf{holds}_{N,\gamma}(t,n,q,P)\right)}
$$

Return $P$ to recover `holds`.<br>
An existential witness inside $P$ is hidden again.

</div>



---

# Recursive mutex rules

<div class="recursive-specifications">

$$
\begin{gathered}
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{rmutex}_{N,\gamma}(P) * \mathsf{holds}_{N,\gamma}(t,n,q,P)\} \\
\texttt{m.lock()} \\
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{rmutex}_{N,\gamma}(P) * \mathsf{holds}_{N,\gamma}(t,n+1,q,P)\}
\\[1em]
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{rmutex}_{N,\gamma}(P) * \mathsf{holds}_{N,\gamma}(t,n+1,q,P)\} \\
\texttt{m.unlock()} \\
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{rmutex}_{N,\gamma}(P) * \mathsf{holds}_{N,\gamma}(t,n,q,P)\}
\end{gathered}
$$

</div>



---

# Problem: $P$ cannot change

<div class="annotated-program">

$$
\begin{array}{l}
\textcolor{#087f8c}{\{\ulcorner n>0 \urcorner * \ldots * \mathsf{holds}_{N,\gamma}(t,n,q,\exists v.\; x \mapsto v)\}} \\
\texttt{m.lock();} \\
\textcolor{#087f8c}{\{\ldots * \mathsf{holds}_{N,\gamma}(t,n+1,q,\exists v.\; x \mapsto v)\}} \\
\textcolor{#087f8c}{\Rightarrow \{\ldots * x \mapsto a\}} \\
\texttt{x++;} \\
\textcolor{#087f8c}{\{\ldots * x \mapsto a+1\}} \\
\textcolor{#087f8c}{\Rightarrow \{\ldots * \mathsf{holds}_{N,\gamma}(t,n+1,q,\exists v.\; x \mapsto v)\}} \\
\texttt{m.unlock();} \\
\textcolor{#087f8c}{\{\ldots * \mathsf{holds}_{N,\gamma}(t,n,q,\exists v.\; x \mapsto v)\}}
\end{array}
$$

</div>



---

# The holds predicate

<div class="recursive-specifications">

$$
\mathsf{holds}_{N,\gamma}(t,n,q,P,\colorbox{#ffe8a3}{$a$})
$$

instead of $P : \mathit{PRED}$, we split it into the fixed $P : A \rightarrow \mathit{PRED}$ and the changing $a : A$.

$$
\mathsf{holds}_{N,\gamma}(t,0,q,P,\colorbox{#ffe8a3}{$a$})
\triangleq \mathsf{not\_locked}_{N,\gamma}(t,q)
$$

$$
\dfrac{n>0}{
\mathsf{holds}_{N,\gamma}(t,n,q,P,\colorbox{#ffe8a3}{$a$})
\vdash \colorbox{#ffe8a3}{$P(a)$} * \left(\colorbox{#ffe8a3}{$\forall b.\; P(b)$} \mathbin{-\!*}
\mathsf{holds}_{N,\gamma}(t,n,q,P,\colorbox{#ffe8a3}{$b$})\right)}
$$

At depth $0$, $a$ is ignored.

</div>



---

# Recursive mutex: lock and unlock

<div class="recursive-specifications">

$$
\begin{gathered}
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{rmutex}_{N,\gamma}(P) * \mathsf{holds}_{N,\gamma}(t,n,q,P,\colorbox{#ffe8a3}{$a$})\} \\
\texttt{m.lock()} \\
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{rmutex}_{N,\gamma}(P) * \colorbox{#ffe8a3}{$\exists a'.$}\;
\mathsf{holds}_{N,\gamma}(t,n+1,q,P,\colorbox{#ffe8a3}{$a'$}) * \colorbox{#ffe8a3}{$\ulcorner n>0 \Rightarrow a'=a \urcorner$}\}
\\[1em]
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{rmutex}_{N,\gamma}(P) * \mathsf{holds}_{N,\gamma}(t,n+1,q,P,\colorbox{#ffe8a3}{$a$})\} \\
\texttt{m.unlock()} \\
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{rmutex}_{N,\gamma}(P) * \mathsf{holds}_{N,\gamma}(t,n,q,P,\colorbox{#ffe8a3}{$a$})\}
\end{gathered}
$$

</div>



---

# Remember $a$ while holding the recursive mutex
<div class="annotated-program">

$$
\begin{array}{l}
\textcolor{#087f8c}{\{\ulcorner n>0 \urcorner * \ldots * \mathsf{holds}_{N,\gamma}(t,n,q,\colorbox{#ffe8a3}{$\lambda v.\; x \mapsto v$},\colorbox{#ffe8a3}{$a$})\}} \\
\texttt{m.lock();} \\
\textcolor{#087f8c}{\{\ldots * \mathsf{holds}_{N,\gamma}(t,n+1,q,\colorbox{#ffe8a3}{$\lambda v.\; x \mapsto v$},\colorbox{#ffe8a3}{$a$})\}} \\
\textcolor{#087f8c}{\Rightarrow \{\ldots * x \mapsto a\}} \\
\texttt{x++;} \\
\textcolor{#087f8c}{\{\ldots * x \mapsto a+1\}} \\
\textcolor{#087f8c}{\Rightarrow \{\ldots * \mathsf{holds}_{N,\gamma}(t,n+1,q,\colorbox{#ffe8a3}{$\lambda v.\; x \mapsto v$},\colorbox{#ffe8a3}{$a+1$})\}} \\
\texttt{m.unlock();} \\
\textcolor{#087f8c}{\{\ldots * \mathsf{holds}_{N,\gamma}(t,n,q,\colorbox{#ffe8a3}{$\lambda v.\; x \mapsto v$},\colorbox{#ffe8a3}{$a+1$})\}}
\end{array}
$$

</div>


---

# shared_mutex: take a fractional `P`

<div class="recursive-specifications">

$$
\begin{gathered}
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{smutex}_{N,\gamma}(P) * \mathsf{not\_locked}_{N,\gamma}(t,q)\} \\
\texttt{m.lock()} \\
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{smutex}_{N,\gamma}(P) * P(1) * \mathsf{sm\_locked}_{N,\gamma}(t,q)\}
\\[1em]
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{smutex}_{N,\gamma}(P) * \mathsf{sm\_locked}_{N,\gamma}(t,q) * \triangleright P(1)\} \\
\texttt{m.unlock()} \\
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{smutex}_{N,\gamma}(P) * \mathsf{not\_locked}_{N,\gamma}(t,q)\}
\end{gathered}
$$

$P$ is a fractional predicate; exclusive acquisition yields $P(1)$.

</div>


---

# shared_mutex: take a fractional `P`

<div class="recursive-specifications">

$$
\begin{gathered}
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{smutex}_{N,\gamma}(P) * \mathsf{not\_locked}_{N,\gamma}(t,q)\} \\
\texttt{m.lock\_shared()} \\
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{smutex}_{N,\gamma}(P) * \exists q_P.\;
\mathsf{sm\_reader\_locked}_{N,\gamma}(t,q,q_P) * P(q_P)\}
\\[1em]
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{smutex}_{N,\gamma}(P) * \mathsf{sm\_reader\_locked}_{N,\gamma}(t,q,q_P) * \triangleright P(q_P)\} \\
\texttt{m.unlock\_shared()} \\
\{\mathsf{tid}(t) * m \mapsto_{\pi} \mathsf{smutex}_{N,\gamma}(P) * \mathsf{not\_locked}_{N,\gamma}(t,q)\}
\end{gathered}
$$

Token fraction $q$ and protected-resource fraction $q_P$ are separate.

</div>

---

# mutex_perm: permission for thread $t$ to "use" the namespace $N$

<div class="annotated-program">

$$
\begin{array}{l}
\texttt{int main() \{} \\
\quad \textcolor{#087f8c}{\{\mathsf{tid}(t_0) * m \mapsto \mathsf{data\_structure\_handle}_{N} * \mathsf{mutex\_perm}_{t_0}(\top)\}} \\
\quad \texttt{std::thread t(op);} \\
\quad \textcolor{#087f8c}{\{\mathsf{tid}(t_0) * m \mapsto_{1/2} \mathsf{data\_structure\_handle}_{N} * \mathsf{mutex\_perm}_{t_0}(\top)\}} \\

\quad \textcolor{#087f8c}{\Rightarrow \{\ldots * \mathsf{mutex\_perm}_{t_0}(N) * \mathsf{mutex\_perm}_{t_0}(\top \setminus N)\}} \\

\quad \texttt{op();} \\
... \\
\texttt{\}}
\end{array}
$$

Assume `op` a concurrent operation, implemented by C++ concurrency primitives. Can still distribute its handle without thinking about thread ID; operation's precondition only require the corresponding $mutex\_perm$.
</div>

---

# BasicLockable: modeling generic lock object


<div class="paper-code"><div class="code-panel"><pre class="example-code"><code>std::mutex m;
int size;
<span></span>
bool get_size() {
<span class="example-focus">  std::lock_guard&lt;std::mutex&gt; lock(m);</span>
<span class="example-focus">  return size;</span>
}</code></pre></div></div>

`lock_guard` is generic over a *BasicLockable* object.

---

# BasicLockable: modeling generic lock object

<div class="lock-interface-specifications basic-lockable-details">

C++ requires BasicLockable to have these operations:

- `lock()`: blocks until the calling thread acquires the "lock".
- `unlock()`: the calling thread must own the lock; releases the lock.
</div>

---

# BasicLockable: modeling generic lock object

<div class="lock-interface-specifications basic-lockable-details">

Formally, an object is BasicLockable if we can define `{ R; do_lock; do_unlock }`

With that, we automatically have:
$$
\begin{gathered}
\{p \mapsto R(q,m) * \mathsf{do\_lock}(p,m,K)\} \\
\texttt{p.lock()} \\
\{p \mapsto R(q,m) * K\}
\\[0.65em]
\{p \mapsto R(q,m) * \mathsf{do\_unlock}(p,m,K)\} \\
\texttt{p.unlock()} \\
\{p \mapsto R(q,m) * K\}
\end{gathered}
$$

</div>






---

# std::lock_guard specifications

<div class="lock-interface-specifications">

$$
\begin{gathered}
\{p \mapsto R(q,m) * \mathsf{do\_lock}(p,m,K)\} \\
\texttt{lock\_guard(p)} \\
\{g.\; g \mapsto \mathsf{lguard}(p,q,m) * K\}
\\[1em]
\{g \mapsto \mathsf{lguard}(p,q,m) * \mathsf{do\_unlock}(p,m,K)\} \\
\texttt{g.}\!\sim\!\texttt{lock\_guard()} \\
\{K * p \mapsto R(q,m)\}
\end{gathered}
$$

</div>


---

# Related work

<div class="related-work">

| Work |  |
| --- | --- |
| **VerCors**<br>Haack et al. (2008); Amighi et al. (2013) | Tracks threads, but is hard to transfer resource (or even impossible), whereas in BRiCk `tid` is the only non-transferrable one. Two specs for locking a recursive mutex, depending on whether a thread holds it or not. 
| **Iris based CSLs**<br>VST, RustBelt, RefinedC ... | Does not track thread ID. | 
| **VeriFast**<br>Verifies Rust stdlib, including Rust mutex proof | Tracks thread ID, but lock predicate properties are axiomatized instead of proved from ghost states.  |

</div>

---

# Conclusion

<div class="conclusion">
Specs for mutexes and RAII wrappers. General locking/unlocking via BasicLockable. Verified reference implementations for the locks. 
  
A modular mutex perm ghost state, an abstraction of the permission to acquire a "mutex-like" data structure. Decouples data structure distribution and mutex permission.  

Specs and ghost states portable to other logic with a notion of `tid`.


Specs designed manually, proof assisted by AI.
</div>


---

# Future work

<div class="rows closing-topic">
  <div class="row"><strong>Other concurrency primitives</strong><span> specifications for condition variables, futures, and RCU.</span></div>
  <div class="row"><strong>Implementation verification</strong><span>Formalize the interfaces of OS primitives such as futex, verify GCC/Clang mutex implementations.</span></div>
  <div class="row"><strong>User program verification</strong><span>Verify real-world programs that use C++ concurrency libraries with our specs.</span></div>
</div>

  On going project, join us!

