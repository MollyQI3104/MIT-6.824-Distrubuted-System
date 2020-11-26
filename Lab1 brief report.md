#  Lab1 要点

## 要求：https://pdos.csail.mit.edu/6.824/labs/lab-mr.html

## 整体思路
1.worker和master通过rpc通信，需约定信息的struct（rpc.go）和响应函数（master.go）</br>
  worker会一直请求task，直到master通知全部任务已完成，即可退出。</br>
2.matser会维护map和reduce的信息列表（使用map[int]Task实现），Task包括状态和时间，状态为“”，“assigned”，“done”，时间会assign给worker的当前时间。</br>
  也可以用一个list，每个list的entry为Task，Task包括状态，时间和一个index信息。</br>
  这些信息有并发访问，需加锁。</br>
3.master维护一个映射，给输入的files编号。FileIndex map[int]string</br>
  方便map任务中文件读取</br>
4.master会先分发map任务，再分发reduce任务</br>
5.worker在完成任务后都会回复master，master修改信息列表中相应task的状态为“done”，已完成的mapTask++或是reduceTask++</br>
6.master分发任务时打上时间戳，在轮询时遇到“assgined”但是与当前时间差大与10s的任务，会重新发配给worker（认为前一个worker宕机）</br>
  为保证输出文件不污染，使用临时文件做储存，完成task时按命名规则来储存文件。</br>
  在这个case中，不存在多个worker去写一个输出文件，不使用临时文件，每次输出文件时都是重新创建也是可以达到目的。</br>
7.master记录完成的task数量，定期执行的done函数会在数量达标后返回true，master退出</br>


## 易忽略的编程点
1.map的task数是文件数，reduce的task数是nreduce。</br>
  使用key的hash来选择reduce的index，同一index是被一个worker来处理的。</br>
  例如mr-0-0，mr-1-0等都是一个worker来reduce任务。</br>
  （否则输出文件里就会有同一个key的多条记录，在输出文件写入方式为追加的前提下）</br>
2.属性的权限程度。大小写开头的字段名。</br>
  调试表现即为没有赋值，因为无法访问。</br>
3.锁的添加与释放。逻辑错误导致的锁不被释放会引起其他worker的堵塞，调试也会有相同表现。</br>

![lab1](https://github.com/MollyQI3104/MIT-6.824-Distrubuted-System/blob/main/Lab1.png)
