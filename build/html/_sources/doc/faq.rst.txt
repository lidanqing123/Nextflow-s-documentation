.. _faq-page:

***
FAQ
***

如何并行处理多个输入文件?
--------------------------------------------------

Q: *我有一组输入文件(e.g. carrots.fa, onions.fa, broccoli.fa). 如何指定每个输入文件以并行方式执行流程?*

A:  这里的想法是创建一个 ``channel`` ，它将触发每个文件的进程执行。首先定义一个参数，指定输入文件的位置:

::

    params.input = "data/*.fa"

数据目录中的每一个文件都可以被做成一个通道:

::

    vegetable_datasets = Channel.fromPath(params.input)

从这里开始，每当将变量 ``vegetable_dataset`` 作为process的输入调用时，该process将对vegetable datasets中的每个文件执行。例如，每个输入文件可能包含一组未对齐的序列。我们可以指定一个process来对它们进行对齐, 如下:

::

    process clustalw2_align {
        input:
        file vegetable_fasta from vegetable_datasets

        output:
        file "${vegetable_fasta.baseName}.aln" into vegetable_alns

        script:
        """
        clustalw2 -INFILE=${vegetable_fasta}
        """
    }

这里将生成三个vegetable fasta files的对齐结果文件 ``carrots.aln``, ``onions.aln`` 和 ``broccoli.aln``.

这些对齐的文件现在位于channel ``vegetable_alns`` 中，可以用作进一步process的输入。


如何根据文件名获得唯一的ID ?
------------------------------------------------

*Q: 如何获得基于数据集文件名(e.g. broccoli from broccoli.fa)的唯一标识符，并将结果保存到特定文件夹(e.g. results/broccoli/)?*

A: 首先，我们可以指定一个结果目录如下所示:

::

    results_path = $PWD/results

最好的实现方法是让 ``channel`` 发出一个元组，其中包含文件名( ``broccoli`` )和完整的文件路径( ``data/broccoli.fa`` ):

::

    datasets = Channel
                    .fromPath(params.input)
                    .map { file -> tuple(file.baseName, file) }

在这个process中我们可以引用这些变量( ``datasetID`` 和 ``datasetFile`` ):

::

    process clustalw2_align {
        publishDir "$results_path/$datasetID"

        input:
        set datasetID, file(datasetFile) from datasets

        output:
        set datasetID, file("${datasetID}.aln") into aligned_files

        script:
        """
        clustalw2 -INFILE=${datasetFile} -OUTFILE=${datasetID}.aln
        """
    }

In our example above would now have the folder ``broccoli`` in the results directory which would
contain the file ``broccoli.aln``.

If the input file has multiple extensions (e.g. ``brocolli.tar.gz``), you will want to use
``file.simpleName`` instead, to strip all of them (available since Nextflow 0.25+).


如何多次使用同一 ``channel`` ?
---------------------------------------------

*Q: 一个channel可以在两个输入语句中使用吗? 例如, 我想同时使用ClustalW and T-Coffee 来对carrots.fa做比对.*


A: 一个channel只能由一个process或operator使用(除非channel只包含一个项)。在将channel调用为不同processes的输入之前，必须复制该channel。

首先创建输入文件的channel:

::

    vegetable_datasets = Channel.fromPath(params.input)

接下来，我们可以使用 :ref:`operator-into` 操作符将其分成两个channels:

::

    vegetable_datasets.into { datasets_clustalw; datasets_tcoffee }

然后我们可以定义一个用 *ClustalW* 对齐数据集的process:

::

    process clustalw2_align {
        input:
        file vegetable_fasta from datasets_clustalw
        
        output:
        file "${vegetable_fasta.baseName}.aln" into clustalw_alns

        script:
        """
        clustalw2 -INFILE=${vegetable_fasta}
        """
    }

和一个用 *T-Coffee* 对齐数据集的process:

::

    process tcoffee_align {
        input:
        file vegetable_fasta from datasets_tcoffee
        
        output:
        file "${vegetable_fasta.baseName}.aln" into tcoffee_alns

        script:
        """
        t_coffee ${vegetable_fasta}
        """
    }

分割通道的好处是，给定我们的三个未对齐的fasta文件(``broccoli.fa``, ``onion.fa`` and ``carrots.fa``)，六个对齐processes (three x ClustalW) + (three x T-Coffee) 将作为并行进程执行。


如何调用自定义脚本和工具?
-----------------------------------------

*Q: 我的代码中有可执行程序，我应该如何在Nextflow中调用它们?*

A: Nextflow将自动将目录 ``bin`` 添加到 ``PATH`` 环境变量中。因此，Nextflow pipeline可以调用 ``bin`` 文件夹中的任何可执行文件，而不需要引用完整的路径。


例如，我们可能希望将问题3中的 *ClustalW* 对齐结果重新转换为 *PHYLIP* 格式。我们将使用工具 ``esl-reformat`` 来完成这项任务。

首先，我们将esl-reformat可执行文件复制(或创建符号链接)到项目的bin文件夹。从上面我们可以看到ClustalW对齐结果在channel ``clustalw_alns`` 中:

::

    process phylip_reformat {
        input:
        file clustalw_alignment from clustalw_alns
        
        output:
        file "${clustalw_alignment.baseName}.phy" to clustalw_phylips

        script:
        """
        esl-reformat phylip ${clustalw_alignment} ${clustalw_alignment.baseName}.phy
        """
    }


    process generate_bootstrap_replicates {
        input:
        file clustalw_phylip from clustalw_phylips
        output:
        file "${clustalw_alignment.baseName}.phy" to clustalw_phylips

        script:
        """
        esl-reformat phylip ${clustalw_alignment} ${clustalw_alignment.baseName}.phy
        """
    }

我如何在一个进程上迭代n次?
-----------------------------------------

要执行一个进程n次，我们可以指定input为 ``each x from y..z`` ,例如:

::

    bootstrapReplicates=100

    process bootstrapReplicateTrees {
        publishDir "$results_path/$datasetID/bootstrapsReplicateTrees"

        input:
        each x from 1..bootstrapReplicates
        set val(datasetID), file(ClustalwPhylips)

        output:
        file "bootstrapTree_${x}.nwk" into bootstrapReplicateTrees

        script:
        // Generate Bootstrap Trees

        """
        raxmlHPC -m PROTGAMMAJTT -n tmpPhylip${x} -s tmpPhylip${x}
        mv "RAxML_bestTree.tmpPhylip${x}" bootstrapTree_${x}.nwk
        """
    }


如何从进程中迭代n个文件?
------------------------------------------------------

*Q:  例如，我有100个文件, 都由一个channel发出。我希望执行一个process，在这个process中迭代每个文件。*

A:  这里的想法是将一个发送多个项(items)的channel转换为另一个channel，该通道将收集所有文件到一个列表对象中，并将该列表作为一个单独的发送(emission)。我们使用 ``collect()`` 操作符进行此操作。然后，process脚本可以使用简单的for循环遍历文件。

如果通道的所有项都需要位于work目录中，那么这也很有用。

::

    process concatenateBootstrapReplicates {
        publishDir "$results_path/$datasetID/concatenate"

        input:
        file bootstrapTreeList from bootstrapReplicateTrees.collect()

        output:
        file "concatenatedBootstrapTrees.nwk"

        // Concatenate Bootstrap Trees
        script:
        """
        for every treeFile in ${bootstrapTreeList}
        do
            cat \$treeFile >> concatenatedBootstrapTrees.nwk
        done

        """
    }

如何使用Nextflow的特定版本?
------------------------------------------------------

*Q:  我需要指定要使用的Nextflow版本，或者我需要拉出一个快照版本。*

A: 有时，为了特定的特性或测试目的，需要使用不同版本的Nextflow。当NXF_VER环境变量被定义在commandline上时，Nextflow可以自动进行转换。

::

    NXF_VER=0.28.0 nextflow run main.nf
