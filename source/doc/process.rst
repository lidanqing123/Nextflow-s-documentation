.. _process-page:

*********
Processes
*********

在Nextflow中，``process`` 是执行用户脚本的基本处理进程。


process定义以关键字 ``process`` 开始，后跟process名称，最后由大括号分隔的process代码块。 ``process`` 正文必须包含表示命令的字符串，或者更一般地说，包含由其执行的脚本。 一个基本 ``process`` 如下例所示::
  process sayHello {

      """
      echo 'Hello world!' > file
      """

  }


一个process可以分别包含五个定义块, 分别是：指令(directives)，输入(inputs)，输出(outputs)，when子句和最后的process脚本。 语法定义如下：
::

  process < name > {

     [ directives ]

     input:
      < process inputs >

     output:
      < process outputs >

     when:
      < condition >

     [script|shell|exec]:
     < user script to be executed >

  }


.. _process-script:

脚本
======

脚本块是一个字符串语句，用于定义process执行其执行任务的命令。

process包含一个且只有一个脚本块，当process包含输入和输出声明时，它必须是最后一个语句。

输入的字符串在主机系统中作为`Bash <http://en.wikipedia.org/wiki/Bash_(Unix_shell)>`_ 脚本执行。 它可以是您通常在终端shell或常见的Bash脚本中使用的任何命令，脚本或它们的组合。

脚本语句中使用的命令的唯一限制是在目标执行系统中那些实用性程序。

脚本块可以是简单的字符串或多行字符串。 后者简化了由跨越多行的多个命令组成的非平凡脚本的编写。 例如::

    process doMoreThings {

      """
      blastp -db $db -query query.fa -outfmt 6 > blast_result
      cat blast_result | head -n 10 | cut -f 2 > top_hits
      blastdbcmd -db $db -entry_batch top_hits > sequences
      """

    }

如脚本教程部分所述，可以使用单引号或双引号定义字符串，多行字符串由三个单引号或三个双引号字符定义。

它们之间存在微妙但重要的区别。 就像在Bash中一样，双引号 ``"`` 的字符串支持变量替换，而单引号 ``'`` 的字符串则不行。


在上面的代码片段中， ``$db`` 变量被流程脚本中某处定义的实际值替换。

.. warning:: 由于Nextflow对字符串中的变量替换使用相同的Bash语法，因此您需要仔细管理它们，具体取决于您是否要在Nextflow上下文中评估变量 - 或者 - 在Bash环境执行中。


当您需要在脚本中访问系统环境变量时，您有两个选择。 第一种选择就像使用单引号字符串定义脚本块一样简单。 例如::

    process printPath {

       '''
       echo The path is: $PATH
       '''

    }


此解决方案的缺点是您将无法在脚本块中访问流程脚本上下文中定义的变量。

要解决此问题，请使用双引号字符串定义脚本，并通过在前面添加反斜杠( ``\`` )字符来转义(`escape` )系统环境变量，如以下示例所示::


    process doOtherThings {

      """
      blastp -db \$DB -query query.fa -outfmt 6 > blast_result
      cat blast_result | head -n $MAX | cut -f 2 > top_hits
      blastdbcmd -db \$DB -entry_batch top_hits > sequences
      """

    }


在此示例中，必须在流程脚本之前的某处定义 ``$MAX`` 变量。 在执行脚本之前，Nextflow会将其替换为实际值。 相反，``$DB`` 变量必须存在于脚本执行环境中，Bash解释器将用实际值替换它。

.. tip::
  或者，您可以使用 :ref:`process-shell` 块定义，该定义允许脚本包含Bash和Nextflow变量，而不必转义第一个。


脚本语言
--------------------

默认情况下，Nextflow Process将流程脚本解释为Bash脚本，但您不必受限于此。

您可以使用自己喜欢的脚本语言（例如Perl，Python，Ruby，R等），甚至可以将它们混合在同一个流程中。

流程可以由执行非常不同的任务的进程组成。 使用Nextflow，您可以选择更适合指定进程执行的任务的脚本语言。 例如，对于某些进程，R可能比Perl更有用，在其他情况下，您可能需要使用Python，因为它可以更好地访问一个库或一个API等。

要使用Bash以外的脚本，只需使用相应的 `shebang <http://en.wikipedia.org/wiki/Shebang_(Unix)>`_ 声明启动您的流程脚本。 例如::

    process perlStuff {

        """
        #!/usr/bin/perl

        print 'Hi there!' . '\n';
        """

    }

    process pyStuff {

        """
        #!/usr/bin/python

        x = 'Hello'
        y = 'world!'
        print "%s - %s" % (x,y)
        """

    }


.. tip:: 由于解释器二进制文件的实际位置可以跨平台更改，为了使脚本更具可移植性，最好使用 ``env`` shell命令后跟解释器的名称，而不是它的绝对路径。 因此，例如，Perl脚本的shebang声明看起来像： ``#!/usr/bin/env perl`` 而不是上面流程片段中的那个。



条件脚本
-------------------


复杂的process脚本可能需要评估输入参数的条件或使用传统的数据流控制语句（即 ``if``, ``switch``,等）以执行特定的脚本命令，具体取决于当前的输入配置.


``process`` 脚本可以通过在脚本块前面加上关键字 ``mode`` 前缀来包含条件语句. 通过这样做，解释器将所有以下语句评估为必须返回要执行的脚本字符串的代码块. 
它比使用解释更容易, 例如::


    seq_to_align = ...
    mode = 'tcoffee'

    process align {
        input:
        file seq_to_aln from sequences

        script:
        if( mode == 'tcoffee' )
            """
            t_coffee -in $seq_to_aln > out_file
            """

        else if( mode == 'mafft' )
            """
            mafft --anysymbol --parttree --quiet $seq_to_aln > out_file
            """

        else if( mode == 'clustalo' )
            """
            clustalo -i $seq_to_aln -o out_file
            """

        else
            error "Invalid alignment mode: ${mode}"

    }



在上面的示例中，process将根据 ``mode`` 参数的值执行脚本片段。 默认情况下，它将执行  ``tcoffee``  命令，将 ``mode`` 变量更改为 ``mafft`` 或 ``clustalo`` 值，其他分支将被执行。

.. _process-template:

模板
--------

process脚本可以通过使用 *template* 文件进行外部化，模板文件可以在不同的process中重复使用，并由整个流程执行独立测试。


模板只是一个shell脚本文件，Nextflow可以使用 ``template`` 函数执行，如下所示::

    process template_example {

        input:
        val STR from 'this', 'that'

        script:
        template 'my_script.sh'

    }


Nextflow在 ``templates`` 目录中查找 ``my_script.sh`` 模板文件，该模板必须存在于Nextflow脚本文件所在的同一文件夹中（可以使用绝对模板路径提供任何其他位置）。

模板脚本可以包含可由底层系统执行的任何代码段。 例如::

  #!/bin/bash
  echo "process started at `date`"
  echo $STR
  :
  echo "process completed"



.. tip::  请注意，当脚本作为Nextflow模板运行时，美元字符（``$``）将被解释为Nextflow变量占位符，而当单独运行时，它将被计算为Bash变量。 这对于自主测试脚本非常有用，即独立于Nextflow执行。 您只需为脚本中存在的每个Nextflow变量提供Bash环境变量。 例如，可以执行以上脚本在shell终端中输入以下命令：``STR='foo' bash templates/my_script.sh``


.. _process-shell:

Shell
-----

.. warning::  这是一个孵化功能。 它可能会在将来的Nextflow版本中发生变化。


 ``shell`` 块是一个字符串语句，它定义process执行的shell命令以执行其任务。 它是 :ref:`process-script` 定义的替代品，具有重要的区别，它使用感叹号``!`` 字符作为Nextflow变量的变量占位符，代替通常的美元字符。


通过这种方式，可以在同一段代码中同时使用Nextflow和Bash变量，而不必逃避后者并使流程脚本更易读和易于维护。 例如::

    process myTask {

        input:
        val str from 'Hello', 'Hola', 'Bonjour'

        shell:
        '''
        echo User $USER says !{str}
        '''

    }


在上面的简单示例中，``$USER`` 变量由Bash解释器管理，而``!{str}``作为由Nextflow管理的process输入变量处理。

.. note::  -Shell脚本定义需要使用单引号 ``'`` 分隔字符串。 当使用双引号 ``"`` 分隔字符串时，美元变量像往常一样被解释为Nextflow变量。请参阅 :ref:`string-interpolation` 。
     -感叹号前缀变量总是需要用大括号括起来，即 ``!{str}`` 是有效变量，而 ``!str`` 被忽略。
     -Shell脚本支持使用 :ref:`process-template` 文件机制。 相同的规则适用于脚本模板中定义的变量。

.. _process-native:

本地执行
----------------

Nextflow process可以执行除系统脚本之外的本机代码，如前面段落所示。

这意味着您可以通过提供一个或多个语言语句来定义它，而不是将process命令指定为字符串脚本，而不是像在流程脚本的其余部分中那样。 只需使用 ``exec:`` 关键字启动脚本定义块，例如::

    x = Channel.from( 'a', 'b', 'c')

    process simpleSum {
        input:
        val x

        exec:
        println "Hello Mr. $x"
    }

Will display::

    Hello Mr. b
    Hello Mr. a
    Hello Mr. c



.. _process-input:

Inputs
======

Nextflow process彼此隔离，但可以在它们之间通过channels发送值进行通信。


``input`` 块定义了process期望从哪些channels接收输入数据。 您一次只能定义一个输入块，并且必须包含一个或多个输入声明。

input块遵循以下语法::

    input:
      <input qualifier> <input name> [from <source channel>] [attributes]


input定义以输入限定符(input `qualifier`)和输入名称(input `name`)开头，后跟关键字 ``from`` 和接收输入的实际通道。 最后，可以指定一些输入可选属性。

.. note:: 当输入名称与channel名称相同时，可以省略声明的 ``from`` 部分。


输入限定符声明要接收的数据类型。 Nextflow使用此信息来应用与每个限定符关联的语义规则，并根据目标执行平台（grid, cloud, etc）正确处理它。

可用的限定符是下表中列出的限定符：

=========== =============
Qualifier   Semantic
=========== =============
val         Lets you access the received input value by its name in the process script.
env         Lets you use the received value to set an environment variable named
            as the specified input name.
file        Lets you handle the received value as a file, staging it properly in the execution context.
stdin       Lets you forward the received value to the process `stdin` special file.
set         Lets you handle a group of input values having one of the above qualifiers.
each        Lets you execute the process for each entry in the input collection.
=========== =============


输入通用值
-----------------------

``val`` 限定符允许您接收任何类型的数据作为输入。 可以使用指定的输入名称在进程脚本中访问它，如以下示例所示::

    num = Channel.from( 1, 2, 3 )

    process basicExample {
      input:
      val x from num

      "echo process job $x"

    }


在上面的例子中，每次从 channel ``num`` 接收一个值并用于处理脚本时，该过程执行三次。 因此，它产生类似于下图所示的输出::

    process job 3
    process job 1
    process job 2

.. note::  channel保证项目(items)的发送顺序与被发送顺序相同 - 但是 - 由于该过程以并行方式执行，因此无法保证按照接收顺序处理它们。 实际上，在上面的例子中，值 ``3`` 在其他之前被处理。


当 ``val`` 与接收数据的channel同名时，可以省略 ``from`` 部分。 因此，上面的例子可以写成如下所示::

    num = Channel.from( 1, 2, 3 )

    process basicExample {
      input:
      val num

      "echo process job $num"

    }


输入文件
--------------

``file`` 限定符允许在process执行上下文中处理文件。 这意味着Nextflow将在process执行目录中将其暂存，并且可以使用输入声明中指定的名称在脚本中进行访问。 例如::

    proteins = Channel.fromPath( '/some/path/*.fa' )

    process blastThemAll {
      input:
      file query_file from proteins

      "blastp -query ${query_file} -db nr"

    }


在上面的例子中，所有以后缀 ``.fa`` 结尾的文件都是通过channel ``proteins`` 发送的。 然后，process将接收这些文件，这些文件将对每个文件执行BLAST比对。


当文件输入名称与channel名称相同时，可以省略输入声明的 ``from`` 部分。 因此，上面的例子可以写成如下所示::

    proteins = Channel.fromPath( '/some/path/*.fa' )

    process blastThemAll {
      input:
      file proteins

      "blastp -query $proteins -db nr"

    }


值得注意的是，在上面的示例中，未触及文件系统中文件的名称，即使不知道其名称也可以访问该文件，因为您可以使用指定名称的变量在process脚本中引用它,在输入文件参数中声明。


在某些情况下，您的任务需要使用名称已修复的文件，而不必随实际提供的文件一起更改。 在这种情况下，您可以通过在输入文件参数声明中指定 ``name`` 属性来指定其名称，如以下示例所示::

    input:
        file query_file name 'query.fa' from proteins


或者使用更短的语法::

    input:
        file 'query.fa' from proteins


使用它，可以重写前面的示例，如下所示::

    proteins = Channel.fromPath( '/some/path/*.fa' )

    process blastThemAll {
      input:
      file 'query.fa' from proteins

      "blastp -query query.fa -db nr"

    }


在此示例中发生的情况是，process接收的每个文件在不同的执行上下文（即执行作业的文件夹）中使用名称 ``query.fa`` 进行暂存，并启动独立的process执行。

.. tip:: 这允许您在不必担心文件名更改的情况下执行各种时间的process命令。 换句话说，Nextflow可帮助您编写由执行环境自包含和解耦的流程任务。 这也是您应该尽可能避免使用引用流程processes中的文件的绝对路径或相对路径的原因。


.. TODO describe that file can handle channels containing any data type not only file


多个输入文件
--------------------

process可以将声明为值集合的channel声明为输入文件，而不是简单值。

在这种情况下，输入文件参数定义的脚本变量将包含文件列表。 您可以如前所示使用它，引用列表中的所有文件，或使用通常的方括号表示法访问特定条目。

当在输入参数中定义目标文件名并且process接收到文件集合时，文件名将附加一个数字后缀，表示其在列表中的序号位置。 例如::

    fasta = Channel.fromPath( "/some/path/*.fa" ).buffer(size:3)

    process blastThemAll {
        input:
        file 'seq' from fasta

        "echo seq*"

    }

Will output::

    seq1 seq2 seq3
    seq1 seq2 seq3
    ...


目标输入文件名可以包含 ``*`` 和 ``?`` 通配符，可用于控制暂存文件的名称。 下表显示了如何根据接收的输入集合的基数替换通配符。

============ ============== ==================================================
Cardinality   Name pattern     Staged file names
============ ============== ==================================================
 any         ``*``           named as the source file
 1           ``file*.ext``   ``file.ext``
 1           ``file?.ext``   ``file1.ext``
 1           ``file??.ext``  ``file01.ext``
 many        ``file*.ext``   ``file1.ext``, ``file2.ext``, ``file3.ext``, ..
 many        ``file?.ext``   ``file1.ext``, ``file2.ext``, ``file3.ext``, ..
 many        ``file??.ext``  ``file01.ext``, ``file02.ext``, ``file03.ext``, ..
 many        ``dir/*``       named as the source file, created in ``dir`` subdirectory
 many        ``dir??/*``     named as the source file, created in a progressively indexed subdirectory e.g. ``dir01/``, ``dir02/``, etc.
 many        ``dir*/*``      (as above)
============ ============== ==================================================


以下片段显示了如何在输入文件声明中使用通配符::


    fasta = Channel.fromPath( "/some/path/*.fa" ).buffer(size:3)

    process blastThemAll {
        input:
        file 'seq?.fa' from fasta

        "cat seq1.fa seq2.fa seq3.fa"

    }


.. note::  根据命名模式重写输入文件名是一项额外功能，完全没有义务。 “文件输入”部分中引入的普通文件输入结构也适用于多个文件的集合。 要处理保留原始文件名的多个输入文件，请使用 ``*`` 通配符作为名称模式或变量标识符。


动态输入文件名
------------------------

使用 ``name``  文件子句或短字符串表示法指定输入文件名时，可以使用其他输入值作为文件名字符串中的变量。 例如::


  process simpleCount {
    input:
    val x from species
    file "${x}.fa" from genomes

    """
    cat ${x}.fa | grep '>'
    """
  }


在上面的示例中，使用 ``x`` 输入值的当前值设置输入文件名。


这允许输入文件在脚本工作目录中暂存，其名称与当前执行上下文一致。

.. tip::  在大多数情况下，您不需要使用动态文件名，因为每个进程都在其自己的专用临时目录中执行，并且输入文件由Nextflow自动转移到此目录。 这可以保证具有相同名称的输入文件不会相互覆盖。


输入'stdin'类型
---------------------

``stdin`` 输入限定符允许您将从channel接收的值转发到process执行的命令的 `standard input <http://en.wikipedia.org/wiki/Standard_streams#Standard_input_.28stdin.29>`_ 。
例如::

    str = Channel.from('hello', 'hola', 'bonjour', 'ciao').map { it+'\n' }

    process printAll {
       input:
       stdin str

       """
       cat -
       """

    }

It will output::

    hola
    bonjour
    ciao
    hello


输入'env'类型
-------------------

``env`` 限定符允许您根据从channel接收的值在process执行上下文中定义环境变量。 例如::

    str = Channel.from('hello', 'hola', 'bonjour', 'ciao')

    process printEnv {

        input:
        env HELLO from str

        '''
        echo $HELLO world!
        '''

    }

::

    hello world!
    ciao world!
    bonjour world!
    hola world!


输入'set'类型
-------------------

``set`` 限定符允许您在单个参数定义中对多个参数进行分组。 当process在输入中接收需要单独处理的值的元组时，它会很有用。 元组中的每个元素都与具有 ``set`` 定义的对应元素相关联。 例如::

     tuple = Channel.from( [1, 'alpha'], [2, 'beta'], [3, 'delta'] )

     process setExample {
         input:
         set val(x), file('latin.txt')  from tuple

         """
         echo Processing $x
         cat - latin.txt > copy
         """

     }


在上面的示例中，``set`` 参数用于定义值 ``x`` 和文件 ``latin.txt``，后者将从同一channel中接收值。

In the ``set`` declaration items can be defined by using the following qualifiers: ``val``, ``env``, ``file`` and ``stdin``.
在 ``set`` 声明中，可以使用以下限定符来定义项：``val``, ``env``, ``file`` 和 ``stdin``

通过应用以下替换规则可以使用更短的表示法：

============== =======
long            short
============== =======
val(x)          x
file(x)         (not supported)
file('name')    'name'
file(x:'name')  x:'name'
stdin           '-'
env(x)          (not supported)
============== =======

因此，前面的示例可以重写如下::

      tuple = Channel.from( [1, 'alpha'], [2, 'beta'], [3, 'delta'] )

      process setExample {
          input:
          set x, 'latin.txt' from tuple

          """
          echo Processing $x
          cat - latin.txt > copy
          """

      }

文件名可以 *动态方式* 定义，如  `动态输入文件名`_  部分所述。


重复输入
---------------

``each`` 限定符允许您在每次收到新数据时为集合中的每个项重复执行过程。 例如::

  sequences = Channel.fromPath('*.fa')
  methods = ['regular', 'expresso', 'psicoffee']

  process alignSequences {
    input:
    file seq from sequences
    each mode from methods

    """
    t_coffee -in $seq -mode $mode > result
    """
  }


在上述示例中，每当过程接收到序列文件作为输入时，它执行三个任务，运行具有  ``mode`` 参数的不同值的T-coffee比对。 当您需要为给定的参数集重复相同的任务时，这非常有用。

从版本0.25+开始，重复输入也可以应用于文件。 例如::

    sequences = Channel.fromPath('*.fa')
    methods = ['regular', 'expresso']
    libraries = [ file('PQ001.lib'), file('PQ002.lib'), file('PQ003.lib') ]

    process alignSequences {
      input:
      file seq from sequences
      each mode from methods
      each file(lib) from libraries

      """
      t_coffee -in $seq -mode $mode -lib $lib > result
      """
    }


.. note::  当声明多个重复时，process将为它们的每个组合执行。


在后一示例中，对于 ``sequences`` channel发出的任何序列输入文件，执行6个比对，3个使用 ``regular`` 方法对每个库文件执行，而其他3个通过使用 ``expresso`` 方法。


.. hint::  如果需要在n元组元素上重复执行进程而不是简单的值或文件，请根据需要创建一个组合输入值的channel，以多次触发process执行。 在这方面，请参阅 :ref:`operator-combine`, :ref:`operator-cross` 和 :ref:`operator-phase` 运算符。


.. _process-understand-how-multiple-input-channels-work:

了解多个输入channels的工作原理
-------------------------------------------


processes的一个关键特性是能够处理来自多个channels的输入。


当两个或多个channels被声明为process输入时，process将停止，直到有完整的输入配置，即它从声明为输入的所有channels接收输入值。


当验证该条件时，它消耗来自各个channels的输入值，并产生任务执行，然后重复相同的逻辑，直到一个或多个channels不再有内容。


这意味着channel一个接一个地串行消耗，第一个空channel导致进程执行停止，即使其他channels中还有其他值。


例如::

  process foo {
    echo true
    input:
    val x from Channel.from(1,2)
    val y from Channel.from('a','b','c')
    script:
     """
     echo $x and $y
     """
  }


process ``foo`` 执行两次，因为第一个输入channel只提供两个值，因此丢弃了 ``c`` 元素。 它打印::

    1 and a
    2 and b


.. warning:: 在使用值通道(*Value channel*)时应用了另一种语义，即单例通道(*Singleton channel*)。

这种channel由 :ref:`Channel.value <channel-value>` factory方法创建，或者在process输入指定 ``from`` 子句中的简单值时隐式创建。

根据定义， *Value channel* 绑定到单个值，可以无限次读取而不消耗其内容。

这些属性使得在将value channel与一个或多个（队列）channels混合时，它不会影响仅仅依赖于其他channels并且其内容被重复应用的process终止。

为了更好地理解此行为，请将前一个示例与以下示例进行比较::

  process bar {
    echo true
    input:
    val x from Channel.value(1)
    val y from Channel.from('a','b','c')
    script:
     """
     echo $x and $y
     """
  }


上面的代码段执行 ``bar``  process三次，因为第一个输入是一个value channel，因此可以根据需要多次读取其内容。 process终止由第二channel的内容确定。它打印::


  1 and a
  1 and b
  1 and c

See also: :ref:`channel-types`.


输出
=======

``output`` 声明块允许定义process用于发送生成结果的channels。


它最多可以定义一个输出块，它可以包含一个或多个输出声明。 输出块遵循以下语法::

    output:
      <output qualifier> <output name> [into <target channel>[,channel,..]] [attribute [,..]]

输出定义以输出限定符和输出名称开头，后跟关键字 ``into`` 以及发送输出的一个或多个channels。 最后，可以指定一些可选属性。

.. note::  当输出名称与channel名称相同时，可以省略 ``into`` 声明的部分内容。

.. TODO the channel is implicitly create if does not exist

可以在输出声明块中使用的限定符是下表中列出的限定符：

=========== =============
Qualifier   Semantic
=========== =============
val         Sends variable's with the name specified over the output channel.
file        Sends a file produced by the process with the name specified over the output channel.
stdout      Sends the executed process `stdout` over the output channel.
set         Lets to send multiple values over the same output channel.
=========== =============


Output values
-------------

``val`` 限定符允许输出脚本上下文中定义的值。 在常见的使用场景中，这是一个已在输入声明块中定义的值，如以下示例所示::

   methods = ['prot','dna', 'rna']

   process foo {
     input:
     val x from methods

     output:
     val x into receiver

     """
     echo $x > file
     """

   }

   receiver.println { "Received: $it" }


有效输出值是文字，输入值标识符，进程范围和值表达式中可访问的变量。 例如::

    process foo {
      input:
      file fasta from 'dummy'

      output:
      val x into var_channel
      val 'BB11' into str_channel
      val "${fasta.baseName}.out" into exp_channel

      script:
      x = fasta.name
      """
      cat $x > file
      """
    }




Output files
------------

``file`` 限定符允许通过指定的channel输出由process生成的一个或多个文件。 例如::


    process randomNum {

       output:
       file 'result.txt' into numbers

       '''
       echo $RANDOM > result.txt
       '''

    }

    numbers.subscribe { println "Received: " + it.text }


在上面的示例中，该过程在执行时会创建一个名为 ``result.txt`` 的文件，其中包含一个随机数。 由于在输出之间声明了使用相同名称的文件参数，因此当任务完成时，该文件通过 ``numbers``  channel发送。 声明与输入相同的channel的下游process将能够接收它。

.. note::  如果先前未在pipeline脚本中声明指定为输出的channel，则它将由输出声明本身隐式创建。


.. TODO explain Path object

Multiple output files
---------------------

当输出文件名包含 ``*`` 或 ``?`` 时 通配符，它被解释为  `glob`_  路径匹配器。 这允许将多个文件捕获到列表对象中并将它们作为唯一的输出。 例如::

    process splitLetters {

        output:
        file 'chunk_*' into letters

        '''
        printf 'Hola' | split -b 1 - chunk_
        '''
    }

    letters
        .flatMap()
        .subscribe { println "File: ${it.name} => ${it.text}" }

It prints::

    File: chunk_aa => H
    File: chunk_ab => o
    File: chunk_ac => l
    File: chunk_ad => a

.. note::  在上面的示例中，运算符 :ref:`operator-flatmap` 用于将 ``letters`` channel发出的文件列表转换为独立发送每个文件对象的channel。


关于glob模式行为的一些警告：
* 输入文件不包含在可能的匹配列表中。
* Glob模式匹配文件和目录路径。
* 当两星模式 ``**`` 用于跨目录匹配时，只匹配文件路径，即目录不包括在结果列表中。

.. warning:: 尽管与结果输出channel匹配的输入文件未包含在结果输出channel中，但这些文件仍可从任务暂存目录传输到目标任务工作目录。 因此，为避免不必要的文件复制，建议在定义输出文件时避免使用松散的通配符，例如 ``file '*'`` 。 相反，使用前缀或后缀命名符号将匹配文件集限制为仅预期的匹配文件，例如 ``file 'prefix_*.sorted.ba'``。

.. tip::  默认情况下，所有与指定的glob模式匹配的文件都由channel作为唯一（列表）项发出。 通过在输出文件声明中添加 ``mode flatten`` 属性，还可以将每个文件作为唯一项输出。

通过使用mode属性，可以重写上一个示例，如下所示::

    process splitLetters {

        output:
        file 'chunk_*' into letters mode flatten

        '''
        printf 'Hola' | split -b 1 - chunk_
        '''
    }

    letters .subscribe { println "File: ${it.name} => ${it.text}" }



Read more about glob syntax at the following link `What is a glob?`_

.. _glob: http://docs.oracle.com/javase/tutorial/essential/io/fileOps.html#glob
.. _What is a glob?: http://docs.oracle.com/javase/tutorial/essential/io/fileOps.html#glob

.. _process-dynoutname:


动态输出文件名
-------------------------

当需要动态表示输出文件名时，可以使用动态计算字符串来定义它，该字符串引用在输入声明块或脚本全局上下文中定义的值。 例如::


  process align {
    input:
    val x from species
    file seq from sequences

    output:
    file "${x}.aln" into genomes

    """
    t_coffee -in $seq > ${x}.aln
    """
  }

在上面的示例中，每次执行该process时，都会生成一个比对文件，其名称取决于 ``x`` 输入的实际值。

.. tip:: 使用Nextflow时，输出文件的管理是一个非常常见的误解。使用其他工具时，通常需要将输出文件组织成某种目录结构或保证唯一的文件名方案，以便结果文件不会相互覆盖，并且可以由下游任务单独引用它们。

  使用Nextflow，在大多数情况下，您不需要处理命名输出文件，因为每个任务都在其自己唯一的临时目录中执行，因此由不同任务生成的文件永远不会相互覆盖。此外，元数据可以通过使用  :ref:`set output <process-set>` 限定符与输出相关联，而不是将它们包含在输出文件名中。

  总而言之，尽可能使用具有静态名称的输出文件而不是动态名称，因为它将导致更简单和更可移植的代码。


.. _process-stdout:

输出 'stdout' 特殊文件
----------------------------

``stdout`` 限定符允许捕获已执行进程的stdout输出，并通过输出参数声明中指定channel发送它。 例如::

    process echoSomething {
        output:
        stdout channel

        "echo Hello world!"

    }

    channel.subscribe { print "I say..  $it" }



.. _process-set:

输出'set'值
----------------------

``set`` 限定符允许将多个值发送到单个channel。 当您需要将同一process的多次执行结果组合在一起时，此功能非常有用，如以下示例所示::

    query = Channel.fromPath '*.fa'
    species = Channel.from 'human', 'cow', 'horse'

    process blast {

    input:
        val species
        file query

    output:
        set val(species), file('result') into blastOuts


    "blast -db nr -query $query" > result

    }


在上面的示例中，针对接收的每对 ``species``  和 ``query`` 执行BLAST任务。 当任务完成一个包含 ``species`` 值的新元组时， ``result`` 文件将被发送到 ``blastOuts``  channel。

set声明可以包含以下描述的以下限定符的任意组合： ``val``, ``file`` 和 ``stdout``.

.. tip:: 变量标识符被解释为值，而字符串文字默认被解释为文件，因此可以使用短符号重写上述输出集，如下所示。
    ::

       output:
           set species, 'result' into blastOuts



文件名可以动态方式定义，如  :ref:`process-dynoutname` 分所述。

可选输出
---------------

在大多数情况下，一个process会生成添加到输出channel的输出。 但是，有些情况下，process无法生成输出。 在这些情况下，可以将 ``optional true`` 添加到输出声明中，如果未创建声明的输出，则声明Nextflow不会使进程失败。

::

    output:
        file("output.txt") optional true into outChannel


在此示例中，通常期望该process生成  ``output.txt``  文件，但在文件合法丢失的情况下，该进程不会失败。  ``outChannel`` 仅由生成 ``output.txt`` 的processes填充。

When
====

``when`` 声明允许您定义必须在验证条件下才能执行该process。 这可以是任何计算布尔值的表达式。

根据各种输入和参数的状态启用/禁用process执行非常有用。 例如::


    process find {
      input:
      file proteins
      val type from dbtype

      when:
      proteins.name =~ /^BB11.*/ && type == 'nr'

      script:
      """
      blastp -query $proteins -db nr
      """

    }


.. _process-directives:

Directives
==========

使用directive声明块，您可以提供将影响当前process执行的可选设置。

它们必须在任何其他声明块（即 ``input``, ``output``, 等）之前输入到process body的顶部，并具有以下语法::

    name value [, value2 [,..]]

某些指令通常可用于所有processes，其他指令取决于当前定义的执行程序。

这些指令是：

* `afterScript`_
* `beforeScript`_
* `cache`_
* `cpus`_
* `conda`_
* `container`_
* `containerOptions`_
* `clusterOptions`_
* `disk`_
* `echo`_
* `errorStrategy`_
* `executor`_
* `ext`_
* `label`_
* `maxErrors`_
* `maxForks`_
* `maxRetries`_
* `memory`_
* `module`_
* `penv`_
* `pod`_
* `publishDir`_
* `queue`_
* `scratch`_
* `stageInMode`_
* `stageOutMode`_
* `storeDir`_
* `tag`_
* `time`_
* `validExitStatus`_

afterScript
-----------

``afterScript``  指令允许您在主进程运行后立即执行自定义（Bash）代码段。 这可能有助于清理您的临时区域。

beforeScript
------------

``beforeScript`` 指令允许您在运行主进程脚本之前执行自定义（Bash）代码段。 这可能对初始化基础集群环境或其他自定义初始化很有用。

For example::

    process foo {

      beforeScript 'source /cluster/bin/setup'

      """
      echo bar
      """

    }


cache
-----

``cache`` 指令允许您将进程结果存储到本地缓存。 启用高速缓存并使用 :ref:`resume <getstart-resume>` 选项启动pipeline时，执行该过程的任何后续尝试以及相同的输入将导致跳过流程执行，从而将存储的数据生成为实际结果。

缓存功能通过索引进程脚本和输入来生成唯一键。 该密钥用于明确识别process执行产生的输出。

默认情况下启用缓存，您可以通过将 ``cache`` 指令设置为 ``false`` 来禁用特定process。 例如::

  process noCacheThis {
    cache false

    script:
    <your command string here>
  }


``cache`` 指令可能的值如下表所示：

===================== =================
Value                 Description
===================== =================
``false``             Disable cache feature.
``true`` (default)    Enable caching. Cache keys are created indexing input files meta-data information (name, size and last update timestamp attributes).
``'deep'``            Enable caching. Cache keys are created indexing input files content.
``'lenient'``         Enable caching. Cache keys are created indexing input files path and size attributes (this policy provides a workaround for incorrect caching invalidation observed on shared file systems due to inconsistent files timestamps; requires version 0.32.x or later).
===================== =================


.. _process-conda:

conda
-----


``conda`` 指令允许使用 `Conda <https://conda.io>`_ 包管理器定义process依赖性。

Nextflow自动为 ``conda`` 指令中列出的给定包名称设置环境。 例如::

  process foo {
    conda 'bwa=0.7.15'

    '''
    your_command --here
    '''
  }


可以指定多个包，用空格分隔它们，例如: ``bwa=0.7.15 fastqc=0.11.5``. 可以使用通常的Conda表示法指定需要下载特定包的channel的名称，即在包之前添加频道名称，如此处所示 ``bioconda::bwa=0.7.15``.

``conda``  目录还允许指定Conda环境文件路径或现有环境目录的路径。 有关更多详细信息，请参阅  :ref:`conda-page` 。

.. _process-container:

container
---------

 ``container`` 指令允许您在 `Docker <http://docker.io>`_ 容器中执行process脚本。

它要求Docker守护程序在执行pipeline的机器中运行，即当通过网格执行器部署pipeline时使用本地执行程序或集群节点时的本地机器。

For example::

    process runThisInDocker {

      container 'dockerbox:tag'

      """
      <your holy script here>
      """

    }


只需在上面的脚本 ``dockerbox:tag`` 中替换您要使用的Docker镜像名称。

.. tip:: 这对于将脚本执行到可复制的自包含环境或在云中部署pipeline非常有用。

.. note:: 对于本机执行的processes  :ref:`executed natively <process-native>`，该指令被忽略。

.. _process-containerOptions:

containerOptions
----------------

``containerOptions`` 指令允许您指定底层容器引擎支持的任何容器执行选项（即Docker，Singularity等）。这对于仅为特定过程提供容器设置是有用的，例如 安装自定义路径::


  process runThisWithDocker {

      container 'busybox:latest'
      containerOptions '--volume /data/db:/db'

      output: file 'output.txt'

      '''
      your_command --data /db > output.txt
      '''
  }


.. warning::  :ref:`awsbatch-executor`  和  :ref:`k8s-executor` executors 不支持此功能。


.. _process-cpus:

cpus
----

``cpus`` 指令允许您定义process任务所需的（逻辑）CPU数量。 例如::

    process big_job {

      cpus 8
      executor 'sge'

      """
      blastp -query input_sequence -num_threads ${task.cpus}
      """
    }


执行多进程或多线程命令/工具的任务需要此指令，并且当通过集群资源管理器执行pipeline任务时，该指令用于保留足够的CPU。

See also: `penv`_, `memory`_, `time`_, `queue`_, `maxForks`_

.. _process-clusterOptions:

clusterOptions
--------------

``clusterOptions``  指令允许使用cluster submit命令接受的任何本机配置选项。 您可以使用它来请求非标准资源或使用特定于您的群集的设置，并且Nextflow不支持开箱即用。

.. note:: 仅当使用基于网格的执行程序(grid based executor)时才考虑此指令:
  :ref:`sge-executor`, :ref:`lsf-executor`, :ref:`slurm-executor`, :ref:`pbs-executor` and
  :ref:`condor-executor` executors.

.. _process-disk:


disk
----

``disk`` 指令允许您定义允许process使用多少本地磁盘存储。 例如::

    process big_job {

        disk '2 GB'
        executor 'cirrus'

        """
        your task script here
        """
    }


指定磁盘值时，可以使用以下内存单元后缀：

======= =============
Unit    Description
======= =============
B       Bytes
KB      Kilobytes
MB      Megabytes
GB      Gigabytes
TB      Terabytes
======= =============

.. note:: This directive currently is taken in account only by the :ref:`ignite-executor`
  and the :ref:`condor-executor` executors.

See also: `cpus`_, `memory`_ `time`_, `queue`_ and `Dynamic computing resources`_.

.. _process-echo:

echo
----

默认情况下，所有processes中执行的命令生成的stdout将被忽略。 将 ``echo``  指令设置为 ``true`` ，可以将process stdout转发到当前运行最高的process stdout文件，并在shell终端中显示它。

For example::

    process sayHello {
      echo true

      script:
      "echo Hello"
    }

::

    Hello


如果不指定  ``echo true`` ，则在执行上述示例时不会看到打印出的 ``Hello``  字符串。


.. _process-page-error-strategy:

errorStrategy
-------------

``errorStrategy`` 指令允许您定义进程管理错误条件的方式。 默认情况下，执行的脚本返回错误状态时，进程立即停止。 这反过来迫使整个pipeline终止。


可用错误策略表：

============== ==================
Name            Executor
============== ==================
``terminate``   Terminates the execution as soon as an error condition is reported. Pending jobs are killed (default)
``finish``      Initiates an orderly pipeline shutdown when an error condition is raised, waiting the completion of any submitted job.
``ignore``      Ignores processes execution errors.
``retry``       Re-submit for execution a process returning an error condition.
============== ==================


设置 ``errorStrategy``  指令为 ``ignore`` , 进程在错误情况下不会停止时，它只会报告一条消息，通知您错误事件。

For example::

    process ignoreAnyError {
       errorStrategy 'ignore'

       script:
       <your command string here>
    }

.. tip:: By definition a command script fails when it ends with a non-zero exit status. To change this behavior
  see `validExitStatus`_.


``retry`` 错误策略允许您重新提交执行过程，返回错误条件。 例如::

    process retryIfFail {
       errorStrategy 'retry'

       script:
       <your command string here>
    }


重新执行失败进程的次数由 `maxRetries`_ 和 `maxErrors`_  指令定义。

.. _process-executor:

executor
--------

``executor`` 定义执行进程的底层系统。 默认情况下，process使用 ``nextflow.config`` 文件中全局定义的执行程序。

``executor`` 指令允许您配置进程必须使用的执行程序，从而覆盖默认配置。 可以使用以下值：

============== ==================
Name            Executor
============== ==================
``local``      The process is executed in the computer where `Nextflow` is launched.
``sge``        The process is executed using the Sun Grid Engine / `Open Grid Engine <http://gridscheduler.sourceforge.net/>`_.
``uge``        The process is executed using the `Univa Grid Engine <https://en.wikipedia.org/wiki/Univa_Grid_Engine/>`_ job scheduler.
``lsf``        The process is executed using the `Platform LSF <http://en.wikipedia.org/wiki/Platform_LSF>`_ job scheduler.
``slurm``      The process is executed using the SLURM job scheduler.
``pbs``        The process is executed using the `PBS/Torque <http://en.wikipedia.org/wiki/Portable_Batch_System>`_ job scheduler.
``condor``     The process is executed using the `HTCondor <https://research.cs.wisc.edu/htcondor/>`_ job scheduler.
``nqsii``      The process is executed using the `NQSII <https://www.rz.uni-kiel.de/en/our-portfolio/hiperf/nec-linux-cluster>`_ job scheduler.
``ignite``     The process is executed using the `Apache Ignite <https://ignite.apache.org/>`_ cluster.
``k8s``        The process is executed using the `Kubernetes <https://kubernetes.io/>`_ cluster.
============== ==================

The following example shows how to set the process's executor::


   process doSomething {

      executor 'sge'

      script:
      <your script here>

   }


.. note:: 每个执行程序都提供自己的一组配置选项，可以在指令声明块中进行设置。 请参阅 :ref:`executor-page` 部分以了解特定执行程序指令。



.. _process-ext:

ext
---

``ext`` 是一个特殊指令，用作用户自定义流程指令的命名空间。 这对高级配置选项很有用。 例如::

    process mapping {
      container "biocontainers/star:${task.ext.version}"

      input:
      file genome from genome_file
      set sampleId, file(reads) from reads_ch

      """
      STAR --genomeDir $genome --readFilesIn $reads
      """
    }


在上面的示例中，该process使用一个容器，其版本由 ``ext.version`` 属性控制。 这可以在 ``nextflow.config`` 文件中定义，如下所示::

    process.ext.version = '2.5.3'



.. _process-maxErrors:

maxErrors
---------

``maxErrors``  指令允许您指定使用 ``retry`` 错误策略时process可能失败的最大次数。 默认情况下，此指令被禁用，您可以如下例所示进行设置::

    process retryIfFail {
      errorStrategy 'retry'
      maxErrors 5

      """
      echo 'do this as that .. '
      """
    }
    
.. note:: 此设置考虑了在所有实例中为给定进程累积的总错误。 如果要控制process实例（也称为task）失败的次数，请使用 ``maxRetries``。.

See also: `errorStrategy`_ and `maxRetries`_.

.. _process-maxForks:

maxForks
--------

``maxForks``  指令允许您定义可以并行执行的最大process实例数。 默认情况下，此值等于可用CPU核心数减1。

如果要以顺序方式执行process，请将此指令设置为1。 例如::

    process doNotParallelizeIt {

       maxForks 1

       '''
       <your script here>
       '''

    }

.. _process-maxRetries:

maxRetries
----------

``maxRetries`` 指令允许您定义在发生故障时可以重新提交process实例的最大次数。 仅在使用 ``retry`` 错误策略时才应用此值。 默认情况下，只允许重试一次，您可以增加此值，如下所示::

    process retryIfFail {
        errorStrategy 'retry'
        maxRetries 3

        """
        echo 'do this as that .. '
        """
    }


.. note:: ``maxRetries`` 和 ``maxErrors`` 指令之间存在微妙但重要的区别。 后者定义了process执行期间允许的错误总数（同一process可以启动不同的执行实例），而  ``maxRetries``  定义了在发生错误时可以重试相同process执行的最大次数。
	

See also: `errorStrategy`_ and `maxErrors`_.


.. _process-memory:

memory
------

``memory`` 指令允许您定义允许process使用多少内存。 例如::

    process big_job {

        memory '2 GB'
        executor 'sge'

        """
        your task script here
        """
    }


The following memory unit suffix can be used when specifying the memory value:

======= =============
Unit    Description
======= =============
B       Bytes
KB      Kilobytes
MB      Megabytes
GB      Gigabytes
TB      Terabytes
======= =============

.. This setting is equivalent to set the ``qsub -l virtual_free=<mem>`` command line option.

See also: `cpus`_, `time`_, `queue`_ and `Dynamic computing resources`_.


.. _process-module:

module
------

`Environment Modules <http://modules.sourceforge.net/>`_ 是一个包管理器，允许您动态配置执行环境，并在同一软件工具的多个版本之间轻松切换.

如果它在您的系统中可用，您可以将其与Nextflow一起使用，以便在pipeline中配置process执行环境。

在process定义中，您可以使用 ``module`` 指令加载要在process执行环境中使用的特定模块版本。 例如::

  process basicExample {

    module 'ncbi-blast/2.2.27'

    """
    blastp -query <etc..>
    """
  }

您可以为需要加载的每个模块重复  ``module`` 指令。 或者，可以在单个 ``module`` 指令中指定多个模块，方法是使用 ``:`` (冒号）字符分隔所有模块名称，如下所示::

   process manyModules {

     module 'ncbi-blast/2.2.27:t_coffee/10.0:clustalw/2.1'

     """
     blastp -query <etc..>
     """
  }


.. _process-penv:

penv
----

``penv`` 指令允许您定义在向 :ref:`SGE <sge-executor>` 资源管理器提交并行任务时要使用的并行环境。 例如::

    process big_job {

      cpus 4
      penv 'smp'
      executor 'sge'

      """
      blastp -query input_sequence -num_threads ${task.cpus}
      """
    }

此配置取决于Grid Engine安装提供的并行环境。 请参阅您的群集文档或与您的管理员联系以了解更多信息。

.. note:: This setting is available when using the :ref:`sge-executor` executor.

See also: `cpus`_, `memory`_, `time`_

.. _process-pod:

pod
---

``pod`` 指令允许在使用 :ref:`k8s-executor` executor 时定义pod特定设置，例如环境变量，机密和配置映射。

For example::

  process your_task {
    pod env: 'FOO', value: 'bar'

    '''
    echo $FOO
    '''
  }

The above snippet defines an environment variable named ``FOO`` which value is ``bar``.
上面的代码片段定义了一个名为 ``FOO``  的环境变量，其值为 ``bar``.

``pod``  指令允许定义以下选项：

================================================= =================================================
``label: <K>, value: <V>``                        Defines a pod label with key ``K`` and value ``V``.
``env: <E>, value: <V>``                          Defines an environment variable with name ``E`` and whose value is given by the ``V`` string.
``env: <E>, config: <C/K>``                       Defines an environment variable with name ``E`` and whose value is given by the entry associated to the key with name ``K`` in the `ConfigMap <https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/>`_ with name ``C``.
``env: <E>, secret: <S/K>``                       Defines an environment variable with name ``E`` and whose value is given by the entry associated to the key with name ``K`` in the `Secret <https://kubernetes.io/docs/concepts/configuration/secret/>`_ with name ``S``.
``config: <C/K>, mountPath: </absolute/path>``    The content of the `ConfigMap <https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/>`_ with name ``C`` with key ``K`` is made available to the path ``/absolute/path``. When the key component is omitted the path is interpreted as a directory and all the `ConfigMap` entries are exposed in that path.
``secret: <S/K>, mountPath: </absolute/path>``    The content of the `Secret <https://kubernetes.io/docs/concepts/configuration/secret/>`_ with name ``S`` with key ``K`` is made available to the path ``/absolute/path``. When the key component is omitted the path is interpreted as a directory and all the `Secret` entries are exposed in that path.
``volumeClaim: <V>, mountPath: </absolute/path>`` Mounts a `Persistent volume claim <https://kubernetes.io/docs/concepts/storage/persistent-volumes/>`_ with name ``V`` to the specified path location. Use the optional `subPath` parameter to mount a directory inside the referenced volume instead of its root.
``imagePullPolicy: <V>``                          Specifies the strategy to be used to pull the container image e.g. ``imagePullPolicy: 'Always'``.
``imagePullSecret: <V>``                          Specifies the secret name to access a private container image registry. See `Kubernetes documentation <https://kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod>`_ for details.
``runAsUser: <UID>``                              Specifies the user ID to be used to run the container.
================================================= =================================================


在Nextflow配置文件中定义时，可以使用规范关联数组语法定义pod设置。 例如::

  process {
    pod = [env: 'FOO', value: 'bar']
  }

当需要提供多个设置时，必须将它们包含在列表定义中，如下所示::

  process {
    pod = [ [env: 'FOO', value: 'bar'], [secret: 'my-secret/key1', mountPath: '/etc/file.txt'] ]
  }


.. _process-publishDir:

publishDir
----------

``publishDir`` 指令允许您将process输出文件输出到指定的文件夹。 例如::

    process foo {

        publishDir '/data/chunks'

        output:
        file 'chunk_*' into letters

        '''
        printf 'Hola' | split -b 1 - chunk_
        '''
    }


上面的示例将字符串 ``Hola`` 拆分为单个字节的文件块。 完成后， ``chunk_*`` 输出文件到 ``/data/chunks`` 文件夹中。

.. tip::  可以多次指定  ``publishDir`` 指令将输出文件发布到不同的目标目录。 此功能需要0.29.0或更高版本。

默认情况下，文件将发布到目标文件夹，为每个process输出创建一个符号链接，该链接将生成的文件链接到进程工作目录。 可以使用``mode`` 参数修改此行为。

可与 ``publishDir`` 指令一起使用的可选参数表:

=============== =================
Name            Description
=============== =================
mode            The file publishing method. See the following table for possible values.
overwrite       When ``true`` any existing file in the specified folder will be overridden (default: ``true`` during normal
                pipeline execution and ``false`` when pipeline execution is `resumed`).
pattern         Specifies a `glob`_ file pattern that selects which files to publish from the overall set of output files.
path            Specifies the directory where files need to be published. **Note**: the syntax ``publishDir '/some/dir'`` is a shortcut for ``publishDir path: '/some/dir'``.
saveAs          A closure which, given the name of the file being published, returns the actual file name or a full path where the file is required to be stored.
                This can be used to rename or change the destination directory of the published files dynamically by using
                a custom strategy.
                Return the value ``null`` from the closure to *not* publish a file.
                This is useful when the process has multiple output files, but you want to publish only some of them.
=============== =================

Table of publish modes:

=============== =================
 Mode           Description
=============== =================
symlink         Creates an absolute `symbolic link` in the published directory for each process output file (default).
rellink         Creates a relative `symbolic link` in the published directory for each process output file.
link            Creates a `hard link` in the published directory for each process output file.
copy            Copies the output files into the published directory.
copyNoFollow    Copies the output files into the published directory without following symlinks ie. copies the links themselves. 
move            Moves the output files into the published directory. **Note**: this is only supposed to be used for a `terminating` process i.e. a process whose output is not consumed by any other downstream process.
=============== =================

.. note:: The `mode` value needs to be specified as a string literal i.e. enclosed by quote characters. Multiple parameters
  need to be separated by a colon character. For example:

::

    process foo {

        publishDir '/data/chunks', mode: 'copy', overwrite: false

        output:
        file 'chunk_*' into letters

        '''
        printf 'Hola' | split -b 1 - chunk_
        '''
    }


.. warning:: Files are copied into the specified directory in an *asynchronous* manner, thus they may not be immediately
  available in the published directory at the end of the process execution. For this reason files published by a process
  must not be accessed by other downstream processes.


.. _process-queue:

queue
-----

``queue`` 目录允许您设置在pipeline中使用基于网格的执行程序时计划作业的队列。 例如::

    process grid_job {

        queue 'long'
        executor 'sge'

        """
        your task script here
        """
    }


可以通过用逗号分隔它们的名称来指定多个队列，例如::

    process grid_job {

        queue 'short,long,cn-el6'
        executor 'sge'

        """
        your task script here
        """
    }


.. note:: This directive is taken in account only by the following executors: :ref:`sge-executor`, :ref:`lsf-executor`,
  :ref:`slurm-executor` and :ref:`pbs-executor` executors.


.. _process-label:

label
-----

``label`` 指令允许使用您选择的助记符标识符对process进行注释。 例如::

  process bigTask {

    label 'big_mem'

    '''
    <task script>
    '''
  }


相同的标签可以应用于多个process，并且可以使用 ``label`` 指令多次将多个标签应用于同一process。

.. note:: A label must consist of alphanumeric characters or ``_``, must start with an alphabetic character
  and must end with an alphanumeric character.


标签对于在单独的组中组织工作process过程非常有用，可以在配置文件中引用它们来选择和配置具有类似计算要求的过程子集。

See the :ref:`config-process-selectors` documentation for details.


.. _process-scratch:

scratch
-------

``scratch`` 指令允许您在执行节点本地的临时文件夹中执行该process。

当您使用网格执行程序启动pipeline时，这非常有用，因为它允许通过在实际执行节点的本地磁盘中的临时目录中运行pipeline进程来减少NFS开销。 只有在process定义中声明为输出的文件才会被复制到pipeline工作区中。

在其基本形式中，只需在指令值中指定 ``true``，如下所示::

  process simpleTask {

    scratch true

    output:
    file 'data_out'

    '''
    <task script>
    '''
  }


通过这样做，它尝试在执行节点中的变量 ``$TMPDIR`` 定义的目录中执行脚本。 如果此变量不存在，它将使用Linux命令 ``mktemp`` 创建一个新的临时目录。

可以通过简单地将其用作临时值来指定除 ``$TMPDIR`` 之外的自定义环境变量，例如::

  scratch '$MY_GRID_TMP'


注意，它必须用单引号字符包装，否则将在pipeline脚本上下文中评估变量。


您还可以将特定文件夹路径提供为临时值，例如::

  scratch '/tmp/my/path'


通过执行此操作，每次执行process时，将在指定的路径中创建新的临时目录。

最后，当 ``ram-disk`` 字符串作为 ``scratch`` 值提供时，该process将在节点RAM虚拟磁盘中执行。

Summary of allowed values:

=========== ==================
scratch     Description
=========== ==================
false       Do not use the scratch folder.
true        Creates a scratch folder in the directory defined by the ``$TMPDIR`` variable; fallback to ``mktemp /tmp`` if that variable do not exists.
$YOUR_VAR   Creates a scratch folder in the directory defined by the ``$YOUR_VAR`` environment variable; fallback to ``mktemp /tmp`` if that variable do not exists.
/my/tmp     Creates a scratch folder in the specified directory.
ram-disk    Creates a scratch folder in the RAM disk ``/dev/shm/`` (experimental).
=========== ==================

.. _process-storeDir:

storeDir
--------

``storeDir`` 指令允许您定义用作process结果的永久缓存的目录。

更详细地说，它以两种主要方式影响process执行:

#. 只有在 ``storeDir``  指令指定的目录中不存在output子句中声明的文件时，才会执行该过程。 当文件存在时，将跳过流程执行，并将这些文件用作实际流程结果。

#. 每当process成功完成时，输出声明块中列出的文件将被移动到 ``storeDir`` 指令指定的目录中.

以下示例显示如何使用 ``storeDir`` 指令为输入参数指定的每个物种创建包含BLAST数据库的目录::

  genomes = Channel.fromPath(params.genomes)

  process formatBlastDatabases {

    storeDir '/db/genomes'

    input:
    file species from genomes

    output:
    file "${dbName}.*" into blastDb

    script:
    dbName = species.baseName
    """
    makeblastdb -dbtype nucl -in ${species} -out ${dbName}
    """

  }


.. warning:: ``storeDir`` 指令用于长期process缓存，不应用于将process生成的文件输出到特定文件夹或组织语义目录结构中的结果数据。 在这些情况下，您可以使用 `publishDir`_ 指令。


.. note:: The use of AWS S3 path is supported however it requires the installation of the `AWS CLI tool <https://aws.amazon.com/cli/>`_
  (ie. ``aws``) in the target computing node.

.. _process-stageInMode:

stageInMode
-----------

``stageInMode`` 指令定义输入文件如何进入process工作目录。 允许以下值：

======= ==================
Value   Description
======= ==================
copy    Input files are staged in the process work directory by creating a copy.
link    Input files are staged in the process work directory by creating an (hard) link for each of them.
symlink Input files are staged in the process work directory by creating a symbolic link with an absolute path for each of them (default).
rellink Input files are staged in the process work directory by creating a symbolic link with a relative path for each of them.
======= ==================


.. _process-stageOutMode:

stageOutMode
------------

``stageOutMode`` 指令定义输出文件如何从暂存目录转移到process工作目录。 允许以下值：

======= ==================
Value   Description
======= ==================
copy    Output files are copied from the scratch directory to the work directory.
move    Output files are moved from the scratch directory to the work directory.
rsync   Output files are copied from the scratch directory to the work directory by using the ``rsync`` utility.
======= ==================

See also: `scratch`_.


.. _process-tag:

tag
---

``tag`` 指令允许您将每个process执行与自定义标签相关联，以便在日志文件或跟踪执行报告中更容易识别它们。 例如::

    process foo {
      tag "$code"

      input:
      val code from 'alpha', 'gamma', 'omega'

      """
      echo $code
      """
    }

上面的代码段将打印一个类似于以下内容的日志，其中process名称包含标记值::

    [6e/28919b] Submitted process > foo (alpha)
    [d2/1c6175] Submitted process > foo (gamma)
    [1c/3ef220] Submitted process > foo (omega)


See also :ref:`Trace execution report <trace-report>`


.. _process-time:

time
----

``time`` 指令允许您定义允许process运行的时间。 例如::

    process big_job {

        time '1h'

        """
        your task script here
        """
    }


指定持续时间值时，可以使用以下时间单位后缀：

======= =============
Unit    Description
======= =============
s       Seconds
m       Minutes
h       Hours
d       Days
======= =============

.. note:: This directive is taken in account only when using one of the following grid based executors:
  :ref:`sge-executor`, :ref:`lsf-executor`, :ref:`slurm-executor`, :ref:`pbs-executor`,
  :ref:`condor-executor` and :ref:`awsbatch-executor` executors.

See also: `cpus`_, `memory`_, `queue`_ and `Dynamic computing resources`_.


.. _process-validExitStatus:

validExitStatus
---------------

当执行的命令返回错误退出状态时，process终止。 默认情况下，除 ``0``  以外的任何错误状态都将被解释为错误条件。

``validExitStatus`` 指令允许您精确控制哪个错误状态将代表成功执行命令。 您可以指定单个值或多个值，如以下示例所示::


    process returnOk {
        validExitStatus 0,1,2

         script:
         """
         echo Hello
         exit 1
         """
    }



在上面的示例中，尽管命令脚本以 ``1`` 退出状态结束，但该process不会返回错误条件，因为值 ``1``  在 ``validExitStatus`` 指令中被声明为有效状态。


Dynamic directives
------------------


可以在process执行期间动态分配指令，以便可以根据一个或多个process输入值来评估其实际值。

为了以动态方式定义，需要使用 :ref:`closure <script-closure>` 语句表达指令的值，如下例所示::

    process foo {

      executor 'sge'
      queue { entries > 100 ? 'long' : 'short' }

      input:
      set entries, file(x) from data

      script:
      """
      < your job here >
      """
    }

在上面的示例中， `queue`_ 指令是动态计算的，具体取决于输入值 ``entries`` 。 当它大于100时，作业将被提交到 ``long`` 队列中，否则将使用 ``short`` 队列作业。


可以将所有指令分配给动态值，但以下情况除外：

* `executor`_
* `maxForks`_


.. note::  您可以使用包含当前process实例中定义的指令值的隐式变量 ``task`` 来检索process脚本中动态指令的当前值。

For example::


   process foo {

      queue { entries > 100 ? 'long' : 'short' }

      input:
      set entries, file(x) from data

      script:
      """
      echo Current queue: ${task.queue}
      """
    }


Dynamic computing resources
---------------------------

这是一种非常常见的情况，即同一process的不同实例在计算资源方面可能有非常不同的需求。 例如，在这种情况下请求一定量的内存太低会导致某些任务失败。 相反，使用适合执行中所有任务的更高限制可能会显着降低作业的执行优先级。

`Dynamic directives`_  评估功能可用于修改在process失败的情况下请求的计算资源量，并尝试使用更高的限制重新执行它。 例如::


    process foo {

        memory { 2.GB * task.attempt }
        time { 1.hour * task.attempt }

        errorStrategy { task.exitStatus == 140 ? 'retry' : 'terminate' }
        maxRetries 3

        script:
        <your job here>

    }


在上面的示例中，动态定义了 `memory`_ 和执行 `time`_ 限制。 第一次执行该process时， ``task.attempt`` 设置为 ``1``，因此它将请求2GB的内存和1小时的最长执行时间。

如果任务执行失败，报告退出状态等于 ``140``，则重新提交任务（否则立即终止）。 这次 ``task.attempt`` 的值为 ``2``，从而将内存量增加到4GB，时间增加到2小时，依此类推。

指令 `maxRetries` 设置可以重新执行同一任务的最大时间。
