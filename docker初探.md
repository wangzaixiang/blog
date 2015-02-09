周六尝试了一下docker，我是在MAC上，通过boot2docker，以及试用prometheus的过程中来了解docker的，虽然现在还不是很熟悉docker，
只是简单的使用了一下，有一个很基本的概念。

1、boot2docker 实际上在mac上安装了一个Linux虚拟机，这个是通过virtual box来完成的，是一个没有图形界面的（甚至是Headless），这个可以
从mac上看出来：

```sh
  501 14955 14522   0 日01下午 ??         3:50.15 /Applications/VirtualBox.app/Contents/MacOS/VBoxHeadless --comment boot2docker-vm --startvm 44985a3e-518e-417e-977b-5e9b8a64274f --vrde config
  501 15310 12942   0 11:09下午 ttys002    0:00.00 grep boot2docker
```

2、我们可以通过 boot2docker ssh 进入到这台虚拟机里面，这个其实就是一个Linux环境了。一般的，这台虚拟机有一个mac可以访问的IP地址：
`192.168.59.103`.

3、在mac上通过docker命令来进行docker的操作，但其实这个和在59.103这台虚拟机内使用docker是一致的，这个实际上由如下环境变量控制的：
```sh
export DOCKER_HOST=tcp://192.168.59.103:2376
```

4、docker run 会启动一个新的 container，并且还会智能的从 docker 官网下载image。可以理解image其实就是一个container的安装盘，
但每个container可以理解为一个实例，它有自己的文件系统和状态。可以使用docker stop、docker start等命令来停止实例、重启实例。
也可以通过 docker exec -it $containerId bash 来在这个实例中运行一个bash会话，从而手动的做一些个管理工作。

5、docker的神奇支持在于，每个container实际上并不是在一个新的”虚拟机“中，实际上，它是一个运行在外部Linux宿主机（在这里就是
virtualbox中的linux虚拟机，因为docker不能直接运行在MAC之中）。每个container中的每一个进程，实际上就是宿主机上的一个进程，
这个可以在container中用ps 查看，也可以在宿主机用ps 查看，当然如果想进入到 /proc/pid 文件系统中，你会发现在宿主机中的proc
文件系统其实大部分信息，如cwd、fd等信息都不是真实有效地。但不管如何，每一个container中的进程就是一个真实的宿主机进程，而不是
运行在一个”新的虚拟机操作系统“之上。这自然带来了最有效的资源利用，以及没有性能上的损耗。但是，在资源的隔离度上，自然也就是
相对有限了。

眼下的docker，貌似是虚拟化路线中的新贵，我的了解，也暂时到此为止，后续，我们也可以尝试使用docker来简单化我们的部署管理。
