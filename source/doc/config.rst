.. _config-page:

*************
配置
*************

配置文件
==================

当启动pipeline脚本时，Nextflow在当前目录和脚本根目录(如果与当前目录不同)中查找名为 ``nextflow.config`` 的文件。最后,检查文件 ``$HOME/.nextflow/config`` 。

在上面的文件中，有多于一个的存在，就会被合并，所以第一个文件中的的设置会覆盖可能出现在第二个文件中的设置，以此类推。

默认配置文件搜索机制可以通过使用命令行选项 ``-c <config file>`` 来扩展额外的配置文件。

.. note::  值得注意的是，通过这样做，文件 ``nextflow.config`` 和 ``$HOME/.nextflow/config`` 不会被忽略，而是像上面解释的那样合并。

.. tip::  如果您想忽略任何默认配置文件，只使用自定义配置文件，请使用命令行选项  ``-C <config file>``.

配置语法
--------------

Nextflow配置文件是一个简单的文本文件，包含一组使用以下语法定义的属性::

  name = value

请注意，字符串值需要用引号括起来，而数字和布尔值( ``true`` 、 ``false`` )则不需要。还请注意，值是类型化的，这意味着，例如， ``1`` 不同于 ``'1'`` ，因为第一个值被解释为数字1，而后者被解释为字符串值。


配置变量
----------------

通过使用通常的 ``$propertyName`` 或 ``${expression}`` 语法，可以将配置属性用作配置文件本身中的变量.

For example::

     propertyOne = 'world'
     anotherProp = "Hello $propertyOne"
     customPath = "$PATH:/my/app/folder"

请注意, :ref:`string-interpolation` 的一般规则是适用的，因此包含变量引用的字符串必须用双引号括起来，而不是单引号。

同样的机制可以让你访问在主机系统中定义的环境变量。在Nextflow配置文件中没有定义名称的一些变量都可能是对具有该名称的环境变量的引用。因此，在上面的示例中，属性 ``customPath`` 被定义为字符串 ``/my/app/folder`` 追加到当前系统的 ``PATH`` 上 。

配置的注释
------------------

配置文件和Groovy或Java编程语言使用的注释有相同的约定。因此，使用 ``//`` 注释一行或 ``/*`` .. ``*/`` 在多行注释一个块。


配置文件导入
--------------

一个配置文件可以使用关键字 ``includeConfig`` 导入一个或多个配置文件。例如::

    process.executor = 'sge'
    process.queue = 'long'
    process.memory = '10G'

    includeConfig 'path/foo.config'

当使用相对路径时，它根据包含文件的实际位置进行解析。


配置文件作用域
===============

配置设置可以在不同的作用域中组织，方法是用点( ``.`` )连接作用域标识符前缀和属性名，或者使用花括号符号将相同作用域中的属性分组。如下例所示::

   alpha.x  = 1
   alpha.y  = 'string value..'

   beta {
        p = 2
        q = 'another string ..'
   }



`env` 域
-----------

``env`` 作用域允许定义一个或多个变量，这些变量将在执行workflow任务的环境中导出。

只需在变量名前面加上 ``env`` 作用域前缀或用大括号括起来，如下所示::

   env.ALPHA = 'some value'
   env.BETA = "$HOME/some/path"

   env {
        DELTA = 'one more'
        GAMMA = "/my/path:$PATH"
   }


`params` 域
--------------

``params`` 作用域允许您定义在pipeline脚本中可访问的参数。只需在参数名前面加上 ``params`` 作用域前缀或用大括号括起来，如下所示::

     params.custom_param = 123
     params.another_param = 'string value .. '

     params {

        alpha_1 = true
        beta_2 = 'another string ..'

     }



.. _config-process:

`process` 域
---------------

``process`` 配置作用域允许您为pipeline中的process提供默认配置。

您可以在这里指定  :ref:`process directive<process-directives>` 和executor部分中描述的任何属性。例子::

  process {
    executor='sge'
    queue='long'
    clusterOptions = '-pe smp 10 -l virtual_free=64G,h_rt=30:00:00'
  }

通过使用此配置，pipeline中的所有process都将按指定的设置, 通过SGE集群执行。


.. _config-process-selectors:

Process选择器
^^^^^^^^^^^^^^^^^

``withLabel`` 选择器允许带有此标签( :ref:`process-label` )指令注释的所有进程使用此配置，如下所示::

    process {
        withLabel: big_mem {
            cpus = 16
            memory = 64.GB
            queue = 'long'
        }
    }


上面的配置示例为使用 ``big_mem`` 标签标注的所有进程分配了16个cpu、64 Gb内存和 ``long`` 队列。


以同样的方式， ``withName`` 选择器允许通过名称配置pipeline中的特定process。
For example::

    process {
        withName: hello {
            cpus = 4
            memory = 8.GB
            queue = 'short'
        }
    }

.. tip::  标签和进程名都不需要用引号括起来，前提是名称包含特殊字符(例如: ``-`` 、``!`` 等)，或者它不是关键字或内置类型标识符。如果有疑问，可以用单引号或双引号括起标签名或进程名。

.. _config-selector-expressions:

选择器表达式
^^^^^^^^^^^^^^^^^^^^

标签和进程名称选择器都允许使用正则表达式，以便对匹配指定模式条件的所有进程应用相同的配置。例如::

    process {
        withLabel: 'foo|bar' {
            cpus = 2
            memory = 4.GB
        }
    }

上面的配置片段为带有标签 ``foo`` 和 ``bar`` 注释的process设置了2个cpu和4 GB内存。

进程选择器可以用特殊字符 ``!`` 作为前缀进行否定。例如::

    process {
        withLabel: 'foo' { cpus = 2 }
        withLabel: '!foo' { cpus = 4 }
        withName: '!align.*' { queue = 'long' }
    }

上面的配置片段为使用 ``foo`` 标签注释的process设置了2个cpu，为未使用该标签注释的所有process设置了4个cpu。最后, 它将不以 ``align`` 开头的所有process设置使用 ``long`` 队列.


.. _config-selectors-priority:

选择器的优先级
^^^^^^^^^^^^^^^^^^

当混合通用流程配置和选择器时，应用以下优先级规则(从低到高):
1. process通用配置。
2. workflow脚本中定义的process特定指令。
3. ``withLabel`` 选择器定义。
4. ``withName`` 选择器定义。

For example::

    process {
        cpus = 4
        withLabel: foo { cpus = 8 }
        withName: bar { cpus = 32 }
    }

使用上面的配置片段，如果workflow脚本中没有另外指定，那么所有workflow process都使用4个cpu。此外，用foo标签注释的process使用8个cpu。最后，名为bar的process使用32个cpu。


.. _config-executor:

`executor` 域
----------------

``executor`` 配置作用域允许您设置可选的executor设置，如下表所示。

===================== =====================
Name                  Description
===================== =====================
name                  要使用的 executor 名字 e.g. ``local``, ``sge``, etc.
queueSize              ``executor`` 将以并行方式处理的任务数量 (default: ``100``).
pollInterval          确定为检查进程终止而进行轮询的频率。
dumpInterval          确定executor状态在应用程序日志文件中写入的频率(默认值:5min)(default: ``5min``).
queueStatInterval     确定从集群系统获取队列状态的频率。此设置仅供 grid executors 使用(默认值: ``1min``).
exitReadTimeout       确定当进程终止但退出文件不存在或为空时，执行程序在返回错误状态之前等待多长时间。此设置仅供 grid executors 使用(默认值: ``270`` 秒).
killBatchSize         确定在单个命令执行中可以杀死的作业的数量 (default: ``100``).
submitRateLimit       确定每个时间单元可以执行的作业的最大速率. 例如: ``'10 sec'`` eg. 每秒最多10个作业(默认: ``unlimited``).
perJobMemLimit        指定平台LSF的 *per-job* 内存限制模式. See :ref:`lsf-executor`.
jobName               确定提交给底层集群执行程序的作业的名称 e.g. ``executor.jobName = { "$task.name - $task.hash" }`` .
cpus                  底层系统提供的最大cpu数量(仅供 ``local`` executor使用).
memory                底层系统提供的最大内存量(only used by the ``local`` executor).
===================== =====================



``executor`` 设置可定义如下:::

    executor {
        name = 'sge'
        queueSize = 200
        pollInterval = '30 sec'
    }


在pipeline中使用两个(或多个)不同的executor时，可以通过在executor名称前面加上符号 ``$`` 并将其用作特殊的作用域标识符来分别指定它们的设置。例如::

  executor {
    $sge {
        queueSize = 100
        pollInterval = '30sec'
    }

    $local {
        cpus = 8
        memory = '32 GB'
    }
  }

上面的配置例子可以用点 ``.`` 表示法重写, 如下::

    executor.$sge.queueSize = 100
    executor.$sge.pollInterval = '30sec'
    executor.$local.cpus = 8
    executor.$local.memory = '32 GB'

.. _config-docker:

`docker` 域
--------------

``docker`` 配置作用域控制Nextflow如何执行 `docker <http://www.docker.io>`_ 容器。

以下设置是可用的:

================== ================
Name                Description
================== ================
enabled             Turn this flag to ``true`` to enable Docker execution (default: ``false``).
envWhitelist        Comma separated list of environment variable names to be included in the container environment.
legacy              Uses command line options removed since version 1.10.x (default: ``false``).
sudo                Executes Docker run command as ``sudo`` (default: ``false``).
tty                 Allocates a pseudo-tty (default: ``false``).
temp                Mounts a path of your choice as the ``/tmp`` directory in the container. Use the special value ``auto`` to create a temporary directory each time a container is created.
remove              Clean-up the container after the execution (default: ``true``). For details see: https://docs.docker.com/engine/reference/run/#clean-up---rm .
runOptions          This attribute can be used to provide any extra command line options supported by the ``docker run`` command. For details see: https://docs.docker.com/engine/reference/run/ .
registry            The registry from where Docker images are pulled. It should be only used to specify a private registry server. It should NOT include the protocol prefix i.e. ``http://``.
fixOwnership        Fixes ownership of files created by the docker container.
engineOptions       This attribute can be used to provide any option supported by the Docker engine i.e. ``docker [OPTIONS]``.
mountFlags          Add the specified flags to the volume mounts e.g. `mountFlags = 'ro,Z'`
================== ================

上面的选项可以用 ``docker`` 作用域作为前缀，也可以用花括号括起来，如下所示::

    process.container = 'nextflow/examples'

    docker {
        enabled = true
        temp = 'auto'
    }



Read :ref:`docker-page` page to lean more how use Docker containers with Nextflow.


.. _config-singularity:

Scope `singularity`
-------------------

The ``singularity`` configuration scope controls how `Singularity <http://singularity.lbl.gov>`_ containers are executed
by Nextflow.

The following settings are available:

================== ================
Name                Description
================== ================
enabled             Turn this flag to ``true`` to enable Singularity execution (default: ``false``).
engineOptions       This attribute can be used to provide any option supported by the Singularity engine i.e. ``singularity [OPTIONS]``.
envWhitelist        Comma separated list of environment variable names to be included in the container environment.
runOptions          This attribute can be used to provide any extra command line options supported by the ``singularity exec``.
autoMounts          When ``true`` Nextflow automatically mounts host paths in the executed contained. It requires the `user bind control` feature enabled in your Singularity installation (default: ``false``).
cacheDir            The directory where remote Singularity images are stored. When using a computing cluster it must be a shared folder accessible to all computing nodes.
pullTimeout         The amount of time the Singularity pull can last, exceeding which the process is terminated (default: ``20 min``).
================== ================


Read :ref:`singularity-page` page to lean more how use Singularity containers with Nextflow.

.. _config-manifest:

`manifest` 域
----------------

``manifest`` 配置作用域允许您在GitHub、BitBucket或GitLab上发布pipeline项目或运行pipeline时定义所需的一些meta-data信息。

The following settings are available:

================== ================
Name                Description
================== ================
author              Project author name (use a comma to separate multiple names).
defaultBranch       Git repository default branch (default: ``master``).
description         Free text describing the workflow project.
homePage            Project home page URL.
mainScript          Project main script (default: ``main.nf``).
name                Project short name.
nextflowVersion     Minimum required Nextflow version.
version             Project version number.
================== ================

上面的选项可以通过在它们前面加上 ``manifest`` 作用域或在它们周围加上花括号来使用。例如::

    manifest {
        homePage = 'http://foo.com'
        description = 'Pipeline does this and that'
        mainScript = 'foo.nf'
        version = '1.0.0'
    }


To learn how to publish your pipeline on GitHub, BitBucket or GitLab code repositories read :ref:`sharing-page`
documentation page.

Nextflow 版本
^^^^^^^^^^^^^^^^

``nextflowVersion`` 设置允许您指定运行pipeline所需的最小版本。
下面的命令可能有助于您确保使用特定的版本::

    nextflowVersion = '1.2.3'        // exact match
    nextflowVersion = '1.2+'         // 1.2 or later (excluding 2 and later)
    nextflowVersion = '>=1.2'        // 1.2 or later
    nextflowVersion = '>=1.2, <=1.5' // any version in the 1.2 .. 1.5 range
    nextflowVersion = '!>=1.2'       // with ! prefix, stop execution if current version
                                        does not match required version.


.. _config-trace:

`trace` 域
-------------

``trace`` 作用域允许您控制由Nextflow生成的执行跟踪文件的布局。

可用的设置包括:

================== ================
Name                Description
================== ================
enabled             当为 ``true`` 时，生成执行跟踪报告文件 (default: ``false``).
fields              报告中要包含的字段的逗号分隔列表。可用字段列在 :ref:`this page <trace-fields>`
file                跟踪文件的名字 (default: ``trace.txt``).
sep                 用于分隔每行中的值的字符 (default: ``\t``).
raw                 当为 ``true``, 报告生成时使用原始数字，即日期和时间报告为毫秒，内存报告为字节数
================== ================

上面的选项可以通过在它们前面加上 ``trace`` 作用域或在它们周围加上花括号来使用。例如::

    trace {
        enabled = true
        file = 'pipeline_trace.txt'
        fields = 'task_id,name,status,exit,realtime,%cpu,rss'
    }


To learn more about the execution report that can be generated by Nextflow read :ref:`trace-report` documentation page.

.. _config-aws:

Scope `aws`
-----------

The ``aws`` scope allows you to configure the access to Amazon S3 storage. Use the attributes ``accessKey`` and ``secretKey``
to specify your bucket credentials. For example::


    aws {
        accessKey = '<YOUR S3 ACCESS KEY>'
        secretKey = '<YOUR S3 SECRET KEY>'
        region = '<REGION IDENTIFIER>'
    }

Click the following link to lean more about `AWS Security Credentials <http://docs.aws.amazon.com/general/latest/gr/aws-security-credentials.html>`_.

Advanced client configuration options can be set by using the ``client`` attribute. The following properties can be used:

=========================== ================
Name                        Description
=========================== ================
connectionTimeout           The amount of time to wait (in milliseconds) when initially establishing a connection before giving up and timing out.
endpoint                    The AWS S3 API entry point e.g. `s3-us-west-1.amazonaws.com`.
maxConnections              The maximum number of allowed open HTTP connections.
maxErrorRetry               The maximum number of retry attempts for failed retryable requests.
protocol                    The protocol (i.e. HTTP or HTTPS) to use when connecting to AWS.
proxyHost                   The proxy host to connect through.
proxyPort                   The port on the proxy host to connect through.
proxyUsername               The user name to use when connecting through a proxy.
proxyPassword               The password to use when connecting through a proxy.
signerOverride              The name of the signature algorithm to use for signing requests made by the client.
socketSendBufferSizeHint    The Size hint (in bytes) for the low level TCP send buffer.
socketRecvBufferSizeHint    The Size hint (in bytes) for the low level TCP receive buffer.
socketTimeout               The amount of time to wait (in milliseconds) for data to be transferred over an established, open connection before the connection is timed out.
storageEncryption           The S3 server side encryption to be used when saving objects on S3 (currently only AES256 is supported)
userAgent                   The HTTP user agent header passed with all HTTP requests.
uploadMaxThreads            The maximum number of threads used for multipart upload.
uploadChunkSize             The size of a single part in a multipart upload (default: `10 MB`).
uploadStorageClass          The S3 storage class applied to stored objects, one of [`STANDARD`, `STANDARD_IA`, `ONEZONE_IA`, `INTELLIGENT_TIERING`] (default: `STANDARD`).
uploadMaxAttempts           The maximum number of upload attempts after which a multipart upload returns an error (default: `5`).
uploadRetrySleep            The time to wait after a failed upload attempt to retry the part upload (default: `100ms`).
=========================== ================

For example::

    aws {
        client {
            maxConnections = 20
            connectionTimeout = 10000
            uploadStorageClass = 'INTELLIGENT_TIERING'
            storageEncryption = 'AES256'
        }
    }


.. _config-cloud:

Scope `cloud`
-------------

The ``cloud`` scope allows you to define the settings of the computing cluster that can be deployed in the cloud
by Nextflow.

The following settings are available:

=========================== ================
Name                        Description
=========================== ================
bootStorageSize             Boot storage volume size e.g. ``10 GB``.
imageId                     Identifier of the virtual machine(s) to launch e.g. ``ami-43f49030``.
instanceRole                IAM role granting required permissions and authorizations in the launched instances.
                            When specifying an IAM role no access/security keys are installed in the cluster deployed in the cloud.
instanceType                Type of the virtual machine(s) to launch e.g. ``m4.xlarge``.
instanceStorageMount        Ephemeral instance storage mount path e.g. ``/mnt/scratch``.
instanceStorageDevice       Ephemeral instance storage device name e.g. ``/dev/xvdc`` (optional).
keyName                     SSH access key name given by the cloud provider.
keyHash                     SSH access public key hash string.
keyFile                     SSH access public key file path.
securityGroup               Identifier of the security group to be applied e.g. ``sg-df72b9ba``.
sharedStorageId             Identifier of the shared file system instance e.g. ``fs-1803efd1``.
sharedStorageMount          Mount path of the shared file system e.g. ``/mnt/efs``.
subnetId                    Identifier of the VPC subnet to be applied e.g. ``subnet-05222a43``.
spotPrice                   Price bid for spot/preemptive instances.
userName                    SSH access user name (don't specify it to use the image default user name).
autoscale                   See below.
=========================== ================

The autoscale configuration group provides the following settings:

=========================== ================
Name                        Description
=========================== ================
enabled                     Enable cluster auto-scaling.
terminateWhenIdle           Enable cluster automatic scale-down i.e. instance terminations when idle (default: ``false``).
idleTimeout                 Amount of time in idle state after which an instance is candidate to be terminated (default: ``5 min``).
starvingTimeout             Amount of time after which one ore more tasks pending for execution trigger an auto-scale request (default: ``5 min``).
minInstances                Minimum number of instances in the cluster.
maxInstances                Maximum number of instances in the cluster.
imageId                     Identifier of the virtual machine(s) to launch when new instances are added to the cluster.
instanceType                Type of the virtual machine(s) to launch when new instances are added to the cluster.
spotPrice                   Price bid for spot/preemptive instances launched while auto-scaling the cluster.
=========================== ================

.. _config-conda:

`conda` 域
-------------

``conda`` 作用域允许定义由conda包管理器控制conda环境创建的配置设置。

以下设置是可用的:

================== ================
Name                Description
================== ================
cacheDir            定义存储Conda环境的路径。在使用计算集群时，请确保提供可从所有计算节点访问的共享文件系统路径。
createTimeout       定义Conda环境创建可以持续的时间。当超过超时时，创建过程将终止(默认值: ``20min``).
================== ================


.. _config-k8s:

Scope `k8s`
-----------

The ``k8s`` scope allows the definition of the configuration settings that control the deployment and execution of
workflow applications in a Kubernetes cluster.

The following settings are available:

================== ================
Name                Description
================== ================
autoMountHostPaths  Automatically mounts host paths in the job pods. Only for development purpose when using a single node cluster (default: ``false``).
context             Defines the Kubernetes `configuration context name <https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/>`_ to use.
namespace           Defines the Kubernetes namespace to use (default: ``default``).
serviceAccount      Defines the Kubernetes `service account name <https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/>`_ to use.
launchDir           Defines the path where the workflow is launched and the user data is stored. This must be a path in a shared K8s persistent volume (default: ``<volume-claim-mount-path>/<user-name>``.
workDir             Defines the path where the workflow temporary data is stored. This must be a path in a shared K8s persistent volume (default:``<user-dir>/work``).
projectDir          Defines the path where Nextflow projects are downloaded. This must be a path in a shared K8s persistent volume (default: ``<volume-claim-mount-path>/projects``).
pod                 Allows the definition of one or more pod configuration options such as environment variables, config maps, secrets, etc. It allows the same settings as the :ref:`process-pod` process directive.
pullPolicy          Defines the strategy to be used to pull the container image e.g. ``pullPolicy: 'Always'``.
runAsUser           Defines the user ID to be used to run the containers.
storageClaimName    The name of the persistent volume claim where store workflow result data.
storageMountPath    The path location used to mount the persistent volume claim (default: ``/workspace``).
storageSubPath      The path in the persistent volume to be mounted (default: root).
volumeClaims        (deprecated)
================== ================

See the :ref:`k8s-page` documentation for more details.

.. _config-timeline:

`timeline` 域
----------------

``timeline`` 作用域允许您启用/禁用Nextflow生成的流程执行timeline报告。

The following settings are available:

================== ================
Name                Description
================== ================
enabled             When ``true`` turns on the generation of the timeline report file (default: ``false``).
file                Timeline file name (default: ``timeline.html``).
================== ================

.. _config-mail:

Scope `mail`
------------

The ``mail`` scope allows you to define the mail server configuration settings needed to send email messages.

================== ================
Name                Description
================== ================
from                Default email sender address.
smtp.host           Host name of the mail server.
smtp.port           Port number of the mail server.
smtp.user           User name to connect to  the mail server.
smtp.password       User password to connect to the mail server.
smtp.proxy.host     Host name of an HTTP web proxy server that will be used for connections to the mail server.
smtp.proxy.port     Port number for the HTTP web proxy server.
smtp.*              Any SMTP configuration property supported by the Java Mail API (see link below).
debug               When ``true`` enables Java Mail logging for debugging purpose.
================== ================

.. note:: Nextflow relies on the `Java Mail API <https://javaee.github.io/javamail/>`_ to send email messages.
  Advanced mail configuration can be provided by using any SMTP configuration property supported by the Java Mail API.
  See the `table of available properties at this link <https://javaee.github.io/javamail/docs/api/com/sun/mail/smtp/package-summary.html#properties>`_.

For example, the following snippet shows how to configure Nextflow to send emails through the
`AWS Simple Email Service <https://aws.amazon.com/ses/>`_::

    mail {
        smtp.host = 'email-smtp.us-east-1.amazonaws.com'
        smtp.port = 587
        smtp.user = '<Your AWS SES access key>'
        smtp.password = '<Your AWS SES secret key>'
        smtp.auth = true
        smtp.starttls.enable = true
        smtp.starttls.required = true
    }

.. _config-notification:

`notification` 域
--------------------

``notification`` 作用域允许您在工作流执行结束时定义自动发送通知电子邮件消息。

================== ================
Name                Description
================== ================
enabled             Enables the sending of a notification message when the workflow execution completes.
to                  Recipient address for the notification email. Multiple addresses can be specified separating them with a comma.
from                Sender address for the notification email message.
template            Path of a template file which provides the content of the notification message.
binding             An associative array modelling the variables in the template file.
================== ================

The notification message is sent my using the STMP server defined in the configuration :ref:`mail scope<config-mail>`.

If no mail configuration is provided, it tries to send the notification message by using the external mail command
eventually provided by the underlying system (eg. ``sendmail`` or ``mail``).

.. _config-report:

Scope `report`
--------------

The ``report`` scope allows you to define configuration setting of the workflow :ref:`execution-report`.

================== ================
Name                Description
================== ================
enabled             If ``true`` it create the workflow execution report.
file                The path of the created execution report file (default: ``report.html``).
================== ================

.. _config-weblog:

Scope `weblog`
--------------

The ``weblog`` scope allows to send detailed :ref:`trace scope<trace-fields>` information as HTTP POST request to a webserver, shipped as a JSON object.

Detailed information about the JSON fields can be found in the :ref:`weblog description<weblog-service>`.

================== ================
Name                Description
================== ================
enabled             If ``true`` it will send HTTP POST requests to a given url.
url                The url where to send HTTP POST requests (default: ``http:localhost``).
================== ================


.. _config-profiles:

Config profiles
===============

配置文件可以包含一个或多个概要文件(profile)的定义。概要文件(profile)是一组配置属性，当使用 ``-profile`` 命令行选项启动pipeline执行时，可以 激活/选择 这些配置属性。

配置profile是通过使用特殊的作用域 ``profiles`` 定义的，这些profiles使用公共前缀将属于同一profile的属性分组。例如::

    profiles {

        standard {
            process.executor = 'local'
        }

        cluster {
            process.executor = 'sge'
            process.queue = 'long'
            process.memory = '10GB'
        }

        cloud {
            process.executor = 'cirrus'
            process.container = 'cbcrg/imagex'
            docker.enabled = true
        }

    }


此配置定义了三个不同的profile: ``standard``, ``cluster`` 和 ``cloud``, 它们根据目标运行时平台设置不同的process配置策略。

.. tip::  可以通过用逗号分隔profile名称来指定两个或多个profiles，例如:
   :: 
           
      nextflow run <your script> -profile standard,cloud


The above feature requires version 0.28.x or higher.

环境变量
=====================

以下环境变量控制Nextflow运行时的配置及其使用的Java虚拟机。

=========================== ================
Name                        Description
=========================== ================
NXF_HOME                    Nextflow home directory (default: ``$HOME/.nextflow``).
NXF_VER                     Defines what version of Nextflow to use.
NXF_ORG                     Default `organization` prefix when looking for a hosted repository (default: ``nextflow-io``).
NXF_GRAB                    Provides extra runtime dependencies downloaded from a Maven repository service.
NXF_OPTS                    Provides extra options for the Java and Nextflow runtime. It must be a blank separated list of ``-Dkey[=value]`` properties.
NXF_CLASSPATH               Allows the extension of the Java runtime classpath with extra JAR files or class folders.
NXF_ASSETS                  Defines the directory where downloaded pipeline repositories are stored (default: ``$NXF_HOME/assets``)
NXF_PID_FILE                Name of the file where the process PID is saved when Nextflow is launched in background.
NXF_WORK                    Directory where working files are stored (usually your *scratch* directory)
NXF_TEMP                    Directory where temporary files are stored
NXF_DEBUG                   Defines scripts debugging level: ``1`` dump task environment variables in the task log file; ``2`` enables command script execution tracing; ``3`` enables command wrapper execution tracing.
NXF_EXECUTOR                Defines the default process executor e.g. `sge`
NXF_CONDA_CACHEDIR          Directory where Conda environments are store. When using a computing cluster it must be a shared folder accessible from all computing nodes.
NXF_SINGULARITY_CACHEDIR    Directory where remote Singularity images are stored. When using a computing cluster it must be a shared folder accessible from all computing nodes.
NXF_JAVA_HOME               Defines the path location of the Java VM installation used to run Nextflow. This variable overrides the ``JAVA_HOME`` variable if defined.
NXF_OFFLINE                 When ``true`` disables the project automatic download and update from remote repositories (default: ``false``).
NXF_CLOUD_DRIVER            Defines the default cloud driver to be used if not specified in the config file or as command line option, either ``aws`` or ``google``.
NXF_ANSI_LOG                Enables/disables ANSI console output (default ``true`` when ANSI terminal is detected).
JAVA_HOME                   Defines the path location of the Java VM installation used to run Nextflow.
JAVA_CMD                    Defines the path location of the Java binary command used to launch Nextflow.
HTTP_PROXY                  Defines the HTTP proxy server
HTTPS_PROXY                 Defines the HTTPS proxy server
=========================== ================
