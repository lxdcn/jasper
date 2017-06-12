---
layout: post
cover: assets/images/cover.jpg
subclass: post
title: "在PowerShell里并行执行脚本"
navigation: true
logo: assets/images/host.png
date: 2014-07-24 16:18:29
categories: lxd
tags: tech powershell chs
comments: true
---


在Bash里并行执行脚本有几种方法，可以在Bash里可以用把一个任务后面加`&`开一个subshell直接甩倒后台去执行，也可以用[GNU Parallel]对并行执行的任务做更精细的控制。

而在PowerShell脚本里并行执行脚本，也有多种方法，一是用`Start-Job`命令启动一个background job，二是用并行的Workflow。

并行的Workflow PowerShell 2不支持，并且PowerShell的Workflow有一个很明确的限制是不能包含交互的命令。比如你需要把一段PowerShell脚本从串行改成并行，但是这段脚本又里使用了`Write-Host`命令直接打印到host的话，PowerShell就会无情的报错`Cannot call the 'Write-Host' command. Other commands from this module have been packaged as workflow activities, but this command was specifically excluded. `，你就得自己看着办了，比如用`Write-Output`。
<br />
我用了background job的方案，相对于Bash的subshell后台执行命令，PowerShell提供了更为丰富的命令来控制background job，我用到了这么几个：

  - `Start-Job`，启动一个background job，这个命令定义一个background job的同时直接启动这个background job，我用到了几个参数：
    - `-ScriptBlock` 需要执行的代码块
    - `-InitializationScript` 初始化代码块
    - `-Name` Job命名
    - `-InputObject` 传给background job的参数，如果需要传多个参数的话需要封装成一个Object，然后在ScriptBlock里再解析出来
  - `Wait-Job`，阻塞等待background job结束，等待的background job可以有多个，作为参数传进去
  - `Receive-Job`，获取background job的结果，我用这个命令来获取background job的终端输出，需要被获取的background job也可以有多个，作为参数传进去
  - `Remove-Job`，当一个background job结束之后就可以用这个命令删除这个job

<br />
一个简单的例子：
{% highlight powershell %}
# background job需要执行的代码
# $Input是传进来的参数，是个Enumerator，需要遍历取出来
$execWrapper = {
    function Exec {
        $Input | %{
            $Param = $_
        }
        Start-Sleep 1
        Write-Output "This is job $Param"
    }
}

# 一般来讲业务上会有一个数组传进来，然后针对每个数组元素创建job并行执行
$data = @('one', 'two', 'three')

# 遍历数组，创建job，Start-Job的同时Job就会执行，创建后将这些job存入$jobs数组
# Start-Job的参数分别是
#   -ScriptBlock 需要执行的代码块
#   -InitializationScript 初始化代码块
#   -Name Job命名
#   -InputObject 传给background job的参数
$jobs = @()
$data | %{
    $job = Start-Job -ScriptBlock {Exec} -InitializationScript $execWrapper -Name $_ -InputObject $_
    $jobs += $job
}

# 等待所有后台job结束
$null = Wait-Job -Job $jobs

# 从Job接收终端输出，打印到当前终端
$jobs | %{
    Write-Output $(Receive-Job -Job $_)
}

# 清理掉job
Remove-Job -Job $jobs
Get-Job
{% endhighlight %}


<br />
但是[韩老师](http://isaachan.github.io/)指出，这种方法有很大缺陷。当background job执行时间很长的时候，如果我们先Wait它们执行完毕，再获取background job的输出，那么造成的效果就是当前的PowerShell session会hang住很长时间，造成很糟糕的运维体验。

那么就不能用`Wait-Job`命令了，要在background job还在running的时候，用`Receive-Job`“实时的”获取到它的终端输出，打印到当前的PowerShell上。

具体的做法是用一个循环来轮询，检查各个background job是否还有数据可读，如果可读，就用`Receive-Job`命令取出这个background job的result，再`Write-Host`到当前的PowerShell里。如果没有可读的的background job，则认为所有job都跑完了，此时就相当于`Wait-Job`释放的那一刻，从循环中跳出，再执行后续操作。

改进后的代码是这样的：
{% highlight powershell %}
# 扩充一下background job需要执行的代码
$execWrapper = {
    function Exec {
        $Input | %{
            $Param = $_
        }
        Start-Sleep 1
        Write-Output "This is job $Param"
        Start-Sleep 1
        Write-Output "This is job $Param again"
        Start-Sleep 1
        Write-Output "This is job $Param again, again"
    }
}

$data = @('one', 'two', 'three')

$jobs = @()
$data | %{
    $job = Start-Job -ScriptBlock {Exec} -InitializationScript $execWrapper -Name $_ -InputObject $_
    $jobs += $job
}

# 因为要实时打印出background job的输出，那么就不能调用这个函数被动等待所有background job结束
# $null = Wait-Job -Job $jobs

# 改用手动循环检查job的状态，同时将background job的输出打印到当前终端
While (1) {
    $JobsRunning = 0
    $jobs | %{
        if ($_.HasMoreData) {
            $JobsRunning += 1
            #在这里将background job的输出打印到当前终端
            Write-Output "This is result of Job $($_.Name) :"
            Write-Output $(Receive-Job -Job $_)
        }
    }
   if ($JobsRunning -eq 0) {
       Break
   }
   Start-Sleep 1
}

Remove-Job -Job $jobs
Get-Job
{% endhighlight %}

<br /><br />
当然在具体场景下面还有更多的坑要去趟，比如要改写在background job里执行的PowerShell代码用的是`Write-Host`，因为`Write-Host`是数据直接写向host，而不是pipeline，那么`Receive-Job`的用法就不一样了，它也会直接写向host。再比如你需要串行改并行的PowerShell脚本本身是在一个PowerShell Module里面，并且background job需要调用当前这个Module里的另外一个函数。

因为Start一个background job是开一个全新的PowerShell session，在这个全新的PowerShell session里没有Import这个Module，所以会提示这个函数找不到。

这时候我采用的方法是在`Start-Job`的`-InputObject`参数里把当前脚本所在的Module的psm1文件路径封装进去传进去，background job启动之后再解析出这个psm1文件路径重新`Import-Module`。

因为PowerShell 2和PowerShell 4的`$PSScriptRoot`含义不同，为了避免兼容性问题，我用了`$MyInvocation`。大概是这样的：
{% highlight powershell %}
$currentModulePath = "$(split-path -parent $MyInvocation.MyCommand.Definition)\psmodule.psm1"
$execWrapper = {
    function Exec {
        $Input | %{
            Import-Module $_['currentModulePath'] -Force | Out-Default
            $Param = $_['data']
        }
        Start-Sleep 1
        Write-Output "This is job $Param"
    }
}

...
$job = Start-Job -ScriptBlock {Exec} -InitializationScript $execWrapper -Name $_ -InputObject @{data=$_; currentModulePath=$currentModulePath}
...

{% endhighlight %}

<br />
完了，有机会再整理一下[GNU Parallel]。
<br /><br />
参考：

  - [Start-Job](http://technet.microsoft.com/en-us/library/hh849698.aspx)
  - [Wait-Job](http://technet.microsoft.com/en-us/library/hh849735.aspx)
  - [Receive-Job](http://technet.microsoft.com/en-us/library/hh849718.aspx)
  - [Background Jobs (PowerShell)](http://vwiki.co.uk/Background_Jobs_%28PowerShell%29)

[GNU Parallel]: http://www.gnu.org/software/parallel/ "GNU Parallel"
