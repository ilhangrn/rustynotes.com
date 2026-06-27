+++
title = "Rust slices and indexing"
date = 2026-06-26
[taxonomies]
categories=["Study Notes"]
tags=["Rust", "Learning", "Slices", "Indexing"]
+++
---
<br>

<h2>(*Eng*) Rust slices and indexing</h2>

<p>Three questions about the same underlying rule — sized vs unsized types.</p>

<h3>How to set an array of zeros</h3>

<pre><code>let zeros: [i32; 5] = [0; 5];</code></pre>

<p>The <code>[val; N]</code> repeat expression evaluates <code>val</code> once and copies it <code>N</code> times. Works for any <code>Copy</code> type.</p>

<pre><code>let zeros: Vec&lt;i32&gt; = vec![0; 5];   // heap version
let zeros: [f64; 10] = [0.0; 10];  // or float</code></pre>

<p><strong>Embedded use case.</strong> Initialise a register buffer in one line, no memset:</p>

<pre><code>let adc_buffer: [u16; 256] = [0; 256];</code></pre>

<p><strong>The gotcha.</strong> The length is part of the type — it must be a compile-time constant.</p>

<pre><code>let n = 5;
// let bad: [i32; n] = [0; n];  // ❌ n is not a const
let good: Vec&lt;i32&gt; = vec![0; n];  // ✅ Vec is runtime-sized</code></pre>

<h3>Why <code>&amp;a[1..4]</code> instead of <code>a[1..4]</code></h3>

<p>You cannot bind a range index by value:</p>

<pre><code>let a = [10, 20, 30, 40, 50];
let slice = a[1..4];  // ❌ expected &amp;[i32], found [i32]</code></pre>

<p>The <code>[]</code> operator desugars into a method call:</p>

<pre><code>// a[1..4] means *a.index(1..4)
// Index::index(&amp;self, idx) -&gt; &amp;Self::Output
// So * dereferences &amp;[i32] back to [i32]
// And [i32] is unsized — the compiler does not know its length at compile time</code></pre>

<p>Unsized types cannot live on the stack. The fix is to borrow the result back into a fat pointer:</p>

<pre><code>let slice = &amp;a[1..4];  // ✅ &amp;[i32] — two words: data pointer + length</code></pre>

<p><code>&amp;[i32]</code> has a known size (16 bytes on 64-bit), so the stack can hold it.</p>

<p><strong>C analogy.</strong> You don&rsquo;t copy array slices by value in C either:</p>

<pre><code>int arr[5] = {10, 20, 30, 40, 50};
int *slice = &amp;arr[1];  // pointer into the original</code></pre>

<p>Rust&rsquo;s <code>&amp;[i32]</code> is that same pointer, but with the length carried alongside — bounds-checked and no off-by-ones.</p>

<p><strong>Panic-safe alternative.</strong></p>

<pre><code>let slice = a.get(1..4);  // Option&lt;&amp;[i32]&gt;</code></pre>

<p>Returns <code>None</code> instead of panicking when the range is out of bounds.</p>

<h3>Why <code>size_of_val</code> can trick you</h3>

<pre><code>let a: [u8; 4] = [10, 20, 30, 40];

size_of_val(&amp;a)       // 4 — size of the [u8; 4] value
size_of_val(&amp;a[1..3]) // 2 — size of the [u8] slice (2 elements)</code></pre>

<p><code>size_of_val</code> looks through the reference and measures the <strong>pointee</strong>, not the pointer.</p>

<pre><code>size_of::&lt;&amp;[u8; 4]&gt;()  // 8  — thin pointer (one word)
size_of::&lt;&amp;[u8]&gt;()     // 16 — fat pointer (data ptr + length)</code></pre>

<p><code>size_of</code> on the reference type measures the <strong>pointer itself</strong>.</p>

<table>
  <thead>
    <tr>
      <th align="left">Expression</th>
      <th align="center">Size</th>
      <th align="left">What it measures</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>size_of_val(&amp;a)</code></td>
      <td align="center">4</td>
      <td>the <code>[u8; 4]</code> value itself</td>
    </tr>
    <tr>
      <td><code>size_of_val(&amp;a[1..3])</code></td>
      <td align="center">2</td>
      <td>the <code>[u8]</code> slice (2 elements)</td>
    </tr>
    <tr>
      <td><code>size_of::&lt;&amp;[u8; 4]&gt;()</code></td>
      <td align="center">8</td>
      <td>the thin pointer (1 word)</td>
    </tr>
    <tr>
      <td><code>size_of::&lt;&amp;[u8]&gt;()</code></td>
      <td align="center">16</td>
      <td>the fat pointer (ptr + len)</td>
    </tr>
  </tbody>
</table>

<p>This is the same reason <code>&amp;a[1..4]</code> is required — <code>[u8]</code> is unsized, but <code>&amp;[u8]</code> is a known-size fat pointer the compiler can place on the stack.</p>

<h3>The big picture</h3>

<table>
  <thead>
    <tr>
      <th align="left">Problem</th>
      <th align="left">C habit</th>
      <th align="left">Rust way</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Zero-init array</td>
      <td><code>int buf[256] = {0}</code></td>
      <td><code>let buf = [0u16; 256]</code> — repeat expr, no memset</td>
    </tr>
    <tr>
      <td>Slice of existing array</td>
      <td><code>int *p = &amp;arr[1]</code></td>
      <td><code>let s = &amp;a[1..4]</code> — <code>[T]</code> unsized, <code>&amp;</code> makes <code>&amp;[T]</code></td>
    </tr>
    <tr>
      <td>Check slice size</td>
      <td><code>sizeof</code> on array</td>
      <td><code>size_of_val(&amp;s)</code> — measures pointee, not pointer</td>
    </tr>
  </tbody>
</table>

<p><strong>The core lesson.</strong> Rust gives you the same low-level control as C — zero-cost slices, no hidden copies — but forces you to be explicit about sized vs unsized types. When the compiler says <code>[i32]</code> is unsized, it is telling you &ldquo;you need a reference, not a bare slice value.&rdquo; Once that clicks, <code>&amp;a[1..4]</code> becomes instinct.</p>
