
#  Lab2 要点

## 要求：https://pdos.csail.mit.edu/6.824/labs/lab-raft.html
任务是实现选举和心跳。

## 整体思路
1.server有三种角色，follower，candidate，leader，注意角色的转换。<br>
![server](https://github.com/MollyQI3104/MIT-6.824-Distrubuted-System/blob/main/Figures/server%20state.png)
2.每个角色下都有相应任务，就2A来说，leader要周期性发送心跳，follower需等待一段时间，无心跳接收即转为candidate开始选举。<br>
  角色切换后相应任务都需要切换，是一个长期后台执行的循环任务。<br>
3.每个server会有一个lastActiveTime（暂时字段写为initTime），在server创建，开始选举，给其他节点投票（接收到投票请求），接收到心跳时都会更新为time.Now()<br>
  意义为最后与其他节点保持联系的时间。<br>
4.每个server会有一个时间段timeout，随机为int(300 + rand.Int31n(200))，随机可使得节点不会一起开始选举，避免全部节点先投给自己，一人一票的局面。<br>
5.term的更新。<br>
  在接受old term的包后丢弃，在接收new term的包（心跳，请求投票）后要意识到自己落后，term更新，角色应为follower。<br>
  leader在发送心跳收到回复后，如果自己的term落后，term更新，应降为follower。<br>
  candidate在收到投票回复后，如果自己的term落后，term更新，应降为follower。<br>

## 易错的编程点

1.rpc包处理的逻辑。<br>
  一方作为client调用rpc的call，rpc调用server的handler函数，server上的handler函数来处理args和reply。<br>
  调用结束后，根据reply，client来处理接下来的逻辑（例如统计选票）<br>
2.条件锁的逻辑。<br>
主线程：<br>
  			var mu sync.Mutex <br>
			cond := sync.NewCond(&mu)<br>
go程：<br>
  mu.Lock() <br>
  defer mu.Unlock()<br>
  ...//收到选票<br>
  cond.Broadcast()<br>
  
主线程：<br>
 mu.Lock() <br>
 			for count < (size+1)/2 && finished != size &&//选举没有结束，等待<br>
				rf.state == "candidate" && //不再是"candidate"则退出<br>
				rf.currentTerm >= maxTerm && //term变化退出<br>
				int(time.Now().Sub(rf.initTime).Milliseconds())<timeout{//超时退出<br>
				cond.Wait()<br>
			}<br>
      ...//选举结束或失格退出后的应对逻辑<br>
 mu.Unlock()<br>
 
 cond.Wait()则是释放锁阻塞，cond.Broadcast()则是每收到选票，notifyall唤醒挂起的线程抢锁向下执行（即验证for循环条件），达到某个条件即可以开结果<br>
 都要在有锁条件下调用<br>
 （signal则对应java中notify）<br>
      
3.server状态的写入读取都应加锁，可以用-race查看冲突情况。<br>
  （如果有volatile这样的关键字，就能避免读取也只能加锁解锁的重量级了？按lab初衷来看确实是不考虑性能的）<br>
  
4.go的闭包函数。<br>
for i ...<br>
  go func(index int){<br>
  }(i)<br>
  传入参数是go程私有，直接使用外部i则外部更新go程更新。<br>
  是非常易错的点。<br>
  
5.server为leader角色时不要给自己发心跳，candidate角色时不要给自己发请求投票。<br>
  即i==rf.me 的go程不需要创建。<br>
  
6.在2A中，心跳每300ms发送一次，timeout时间是300-500的随机数。timeout时间过长可能无法在5s内完成选举（测试脚本的限制）

![lab2A](https://github.com/MollyQI3104/MIT-6.824-Distrubuted-System/blob/main/Lab%202A.png)
