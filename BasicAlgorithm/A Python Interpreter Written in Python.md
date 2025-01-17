<!DOCTYPE html>
<!-- saved from url=(0034)http://qingyunha.github.io/taotao/ -->
<html lang="en-us"><head><meta http-equiv="Content-Type" content="text/html; charset=UTF-8">

    <title>A Python Interpreter Written in Python by qingyunha</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="stylesheet" type="text/css" href="./A Python Interpreter Written in Python by qingyunha_files/normalize.css" media="screen">
    <link href="./A Python Interpreter Written in Python by qingyunha_files/css" rel="stylesheet" type="text/css">
    <link rel="stylesheet" type="text/css" href="./A Python Interpreter Written in Python by qingyunha_files/stylesheet.css" media="screen">
    <link rel="stylesheet" type="text/css" href="./A Python Interpreter Written in Python by qingyunha_files/github-light.css" media="screen">
  </head>
  <body>
    <section class="page-header">
      <h1 class="project-name">A Python Interpreter Written in Python</h1>
      <h2 class="project-tagline">Allison Kaptur</h2>
    </section>

    <section class="main-content">
<p>原文地址: <a href="http://aosabook.org/en/500L/a-python-interpreter-written-in-python.html">500lines</a></p>

<p><em>Allison是Dropbox的工程师，在那里她维护着世界上最大的由Python客户组成的网络。在Dropbox之前，她是Recurse Center的引导师, ... 她在北美的PyCon做过关于Python内部机制的演讲，并且她喜欢奇怪的bugs。她的博客地址是<a href="http://akaptur.com/">akaptur.com</a>.</em></p>

<h2>
<a id="introduction" class="anchor" href="http://qingyunha.github.io/taotao/#introduction" aria-hidden="true"><span class="octicon octicon-link"></span></a>Introduction</h2>

<p>Byterun是一个用Python实现的Python解释器。随着我在Byterun上的工作，我惊讶并很高兴地的发现，这个Python解释器的基础结构可以满足500行的限制。在这一章我们会搞清楚这个解释器的结构，给你足够的知识探索下去。我们的目标不是向你展示解释器的每个细节---像编程和计算机科学其他有趣的领域一样，你可能会投入几年的时间去搞清楚这个主题。</p>

<p>Byterun是Ned Batchelder和我完成的，建立在Paul Swartz的工作之上。它的结构和主要的Python实现（CPython）差不多，所以理解Byterun会帮助你理解大多数解释器特别是CPython解释器。（如果你不知道你用的是什么Python，那么很可能它就是CPython）。尽管Byterun很小，但它能执行大多数简单的Python程序。</p>

<h3>
<a id="a-python-interpreter" class="anchor" href="http://qingyunha.github.io/taotao/#a-python-interpreter" aria-hidden="true"><span class="octicon octicon-link"></span></a>A Python Interpreter</h3>

<p>在开始之前，让我们缩小一下“Pyhton解释器”的意思。在讨论Python的时候，“解释器”这个词可以用在很多不同的地方。有的时候解释器指的是REPL，当你在命令行下敲下<code>python</code>时所得到的交互式环境。有时候人们会相互替代的使用Python解释器和Python来说明执行Python代码的这一过程。在本章，“解释器”有一个更精确的意思：执行Python程序过程中的最后一步。</p>

<p>在解释器接手之前，Python会执行其他3个步骤：词法分析，语法解析和编译。这三步合起来把源代码转换成<em>code object</em>,它包含着解释器可以理解的指令。而解释器的工作就是解释code object中的指令。</p>

<p>你可能很奇怪执行Python代码会有编译这一步。Python通常被称为解释型语言，就像Ruby，Perl一样，它们和编译型语言相对，比如C，Rust。然而，这里的术语并不是它看起来的那样精确。大多数解释型语言包括Python，确实会有编译这一步。而Python被称为解释型的原因是相对于编译型语言，它在编译这一步的工作相对较少（解释器做相对多的工作）。在这章后面你会看到，Python的编译器比C语言编译器需要更少的关于程序行为的信息。</p>

<h3>
<a id="a-python-python-interpreter" class="anchor" href="http://qingyunha.github.io/taotao/#a-python-python-interpreter" aria-hidden="true"><span class="octicon octicon-link"></span></a>A Python Python Interpreter</h3>

<p>Byterun是一个用Python写的Python解释器，这点可能让你感到奇怪，但没有比用C语言写C语言编译器更奇怪。（事实上，广泛使用的gcc编译器就是用C语言本身写的）你可以用几乎的任何语言写一个Python解释器。</p>

<p>用Python写Python既有优点又有缺点。最大的缺点就是速度：用Byterun执行代码要比用CPython执行慢的多，CPython解释器是用C语言实现的并做了优化。然而Byterun是为了学习而设计的，所以速度对我们不重要。使用Python最大优点是我们可以<em>仅仅</em>实现解释器，而不用担心Python运行时的部分，特别是对象系统。比如当Byterun需要创建一个类时，它就会回退到“真正”的Python。另外一个优势是Byterun很容易理解，部分原因是它是用高级语言写的（Python！）（另外我们不会对解释器做优化 --- 再一次，清晰和简单比速度更重要）</p>

<h2>
<a id="building-an-interpreter" class="anchor" href="http://qingyunha.github.io/taotao/#building-an-interpreter" aria-hidden="true"><span class="octicon octicon-link"></span></a>Building an Interpreter</h2>

<p>在我们考察Byterun代码之前，我们需要一些对解释器结构的高层次视角。Python解释器是如何工作的？</p>

<p>Python解释器是一个<em>虚拟机</em>,模拟真实计算机的软件。我们这个虚拟机是栈机器，它用几个栈来完成操作（与之相对的是寄存器机器，它从特定的内存地址读写数据）。</p>

<p>Python解释器是一个<em>字节码解释器</em>：它的输入是一些命令集合称作<em>字节码</em>。当你写Python代码时，词法分析器，语法解析器和编译器生成code object让解释器去操作。每个code object都包含一个要被执行的指令集合 --- 它就是字节码 --- 另外还有一些解释器需要的信息。字节码是Python代码的一个<em>中间层表示</em>：它以一种解释器可以理解的方式来表示源代码。这和汇编语言作为C语言和机器语言的中间表示很类似。</p>

<h3>
<a id="a-tiny-interpreter" class="anchor" href="http://qingyunha.github.io/taotao/#a-tiny-interpreter" aria-hidden="true"><span class="octicon octicon-link"></span></a>A Tiny Interpreter</h3>

<p>为了让说明更具体，让我们从一个非常小的解释器开始。它只能计算两个数的和，只能理解三个指令。它执行的所有代码只是这三个指令的不同组合。下面就是这三个指令：</p>

<ul>
<li><code>LOAD_VALUE</code></li>
<li><code>ADD_TWO_VALUES</code></li>
<li><code>PRINT_ANSWER</code></li>
</ul>

<p>我们不关心词法，语法和编译，所以我们也不在乎这些指令是如何产生的。你可以想象，你写下<code>7 + 5</code>，然后一个编译器为你生成那三个指令的组合。如果你有一个合适的编译器，你甚至可以用Lisp的语法来写，只要它能生成相同的指令。</p>

<p>假设</p>

<div class="highlight highlight-source-python"><pre><span class="pl-c1">7</span> <span class="pl-k">+</span> <span class="pl-c1">5</span></pre></div>

<p>生成这样的指令集：</p>

<div class="highlight highlight-source-python"><pre>what_to_execute <span class="pl-k">=</span> {
    <span class="pl-s"><span class="pl-pds">"</span>instructions<span class="pl-pds">"</span></span>: [(<span class="pl-s"><span class="pl-pds">"</span>LOAD_VALUE<span class="pl-pds">"</span></span>, <span class="pl-c1">0</span>),  <span class="pl-c"># the first number</span>
                     (<span class="pl-s"><span class="pl-pds">"</span>LOAD_VALUE<span class="pl-pds">"</span></span>, <span class="pl-c1">1</span>),  <span class="pl-c"># the second number</span>
                     (<span class="pl-s"><span class="pl-pds">"</span>ADD_TWO_VALUES<span class="pl-pds">"</span></span>, <span class="pl-c1">None</span>),
                     (<span class="pl-s"><span class="pl-pds">"</span>PRINT_ANSWER<span class="pl-pds">"</span></span>, <span class="pl-c1">None</span>)],
    <span class="pl-s"><span class="pl-pds">"</span>numbers<span class="pl-pds">"</span></span>: [<span class="pl-c1">7</span>, <span class="pl-c1">5</span>] }</pre></div>

<p>Python解释器是一个<em>栈机器</em>，所以它必须通过操作栈来完成这个加法。(<a href="http://qingyunha.github.io/taotao/#figure-1.1">Figure 1.1</a>)解释器先执行第一条指令，<code>LOAD_VALUE</code>，把第一个数压到栈中。接着它把第二个数也压到栈中。然后，第三条指令，<code>ADD_TWO_VALUES</code>,先把两个数从栈中弹出，加起来，再把结果压入栈中。最后一步，把结果弹出并输出。</p>

<a name="figure-1.1"></a><img src="./A Python Interpreter Written in Python by qingyunha_files/interpreter-stack.png" alt="Figure 1.1 - A stack machine" title="Figure 1.1 - A stack machine">

<p><code>LOAD_VALUE</code>这条指令告诉解释器把一个数压入栈中，但指令本身并没有指明这个数是多少。指令需要一个额外的信息告诉解释器去哪里找到这个数。所以我们的指令集有两个部分：指令本身和一个常量列表。（在Python中，字节码就是我们称为的“指令”，而解释器执行的是<em>code object</em>。）</p>

<p>为什么不把数字直接嵌入指令之中？想象一下，如果我们加的不是数字，而是字符串。我们可不想把字符串这样的东西加到指令中，因为它可以有任意的长度。另外，我们这种设计也意味着我们只需要对象的一份拷贝，比如这个加法 <code>7 + 7</code>, 现在常量表 <code>"numbers"</code>只需包含一个<code>7</code>。</p>

<p>你可能会想为什么会需要除了<code>ADD_TWO_VALUES</code>之外的指令。的确，对于我们两个数加法，这个例子是有点人为制作的意思。然而，这个指令却是建造更复杂程序的轮子。比如，就我们目前定义的三个指令，只要给出正确的指令组合，我们可以做三个数的加法，或者任意个数的加法。同时，栈提供了一个清晰的方法去跟踪解释器的状态，这为我们增长的复杂性提供了支持。</p>

<p>现在让我们来完成我们的解释器。解释器对象需要一个栈，它可以用一个列表来表示。它还需要一个方法来描述怎样执行每条指令。比如，<code>LOAD_VALUE</code>会把一个值压入栈中。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">class</span> <span class="pl-en">Interpreter</span>:
    <span class="pl-k">def</span> <span class="pl-en"><span class="pl-c1">__init__</span></span>(<span class="pl-smi">self</span>):
        <span class="pl-v">self</span>.stack <span class="pl-k">=</span> []

    <span class="pl-k">def</span> <span class="pl-en">LOAD_VALUE</span>(<span class="pl-smi">self</span>, <span class="pl-smi">number</span>):
        <span class="pl-v">self</span>.stack.append(number)
    
    <span class="pl-k">def</span> <span class="pl-en">PRINT_ANSWER</span>(<span class="pl-smi">self</span>):
        answer <span class="pl-k">=</span> <span class="pl-v">self</span>.stack.pop()
        <span class="pl-k">print</span>(answer)
    
    <span class="pl-k">def</span> <span class="pl-en">ADD_TWO_VALUES</span>(<span class="pl-smi">self</span>):
        first_num <span class="pl-k">=</span> <span class="pl-v">self</span>.stack.pop()
        second_num <span class="pl-k">=</span> <span class="pl-v">self</span>.stack.pop()
        total <span class="pl-k">=</span> first_num <span class="pl-k">+</span> second_num
        <span class="pl-v">self</span>.stack.append(total)</pre></div>

<p>这三个方法完成了解释器所理解的三条指令。但解释器还需要一样东西：一个能把所有东西结合在一起并执行的方法。这个方法就叫做<code>run_code</code>, 它把我们前面定义的字典结构<code>what-to-execute</code>作为参数，循环执行里面的每条指令，如何指令有参数，处理参数，然后调用解释器对象中相应的方法。</p>

<div class="highlight highlight-source-python"><pre>    <span class="pl-k">def</span> <span class="pl-en">run_code</span>(<span class="pl-smi">self</span>, <span class="pl-smi">what_to_execute</span>):
        instructions <span class="pl-k">=</span> what_to_execute[<span class="pl-s"><span class="pl-pds">"</span>instructions<span class="pl-pds">"</span></span>]
        numbers <span class="pl-k">=</span> what_to_execute[<span class="pl-s"><span class="pl-pds">"</span>numbers<span class="pl-pds">"</span></span>]
        <span class="pl-k">for</span> each_step <span class="pl-k">in</span> instructions:
            instruction, argument <span class="pl-k">=</span> each_step
            <span class="pl-k">if</span> instruction <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">"</span>LOAD_VALUE<span class="pl-pds">"</span></span>:
                number <span class="pl-k">=</span> numbers[argument]
                <span class="pl-v">self</span>.LOAD_VALUE(number)
            <span class="pl-k">elif</span> instruction <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">"</span>ADD_TWO_VALUES<span class="pl-pds">"</span></span>:
                <span class="pl-v">self</span>.ADD_TWO_VALUES()
            <span class="pl-k">elif</span> instruction <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">"</span>PRINT_ANSWER<span class="pl-pds">"</span></span>:
                <span class="pl-v">self</span>.PRINT_ANSWER()</pre></div>

<p>为了测试，我们创建一个解释器对象，然后用前面定义的 7 + 5 的指令集来调用<code>run_code</code>。</p>

<div class="highlight highlight-source-python"><pre>    interpreter <span class="pl-k">=</span> Interpreter()
    interpreter.run_code(what_to_execute)</pre></div>

<p>显然，它会输出12</p>

<p>尽管我们的解释器功能受限，但这个加法过程几乎和真正的Python解释器是一样的。这里，我们还有几点要注意。</p>

<p>首先，一些指令需要参数。在真正的Python bytecode中，大概有一半的指令有参数。像我们的例子一样，参数和指令打包在一起。注意<em>指令</em>的参数和传递给对应方法的参数是不同的。</p>

<p>第二，指令<code>ADD_TWO_VALUES</code>不需要任何参数，它从解释器栈中弹出所需的值。这正是以栈为基础的解释器的特点。</p>

<p>记得我们说过只要给出合适的指令集，不需要对解释器做任何改变，我们做多个数的加法。考虑下面的指令集，你觉得会发生什么？如果你有一个合适的编译器，什么代码才能编译出下面的指令集？</p>

<div class="highlight highlight-source-python"><pre>    what_to_execute <span class="pl-k">=</span> {
        <span class="pl-s"><span class="pl-pds">"</span>instructions<span class="pl-pds">"</span></span>: [(<span class="pl-s"><span class="pl-pds">"</span>LOAD_VALUE<span class="pl-pds">"</span></span>, <span class="pl-c1">0</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>LOAD_VALUE<span class="pl-pds">"</span></span>, <span class="pl-c1">1</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>ADD_TWO_VALUES<span class="pl-pds">"</span></span>, <span class="pl-c1">None</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>LOAD_VALUE<span class="pl-pds">"</span></span>, <span class="pl-c1">2</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>ADD_TWO_VALUES<span class="pl-pds">"</span></span>, <span class="pl-c1">None</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>PRINT_ANSWER<span class="pl-pds">"</span></span>, <span class="pl-c1">None</span>)],
        <span class="pl-s"><span class="pl-pds">"</span>numbers<span class="pl-pds">"</span></span>: [<span class="pl-c1">7</span>, <span class="pl-c1">5</span>, <span class="pl-c1">8</span>] }</pre></div>

<p>从这点出发，我们开始看到这种结构的可扩展性：我们可以通过向解释器对象增加方法来描述更多的操作（只要有一个编译器能为我们生成组织良好的指令集）。</p>

<h4>
<a id="variables" class="anchor" href="http://qingyunha.github.io/taotao/#variables" aria-hidden="true"><span class="octicon octicon-link"></span></a>Variables</h4>

<p>接下来给我们的解释器增加变量的支持。我们需要一个保存变量值的指令，<code>STORE_NAME</code>;一个取变量值的指令<code>LOAD_NAME</code>;和一个变量到值的映射关系。目前，我们会忽略命名空间和作用域，所以我们可以把变量和值的映射直接存储在解释器对象中。最后，我们要保证<code>what_to_execute</code>除了一个常量列表，还要有个变量名字的列表。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">&gt;&gt;&gt;</span> <span class="pl-k">def</span> s():
...     a <span class="pl-k">=</span> <span class="pl-c1">1</span>
...     b <span class="pl-k">=</span> <span class="pl-c1">2</span>
...     <span class="pl-k">print</span>(a <span class="pl-k">+</span> b)
<span class="pl-c"># a friendly compiler transforms `s` into:</span>
    what_to_execute <span class="pl-k">=</span> {
        <span class="pl-s"><span class="pl-pds">"</span>instructions<span class="pl-pds">"</span></span>: [(<span class="pl-s"><span class="pl-pds">"</span>LOAD_VALUE<span class="pl-pds">"</span></span>, <span class="pl-c1">0</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>STORE_NAME<span class="pl-pds">"</span></span>, <span class="pl-c1">0</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>LOAD_VALUE<span class="pl-pds">"</span></span>, <span class="pl-c1">1</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>STORE_NAME<span class="pl-pds">"</span></span>, <span class="pl-c1">1</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>LOAD_NAME<span class="pl-pds">"</span></span>, <span class="pl-c1">0</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>LOAD_NAME<span class="pl-pds">"</span></span>, <span class="pl-c1">1</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>ADD_TWO_VALUES<span class="pl-pds">"</span></span>, <span class="pl-c1">None</span>),
                         (<span class="pl-s"><span class="pl-pds">"</span>PRINT_ANSWER<span class="pl-pds">"</span></span>, <span class="pl-c1">None</span>)],
        <span class="pl-s"><span class="pl-pds">"</span>numbers<span class="pl-pds">"</span></span>: [<span class="pl-c1">1</span>, <span class="pl-c1">2</span>],
        <span class="pl-s"><span class="pl-pds">"</span>names<span class="pl-pds">"</span></span>:   [<span class="pl-s"><span class="pl-pds">"</span>a<span class="pl-pds">"</span></span>, <span class="pl-s"><span class="pl-pds">"</span>b<span class="pl-pds">"</span></span>] }</pre></div>

<p>我们的新的的实现在下面。为了跟踪哪名字绑定到那个值，我们在<code>__init__</code>方法中增加一个<code>environment</code>字典。我们也增加了<code>STORE_NAME</code>和<code>LOAD_NAME</code>方法，它们获得变量名，然后从<code>environment</code>字典中设置或取出这个变量值。</p>

<p>现在指令参数就有两个不同的意思，它可能是<code>numbers</code>列表的索引，也可能是<code>names</code>列表的索引。解释器通过检查所执行的指令就能知道是那种参数。而我们打破这种逻辑 ，把指令和它所用何种参数的映射关系放在另一个单独的方法中。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">class</span> <span class="pl-en">Interpreter</span>:
    <span class="pl-k">def</span> <span class="pl-en"><span class="pl-c1">__init__</span></span>(<span class="pl-smi">self</span>):
        <span class="pl-v">self</span>.stack <span class="pl-k">=</span> []
        <span class="pl-v">self</span>.environment <span class="pl-k">=</span> {}

    <span class="pl-k">def</span> <span class="pl-en">STORE_NAME</span>(<span class="pl-smi">self</span>, <span class="pl-smi">name</span>):
        val <span class="pl-k">=</span> <span class="pl-v">self</span>.stack.pop()
        <span class="pl-v">self</span>.environment[name] <span class="pl-k">=</span> val
    
    <span class="pl-k">def</span> <span class="pl-en">LOAD_NAME</span>(<span class="pl-smi">self</span>, <span class="pl-smi">name</span>):
        val <span class="pl-k">=</span> <span class="pl-v">self</span>.environment[name]
        <span class="pl-v">self</span>.stack.append(val)
    
    <span class="pl-k">def</span> <span class="pl-en">parse_argument</span>(<span class="pl-smi">self</span>, <span class="pl-smi">instruction</span>, <span class="pl-smi">argument</span>, <span class="pl-smi">what_to_execute</span>):
        <span class="pl-s"><span class="pl-pds">"""</span> Understand what the argument to each instruction means.<span class="pl-pds">"""</span></span>
        numbers <span class="pl-k">=</span> [<span class="pl-s"><span class="pl-pds">"</span>LOAD_VALUE<span class="pl-pds">"</span></span>]
        names <span class="pl-k">=</span> [<span class="pl-s"><span class="pl-pds">"</span>LOAD_NAME<span class="pl-pds">"</span></span>, <span class="pl-s"><span class="pl-pds">"</span>STORE_NAME<span class="pl-pds">"</span></span>]
    
        <span class="pl-k">if</span> instruction <span class="pl-k">in</span> numbers:
            argument <span class="pl-k">=</span> what_to_execute[<span class="pl-s"><span class="pl-pds">"</span>numbers<span class="pl-pds">"</span></span>][argument]
        <span class="pl-k">elif</span> instruction <span class="pl-k">in</span> names:
            argument <span class="pl-k">=</span> what_to_execute[<span class="pl-s"><span class="pl-pds">"</span>names<span class="pl-pds">"</span></span>][argument]
    
        <span class="pl-k">return</span> argument
    
    <span class="pl-k">def</span> <span class="pl-en">run_code</span>(<span class="pl-smi">self</span>, <span class="pl-smi">what_to_execute</span>):
        instructions <span class="pl-k">=</span> what_to_execute[<span class="pl-s"><span class="pl-pds">"</span>instructions<span class="pl-pds">"</span></span>]
        <span class="pl-k">for</span> each_step <span class="pl-k">in</span> instructions:
            instruction, argument <span class="pl-k">=</span> each_step
            argument <span class="pl-k">=</span> <span class="pl-v">self</span>.parse_argument(instruction, argument, what_to_execute)
    
            <span class="pl-k">if</span> instruction <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">"</span>LOAD_VALUE<span class="pl-pds">"</span></span>:
                <span class="pl-v">self</span>.LOAD_VALUE(argument)
            <span class="pl-k">elif</span> instruction <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">"</span>ADD_TWO_VALUES<span class="pl-pds">"</span></span>:
                <span class="pl-v">self</span>.ADD_TWO_VALUES()
            <span class="pl-k">elif</span> instruction <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">"</span>PRINT_ANSWER<span class="pl-pds">"</span></span>:
                <span class="pl-v">self</span>.PRINT_ANSWER()
            <span class="pl-k">elif</span> instruction <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">"</span>STORE_NAME<span class="pl-pds">"</span></span>:
                <span class="pl-v">self</span>.STORE_NAME(argument)
            <span class="pl-k">elif</span> instruction <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">"</span>LOAD_NAME<span class="pl-pds">"</span></span>:
                <span class="pl-v">self</span>.LOAD_NAME(argument)</pre></div>

<p>仅仅五个指令，<code>run_code</code>这个方法已经开始变得冗长了。如果保持这种结构，那么每条指令都需要一个<code>if</code>分支。这里，我们要利用Python的动态方法查找。我们总会给一个称为<code>FOO</code>的指令定义一个名为<code>FOO</code>的方法，这样我们就可用Python的<code>getattr</code>函数在运行时动态查找方法，而不用这个大大的分支结构。<code>run_code</code>方法现在是这样：</p>

<div class="highlight highlight-source-python"><pre>    <span class="pl-k">def</span> <span class="pl-en">execute</span>(<span class="pl-smi">self</span>, <span class="pl-smi">what_to_execute</span>):
        instructions <span class="pl-k">=</span> what_to_execute[<span class="pl-s"><span class="pl-pds">"</span>instructions<span class="pl-pds">"</span></span>]
        <span class="pl-k">for</span> each_step <span class="pl-k">in</span> instructions:
            instruction, argument <span class="pl-k">=</span> each_step
            argument <span class="pl-k">=</span> <span class="pl-v">self</span>.parse_argument(instruction, argument, what_to_execute)
            bytecode_method <span class="pl-k">=</span> <span class="pl-c1">getattr</span>(<span class="pl-v">self</span>, instruction)
            <span class="pl-k">if</span> argument <span class="pl-k">is</span> <span class="pl-c1">None</span>:
                bytecode_method()
            <span class="pl-k">else</span>:
                bytecode_method(argument)</pre></div>

<h2>
<a id="real-python-bytecode" class="anchor" href="http://qingyunha.github.io/taotao/#real-python-bytecode" aria-hidden="true"><span class="octicon octicon-link"></span></a>Real Python Bytecode</h2>

<p>现在，放弃我们的小指令集，去看看真正的Python字节码。字节码的结构和我们的小解释器的指令集差不多，除了字节码用一个字节而不是一个名字来指示这条指令。为了理解它的结构，我们将考察一个函数的字节码。考虑下面这个例子：</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">&gt;&gt;&gt;</span> <span class="pl-k">def</span> cond():
...     x <span class="pl-k">=</span> <span class="pl-c1">3</span>
...     <span class="pl-k">if</span> x <span class="pl-k">&lt;</span> <span class="pl-c1">5</span>:
...         <span class="pl-k">return</span> <span class="pl-s"><span class="pl-pds">'</span>yes<span class="pl-pds">'</span></span>
...     <span class="pl-k">else</span>:
...         <span class="pl-k">return</span> <span class="pl-s"><span class="pl-pds">'</span>no<span class="pl-pds">'</span></span>
...</pre></div>

<p>Python在运行时会暴露一大批内部信息，并且我们可以通过REPL直接访问这些信息。对于函数对象<code>cond</code>，<code>cond.__code__</code>是与其关联的code object，而<code>cond.__code__.co_code</code>就是它的字节码。当你写Python代码时，你永远也不会想直接使用这些属性，但是这可以让我们做出各种恶作剧，同时也可以看看内部机制。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">&gt;&gt;&gt;</span> cond.<span class="pl-c1">__code__</span>.co_code  <span class="pl-c"># the bytecode as raw bytes</span>
b<span class="pl-s"><span class="pl-pds">'</span>d<span class="pl-cce">\x01\x00</span>}<span class="pl-cce">\x00\x00</span>|<span class="pl-cce">\x00\x00</span>d<span class="pl-cce">\x02\x00</span>k<span class="pl-cce">\x00\x00</span>r<span class="pl-cce">\x16\x00</span>d<span class="pl-cce">\x03\x00</span>Sd<span class="pl-cce">\x04\x00</span>Sd<span class="pl-cce">\x00</span><span class="pl-ii"></span></span>
   \<span class="pl-ii">x00S'</span>
<span class="pl-k">&gt;&gt;&gt;</span> <span class="pl-c1">list</span>(cond.<span class="pl-c1">__code__</span>.co_code)  <span class="pl-c"># the bytecode as numbers</span>
[<span class="pl-c1">100</span>, <span class="pl-c1">1</span>, <span class="pl-c1">0</span>, <span class="pl-c1">125</span>, <span class="pl-c1">0</span>, <span class="pl-c1">0</span>, <span class="pl-c1">124</span>, <span class="pl-c1">0</span>, <span class="pl-c1">0</span>, <span class="pl-c1">100</span>, <span class="pl-c1">2</span>, <span class="pl-c1">0</span>, <span class="pl-c1">107</span>, <span class="pl-c1">0</span>, <span class="pl-c1">0</span>, <span class="pl-c1">114</span>, <span class="pl-c1">22</span>, <span class="pl-c1">0</span>, <span class="pl-c1">100</span>, <span class="pl-c1">3</span>, <span class="pl-c1">0</span>, <span class="pl-c1">83</span>, 
 <span class="pl-c1">100</span>, <span class="pl-c1">4</span>, <span class="pl-c1">0</span>, <span class="pl-c1">83</span>, <span class="pl-c1">100</span>, <span class="pl-c1">0</span>, <span class="pl-c1">0</span>, <span class="pl-c1">83</span>]</pre></div>

<p>当我们直接输出这个字节码，它看起来完全无法理解 --- 唯一我们了解的是它是一串字节。很幸运，我们有一个很强大的工具可以用：Python标准库中的<code>dis</code> module。</p>

<p><code>dis</code>是一个字节码反汇编器。反汇编器以为机器而写的底层代码作为输入，比如汇编代码和字节码，然后以人类可读的方式输出。当我们运行<code>dis.dis</code>, 它输出每个字节码的解释。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">&gt;&gt;&gt;</span> dis.dis(cond)
  <span class="pl-c1">2</span>           <span class="pl-c1">0</span> LOAD_CONST               <span class="pl-c1">1</span> (<span class="pl-c1">3</span>)
              <span class="pl-c1">3</span> STORE_FAST               <span class="pl-c1">0</span> (x)

  <span class="pl-c1">3</span>           <span class="pl-c1">6</span> LOAD_FAST                <span class="pl-c1">0</span> (x)
              <span class="pl-c1">9</span> LOAD_CONST               <span class="pl-c1">2</span> (<span class="pl-c1">5</span>)
             <span class="pl-c1">12</span> COMPARE_OP               <span class="pl-c1">0</span> (<span class="pl-k">&lt;</span>)
             <span class="pl-c1">15</span> POP_JUMP_IF_FALSE       <span class="pl-c1">22</span>

  <span class="pl-c1">4</span>          <span class="pl-c1">18</span> LOAD_CONST               <span class="pl-c1">3</span> (<span class="pl-s"><span class="pl-pds">'</span>yes<span class="pl-pds">'</span></span>)
             <span class="pl-c1">21</span> RETURN_VALUE

  <span class="pl-c1">6</span>     <span class="pl-k">&gt;&gt;</span>   <span class="pl-c1">22</span> LOAD_CONST               <span class="pl-c1">4</span> (<span class="pl-s"><span class="pl-pds">'</span>no<span class="pl-pds">'</span></span>)
             <span class="pl-c1">25</span> RETURN_VALUE
             <span class="pl-c1">26</span> LOAD_CONST               <span class="pl-c1">0</span> (<span class="pl-c1">None</span>)
             <span class="pl-c1">29</span> RETURN_VALUE</pre></div>

<p>这些都是什么意思？让我们以第一条指令<code>LOAD_CONST</code>为例子。第一列的数字（<code>2</code>）表示对应源代码的行数。第二列的数字是字节码的索引，告诉我们指令<code>LOAD_CONST</code>在0位置。第三列是指令本身对应的人类可读的名字。如果第四列存在，它表示指令的参数。如果第5列存在，它是一个关于参数是什么的提示。</p>

<p>考虑这个字节码的前几个字节：[100, 1, 0, 125, 0, 0]。这6个字节表示两条带参数的指令。我们可以使用<code>dis.opname</code>，一个字节到可读字符串的映射，来找到指令100和指令125代表是什么：</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">&gt;&gt;&gt;</span> dis.opname[<span class="pl-c1">100</span>]
<span class="pl-s"><span class="pl-pds">'</span>LOAD_CONST<span class="pl-pds">'</span></span>
<span class="pl-k">&gt;&gt;&gt;</span> dis.opname[<span class="pl-c1">125</span>]
<span class="pl-s"><span class="pl-pds">'</span>STORE_FAST<span class="pl-pds">'</span></span></pre></div>

<p>第二和第三个字节 --- 1 ，0 ---是<code>LOAD_CONST</code>的参数，第五和第六个字节 --- 0，0 --- 是<code>STORE_FAST</code>的参数。就像我们前面的小例子，<code>LOAD_CONST</code>需要知道的到哪去找常量，<code>STORE_FAST</code>需要找到名字。（Python的<code>LOAD_CONST</code>和我们小例子中的<code>LOAD_VALUE</code>一样，<code>LOAD_FAST</code>和<code>LOAD_NAME</code>一样）。所以这六个字节代表第一行源代码<code>x = 3</code>.(为什么用两个字节表示指令的参数？如果Python使用一个字节，每个code object你只能有256个常量/名字，而用两个字节，就增加到了256的平方，65536个）。</p>

<h3>
<a id="conditionals-and-loops" class="anchor" href="http://qingyunha.github.io/taotao/#conditionals-and-loops" aria-hidden="true"><span class="octicon octicon-link"></span></a>Conditionals and Loops</h3>

<p>到目前为止，我们的解释器只能一条接着一条的执行指令。这有个问题，我们经常会想多次执行某个指令，或者在特定的条件下略过它们。为了可以写循环和分支结构，解释器必须能够在指令中跳转。在某种程度上，Python在字节码中使用<code>GOTO</code>语句来处理循环和分支！让我们再看一个<code>cond</code>函数的反汇编结果：</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">&gt;&gt;&gt;</span> dis.dis(cond)
  <span class="pl-c1">2</span>           <span class="pl-c1">0</span> LOAD_CONST               <span class="pl-c1">1</span> (<span class="pl-c1">3</span>)
              <span class="pl-c1">3</span> STORE_FAST               <span class="pl-c1">0</span> (x)

  <span class="pl-c1">3</span>           <span class="pl-c1">6</span> LOAD_FAST                <span class="pl-c1">0</span> (x)
              <span class="pl-c1">9</span> LOAD_CONST               <span class="pl-c1">2</span> (<span class="pl-c1">5</span>)
             <span class="pl-c1">12</span> COMPARE_OP               <span class="pl-c1">0</span> (<span class="pl-k">&lt;</span>)
             <span class="pl-c1">15</span> POP_JUMP_IF_FALSE       <span class="pl-c1">22</span>

  <span class="pl-c1">4</span>          <span class="pl-c1">18</span> LOAD_CONST               <span class="pl-c1">3</span> (<span class="pl-s"><span class="pl-pds">'</span>yes<span class="pl-pds">'</span></span>)
             <span class="pl-c1">21</span> RETURN_VALUE

  <span class="pl-c1">6</span>     <span class="pl-k">&gt;&gt;</span>   <span class="pl-c1">22</span> LOAD_CONST               <span class="pl-c1">4</span> (<span class="pl-s"><span class="pl-pds">'</span>no<span class="pl-pds">'</span></span>)
             <span class="pl-c1">25</span> RETURN_VALUE
             <span class="pl-c1">26</span> LOAD_CONST               <span class="pl-c1">0</span> (<span class="pl-c1">None</span>)
             <span class="pl-c1">29</span> RETURN_VALUE</pre></div>

<p>第三行的条件表达式<code>if x &lt; 5</code>被编译成四条指令：<code>LOAD_FAST</code>, <code>LOAD_CONST</code>, <code>COMPARE_OP</code>和 <code>POP_JUMP_IF_FALSE</code>。<code>x &lt; 5</code>对应加载<code>x</code>，加载5，比较这两个值。指令<code>POP_JUMP_IF_FALSE</code>完成<code>if</code>语句。这条指令把栈顶的值弹出，如果值为真，什么都不发生。如果值为假，解释器会跳转到另一条指令。</p>

<p>这条将被加载的指令称为跳转目标，它作为指令<code>POP_JUMP</code>的参数。这里，跳转目标是22，索引为22的指令是<code>LOAD_CONST</code>,对应源码的第6行。（<code>dis</code>用<code>&gt;&gt;</code>标记跳转目标。）如果<code>X &lt; 5</code>为假，解释器会忽略第四行（<code>return yes</code>）,直接跳转到第6行（<code>return "no"</code>）。因此解释器通过跳转指令选择性的执行指令。</p>

<p>Python的循环也依赖于跳转。在下面的字节码中，<code>while x &lt; 5</code>这一行产生了和<code>if x &lt; 10</code>几乎一样的字节码。在这两种情况下，解释器都是先执行比较，然后执行<code>POP_JUMP_IF_FALSE</code>来控制下一条执行哪个指令。第四行的最后一条字节码<code>JUMP_ABSOLUT</code>(循环体结束的地方），让解释器返回到循环开始的第9条指令处。当 <code>x &lt; 10</code>变为假，<code>POP_JUMP_IF_FALSE</code>会让解释器跳到循环的终止处，第34条指令。 </p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">&gt;&gt;&gt;</span> <span class="pl-k">def</span> loop():
...      x <span class="pl-k">=</span> <span class="pl-c1">1</span>
...      <span class="pl-k">while</span> x <span class="pl-k">&lt;</span> <span class="pl-c1">5</span>:
...          x <span class="pl-k">=</span> x <span class="pl-k">+</span> <span class="pl-c1">1</span>
...      <span class="pl-k">return</span> x
...
<span class="pl-k">&gt;&gt;&gt;</span> dis.dis(loop)
  <span class="pl-c1">2</span>           <span class="pl-c1">0</span> LOAD_CONST               <span class="pl-c1">1</span> (<span class="pl-c1">1</span>)
              <span class="pl-c1">3</span> STORE_FAST               <span class="pl-c1">0</span> (x)

  <span class="pl-c1">3</span>           <span class="pl-c1">6</span> SETUP_LOOP              <span class="pl-c1">26</span> (to <span class="pl-c1">35</span>)
        <span class="pl-k">&gt;&gt;</span>    <span class="pl-c1">9</span> LOAD_FAST                <span class="pl-c1">0</span> (x)
             <span class="pl-c1">12</span> LOAD_CONST               <span class="pl-c1">2</span> (<span class="pl-c1">5</span>)
             <span class="pl-c1">15</span> COMPARE_OP               <span class="pl-c1">0</span> (<span class="pl-k">&lt;</span>)
             <span class="pl-c1">18</span> POP_JUMP_IF_FALSE       <span class="pl-c1">34</span>

  <span class="pl-c1">4</span>          <span class="pl-c1">21</span> LOAD_FAST                <span class="pl-c1">0</span> (x)
             <span class="pl-c1">24</span> LOAD_CONST               <span class="pl-c1">1</span> (<span class="pl-c1">1</span>)
             <span class="pl-c1">27</span> BINARY_ADD
             <span class="pl-c1">28</span> STORE_FAST               <span class="pl-c1">0</span> (x)
             <span class="pl-c1">31</span> JUMP_ABSOLUTE            <span class="pl-c1">9</span>
        <span class="pl-k">&gt;&gt;</span>   <span class="pl-c1">34</span> POP_BLOCK

  <span class="pl-c1">5</span>     <span class="pl-k">&gt;&gt;</span>   <span class="pl-c1">35</span> LOAD_FAST                <span class="pl-c1">0</span> (x)
             <span class="pl-c1">38</span> RETURN_VALUE</pre></div>

<h3>
<a id="explore-bytecode" class="anchor" href="http://qingyunha.github.io/taotao/#explore-bytecode" aria-hidden="true"><span class="octicon octicon-link"></span></a>Explore Bytecode</h3>

<p>我鼓励你用<code>dis.dis</code>来试试你自己写的函数。一些有趣的问题值得探索：</p>

<ul>
<li>对解释器而言for循环和while循环有什么不同？</li>
<li>能不能写出两个不同函数，却能产生相同的字节码?</li>
<li>
<code>elif</code>是怎么工作的？列表推导呢？</li>
</ul>

<h2>
<a id="frames" class="anchor" href="http://qingyunha.github.io/taotao/#frames" aria-hidden="true"><span class="octicon octicon-link"></span></a>Frames</h2>

<p>到目前为止，我们已经知道了Python虚拟机是一个栈机器。它能顺序执行指令，在指令间跳转，压入或弹出栈值。但是这和我们心想的解释器还有一定距离。在前面的那个例子中，最后一条指令是<code>RETURN_VALUE</code>,它和<code>return</code>语句想对应。但是它返回到哪里去呢？</p>

<p>为了回答这个问题，我们必须严增加一层复杂性：frame。一个frame是一些信息的集合和代码的执行上下文。frames在Python代码执行时动态的创建和销毁。每个frame对应函数的一次调用。--- 所以每个frame只有一个code object与之关联，而一个code object可以很多frame。比如你有一个函数递归的调用自己10次，这时有11个frame。总的来说，Python程序的每个作用域有一个frame，比如，每个module，每个函数调用，每个类定义。</p>

<p>Frame存在于<em>调用栈</em>中，一个和我们之前讨论的完全不同的栈。（你最熟悉的栈就是调用栈，就是你经常看到的异常回溯，每个以"File 'program.py'"开始的回溯对应一个frame。）解释器在执行字节码时操作的栈，我们叫它<em>数据栈</em>。其实还有第三个栈，叫做<em>块栈</em>，用于特定的控制流块，比如循环和异常处理。调用栈中的每个frame都有它自己的数据栈和块栈。</p>

<p>让我们用一个具体的例子来说明。假设Python解释器执行到标记为3的地方。解释器正在<code>foo</code>函数的调用中，它接着调用<code>bar</code>。下面是frame调用栈，块栈和数据栈的示意图。我们感兴趣的是解释器先从最底下的<code>foo()</code>开始，接着执行<code>foo</code>的函数体，然后到达<code>bar</code>。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">&gt;&gt;&gt;</span> <span class="pl-k">def</span> bar(y):
...     z <span class="pl-k">=</span> y <span class="pl-k">+</span> <span class="pl-c1">3</span>     <span class="pl-c"># &lt;--- (3) ... and the interpreter is here.</span>
...     <span class="pl-k">return</span> z
...
<span class="pl-k">&gt;&gt;&gt;</span> <span class="pl-k">def</span> foo():
...     a <span class="pl-k">=</span> <span class="pl-c1">1</span>
...     b <span class="pl-k">=</span> <span class="pl-c1">2</span>
...     <span class="pl-k">return</span> a <span class="pl-k">+</span> bar(b) <span class="pl-c"># &lt;--- (2) ... which is returning a call to bar ...</span>
...
<span class="pl-k">&gt;&gt;&gt;</span> foo()             <span class="pl-c"># &lt;--- (1) We're in the middle of a call to foo ...</span>
<span class="pl-c1">3</span></pre></div>

<a name="figure-1.2"></a><img src="./A Python Interpreter Written in Python by qingyunha_files/interpreter-callstack.png" alt="Figure 1.2 - The call stack" title="Figure 1.2 - The call stack">

<p>现在，解释器在<code>bar</code>函数的调用中。调用栈中有3个fram：一个对应于module层，一个对应函数<code>foo</code>,别一个对应函数<code>bar</code>。(<a href="http://qingyunha.github.io/taotao/#figure-1.2">Figure 1.2</a>)一旦<code>bar</code>返回，与它对应的frame就会从调用栈中弹出并丢弃。</p>

<p>字节码指令<code>RETURN_VALUE</code>告诉解释器在frame间传递一个值。首先，它把位于调用栈栈顶的frame中的数据栈的栈顶值弹出。然后把整个frame弹出丢弃。最后把这个值压到下一个frame的数据栈中。</p>

<p>当Ned Batchelder和我在写Byterun时，很长一段时间我们的实现中一直有个重大的错误。我们整个虚拟机中只有一个数据栈，而不是每个frame都有个一个。我们做了很多测试，同时在Byterun和真正的Python上，为了保证结果一致。我们几乎通过了所有测试，只有一样东西不能通过，那就是生成器。最后，通过仔细的阅读Cpython的源码，我们发现了错误所在。把数据栈移到每个frame就解决了这个问题。</p>


<p>回头在看看这个bug，我惊讶的发现Python真的很少依赖于每个frame有一个数据栈这个特性。在Python中几乎所有的操作都会清空数据栈，所以所有的frame公用一个数据栈是没问题的。在上面的例子中，当<code>bar</code>执行完后，它的数据栈为空。即使<code>foo</code>公用这一个栈，它的值也不会受影响。然而，对应生成器，一个关键的特点是它能暂停一个frame的执行，返回到其他的frame，一段时间后它能返回到原来的frame，并以它离开时的同样的状态继续执行。</p>

<h2>
<a id="byterun" class="anchor" href="http://qingyunha.github.io/taotao/#byterun" aria-hidden="true"><span class="octicon octicon-link"></span></a>Byterun</h2>

<p>现在我们有足够的Python解释器的知识背景去考察Byterun。</p>

<p>Byterun中有四种对象。</p>

<ul>
<li>
<code>VirtualMachine</code>类，它管理高层结构，frame调用栈，指令到操作的映射。这是一个比前面<code>Inteprter</code>对象更复杂的版本。</li>
<li>
<code>Frame</code>类，每个<code>Frame</code>类都有一个code object，并且管理者其他一些必要的状态信息，全局和局部命名空间，指向调用它的frame的指针和最后执行的字节码指令。 </li>
<li>
<code>Function</code>类，它被用来代替真正的Python函数。回想一下，调用函数时会创建一个新的frame。我们自己实现<code>Function</code>，所以我们控制新frame的创建。</li>
<li>
<code>Block</code>类，它只是包装了代码块的3个属性。（代码块的细节不是解释器的核心，我们不会花时间在它身上，把它列在这里，是因为Byterun需要它。）</li>
</ul>

<h3>
<a id="the-virtualmachine-class" class="anchor" href="http://qingyunha.github.io/taotao/#the-virtualmachine-class" aria-hidden="true"><span class="octicon octicon-link"></span></a>The <code>VirtualMachine</code> Class</h3>

<p>程序运行时只有一个<code>VirtualMachine</code>被创建，因为我们只有一个解释器。<code>VirtualMachine</code>保存调用栈，异常状态，在frame中传递的返回值。它的入口点是<code>run_code</code>方法，它以编译后的code object为参数，以创建一个frame为开始，然后运行这个frame。这个frame可能再创建出新的frame；调用栈随着程序的运行增长缩短。当第一个frame返回时，执行结束。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">class</span> <span class="pl-en">VirtualMachineError</span>(<span class="pl-e"><span class="pl-c1">Exception</span></span>):
    <span class="pl-k">pass</span>

<span class="pl-k">class</span> <span class="pl-en">VirtualMachine</span>(<span class="pl-e"><span class="pl-c1">object</span></span>):
    <span class="pl-k">def</span> <span class="pl-en"><span class="pl-c1">__init__</span></span>(<span class="pl-smi">self</span>):
        <span class="pl-v">self</span>.frames <span class="pl-k">=</span> []   <span class="pl-c"># The call stack of frames.</span>
        <span class="pl-v">self</span>.frame <span class="pl-k">=</span> <span class="pl-c1">None</span>  <span class="pl-c"># The current frame.</span>
        <span class="pl-v">self</span>.return_value <span class="pl-k">=</span> <span class="pl-c1">None</span>
        <span class="pl-v">self</span>.last_exception <span class="pl-k">=</span> <span class="pl-c1">None</span>

    <span class="pl-k">def</span> <span class="pl-en">run_code</span>(<span class="pl-smi">self</span>, <span class="pl-smi">code</span>, <span class="pl-smi">global_names</span><span class="pl-k">=</span><span class="pl-c1">None</span>, <span class="pl-smi">local_names</span><span class="pl-k">=</span><span class="pl-c1">None</span>):
        <span class="pl-s"><span class="pl-pds">"""</span> An entry point to execute code using the virtual machine.<span class="pl-pds">"""</span></span>
        frame <span class="pl-k">=</span> <span class="pl-v">self</span>.make_frame(code, <span class="pl-smi">global_names</span><span class="pl-k">=</span>global_names, 
                                <span class="pl-smi">local_names</span><span class="pl-k">=</span>local_names)
        <span class="pl-v">self</span>.run_frame(frame)
</pre></div>

<h3>
<a id="the-frame-class" class="anchor" href="http://qingyunha.github.io/taotao/#the-frame-class" aria-hidden="true"><span class="octicon octicon-link"></span></a>The <code>Frame</code> Class</h3>

<p>接下来，我们来写<code>Frame</code>对象。frame是一个属性的集合，它没有任何方法。前面提到过，这些属性包括由编译器生成的code object；局部，全局和内置命名空间；前一个frame的引用；一个数据栈；一个块栈；最后执行的指令。（对于内置命名空间我们需要多做一点工作，Python对待这个命名空间不同；但这个细节对我们的虚拟机不重要。）</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">class</span> <span class="pl-en">Frame</span>(<span class="pl-e"><span class="pl-c1">object</span></span>):
    <span class="pl-k">def</span> <span class="pl-en"><span class="pl-c1">__init__</span></span>(<span class="pl-smi">self</span>, <span class="pl-smi">code_obj</span>, <span class="pl-smi">global_names</span>, <span class="pl-smi">local_names</span>, <span class="pl-smi">prev_frame</span>):
        <span class="pl-v">self</span>.code_obj <span class="pl-k">=</span> code_obj
        <span class="pl-v">self</span>.global_names <span class="pl-k">=</span> global_names
        <span class="pl-v">self</span>.local_names <span class="pl-k">=</span> local_names
        <span class="pl-v">self</span>.prev_frame <span class="pl-k">=</span> prev_frame
        <span class="pl-v">self</span>.stack <span class="pl-k">=</span> []
        <span class="pl-k">if</span> prev_frame:
            <span class="pl-v">self</span>.builtin_names <span class="pl-k">=</span> prev_frame.builtin_names
        <span class="pl-k">else</span>:
            <span class="pl-v">self</span>.builtin_names <span class="pl-k">=</span> local_names[<span class="pl-s"><span class="pl-pds">'</span>__builtins__<span class="pl-pds">'</span></span>]
            <span class="pl-k">if</span> <span class="pl-c1">hasattr</span>(<span class="pl-v">self</span>.builtin_names, <span class="pl-s"><span class="pl-pds">'</span>__dict__<span class="pl-pds">'</span></span>):
                <span class="pl-v">self</span>.builtin_names <span class="pl-k">=</span> <span class="pl-v">self</span>.builtin_names.<span class="pl-c1">__dict__</span>

        <span class="pl-v">self</span>.last_instruction <span class="pl-k">=</span> <span class="pl-c1">0</span>
        <span class="pl-v">self</span>.block_stack <span class="pl-k">=</span> []</pre></div>

<p>接着，我们在虚拟机中增加对frame的操作。这有3个帮助函数：一个创建新的frame的方法，和压栈和出栈的方法。第四个函数，<code>run_frame</code>,完成执行frame的主要工作，待会我们再讨论这个方法。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">class</span> <span class="pl-en">VirtualMachine</span>(<span class="pl-e"><span class="pl-c1">object</span></span>):
    [... snip ...]

    <span class="pl-c"># Frame manipulation</span>
    <span class="pl-k">def</span> <span class="pl-en">make_frame</span>(<span class="pl-smi">self</span>, <span class="pl-smi">code</span>, <span class="pl-smi">callargs</span><span class="pl-k">=</span>{}, <span class="pl-smi">global_names</span><span class="pl-k">=</span><span class="pl-c1">None</span>, <span class="pl-smi">local_names</span><span class="pl-k">=</span><span class="pl-c1">None</span>):
        <span class="pl-k">if</span> global_names <span class="pl-k">is</span> <span class="pl-k">not</span> <span class="pl-c1">None</span> <span class="pl-k">and</span> local_names <span class="pl-k">is</span> <span class="pl-k">not</span> <span class="pl-c1">None</span>:
            local_names <span class="pl-k">=</span> global_names
        <span class="pl-k">elif</span> <span class="pl-v">self</span>.frames:
            global_names <span class="pl-k">=</span> <span class="pl-v">self</span>.frame.global_names
            local_names <span class="pl-k">=</span> {}
        <span class="pl-k">else</span>:
            global_names <span class="pl-k">=</span> local_names <span class="pl-k">=</span> {
                <span class="pl-s"><span class="pl-pds">'</span>__builtins__<span class="pl-pds">'</span></span>: __builtins__,
                <span class="pl-s"><span class="pl-pds">'</span>__name__<span class="pl-pds">'</span></span>: <span class="pl-s"><span class="pl-pds">'</span>__main__<span class="pl-pds">'</span></span>,
                <span class="pl-s"><span class="pl-pds">'</span>__doc__<span class="pl-pds">'</span></span>: <span class="pl-c1">None</span>,
                <span class="pl-s"><span class="pl-pds">'</span>__package__<span class="pl-pds">'</span></span>: <span class="pl-c1">None</span>,
            }
        local_names.update(callargs)
        frame <span class="pl-k">=</span> Frame(code, global_names, local_names, <span class="pl-v">self</span>.frame)
        <span class="pl-k">return</span> frame
    
    <span class="pl-k">def</span> <span class="pl-en">push_frame</span>(<span class="pl-smi">self</span>, <span class="pl-smi">frame</span>):
        <span class="pl-v">self</span>.frames.append(frame)
        <span class="pl-v">self</span>.frame <span class="pl-k">=</span> frame
    
    <span class="pl-k">def</span> <span class="pl-en">pop_frame</span>(<span class="pl-smi">self</span>):
        <span class="pl-v">self</span>.frames.pop()
        <span class="pl-k">if</span> <span class="pl-v">self</span>.frames:
            <span class="pl-v">self</span>.frame <span class="pl-k">=</span> <span class="pl-v">self</span>.frames[<span class="pl-k">-</span><span class="pl-c1">1</span>]
        <span class="pl-k">else</span>:
            <span class="pl-v">self</span>.frame <span class="pl-k">=</span> <span class="pl-c1">None</span>
    
    <span class="pl-k">def</span> <span class="pl-en">run_frame</span>(<span class="pl-smi">self</span>):
        <span class="pl-k">pass</span>
        <span class="pl-c"># we'll come back to this shortly</span></pre></div>

<h3>
<a id="the-function-class" class="anchor" href="http://qingyunha.github.io/taotao/#the-function-class" aria-hidden="true"><span class="octicon octicon-link"></span></a>The <code>Function</code> Class</h3>

<p><code>Function</code>的实现有点扭曲，但是大部分的细节对理解解释器不重要。重要的是当调用函数时 --- <code>__call__</code>方法被调用 --- 它创建一个新的<code>Frame</code>并运行它。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">class</span> <span class="pl-en">Function</span>(<span class="pl-e"><span class="pl-c1">object</span></span>):
    <span class="pl-s"><span class="pl-pds">"""</span></span>
<span class="pl-s">    Create a realistic function object, defining the things the interpreter expects.</span>
<span class="pl-s">    <span class="pl-pds">"""</span></span>
    <span class="pl-c1">__slots__</span> <span class="pl-k">=</span> [
        <span class="pl-s"><span class="pl-pds">'</span>func_code<span class="pl-pds">'</span></span>, <span class="pl-s"><span class="pl-pds">'</span>func_name<span class="pl-pds">'</span></span>, <span class="pl-s"><span class="pl-pds">'</span>func_defaults<span class="pl-pds">'</span></span>, <span class="pl-s"><span class="pl-pds">'</span>func_globals<span class="pl-pds">'</span></span>,
        <span class="pl-s"><span class="pl-pds">'</span>func_locals<span class="pl-pds">'</span></span>, <span class="pl-s"><span class="pl-pds">'</span>func_dict<span class="pl-pds">'</span></span>, <span class="pl-s"><span class="pl-pds">'</span>func_closure<span class="pl-pds">'</span></span>,
        <span class="pl-s"><span class="pl-pds">'</span>__name__<span class="pl-pds">'</span></span>, <span class="pl-s"><span class="pl-pds">'</span>__dict__<span class="pl-pds">'</span></span>, <span class="pl-s"><span class="pl-pds">'</span>__doc__<span class="pl-pds">'</span></span>,
        <span class="pl-s"><span class="pl-pds">'</span>_vm<span class="pl-pds">'</span></span>, <span class="pl-s"><span class="pl-pds">'</span>_func<span class="pl-pds">'</span></span>,
    ]

    <span class="pl-k">def</span> <span class="pl-en"><span class="pl-c1">__init__</span></span>(<span class="pl-smi">self</span>, <span class="pl-smi">name</span>, <span class="pl-smi">code</span>, <span class="pl-smi">globs</span>, <span class="pl-smi">defaults</span>, <span class="pl-smi">closure</span>, <span class="pl-smi">vm</span>):
        <span class="pl-s"><span class="pl-pds">"""</span>You don't need to follow this closely to understand the interpreter.<span class="pl-pds">"""</span></span>
        <span class="pl-v">self</span>._vm <span class="pl-k">=</span> vm
        <span class="pl-v">self</span>.func_code <span class="pl-k">=</span> code
        <span class="pl-v">self</span>.func_name <span class="pl-k">=</span> <span class="pl-v">self</span>.<span class="pl-c1">__name__</span> <span class="pl-k">=</span> name <span class="pl-k">or</span> code.co_name
        <span class="pl-v">self</span>.func_defaults <span class="pl-k">=</span> <span class="pl-c1">tuple</span>(defaults)
        <span class="pl-v">self</span>.func_globals <span class="pl-k">=</span> globs
        <span class="pl-v">self</span>.func_locals <span class="pl-k">=</span> <span class="pl-v">self</span>._vm.frame.f_locals
        <span class="pl-v">self</span>.<span class="pl-c1">__dict__</span> <span class="pl-k">=</span> {}
        <span class="pl-v">self</span>.func_closure <span class="pl-k">=</span> closure
        <span class="pl-v">self</span>.<span class="pl-c1">__doc__</span> <span class="pl-k">=</span> code.co_consts[<span class="pl-c1">0</span>] <span class="pl-k">if</span> code.co_consts <span class="pl-k">else</span> <span class="pl-c1">None</span>
    
        <span class="pl-c"># Sometimes, we need a real Python function.  This is for that.</span>
        kw <span class="pl-k">=</span> {
            <span class="pl-s"><span class="pl-pds">'</span>argdefs<span class="pl-pds">'</span></span>: <span class="pl-v">self</span>.func_defaults,
        }
        <span class="pl-k">if</span> closure:
            kw[<span class="pl-s"><span class="pl-pds">'</span>closure<span class="pl-pds">'</span></span>] <span class="pl-k">=</span> <span class="pl-c1">tuple</span>(make_cell(<span class="pl-c1">0</span>) <span class="pl-k">for</span> _ <span class="pl-k">in</span> closure)
        <span class="pl-v">self</span>._func <span class="pl-k">=</span> types.FunctionType(code, globs, <span class="pl-k">**</span>kw)
    
    <span class="pl-k">def</span> <span class="pl-en"><span class="pl-c1">__call__</span></span>(<span class="pl-smi">self</span>, *<span class="pl-smi">args</span>, **<span class="pl-smi">kwargs</span>):
        <span class="pl-s"><span class="pl-pds">"""</span>When calling a Function, make a new frame and run it.<span class="pl-pds">"""</span></span>
        callargs <span class="pl-k">=</span> inspect.getcallargs(<span class="pl-v">self</span>._func, <span class="pl-k">*</span>args, <span class="pl-k">**</span>kwargs)
        <span class="pl-c"># Use callargs to provide a mapping of arguments: values to pass into the new </span>
        <span class="pl-c"># frame.</span>
        frame <span class="pl-k">=</span> <span class="pl-v">self</span>._vm.make_frame(
            <span class="pl-v">self</span>.func_code, callargs, <span class="pl-v">self</span>.func_globals, {}
        )
        <span class="pl-k">return</span> <span class="pl-v">self</span>._vm.run_frame(frame)

<span class="pl-k">def</span> <span class="pl-en">make_cell</span>(<span class="pl-smi">value</span>):
    <span class="pl-s"><span class="pl-pds">"""</span>Create a real Python closure and grab a cell.<span class="pl-pds">"""</span></span>
    <span class="pl-c"># Thanks to Alex Gaynor for help with this bit of twistiness.</span>
    fn <span class="pl-k">=</span> (<span class="pl-k">lambda</span> <span class="pl-smi">x</span>: <span class="pl-k">lambda</span>: x)(value)
    <span class="pl-k">return</span> fn.<span class="pl-c1">__closure__</span>[<span class="pl-c1">0</span>]</pre></div>

<p>接着，回到<code>VirtualMachine</code>对象，我们对数据栈的操作也增加一些帮助方法。字节码操作的栈总是在当前frame的数据栈。这让我们能完成<code>POP_TOP</code>,<code>LOAD_FAST</code>,并且让其他操作栈的指令可读性更高。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">class</span> <span class="pl-en">VirtualMachine</span>(<span class="pl-e"><span class="pl-c1">object</span></span>):
    [... snip ...]

    <span class="pl-c"># Data stack manipulation</span>
    <span class="pl-k">def</span> <span class="pl-en">top</span>(<span class="pl-smi">self</span>):
        <span class="pl-k">return</span> <span class="pl-v">self</span>.frame.stack[<span class="pl-k">-</span><span class="pl-c1">1</span>]
    
    <span class="pl-k">def</span> <span class="pl-en">pop</span>(<span class="pl-smi">self</span>):
        <span class="pl-k">return</span> <span class="pl-v">self</span>.frame.stack.pop()
    
    <span class="pl-k">def</span> <span class="pl-en">push</span>(<span class="pl-smi">self</span>, *<span class="pl-smi">vals</span>):
        <span class="pl-v">self</span>.frame.stack.extend(vals)
    
    <span class="pl-k">def</span> <span class="pl-en">popn</span>(<span class="pl-smi">self</span>, <span class="pl-smi">n</span>):
        <span class="pl-s"><span class="pl-pds">"""</span>Pop a number of values from the value stack.</span>
<span class="pl-s">        A list of `n` values is returned, the deepest value first.</span>
<span class="pl-s">        <span class="pl-pds">"""</span></span>
        <span class="pl-k">if</span> n:
            ret <span class="pl-k">=</span> <span class="pl-v">self</span>.frame.stack[<span class="pl-k">-</span>n:]
            <span class="pl-v">self</span>.frame.stack[<span class="pl-k">-</span>n:] <span class="pl-k">=</span> []
            <span class="pl-k">return</span> ret
        <span class="pl-k">else</span>:
            <span class="pl-k">return</span> []</pre></div>

<p>在我们运行frame之前，我们还需两个方法。</p>

<p>第一个方法，<code>parse_byte_and_args</code>,以一个字节码为输入，先检查它是否有参数，如果有，就解析它的参数。这个方法同时也更新frame的<code>last_instruction</code>属性，它指向最后执行的指令。一条没有参数的指令只有一个字节长度，而有参数的字节有3个字节长。参数的意义依赖于指令是什么。比如，前面说过，指令<code>POP_JUMP_IF_FALSE</code>,它的参数指的是跳转目标。<code>BUILD_LIST</code>, 它的参数是列表的个数。<code>LOAD_CONST</code>,它的参数是常量的索引。</p>

<p>一些指令用简单的数字作为参数。对于另一些，虚拟机需要一点努力去发现它意义。标准库中的<code>dis</code>module中有一个备忘单解释什么参数有什么意思，这让我们的代码更加简洁。比如，列表<code>dis.hasname</code>告诉我们<code>LOAD_NAME</code>, <code>IMPORT_NAME</code>,<code>LOAD_GLOBAL</code>,和另外的9个指令都有同样的意思：名字列表的索引。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">class</span> <span class="pl-en">VirtualMachine</span>(<span class="pl-e"><span class="pl-c1">object</span></span>):
    [... snip ...]

    <span class="pl-k">def</span> <span class="pl-en">parse_byte_and_args</span>(<span class="pl-smi">self</span>):
        f <span class="pl-k">=</span> <span class="pl-v">self</span>.frame
        opoffset <span class="pl-k">=</span> f.last_instruction
        byteCode <span class="pl-k">=</span> f.code_obj.co_code[opoffset]
        f.last_instruction <span class="pl-k">+=</span> <span class="pl-c1">1</span>
        byte_name <span class="pl-k">=</span> dis.opname[byteCode]
        <span class="pl-k">if</span> byteCode <span class="pl-k">&gt;=</span> dis.HAVE_ARGUMENT:
            <span class="pl-c"># index into the bytecode</span>
            arg <span class="pl-k">=</span> f.code_obj.co_code[f.last_instruction:f.last_instruction<span class="pl-k">+</span><span class="pl-c1">2</span>]  
            f.last_instruction <span class="pl-k">+=</span> <span class="pl-c1">2</span>   <span class="pl-c"># advance the instruction pointer</span>
            arg_val <span class="pl-k">=</span> arg[<span class="pl-c1">0</span>] <span class="pl-k">+</span> (arg[<span class="pl-c1">1</span>] <span class="pl-k">*</span> <span class="pl-c1">256</span>)
            <span class="pl-k">if</span> byteCode <span class="pl-k">in</span> dis.hasconst:   <span class="pl-c"># Look up a constant</span>
                arg <span class="pl-k">=</span> f.code_obj.co_consts[arg_val]
            <span class="pl-k">elif</span> byteCode <span class="pl-k">in</span> dis.hasname:  <span class="pl-c"># Look up a name</span>
                arg <span class="pl-k">=</span> f.code_obj.co_names[arg_val]
            <span class="pl-k">elif</span> byteCode <span class="pl-k">in</span> dis.haslocal: <span class="pl-c"># Look up a local name</span>
                arg <span class="pl-k">=</span> f.code_obj.co_varnames[arg_val]
            <span class="pl-k">elif</span> byteCode <span class="pl-k">in</span> dis.hasjrel:  <span class="pl-c"># Calculate a relative jump</span>
                arg <span class="pl-k">=</span> f.last_instruction <span class="pl-k">+</span> arg_val
            <span class="pl-k">else</span>:
                arg <span class="pl-k">=</span> arg_val
            argument <span class="pl-k">=</span> [arg]
        <span class="pl-k">else</span>:
            argument <span class="pl-k">=</span> []
    
        <span class="pl-k">return</span> byte_name, argument</pre></div>

<p>下一个方法是<code>dispatch</code>,它查看给定的指令并执行相应的操作。在CPython中，这个分派函数用一个巨大的switch语句完成，有超过1500行的代码。幸运的是，我们用的是Python，我们的代码会简洁的多。我们会为每一个字节码名字定义一个方法，然后用<code>getattr</code>来查找。就像我们前面的小解释器一样，如果一条指令叫做<code>FOO_BAR</code>，那么它对应的方法就是<code>byte_FOO_BAR</code>。现在，我们先把这些方法当做一个黑盒子。每个指令方法都会返回<code>None</code>或者一个字符串<code>why</code>,有些情况下虚拟机需要这个额外<code>why</code>信息。这些指令方法的返回值，仅作为解释器状态的内部指示，千万不要和执行frame的返回值相混淆。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">class</span> <span class="pl-en">VirtualMachine</span>(<span class="pl-e"><span class="pl-c1">object</span></span>):
    [... snip ...]

    <span class="pl-k">def</span> <span class="pl-en">dispatch</span>(<span class="pl-smi">self</span>, <span class="pl-smi">byte_name</span>, <span class="pl-smi">argument</span>):
        <span class="pl-s"><span class="pl-pds">"""</span> Dispatch by bytename to the corresponding methods.</span>
<span class="pl-s">        Exceptions are caught and set on the virtual machine.<span class="pl-pds">"""</span></span>

        <span class="pl-c"># When later unwinding the block stack,</span>
        <span class="pl-c"># we need to keep track of why we are doing it.</span>
        why <span class="pl-k">=</span> <span class="pl-c1">None</span>
        <span class="pl-k">try</span>:
            bytecode_fn <span class="pl-k">=</span> <span class="pl-c1">getattr</span>(<span class="pl-v">self</span>, <span class="pl-s"><span class="pl-pds">'</span>byte_<span class="pl-c1">%s</span><span class="pl-pds">'</span></span> <span class="pl-k">%</span> byte_name, <span class="pl-c1">None</span>)
            <span class="pl-k">if</span> bytecode_fn <span class="pl-k">is</span> <span class="pl-c1">None</span>:
                <span class="pl-k">if</span> byte_name.startswith(<span class="pl-s"><span class="pl-pds">'</span>UNARY_<span class="pl-pds">'</span></span>):
                    <span class="pl-v">self</span>.unaryOperator(byte_name[<span class="pl-c1">6</span>:])
                <span class="pl-k">elif</span> byte_name.startswith(<span class="pl-s"><span class="pl-pds">'</span>BINARY_<span class="pl-pds">'</span></span>):
                    <span class="pl-v">self</span>.binaryOperator(byte_name[<span class="pl-c1">7</span>:])
                <span class="pl-k">else</span>:
                    <span class="pl-k">raise</span> VirtualMachineError(
                        <span class="pl-s"><span class="pl-pds">"</span>unsupported bytecode type: <span class="pl-c1">%s</span><span class="pl-pds">"</span></span> <span class="pl-k">%</span> byte_name
                    )
            <span class="pl-k">else</span>:
                why <span class="pl-k">=</span> bytecode_fn(<span class="pl-k">*</span>argument)
        <span class="pl-k">except</span>:
            <span class="pl-c"># deal with exceptions encountered while executing the op.</span>
            <span class="pl-v">self</span>.last_exception <span class="pl-k">=</span> sys.exc_info()[:<span class="pl-c1">2</span>] <span class="pl-k">+</span> (<span class="pl-c1">None</span>,)
            why <span class="pl-k">=</span> <span class="pl-s"><span class="pl-pds">'</span>exception<span class="pl-pds">'</span></span>
    
        <span class="pl-k">return</span> why
    
    <span class="pl-k">def</span> <span class="pl-en">run_frame</span>(<span class="pl-smi">self</span>, <span class="pl-smi">frame</span>):
        <span class="pl-s"><span class="pl-pds">"""</span>Run a frame until it returns (somehow).</span>
<span class="pl-s">        Exceptions are raised, the return value is returned.</span>
<span class="pl-s">        <span class="pl-pds">"""</span></span>
        <span class="pl-v">self</span>.push_frame(frame)
        <span class="pl-k">while</span> <span class="pl-c1">True</span>:
            byte_name, arguments <span class="pl-k">=</span> <span class="pl-v">self</span>.parse_byte_and_args()

            why <span class="pl-k">=</span> <span class="pl-v">self</span>.dispatch(byte_name, arguments)
    
            <span class="pl-c"># Deal with any block management we need to do</span>
            <span class="pl-k">while</span> why <span class="pl-k">and</span> frame.block_stack:
                why <span class="pl-k">=</span> <span class="pl-v">self</span>.manage_block_stack(why)
    
            <span class="pl-k">if</span> why:
                <span class="pl-k">break</span>
    
        <span class="pl-v">self</span>.pop_frame()
    
        <span class="pl-k">if</span> why <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">'</span>exception<span class="pl-pds">'</span></span>:
            exc, val, tb <span class="pl-k">=</span> <span class="pl-v">self</span>.last_exception
            e <span class="pl-k">=</span> exc(val)
            e.__traceback__ <span class="pl-k">=</span> tb
            <span class="pl-k">raise</span> e
    
        <span class="pl-k">return</span> <span class="pl-v">self</span>.return_value</pre></div>

<h3>
<a id="the-block-class" class="anchor" href="http://qingyunha.github.io/taotao/#the-block-class" aria-hidden="true"><span class="octicon octicon-link"></span></a>The <code>Block</code> Class</h3>

<p>在我们完成每个字节码方法前，我们简单的讨论一下块。一个块被用于某种控制流，特别是异常处理和循环。它负责保证当操作完成后数据栈处于正确的状态。比如，在一个循环中，一个特殊的迭代器会存在栈中，当循环完成时它从栈中弹出。解释器需要检查循环仍在继续还是已经停止。</p>

<p>为了跟踪这些额外的信息，解释器设置了一个标志来指示它的状态。我们用一个变量<code>why</code>实现这个标志，它可以<code>None</code>或者是下面几个字符串这一，<code>"continue"</code>, <code>"break"</code>,<code>"excption"</code>,<code>return</code>。他们指示对块栈和数据栈进行什么操作。回到我们迭代器的例子，如果块栈的栈顶是一个<code>loop</code>块，<code>why</code>是<code>continue</code>,迭代器就因该保存在数据栈上，不是如果<code>why</code>是<code>break</code>,迭代器就会被弹出。</p>

<p>块操作的细节比较精密，我们不会花时间在这上面，但是有兴趣的读者值得仔细的看看。</p>

<div class="highlight highlight-source-python"><pre>Block <span class="pl-k">=</span> collections.namedtuple(<span class="pl-s"><span class="pl-pds">"</span>Block<span class="pl-pds">"</span></span>, <span class="pl-s"><span class="pl-pds">"</span>type, handler, stack_height<span class="pl-pds">"</span></span>)

<span class="pl-k">class</span> <span class="pl-en">VirtualMachine</span>(<span class="pl-e"><span class="pl-c1">object</span></span>):
    [... snip ...]

    <span class="pl-c"># Block stack manipulation</span>
    <span class="pl-k">def</span> <span class="pl-en">push_block</span>(<span class="pl-smi">self</span>, <span class="pl-smi">b_type</span>, <span class="pl-smi">handler</span><span class="pl-k">=</span><span class="pl-c1">None</span>):
        level <span class="pl-k">=</span> <span class="pl-c1">len</span>(<span class="pl-v">self</span>.frame.stack)
        <span class="pl-v">self</span>.frame.block_stack.append(Block(b_type, handler, stack_height))
    
    <span class="pl-k">def</span> <span class="pl-en">pop_block</span>(<span class="pl-smi">self</span>):
        <span class="pl-k">return</span> <span class="pl-v">self</span>.frame.block_stack.pop()
    
    <span class="pl-k">def</span> <span class="pl-en">unwind_block</span>(<span class="pl-smi">self</span>, <span class="pl-smi">block</span>):
        <span class="pl-s"><span class="pl-pds">"""</span>Unwind the values on the data stack corresponding to a given block.<span class="pl-pds">"""</span></span>
        <span class="pl-k">if</span> block.type <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">'</span>except-handler<span class="pl-pds">'</span></span>:
            <span class="pl-c"># The exception itself is on the stack as type, value, and traceback.</span>
            offset <span class="pl-k">=</span> <span class="pl-c1">3</span>  
        <span class="pl-k">else</span>:
            offset <span class="pl-k">=</span> <span class="pl-c1">0</span>
    
        <span class="pl-k">while</span> <span class="pl-c1">len</span>(<span class="pl-v">self</span>.frame.stack) <span class="pl-k">&gt;</span> block.level <span class="pl-k">+</span> offset:
            <span class="pl-v">self</span>.pop()
    
        <span class="pl-k">if</span> block.type <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">'</span>except-handler<span class="pl-pds">'</span></span>:
            traceback, value, exctype <span class="pl-k">=</span> <span class="pl-v">self</span>.popn(<span class="pl-c1">3</span>)
            <span class="pl-v">self</span>.last_exception <span class="pl-k">=</span> exctype, value, traceback
    
    <span class="pl-k">def</span> <span class="pl-en">manage_block_stack</span>(<span class="pl-smi">self</span>, <span class="pl-smi">why</span>):
        <span class="pl-s"><span class="pl-pds">"""</span> <span class="pl-pds">"""</span></span>
        frame <span class="pl-k">=</span> <span class="pl-v">self</span>.frame
        block <span class="pl-k">=</span> frame.block_stack[<span class="pl-k">-</span><span class="pl-c1">1</span>]
        <span class="pl-k">if</span> block.type <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">'</span>loop<span class="pl-pds">'</span></span> <span class="pl-k">and</span> why <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">'</span>continue<span class="pl-pds">'</span></span>:
            <span class="pl-v">self</span>.jump(<span class="pl-v">self</span>.return_value)
            why <span class="pl-k">=</span> <span class="pl-c1">None</span>
            <span class="pl-k">return</span> why
    
        <span class="pl-v">self</span>.pop_block()
        <span class="pl-v">self</span>.unwind_block(block)
    
        <span class="pl-k">if</span> block.type <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">'</span>loop<span class="pl-pds">'</span></span> <span class="pl-k">and</span> why <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">'</span>break<span class="pl-pds">'</span></span>:
            why <span class="pl-k">=</span> <span class="pl-c1">None</span>
            <span class="pl-v">self</span>.jump(block.handler)
            <span class="pl-k">return</span> why
    
        <span class="pl-k">if</span> (block.type <span class="pl-k">in</span> [<span class="pl-s"><span class="pl-pds">'</span>setup-except<span class="pl-pds">'</span></span>, <span class="pl-s"><span class="pl-pds">'</span>finally<span class="pl-pds">'</span></span>] <span class="pl-k">and</span> why <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">'</span>exception<span class="pl-pds">'</span></span>):
            <span class="pl-v">self</span>.push_block(<span class="pl-s"><span class="pl-pds">'</span>except-handler<span class="pl-pds">'</span></span>)
            exctype, value, tb <span class="pl-k">=</span> <span class="pl-v">self</span>.last_exception
            <span class="pl-v">self</span>.push(tb, value, exctype)
            <span class="pl-v">self</span>.push(tb, value, exctype) <span class="pl-c"># yes, twice</span>
            why <span class="pl-k">=</span> <span class="pl-c1">None</span>
            <span class="pl-v">self</span>.jump(block.handler)
            <span class="pl-k">return</span> why
    
        <span class="pl-k">elif</span> block.type <span class="pl-k">==</span> <span class="pl-s"><span class="pl-pds">'</span>finally<span class="pl-pds">'</span></span>:
            <span class="pl-k">if</span> why <span class="pl-k">in</span> (<span class="pl-s"><span class="pl-pds">'</span>return<span class="pl-pds">'</span></span>, <span class="pl-s"><span class="pl-pds">'</span>continue<span class="pl-pds">'</span></span>):
                <span class="pl-v">self</span>.push(<span class="pl-v">self</span>.return_value)
    
            <span class="pl-v">self</span>.push(why)
    
            why <span class="pl-k">=</span> <span class="pl-c1">None</span>
            <span class="pl-v">self</span>.jump(block.handler)
            <span class="pl-k">return</span> why
        <span class="pl-k">return</span> why</pre></div>

<h2>
<a id="the-instructions" class="anchor" href="http://qingyunha.github.io/taotao/#the-instructions" aria-hidden="true"><span class="octicon octicon-link"></span></a>The Instructions</h2>

<p>剩下了的就是完成那些指令方法了：<code>byte_LOAD_FAST</code>,<code>byte_BINARY_MODULO</code>等等。而这些指令的实现并不是很有趣，这里我们只展示了一小部分，完整的实现在<a href="https://github.com/nedbat/byterun">这儿</a>（足够执行我们前面所述的所有代码了。）</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">class</span> <span class="pl-en">VirtualMachine</span>(<span class="pl-e"><span class="pl-c1">object</span></span>):
    [... snip ...]

    <span class="pl-c">## Stack manipulation</span>
    
    <span class="pl-k">def</span> <span class="pl-en">byte_LOAD_CONST</span>(<span class="pl-smi">self</span>, <span class="pl-smi">const</span>):
        <span class="pl-v">self</span>.push(const)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_POP_TOP</span>(<span class="pl-smi">self</span>):
        <span class="pl-v">self</span>.pop()
    
    <span class="pl-c">## Names</span>
    <span class="pl-k">def</span> <span class="pl-en">byte_LOAD_NAME</span>(<span class="pl-smi">self</span>, <span class="pl-smi">name</span>):
        frame <span class="pl-k">=</span> <span class="pl-v">self</span>.frame
        <span class="pl-k">if</span> name <span class="pl-k">in</span> frame.f_locals:
            val <span class="pl-k">=</span> frame.f_locals[name]
        <span class="pl-k">elif</span> name <span class="pl-k">in</span> frame.f_globals:
            val <span class="pl-k">=</span> frame.f_globals[name]
        <span class="pl-k">elif</span> name <span class="pl-k">in</span> frame.f_builtins:
            val <span class="pl-k">=</span> frame.f_builtins[name]
        <span class="pl-k">else</span>:
            <span class="pl-k">raise</span> <span class="pl-c1">NameError</span>(<span class="pl-s"><span class="pl-pds">"</span>name '<span class="pl-c1">%s</span>' is not defined<span class="pl-pds">"</span></span> <span class="pl-k">%</span> name)
        <span class="pl-v">self</span>.push(val)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_STORE_NAME</span>(<span class="pl-smi">self</span>, <span class="pl-smi">name</span>):
        <span class="pl-v">self</span>.frame.f_locals[name] <span class="pl-k">=</span> <span class="pl-v">self</span>.pop()
    
    <span class="pl-k">def</span> <span class="pl-en">byte_LOAD_FAST</span>(<span class="pl-smi">self</span>, <span class="pl-smi">name</span>):
        <span class="pl-k">if</span> name <span class="pl-k">in</span> <span class="pl-v">self</span>.frame.f_locals:
            val <span class="pl-k">=</span> <span class="pl-v">self</span>.frame.f_locals[name]
        <span class="pl-k">else</span>:
            <span class="pl-k">raise</span> <span class="pl-c1">UnboundLocalError</span>(
                <span class="pl-s"><span class="pl-pds">"</span>local variable '<span class="pl-c1">%s</span>' referenced before assignment<span class="pl-pds">"</span></span> <span class="pl-k">%</span> name
            )
        <span class="pl-v">self</span>.push(val)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_STORE_FAST</span>(<span class="pl-smi">self</span>, <span class="pl-smi">name</span>):
        <span class="pl-v">self</span>.frame.f_locals[name] <span class="pl-k">=</span> <span class="pl-v">self</span>.pop()
    
    <span class="pl-k">def</span> <span class="pl-en">byte_LOAD_GLOBAL</span>(<span class="pl-smi">self</span>, <span class="pl-smi">name</span>):
        f <span class="pl-k">=</span> <span class="pl-v">self</span>.frame
        <span class="pl-k">if</span> name <span class="pl-k">in</span> f.f_globals:
            val <span class="pl-k">=</span> f.f_globals[name]
        <span class="pl-k">elif</span> name <span class="pl-k">in</span> f.f_builtins:
            val <span class="pl-k">=</span> f.f_builtins[name]
        <span class="pl-k">else</span>:
            <span class="pl-k">raise</span> <span class="pl-c1">NameError</span>(<span class="pl-s"><span class="pl-pds">"</span>global name '<span class="pl-c1">%s</span>' is not defined<span class="pl-pds">"</span></span> <span class="pl-k">%</span> name)
        <span class="pl-v">self</span>.push(val)
    
    <span class="pl-c">## Operators</span>
    
    BINARY_OPERATORS <span class="pl-k">=</span> {
        <span class="pl-s"><span class="pl-pds">'</span>POWER<span class="pl-pds">'</span></span>:    <span class="pl-c1">pow</span>,
        <span class="pl-s"><span class="pl-pds">'</span>MULTIPLY<span class="pl-pds">'</span></span>: operator.mul,
        <span class="pl-s"><span class="pl-pds">'</span>FLOOR_DIVIDE<span class="pl-pds">'</span></span>: operator.floordiv,
        <span class="pl-s"><span class="pl-pds">'</span>TRUE_DIVIDE<span class="pl-pds">'</span></span>:  operator.truediv,
        <span class="pl-s"><span class="pl-pds">'</span>MODULO<span class="pl-pds">'</span></span>:   operator.mod,
        <span class="pl-s"><span class="pl-pds">'</span>ADD<span class="pl-pds">'</span></span>:      operator.add,
        <span class="pl-s"><span class="pl-pds">'</span>SUBTRACT<span class="pl-pds">'</span></span>: operator.sub,
        <span class="pl-s"><span class="pl-pds">'</span>SUBSCR<span class="pl-pds">'</span></span>:   operator.getitem,
        <span class="pl-s"><span class="pl-pds">'</span>LSHIFT<span class="pl-pds">'</span></span>:   operator.lshift,
        <span class="pl-s"><span class="pl-pds">'</span>RSHIFT<span class="pl-pds">'</span></span>:   operator.rshift,
        <span class="pl-s"><span class="pl-pds">'</span>AND<span class="pl-pds">'</span></span>:      operator.and_,
        <span class="pl-s"><span class="pl-pds">'</span>XOR<span class="pl-pds">'</span></span>:      operator.xor,
        <span class="pl-s"><span class="pl-pds">'</span>OR<span class="pl-pds">'</span></span>:       operator.or_,
    }
    
    <span class="pl-k">def</span> <span class="pl-en">binaryOperator</span>(<span class="pl-smi">self</span>, <span class="pl-smi">op</span>):
        x, y <span class="pl-k">=</span> <span class="pl-v">self</span>.popn(<span class="pl-c1">2</span>)
        <span class="pl-v">self</span>.push(<span class="pl-v">self</span>.BINARY_OPERATORS[op](x, y))
    
    COMPARE_OPERATORS <span class="pl-k">=</span> [
        operator.lt,
        operator.le,
        operator.eq,
        operator.ne,
        operator.gt,
        operator.ge,
        <span class="pl-k">lambda</span> <span class="pl-smi">x</span>, <span class="pl-smi">y</span>: x <span class="pl-k">in</span> y,
        <span class="pl-k">lambda</span> <span class="pl-smi">x</span>, <span class="pl-smi">y</span>: x <span class="pl-k">not</span> <span class="pl-k">in</span> y,
        <span class="pl-k">lambda</span> <span class="pl-smi">x</span>, <span class="pl-smi">y</span>: x <span class="pl-k">is</span> y,
        <span class="pl-k">lambda</span> <span class="pl-smi">x</span>, <span class="pl-smi">y</span>: x <span class="pl-k">is</span> <span class="pl-k">not</span> y,
        <span class="pl-k">lambda</span> <span class="pl-smi">x</span>, <span class="pl-smi">y</span>: <span class="pl-c1">issubclass</span>(x, <span class="pl-c1">Exception</span>) <span class="pl-k">and</span> <span class="pl-c1">issubclass</span>(x, y),
    ]
    
    <span class="pl-k">def</span> <span class="pl-en">byte_COMPARE_OP</span>(<span class="pl-smi">self</span>, <span class="pl-smi">opnum</span>):
        x, y <span class="pl-k">=</span> <span class="pl-v">self</span>.popn(<span class="pl-c1">2</span>)
        <span class="pl-v">self</span>.push(<span class="pl-v">self</span>.COMPARE_OPERATORS[opnum](x, y))
    
    <span class="pl-c">## Attributes and indexing</span>
    
    <span class="pl-k">def</span> <span class="pl-en">byte_LOAD_ATTR</span>(<span class="pl-smi">self</span>, <span class="pl-smi">attr</span>):
        obj <span class="pl-k">=</span> <span class="pl-v">self</span>.pop()
        val <span class="pl-k">=</span> <span class="pl-c1">getattr</span>(obj, attr)
        <span class="pl-v">self</span>.push(val)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_STORE_ATTR</span>(<span class="pl-smi">self</span>, <span class="pl-smi">name</span>):
        val, obj <span class="pl-k">=</span> <span class="pl-v">self</span>.popn(<span class="pl-c1">2</span>)
        <span class="pl-c1">setattr</span>(obj, name, val)
    
    <span class="pl-c">## Building</span>
    
    <span class="pl-k">def</span> <span class="pl-en">byte_BUILD_LIST</span>(<span class="pl-smi">self</span>, <span class="pl-smi">count</span>):
        elts <span class="pl-k">=</span> <span class="pl-v">self</span>.popn(count)
        <span class="pl-v">self</span>.push(elts)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_BUILD_MAP</span>(<span class="pl-smi">self</span>, <span class="pl-smi">size</span>):
        <span class="pl-v">self</span>.push({})
    
    <span class="pl-k">def</span> <span class="pl-en">byte_STORE_MAP</span>(<span class="pl-smi">self</span>):
        the_map, val, key <span class="pl-k">=</span> <span class="pl-v">self</span>.popn(<span class="pl-c1">3</span>)
        the_map[key] <span class="pl-k">=</span> val
        <span class="pl-v">self</span>.push(the_map)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_LIST_APPEND</span>(<span class="pl-smi">self</span>, <span class="pl-smi">count</span>):
        val <span class="pl-k">=</span> <span class="pl-v">self</span>.pop()
        the_list <span class="pl-k">=</span> <span class="pl-v">self</span>.frame.stack[<span class="pl-k">-</span>count] <span class="pl-c"># peek</span>
        the_list.append(val)
    
    <span class="pl-c">## Jumps</span>
    
    <span class="pl-k">def</span> <span class="pl-en">byte_JUMP_FORWARD</span>(<span class="pl-smi">self</span>, <span class="pl-smi">jump</span>):
        <span class="pl-v">self</span>.jump(jump)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_JUMP_ABSOLUTE</span>(<span class="pl-smi">self</span>, <span class="pl-smi">jump</span>):
        <span class="pl-v">self</span>.jump(jump)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_POP_JUMP_IF_TRUE</span>(<span class="pl-smi">self</span>, <span class="pl-smi">jump</span>):
        val <span class="pl-k">=</span> <span class="pl-v">self</span>.pop()
        <span class="pl-k">if</span> val:
            <span class="pl-v">self</span>.jump(jump)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_POP_JUMP_IF_FALSE</span>(<span class="pl-smi">self</span>, <span class="pl-smi">jump</span>):
        val <span class="pl-k">=</span> <span class="pl-v">self</span>.pop()
        <span class="pl-k">if</span> <span class="pl-k">not</span> val:
            <span class="pl-v">self</span>.jump(jump)
    
    <span class="pl-c">## Blocks</span>
    
    <span class="pl-k">def</span> <span class="pl-en">byte_SETUP_LOOP</span>(<span class="pl-smi">self</span>, <span class="pl-smi">dest</span>):
        <span class="pl-v">self</span>.push_block(<span class="pl-s"><span class="pl-pds">'</span>loop<span class="pl-pds">'</span></span>, dest)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_GET_ITER</span>(<span class="pl-smi">self</span>):
        <span class="pl-v">self</span>.push(<span class="pl-c1">iter</span>(<span class="pl-v">self</span>.pop()))
    
    <span class="pl-k">def</span> <span class="pl-en">byte_FOR_ITER</span>(<span class="pl-smi">self</span>, <span class="pl-smi">jump</span>):
        iterobj <span class="pl-k">=</span> <span class="pl-v">self</span>.top()
        <span class="pl-k">try</span>:
            v <span class="pl-k">=</span> <span class="pl-c1">next</span>(iterobj)
            <span class="pl-v">self</span>.push(v)
        <span class="pl-k">except</span> <span class="pl-c1">StopIteration</span>:
            <span class="pl-v">self</span>.pop()
            <span class="pl-v">self</span>.jump(jump)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_BREAK_LOOP</span>(<span class="pl-smi">self</span>):
        <span class="pl-k">return</span> <span class="pl-s"><span class="pl-pds">'</span>break<span class="pl-pds">'</span></span>
    
    <span class="pl-k">def</span> <span class="pl-en">byte_POP_BLOCK</span>(<span class="pl-smi">self</span>):
        <span class="pl-v">self</span>.pop_block()
    
    <span class="pl-c">## Functions</span>
    
    <span class="pl-k">def</span> <span class="pl-en">byte_MAKE_FUNCTION</span>(<span class="pl-smi">self</span>, <span class="pl-smi">argc</span>):
        name <span class="pl-k">=</span> <span class="pl-v">self</span>.pop()
        code <span class="pl-k">=</span> <span class="pl-v">self</span>.pop()
        defaults <span class="pl-k">=</span> <span class="pl-v">self</span>.popn(argc)
        globs <span class="pl-k">=</span> <span class="pl-v">self</span>.frame.f_globals
        fn <span class="pl-k">=</span> Function(name, code, globs, defaults, <span class="pl-c1">None</span>, <span class="pl-v">self</span>)
        <span class="pl-v">self</span>.push(fn)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_CALL_FUNCTION</span>(<span class="pl-smi">self</span>, <span class="pl-smi">arg</span>):
        lenKw, lenPos <span class="pl-k">=</span> <span class="pl-c1">divmod</span>(arg, <span class="pl-c1">256</span>) <span class="pl-c"># KWargs not supported here</span>
        posargs <span class="pl-k">=</span> <span class="pl-v">self</span>.popn(lenPos)
    
        func <span class="pl-k">=</span> <span class="pl-v">self</span>.pop()
        frame <span class="pl-k">=</span> <span class="pl-v">self</span>.frame
        retval <span class="pl-k">=</span> func(<span class="pl-k">*</span>posargs)
        <span class="pl-v">self</span>.push(retval)
    
    <span class="pl-k">def</span> <span class="pl-en">byte_RETURN_VALUE</span>(<span class="pl-smi">self</span>):
        <span class="pl-v">self</span>.return_value <span class="pl-k">=</span> <span class="pl-v">self</span>.pop()
        <span class="pl-k">return</span> <span class="pl-s"><span class="pl-pds">"</span>return<span class="pl-pds">"</span></span></pre></div>

<h2>
<a id="dynamic-typing-what-the-compiler-doesnt-know" class="anchor" href="http://qingyunha.github.io/taotao/#dynamic-typing-what-the-compiler-doesnt-know" aria-hidden="true"><span class="octicon octicon-link"></span></a>Dynamic Typing: What the Compiler Doesn't Know</h2>

<p>你可能听过Python是一种动态语言 --- 是它是动态类型的。在我们建造解释器的过程中，已经流露出这个描述。</p>

<p>动态的一个意思是很多工作在运行时完成。前面我们看到Python的编译器没有很多关于代码真正做什么的信息。举个例子，考虑下面这个简单的函数<code>mod</code>。它区两个参数，返回它们的模运算值。从它的字节码中，我们看到变量<code>a</code>和<code>b</code>首先被加载，然后字节码<code>BINAY_MODULO</code>完成这个模运算。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">&gt;&gt;&gt;</span> <span class="pl-k">def</span> mod(a, b):
...    <span class="pl-k">return</span> a <span class="pl-k">%</span> b
<span class="pl-k">&gt;&gt;&gt;</span> dis.dis(mod)
  <span class="pl-c1">2</span>           <span class="pl-c1">0</span> LOAD_FAST                <span class="pl-c1">0</span> (a)
              <span class="pl-c1">3</span> LOAD_FAST                <span class="pl-c1">1</span> (b)
              <span class="pl-c1">6</span> BINARY_MODULO
              <span class="pl-c1">7</span> RETURN_VALUE
<span class="pl-k">&gt;&gt;&gt;</span> mod(<span class="pl-c1">19</span>, <span class="pl-c1">5</span>)
<span class="pl-c1">4</span></pre></div>

<p>计算19 % 5得4，--- 一点也不奇怪。如果我们用不同类的参数呢？</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">&gt;&gt;&gt;</span> mod(<span class="pl-s"><span class="pl-pds">"</span>by<span class="pl-c1">%s</span>de<span class="pl-pds">"</span></span>, <span class="pl-s"><span class="pl-pds">"</span>teco<span class="pl-pds">"</span></span>)
<span class="pl-s"><span class="pl-pds">'</span>bytecode<span class="pl-pds">'</span></span></pre></div>

<p>刚才发生了什么？你可能见过这样的语法，格式化字符串。</p>

<pre><code>&gt;&gt;&gt; print("by%sde" % "teco")
bytecode
</code></pre>

<p>用符号<code>%</code>去格式化字符串会调用字节码<code>BUNARY_MODULO</code>.它取栈顶的两个值求模，不管这两个值是字符串，数字或是你自己定义的类的实例。字节码在函数编译时生成（或者说，函数定义时）相同的字节码会用于不同类的参数。</p>

<p>Python的编译器关于字节码的功能知道的很少。而取决于解释器来决定<code>BINAYR_MODULO</code>应用于什么类型的对象并完成真确的操作。这就是为什么Python被描述为<em>动态类型</em>：直到你运行前你不必知道这个函数参数的类型。相反，在一个静态类型语言中，程序员需要告诉编译器参数的类型是什么（或者编译器自己推断出参数的类型。）</p>

<p>编译器的无知是优化Python的一个挑战 --- 只看字节码，而不真正运行它，你就不知道每条字节码在干什么！你可以定义一个类，实现<code>__mod__</code>方法，当你对这个类的实例使用<code>%</code>时，Python就会自动调用这个方法。所以，<code>BINARY_MODULO</code>其实可以运行任何代码。</p>

<p> 看看下面的代码，第一个<code>a % b</code>看起来没有用。</p>

<div class="highlight highlight-source-python"><pre><span class="pl-k">def</span> <span class="pl-en">mod</span>(<span class="pl-smi">a</span>,<span class="pl-smi">b</span>):
    a <span class="pl-k">%</span> b
    <span class="pl-k">return</span> a <span class="pl-k">%</span>b</pre></div>

<p>不幸的是，对这段代码进行静态分析 --- 不运行它 --- 不能确定第一个<code>a % b</code>没有做任何事。用 <code>%</code>调用<code>__mod__</code>可能会写一个文件，或是和程序的其他部分交互，或者其他任何可以在Python中完成的事。很难优化一个你不知道它会做什么的函数。在Russell Power和Alex Rubinsteyn的优秀论文中写道，“我们可用多快的速度解释Python？”，他们说，“在普遍缺乏类型信息下，每条指令必须被看作一个<code>INVOKE_ARBITRARY_METHOD</code>。”</p>

<h2>
<a id="conclusion" class="anchor" href="http://qingyunha.github.io/taotao/#conclusion" aria-hidden="true"><span class="octicon octicon-link"></span></a>Conclusion</h2>

<p>Byterun是一个比CPython容易理解的简洁的Python解释器。Byterun复制了CPython的主要结构：一个基于栈的指令集称为字节码，它们顺序执行或在指令间跳转，向栈中压入和从中弹出数据。解释器随着函数和生成器的调用和返回，动态的创建，销毁frame，并在frame间跳转。Byterun也有着和真正解释器一样的限制：因为Python使用动态类型，解释器必须在运行时决定指令的正确行为。</p>

<p>我鼓励你去反汇编你的程序，然后用Byterun来运行。你很快会发现这个缩短版的Byterun所没有实现的指令。完整的实现在<a href="https://github.com/nedbat/byterun">这里</a>或者仔细阅读真正的CPython解释器<code>ceval.c</code>,你也可以实现自己的解释器！</p>

<h2>
<a id="acknowledgements" class="anchor" href="http://qingyunha.github.io/taotao/#acknowledgements" aria-hidden="true"><span class="octicon octicon-link"></span></a>Acknowledgements</h2>

<p>Thanks to Ned Batchelder for originating this project and guiding my contributions, Michael Arntzenius for his help debugging the code and editing the prose, Leta Montopoli for her edits, and the entire Recurse Center community for their support and interest. Any errors are my own.</p>

      <footer class="site-footer">
        <span class="site-footer-owner"><a href="https://github.com/qingyunha/taotao">A Python Interpreter Written in Python</a> is maintained by <a href="https://github.com/qingyunha">qingyunha</a>.</span>
    
        <span class="site-footer-credits">This page was generated by <a href="https://pages.github.com/">GitHub Pages</a> using the <a href="https://github.com/jasonlong/cayman-theme">Cayman theme</a> by <a href="https://twitter.com/jasonlong">Jason Long</a>.</span>
      </footer>
    
    </section>

  

  

</body></html>