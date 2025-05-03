

Raft 是一种为了更容易理解而设计的共识算法，由 Diego Ongaro 和 John Ousterhout 在 2013 年提出。它等同于 Paxos 算法的功能，但结构不同，更容易理解和实现。

## 基本概念

Raft 将一致性问题分解为三个相对独立的子问题：

1. **领导者选举**：当现有领导者失效时，必须选出一个新的领导者
2. **日志复制**：领导者必须从客户端接收日志条目，并将其复制到集群中的其他服务器上
3. **安全性**：如果任何服务器已经应用了特定的日志条目到其状态机，则其他服务器不会在相同的日志索引处应用不同的命令

在 Raft 中，服务器可以处于三种状态之一：

- **领导者（Leader）**：处理所有客户端请求
- **跟随者（Follower）**：被动响应来自领导者或候选人的请求
- **候选人（Candidate）**：用于选举新的领导者

## 算法流程

### 领导者选举

1. 所有服务器初始化为跟随者状态
2. 如果跟随者在一段时间内没有收到领导者的心跳，它会转变为候选人状态
3. 候选人增加当前任期号，为自己投票，并向其他服务器发送请求投票的 RPC
4. 服务器会投票给第一个它收到的、具有更新的任期号的候选人
5. 候选人获得多数票后成为领导者
6. 领导者定期发送心跳（空的附加日志 RPC）来维持其权威

### 日志复制

1. 客户端将命令发送给领导者
2. 领导者将命令作为新的日志条目附加到其日志中
3. 领导者并行发送附加日志 RPC 给跟随者
4. 当日志条目被安全复制后，领导者应用该条目到其状态机并向客户端返回结果
5. 如果跟随者崩溃或运行缓慢，领导者会无限重试附加日志 RPC

### 安全性

Raft 通过以下机制确保安全性：

- **选举限制**：候选人必须包含所有已提交的日志条目才能当选为领导者
- **提交之前的日志条目**：领导者只提交当前任期内的日志条目，之前任期的日志条目通过提交当前任期的日志条目间接提交
- **日志匹配特性**：如果两个日志包含具有相同索引和任期的条目，则这两个日志在该索引之前的所有条目都相同

## 集群成员变更

Raft 使用两阶段方法来处理集群成员变更：

1. 首先切换到一个过渡配置（称为联合共识），这个配置包括新旧配置的所有服务器
2. 一旦联合共识被提交，系统就切换到新配置

## 伪代码实现

```python
# 领导者选举
def become_candidate():
    current_term += 1
    voted_for = self_id
    votes_received = 1  # 投给自己
    send_request_vote_to_all_servers()

def handle_request_vote(candidate_term, candidate_id, last_log_index, last_log_term):
    if candidate_term < current_term:
        return False
    
    if (voted_for is None or voted_for == candidate_id) and 
       candidate_log_is_up_to_date(last_log_index, last_log_term):
        voted_for = candidate_id
        return True
    else:
        return False

# 日志复制
def append_entries(entries):
    log.append(entries)
    send_append_entries_to_all_followers()

def handle_append_entries(leader_term, prev_log_index, prev_log_term, entries, leader_commit):
    if leader_term < current_term:
        return False
    
    if log[prev_log_index].term != prev_log_term:
        return False
    
    # 删除冲突的条目，追加新条目
    log.delete_entries_from(prev_log_index + 1)
    log.append(entries)
    
    # 更新提交索引
    if leader_commit > commit_index:
        commit_index = min(leader_commit, log.last_index)
    
    return True
```
## 参考资料
1. Ongaro, D., & Ousterhout, J. (2014). In search of an understandable consensus algorithm. In USENIX Annual Technical Conference (pp. 305-319).
2. Raft 官方网站： https://raft.github.io/