.. _getstart-page:

开始
=========

.. _getstart-requirement:

所需环境
----------

`Nextflow` can be used on any POSIX compatible system (Linux, OS X, etc).
It requires Bash 3.2 (or later) and `Java 8 (or later) <http://www.oracle.com/technetwork/java/javase/downloads/index.html>`_ to be installed.

Windows systems may be supported using a POSIX compatibility layer like `Cygwin <http://www.cygwin.com>`_ (unverified) or,
alternatively, installing it into a Linux VM using virtualization software like `VirtualBox <http://www.virtualbox.org>`_
or `VMware <http://www.vmware.com/>`_.

.. _getstart-install:

安装
--------

Nextflow是一个独立的可执行包, 这意味着它不需要特殊的安装过程.

它仅仅简单的两步:

1. 下载可执行包. 复制粘贴后面的命令到你的终端窗口中:``wget -qO- https://get.nextflow.io | bash``. 它将会在当前目录创建一个可执行文件 ``nextflow``.

2. (此步可选)将 ``nextflow`` 文件转移到 ``$PATH`` 变量可以找到的目录中.(这仅仅是为了避免你每次运行它的时候都需要记住或输入 ``nextflow`` 的绝对路径).

.. tip::
   在你没有安装 ``wget`` 的情况下, 你可以使用 ``curl`` 来代替, 在命令行中输入后面的命令: ``curl -s https://get.nextflow.io | bash``.


.. _getstart-first:

第一个脚本
-----------

将下面的例子复制到你习惯的文本编辑器中, 然后保存到名字为 ``tutorial.nf`` 的文件中.

 ::
 
  #!/usr/bin/env nextflow
  
  params.str = 'Hello world!'
  
  process splitLetters {
  
      output:
      file 'chunk_*' into letters mode flatten
  
      """
      printf '${params.str}' | split -b 6 - chunk_
      """
  }
  
  
  process convertToUpper {
  
      input:
      file x from letters
  
      output:
      stdout result
  
      """
      cat $x | tr '[a-z]' '[A-Z]'
      """
  }
  
  result.subscribe {
      println it.trim()
  }
  

这个脚本定义了2个processes.  


在终端中输入下面的命令来执行这个脚本::

  nextflow run tutorial.nf


它将输出一些东西,类似下面的文本::

   N E X T F L O W  ~  version 19.01.0
   Launching `tutorial.nf` [prickly_goldwasser] - revision: 361b274147
   [warm up] executor > local
   [4b/65537c] Submitted process > splitLetters
   [7d/c27fbe] Submitted process > convertToUpper (1)
   [62/9488a7] Submitted process > convertToUpper (2)
   WORLD!
   HELLO
   

可以看到,第一个process执行了一次,而第二个执行了两次. 最后, 结果字符串被打印出来.

It's worth noting that the process ``convertToUpper`` is executed in parallel, so there's no guarantee that the instance
processing the first split (the chunk `Hello`) will be executed before before the one processing the second split (the chunk `world!`).

Thus, it is perfectly possible that you will get the final result printed out in a different order::

    WORLD!
    HELLO

.. tip:: The hexadecimal numbers, like ``22/7548fa``, identify the unique process execution. These numbers are
  also the prefix of the directories where each process is executed. You can inspect the files produced by them
  changing to the directory ``$PWD/work`` and using these numbers to find the process-specific execution path.

.. _getstart-resume:




Modify and resume
^^^^^^^^^^^^^^^^^^^

Nextflow keeps track of all the processes executed in your pipeline. If you modify some parts of your script,
only the processes that are actually changed will be re-executed. The execution of the processes that are not changed
will be skipped and the cached result used instead.

This helps a lot when testing or modifying part of your pipeline without having to re-execute it from scratch.

For the sake of this tutorial, modify the ``convertToUpper`` process in the previous example, replacing the
process script with the string ``rev $x``, so that the process looks like this::

    process convertToUpper {

        input:
        file x from letters

        output:
        stdout result

        """
        rev $x
        """
    }

Then save the file with the same name, and execute it by adding the ``-resume`` option to the command line::

    nextflow run tutorial.nf -resume


It will print output similar to this::

    N E X T F L O W  ~  version 18.10.1
    [warm up] executor > local
    [22/7548fa] Cached process > splitLetters (1)
    [d0/7b79a3] Submitted process > convertToUpper (1)
    [b0/c99ef9] Submitted process > convertToUpper (2)
    olleH
    !dlrow


You will see that the execution of the process ``splitLetters`` is actually skipped (the process ID is the same), and
its results are retrieved from the cache. The second process is executed as expected, printing the reversed strings.


.. tip:: The pipeline results are cached by default in the directory ``$PWD/work``. Depending on your script, this folder
  can take of lot of disk space. If your are sure you won't resume your pipeline execution, clean this folder periodically.

.. _getstart-params:



Pipeline parameters
^^^^^^^^^^^^^^^^^^^^


Pipeline parameters are simply declared by prepending to a variable name the prefix ``params``, separated by dot character.
Their value can be specified on the command line by prefixing the parameter name with a double `dash` character, i.e. ``--paramName``

For the sake of this tutorial, you can try to execute the previous example specifying a different input
string parameter, as shown below::

  nextflow run tutorial.nf --str 'Hola mundo'


The string specified on the command line will override the default value of the parameter. The output
will look like this::

    N E X T F L O W  ~  version 18.10.1
    [warm up] executor > local
    [6d/54ab39] Submitted process > splitLetters (1)
    [a1/88716d] Submitted process > convertToUpper (2)
    [7d/3561b6] Submitted process > convertToUpper (1)
    odnu
    m aloH




