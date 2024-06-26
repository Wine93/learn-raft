流程详解
===

流程概览
---

1. 节点在选举超时时间（`election_timeout_ms`）内未收到任何心跳而触发选举
2. 向所有节点广播 `PreVote` 请求，若收到大多数赞成票则进行正式选举，否则重新等待选举超时
3. 将自身角色转变为 Candidate，并将自身 `term` 加一，向所有节点广播 `RequestVote` 请求
4. 在投票超时时间（`vote_timeout_ms`）内若收到足够多的选票则成为 Leader；若有收到更高 `term` 的响应则转变为 Follower 并重复步骤 1；否则等待投票超时后转变为 Follower 并重复步骤 2
5. 成为 Leader
    * 5.1 将自身角色转变为 Leader
    * 5.2 为所有 Follower 创建 `Replicator`，并对所有 Follower 定期广播心跳
    * 5.3 通过发送空的 `AppendEntries` 请求来确定各 Follower 的 `nextIndex`
    * 5.4 将之前任期的日志全部复制给 Follower（**只复制不提交，不更新 commitIndex**）
    * 5.5 通过复制并提交一条本任期的配置日志来提交之前任期的日志，提交后日志就可以 apply了
    * 5.6 调用用户状态机的 `on_apply` 来回放之前任期的日志，以恢复状态机
    * 5.7 待日志全部回放完后，调用用户状态机的 `on_configuration_committed` 来应用上述配置
    * 5.8 调用用户状态机的 `on_leader_start`
6. 至此，Leader 可以正式对外服务

上述流程可分为 PreVote (1-2)、RequestVote (3-4)、成为 Leader（5-6）这三个阶段

<!---
流程注解
---

* `PreVote` 请求中的 Term
* 二元组 `<currentTerm, votedFor>` 会持久化，即使重启也能确保一个任期能只给一个节点投票，这是确保在同一个 Term 内只会产生一个 Leader 的关键
* 4
* 5.3 只有确定了各 Follower 的 `nextIndex` 才能发送日志，不然不知道从哪里开始发送日志
* 5.4 对于上一任期的日志只复制不提交是为了防止幽灵日志问题，需通过 5.5 提交
* 5.5 braft 中成为 Leader 后提交本任期内的第一条日志是配置日志，并非 `no-op`
* 5.5 `CommitIndex` 并不会持久化，Leader 只有提交了配置日志才能确认，Follower 则在之后的心跳中由 Leader 传递，只有确认了 `CommitIndex` 后才能开始回放日志
* 5.5 日志复制，这样确保了在调用 `on_leader_start` 前，上一任期的所有日志都已经回调 `on_apply`
-->

投票规则
---

**相关 RPC：**
```proto
message RequestVoteRequest {
    required string group_id = 1;
    required string server_id = 2;
    required string peer_id = 3;
    required int64 term = 4;
    required int64 last_log_term = 5;
    required int64 last_log_index = 6;
    optional TermLeader disrupted_leader = 7;
};

message RequestVoteResponse {
    required int64 term = 1;
    required bool granted = 2;
    optional bool disrupted = 3;
    optional int64 previous_term = 4;
    optional bool rejected_by_lease = 5;
};

service RaftService {
    rpc pre_vote(RequestVoteRequest) returns (RequestVoteResponse);
    rpc request_vote(RequestVoteRequest) returns (RequestVoteResponse);
}
```

在同一任期内，节点发出的 `PreVote` 请求和 `RequestVote` 请求的内容是一样的，区别在于：
  * `PreVote` 请求中的 `term` 为自身的 `term` 加上 1
  * 而发送 `RequestVote` 请求前会先将自身的 `term` 加 1，再将其作为请求中的 `term`

节点对于 `RequestVote` 请求投赞成票需要同时满足以下 3 个条件，而 `PreVote` 只需满足前 2 个条件：

* term：候选人的 `term` 要大于等于自己的 `term`
* lastLog：候选人的最后一条日志要和自己的一样新或者新于自己
* votedFor：自己的 `votedFor` 为空或者等于候选人的 ID

`PreVote` 与 `RequestVote` 的差异：

* 处理 `RequestVote` 请求时会记录 `votedFor`，确保在同一个任期内只会给一个候选人投票；而 `PreVote` 则可以同时投票给多个候选人，只要其满足上述的 2 个条件
* 处理 `RequestVote` 请求时若发现请求中的 `term` 比自身的大，会 `step_down` 成 Follower，而 `PreVote` 则不会，这点可以确保不会在 `PreVote` 阶段打断当前 Leader

从以上差异可以看出，`PreVote` 更像是一次预检，检测其连通性和合法性，并没有实际的动作。

> **日志新旧比较**
>
> 日志由 `term` 和 `index` 组成，对于 2 条日志 `a` 和 `b` 来说：
> * 若其 `term` 和 `index` 都一样，则 2 条日志一样新
> * 若 `(a.term > b.term) || (a.term == b.term && a.index > b.index)`，则日志 `a` 新于日志 `b`

votedFor
---

每个节点都会记录当前 `term` 内投给了谁（即 `votedFor`），如果在相同 `term` 内已经投过票了，就不会再投票给其他人，这是确保在同一任期内只会产生一个 Leader 的关键。

除此之外，Raft 会将 `term` 与 `votedFor` 进行持久化，防止在投给某一个候选人后节点 Crash，重新启动后又投给了另一候选人。 在具体实现方面，节点会先持久化 `term` 与 `votedFor`，再向候选人返回赞成响应。

幽灵日志
---

![图 3.1  幽灵日志](image/3.1.png)

上述选举流程中提到，Leader 并不是通过 `Quorum` 机制来提交之前任期的日志，而是通过提交本任期的一条日志，顺带提交上一任期的日志。这主要是为了解决 Raft 论文在 `5.4 Safety` 一节中提到的幽灵日志问题，因为该问题会破坏系统的线性一致性，正如上图所示：

* (a)：`S1` 当选 `term 2` 的 Leader，并将日志复制给 `S2`，之后 Crash
* (b)：`S5` 被 `S3,S4,S5` 选为 `term 3` 的 Leader，在本地写入一条日志后 Crash
* (c)：`S1` 被 `S1,S2,S3` 选为 `term 4` 的 Leader，并将 `index=2` 的日志复制给 `S3`，达到 `Quorum` 并应用到状态机；在本地写入一条日志，然后 Crash
* (d1)：`S5` 被 `S2,S3,S4,S5` 选为 `term 5` 的 Leader，并将 `index=2` 的日志复制给所有节点，从而覆盖了原来的日志。

从上面流程可以看到，在 `(c)` 时刻 `index=2` 的日志即使被提交了，但在 `(d1)` 时又被覆盖了。如果我们在 `(c)` 时刻去 Leader `S1` 读取 `x` 的值，得到的将是 `1`，之后我们又在 `(d1)` 时刻去 Leader `S5` 读取 `x` 的值，得到的将是 `0`，这明显违背了线性一致性。

所以论文里提出不能通过 `Quorum` 机制提交之前任期的日志，而是需要通过提交本任期的一条日志，顺带提交上一任期的日志，正如 `(d2)` 所示。一般 Raft 实现会在节点当选 Leader 后提交一条本任期的 `no-op` 日志，而 braft 中提交的是本任期的配置日志，这主要是在实现上和[节点配置变更](/ch06/)的特性结合到一起了，但其起到的作用是一样的，只要是本任期内的日志都能解决幽灵日志问题，具体实现见以下[提交 no-op 日志](#ti-jiao-noop-ri-zhi)。

> 特别需要注意的是，以上的读操作是指除 `Raft Log Read` 之外的其他读取方式，因为对于 `Raft Log Read` 来说，其读操作就是一条本任期的日志。

相关接口
---

一些会在选举过程中调用的状态机接口：

```cpp
class StateMachine {
public:
    // Update the StateMachine with a batch a tasks that can be accessed
    // through |iterator|.
    //
    // Invoked when one or more tasks that were passed to Node::apply have been
    // committed to the raft group (quorum of the group peers have received
    // those tasks and stored them on the backing storage).
    //
    // Once this function returns to the caller, we will regard all the iterated
    // tasks through |iter| have been successfully applied. And if you didn't
    // apply all the the given tasks, we would regard this as a critical error
    // and report a error whose type is ERROR_TYPE_STATE_MACHINE.
    virtual void on_apply(::braft::Iterator& iter) = 0;

    // Invoked when the belonging node becomes the leader of the group at |term|
    // Default: Do nothing
    virtual void on_leader_start(int64_t term);

    // Invoked when this node steps down from the leader of the replication
    // group and |status| describes detailed information
    virtual void on_leader_stop(const butil::Status& status);

    // Invoked when a configuration has been committed to the group
    virtual void on_configuration_committed(const ::braft::Configuration& conf);
    virtual void on_configuration_committed(const ::braft::Configuration& conf, int64_t index);

    // this method is called when a follower stops following a leader and its leader_id becomes NULL,
    // situations including:
    // 1. handle election_timeout and start pre_vote
    // 2. receive requests with higher term such as vote_request from a candidate
    // or append_entries_request from a new leader
    // 3. receive timeout_now_request from current leader and start request_vote
    // the parameter ctx gives the information(leader_id, term and status) about
    // the very leader whom the follower followed before.
    // User can reset the node's information as it stops following some leader.
    virtual void on_stop_following(const ::braft::LeaderChangeContext& ctx);

    // this method is called when a follower or candidate starts following a leader and its leader_id
    // (should be NULL before the method is called) is set to the leader's id,
    // situations including:
    // 1. a candidate receives append_entries from a leader
    // 2. a follower(without leader) receives append_entries from a leader
    // the parameter ctx gives the information(leader_id, term and status) about
    // the very leader whom the follower starts to follow.
    // User can reset the node's information as it starts to follow some leader.
    virtual void on_start_following(const ::braft::LeaderChangeContext& ctx);
};
```

阶段一：PreVote
===

![图 3.2  PreVote 整体流程](image/3.2.png)

触发投票
---

节点在启动时就会启动选举定时器：

```cpp
int NodeImpl::init(const NodeOptions& options) {
    ...
    // 只有当前节点的集群列表不为空，才会调用 step_down 启动选举定时器
    if (!_conf.empty()) {
        step_down(_current_term, false, butil::Status::OK());
    }
    ...
}

void NodeImpl::step_down(const int64_t term, bool wakeup_a_candidate,
                         const butil::Status& status) {
    ...
    _election_timer.start();  // 启动选举定时器
}
```

待定时器超时后就会调用 `pre_vote` 进行 `PreVote`：

```cpp
// 定时器超时的 handler
void ElectionTimer::run() {
    _node->handle_election_timeout();
}

void NodeImpl::handle_election_timeout() {
    ...
    reset_leader_id(empty_id, status);

    return pre_vote(&lck, triggered);
    // Don't touch any thing of *this ever after
}
```

发送请求
---

在 `pre_vote` 函数中会向所有节点发送 `PreVote` 请求，并设置 RPC 响应的回调函数为 `OnPreVoteRPCDone`，最后调用 `grant_slef` 给自己投一票后，就等待其他节点的 `PreVote` 响应：

```cpp
void NodeImpl::pre_vote(std::unique_lock<raft_mutex_t>* lck, bool triggered) {
    ...
    // (1) 如果当前节点已经被集群移除了，则不再参与选举
    if (!_conf.contains(_server_id)) {
        ...
        return;
    }
    ...
    // (2) 获取节点的 lastLog
    const LogId last_log_id = _log_manager->last_log_id(true);

    _pre_vote_ctx.init(this, triggered);
    std::set<PeerId> peers;
    _conf.list_peers(&peers);

    // (3) 向组内所有节点发送 `PreVote` 请求
    for (std::set<PeerId>::const_iterator
            iter = peers.begin(); iter != peers.end(); ++iter) {
        ...
        // (4) 设置回调函数
        OnPreVoteRPCDone* done = new OnPreVoteRPCDone(
                *iter, _current_term, _pre_vote_ctx.version(), this);
        ...
        done->request.set_term(_current_term + 1); // next term
        done->request.set_last_log_index(last_log_id.index);
        done->request.set_last_log_term(last_log_id.term);

        RaftService_Stub stub(&channel);
        stub.pre_vote(&done->cntl, &done->request, &done->response, done);
    }
    // (5) 给自己投一票
    grant_self(&_pre_vote_ctx, lck);
}
```

处理请求
---

其他节点在收到 `PreVote` 请求后会调用 `handle_pre_vote_request` 处理请求：

```cpp
int NodeImpl::handle_pre_vote_request(const RequestVoteRequest* request,
                                      RequestVoteResponse* response) {
    ...
    do {
        // (1) 判断 Term
        if (request->term() < _current_term) {
            ...
            break;
        }

        // (2) 判断 LastLogId
        ...
        LogId last_log_id = _log_manager->last_log_id(true);
        ...
        bool grantable = (LogId(request->last_log_index(), request->last_log_term())
                        >= last_log_id);
        if (grantable) {
            granted = (votable_time == 0);  // votable_time 是 Follower Lease 的特性，将在 <3.2 选举优化中> 详细介绍，这里可忽略
        }
        ...
    } while (0);

    // (3) 设置响应
    ...
    response->set_term(_current_term);  // 携带自身的 term
    response->set_granted(granted);  // true 代表投赞成票
    ...

    return 0;
}
```

处理响应
---

在收到其他节点的 `PreVote` 响应后，会调用之前设置的回调函数 `OnPreVoteRPCDone->Run()`，在回调函数中会调用 `handle_pre_vote_response` 处理 `PreVote` 响应：

```cpp
struct OnPreVoteRPCDone : public google::protobuf::Closure {
    ...
    void Run() {
            if (cntl.ErrorCode() != 0) {
                ...
                break;
            }
            node->handle_pre_vote_response(peer, term, ctx_version, response);
    }
    ...
};
```

处理响应，如果在处理响应后发现收到的选票数已达到 `Quorum`，则调用 `elect_self` 进行正式选举：

```cpp
void NodeImpl::handle_pre_vote_response(const PeerId& peer_id, const int64_t term,
                                        const int64_t ctx_version,
                                        const RequestVoteResponse& response) {
    ...
    // (1) 若发现有节点 term 比自己高，直接 step_down 成 Follower
    // check response term
    if (response.term() > _current_term) {
        ...
        butil::Status status;
        status.set_error(EHIGHERTERMRESPONSE, "Raft node receives higher term "
                "pre_vote_response.");
        step_down(response.term(), false, status);
        return;
    }

    // (2) 收到拒绝投票
    if (!response.granted() && !response.rejected_by_lease()) {
        return;
    }
    ...
    // (3) 增加一个选票
    _pre_vote_ctx.grant(*it);
    ...
    // (4) 如果收到的选票数达到 `Quorum`，则调用 `elect_self` 进行正式选举
    if (_pre_vote_ctx.granted()) {
        elect_self(&lck);
    }
}

```

<!--
TODO(Wine93):
投票失败
---
-->

阶段二：RequestVote
===

![图 3.3  RequestVote 整体流程](image/3.3.png)

发送请求
---

当 `PreVote` 阶段获得大多数节点的支持后，将调用 `elect_self` 正式进 `RequestVote` 阶段。在 `elect_self` 会将角色转变为 Candidte，并加自身的 `term` 加一，向所有的节点发送 `RequestVote` 请求，最后给自己投一票后，就等待其他节点的 `RequestVote` 响应：

```cpp
void NodeImpl::elect_self(std::unique_lock<raft_mutex_t>* lck,
                          bool old_leader_stepped_down) {
    ...
    // (1) 如果当前节点已经被集群移除了，则不再参与选举
    if (!_conf.contains(_server_id)) {
        ...
        return;
    }
    ...
    _state = STATE_CANDIDATE;  // (2) 将自身角色转变为 Candidate
    _current_term++;           // (3) 将自身的 term+1
    _voted_id = _server_id;    // (4) 记录 votedFor 投给自己

    ...
    // (5) 启动投票超时器：如果在 vote_timeout_ms 未得到足够多的选票，则变为 Follower 重新进行 PreVote
    _vote_timer.start();

    // (6) 获取 LastLog 的 index 和 term（LogId 由 index 和 term 组成）
    const LogId last_log_id = _log_manager->last_log_id(true);
    ...
    _vote_ctx.set_last_log_id(last_log_id);

    // (7) 向所有节点广播 `RequestVote` 请求
    std::set<PeerId> peers;
    _conf.list_peers(&peers);
    request_peers_to_vote(peers, _vote_ctx.disrupted_leader());

    // (8) 持久化 currentTerm，votedFor
    status = _meta_storage->
                    set_term_and_votedfor(_current_term, _server_id, _v_group_id);

    // (9) 给自己投一票
    grant_self(&_vote_ctx, lck);
}
```

`request_peers_to_vote` 负责向所有节点发送 `RequestVote` 请求：

```cpp
void NodeImpl::request_peers_to_vote(const std::set<PeerId>& peers,
                                     const DisruptedLeader& disrupted_leader) {
    // (1) 遍历所有节点，向其发送 `RequestVote` 请求
    for (std::set<PeerId>::const_iterator
        iter = peers.begin(); iter != peers.end(); ++iter) {
        ...

        // (2) 准备响应回调函数
        OnRequestVoteRPCDone* done =
            new OnRequestVoteRPCDone(*iter, _current_term, _vote_ctx.version(), this);
        ...
        // (3) 设置请求
        done->request.set_term(_current_term);
        done->request.set_last_log_index(_vote_ctx.last_log_id().index);
        done->request.set_last_log_term(_vote_ctx.last_log_id().term);

        // (4) 正式发送 RPC
        RaftService_Stub stub(&channel);
        stub.request_vote(&done->cntl, &done->request, &done->response, done);
    }
}
```

处理请求
---

节点在收到 `RequestVote` 请求后，会调用 `handle_request_vote_request` 处理 `RequestVote` 请求：

```cpp
int NodeImpl::handle_request_vote_request(const RequestVoteRequest* request,
                                          RequestVoteResponse* response) {
    ...
    do {
        // (1) 投票条件 1：候选人的 term 要大于或等于自己的 term
        // ignore older term
        if (request->term() < _current_term) {
            ...
            break;
        }

        // (2) 投票条件 2：候选人的最后一条日志要和自己的一样新或者新于自己
        LogId last_log_id = _log_manager->last_log_id(true);
        ...
        bool log_is_ok = (LogId(request->last_log_index(), request->last_log_term()) >= last_log_id);
        ...
        // (3) 如果发现有比自己 term 大的节点，则调用 step_down：
        //     (1) 转变为 Follower
        //     (2) 设置 _current_term 为 request->term()
        //     (3) 重置 votedFor 为空
        // increase current term, change state to follower
        if (request->term() > _current_term) {
            ...
            step_down(request->term(), false, status);
        }

        // (4) 投票条件 3：当前 term 没有给其他节点投过票
        if (log_is_ok && _voted_id.is_empty()) {
            ...
            step_down(request->term(), false, status);
            _voted_id = candidate_id;  // 记录 votedFor
            status = _meta_storage->   // 持久化 currentTerm 和 votedFor
                    set_term_and_votedfor(_current_term, candidate_id, _v_group_id);
        }
    } while (0);

    // (5) 设置响应
    ...
    response->set_term(_current_term);
    response->set_granted(request->term() == _current_term && _voted_id == candidate_id);
    ...

    return 0;
}
```

处理响应
---

候选人在收到其他节点的 `RequestVote` 响应后，会调用之前设置的回调函数 `OnRequestVoteRPCDone->Run()`，在回调函数中会调用 `handle_request_vote_response` 处理 `RequestVote` 响应：

```cpp
struct OnRequestVoteRPCDone : public google::protobuf::Closure {
    ...
    void Run() {
            if (cntl.ErrorCode() != 0) {
                ...
                break;
            }
            node->handle_request_vote_response(peer, term, ctx_version, response);
    }
    ...
};
```

处理响应，如果在处理响应后发现收到的选票数已达到 `Quorum`，则调用 `become_leader` 成为 Leader：

```cpp
void NodeImpl::handle_request_vote_response(const PeerId& peer_id, const int64_t term,
                                            const int64_t ctx_version,
                                            const RequestVoteResponse& response) {
    ...
    // (1) 发现有比自己 term 高的节点，则 step_down 成 Follower
    if (response.term() > _current_term) {
        ...
        step_down(response.term(), false, status);
        return;
    }
    ...
    // (2) 收到拒绝投票
    if (!response.granted() && !response.rejected_by_lease()) {
        return;
    }

    if (response.granted()) {
        // (3) 增加一个选票
        _vote_ctx.grant(peer_id);
        ...
        // (4) 如果收到的选票数达到 `Quorum`，则调用 `become_leader` 正式成为 Leader
        if (_vote_ctx.granted()) {
            return become_leader();
        }
    }
    ...
}
```

投票超时
---

`VoteTimer` 在节点开始正式投票（即调用 `elect_self`）就开始计时了。若在投票超时时间内未收到足够多的选票，`VoteTimer` 就会调用 `handle_vote_timeout` 将当前节点 `step_down` 并进行重新 PreVote；反之在收到足够多选票成为 Leader 后将会停止该 Timer：

```cpp
void VoteTimer::run() {
    _node->handle_vote_timeout();
}

void NodeImpl::handle_vote_timeout() {
    ...
    step_down(_current_term, false, status);
    pre_vote(&lck, false);
    ...
}
```

阶段三：成为 Leader
===

![图 3.4  become_leader 整体流程](image/3.4.png)

## 成为 Leader

节点在 `RequestVote` 阶段收到足够多的选票后，会调用 `become_leader` 正式成为 Leader，在该函数中主要执行成为 Leader 前的准备工作，特别需要注意的是，只有当这些工作全部完成后才会回调用户状态机的 `on_leader_start`，每一项工作将在下面的小节中逐一介绍：

```cpp
// in lock
void NodeImpl::become_leader() {
    ...
    // cancel candidate vote timer
    _vote_timer.stop();  // (1) 停止投票计时器
    ...
    _state = STATE_LEADER;  // (2) 将自身角色转变为 Leader
    _leader_id = _server_id;  // (3) 记录当前 leaderId 为自身的 PeerId
    ...
    // (3) 为所有 Follower 创建 Replicator，Replicator 主要负责向 Follower 发送心跳、日志等
    std::set<PeerId> peers;
    _conf.list_peers(&peers);
    for (std::set<PeerId>::const_iterator
            iter = peers.begin(); iter != peers.end(); ++iter) {
        ...
        _replicator_group.add_replicator(*iter);
    }

    // (4) 设置最小可以提交的 logIndex，在这之前的日志就算复制达到了 Quorum，也不会更新 commitIndex
    //     注意：这是实现只复制但不提交上一任期日志的关键
    // init commit manager
    _ballot_box->reset_pending_index(_log_manager->last_log_index() + 1);

    // (5) 复制并提交本一条任期的配置日志
    // Register _conf_ctx to reject configuration changing before the first log
    // is committed.
    _conf_ctx.flush(_conf.conf, _conf.old_conf);

    // (6) 启动 StepdownTimer，用于实现 Check Quorum 优化
    //     详见 <3.2 选举优化> 小节
    _stepdown_timer.start();
}
```

创建 Replicator
---

Leader 会为每个 Follower 创建对应 `Replicator`，并将其启动。每个 `Replicator` 都是单独的 `bthread`，它主要有以下 3 个作用：

* 记录 Follower 的一些状态，比如 `nextIndex`、`flyingAppendEntriesSize` 等
* 作为 RPC Client，所有从 Leader 发往 Follower 的 RPC 请求都会通过它，包括心跳、`AppendEntriesRequest`、`InstallSnapshotRequest`
* 同步日志：`Replicator` 会不断地向 Follower 同步日志，直到 Follower 成功复制了 Leader 的所有日志后，其会在后台等待新日志的到来

调用 `Replicator::start` 来创建 `Replicator`，并将其启动：

```cpp
int ReplicatorGroup::add_replicator(const PeerId& peer) {
    ...
    if (Replicator::start(options, &rid) != 0) {
        ...
    }
    ...
}

int Replicator::start(const ReplicatorOptions& options, ReplicatorId *id) {
    // (1) 创建 Replicator
    Replicator* r = new Replicator();

    // (2) 初始化与 Follower 的 channel
    //     注意：这里的 channel 的 timeout_ms 为 -1
    brpc::ChannelOptions channel_opt;
    channel_opt.connect_timeout_ms = FLAGS_raft_rpc_channel_connect_timeout_ms;
    channel_opt.timeout_ms = -1; // We don't need RPC timeout
    if (r->_sending_channel.Init(options.peer_id.addr, &channel_opt) != 0) {
        ...
        return -1;
    }

    // (4）初始化 Follower 的 nextIndex，该值将在步骤 7 中将被矫正
    r->_next_index = r->_options.log_manager->last_log_index() + 1;

    // (5) 创建 bthread 运行 Replicator
    if (bthread_id_create(&r->_id, r, _on_error) != 0) {
        ...
        return -1;
    }

    // (6) 启动心跳定时器，其会每隔 100 毫秒向 Follower 发送心跳
    r->_start_heartbeat_timer(butil::gettimeofday_us());

    // (7) 向 Follower 发送空的 AppendEntries 请求来确认 nextIndex
    r->_send_empty_entries(false);
    return 0;
}
```

定时发送心跳
---

`Replicator` 调用 `_start_heartbeat_timer` 启动心跳定时器，其每隔一段时间会发送 `ETIMEDOUT` 状态码，而 `Replicator` 收到该状态码后，会调用 `_send_heartbeat` 发送心跳。

调用 `_start_heartbeat_timer` 启动心跳定时器，心跳间隔时间在节点初始化时通过 `heartbeat_timeout` 函数算得：
```cpp
void Replicator::_start_heartbeat_timer(long start_time_us) {
    const timespec due_time = butil::milliseconds_from(
            butil::microseconds_to_timespec(start_time_us),
            *_options.dynamic_heartbeat_timeout_ms);

    // 增加 timer，其 handler 为 _on_timedout
    if (bthread_timer_add(&_heartbeat_timer, due_time,
                       _on_timedout, (void*)_id.value) != 0) {
        ...
    }
}

// 计算心跳间隔，默认为 election_timeout_ms / 10
static inline int heartbeat_timeout(int election_timeout) {
    if (FLAGS_raft_election_heartbeat_factor <= 0){
        ...
        FLAGS_raft_election_heartbeat_factor = 10;
    }
    return std::max(election_timeout / FLAGS_raft_election_heartbeat_factor, 10);
}
```

定时器会每隔一段时间向 `Replicator` 发送 `ETIMEDOUT` 状态码：
```cpp
void Replicator::_on_timedout(void* arg) {
    bthread_id_t id = { (uint64_t)arg };
    bthread_id_error(id, ETIMEDOUT);
}
```

`Replicator` 收到该状态码后，会调用 `_send_heartbeat` 发送心跳：

```cpp
int Replicator::_on_error(bthread_id_t id, void* arg, int error_code) {
    Replicator* r = (Replicator*)arg;
    if (error_code == ESTOP) {
        ...
    } else if (error_code == ETIMEDOUT) {
        if (bthread_start_urgent(&tid, NULL, _send_heartbeat,
                                 reinterpret_cast<void*>(id.value)) != 0) {
            ...
        }
        return 0;
    }
    ...
}

void* Replicator::_send_heartbeat(void* arg) {
    ...
    r->_send_empty_entries(true);  // 发送空的 `AppendEntries` 请求
    return NULL;
}
```

确定 nextIndex
---

Leader 通过发送空的 `AppendEntries` 请求来探测 Follower 的 `nextIndex`，
只有确定了 `nextIndex` 才能正式向 Follower 发送日志。这里忽略了很多细节，关于 `nextIndex` 的作用和匹配算法，以及相关实现可参考 [4.1 日志复制](/ch04/4.1/replicate.md)中的相关内容：
* [nextIndex](/ch04/4.1/replicate.md#nextindex)
* [具体实现](/ch04/4.1/replicate.md#qian-zhi-jie-duan-que-ding-nextindex)

```cpp
void Replicator::_send_empty_entries(bool is_heartbeat) {
    ...
    // (1) 设置响应回调函数为 _on_rpc_returned
    google::protobuf::Closure* done = brpc::NewCallback(
                is_heartbeat ? _on_heartbeat_returned : _on_rpc_returned, ...);
    ...
    // (2) 发送空的 AppendEntries 请求用于探测 nextIndex
    RaftService_Stub stub(&_sending_channel);
    stub.append_entries(cntl.release(), request.release(),
                        response.release(), done);
    ...
}

void Replicator::_on_rpc_returned(ReplicatorId id, brpc::Controller* cntl,
                     AppendEntriesRequest* request,
                     AppendEntriesResponse* response,
                     int64_t rpc_send_time) {
    ...
    // (3) 更新 Follower 的 nextIndex
    r->_next_index = response->last_log_index() + 1;
    ...
}
```

复制之前任期日志
---

上述我们已经为每一个 Follower 创建了 `Replicator`，并且确认了每个 Follower 的 `nextIndex`，这时候 `Replicator` 通过 `nextIndex` 判断 Follower 日志还落后于 Leader，将自动向 Follower 同步日志，直至与 Leader 对齐为止。

注意，这些日志只复制并不提交。通常情况下，Leader 每向一个 Follower 成功复制日志后，都会调用 `BallotBox::commit_at` 将对应日志的投票数加一，当投票数达到 `Quorum` 时，Leader 就会更新 `commitIndex`，并应用这些日志。

节点在刚成为 Leader 时通过调用以下函数将第一条可以提交的 `logIndex` （即 `_pending_index`）设为了 Leader 的 `lastLogIndex+1`：

```cpp
void NodeImpl::become_leader() {
    ...
    _ballot_box->reset_pending_index(_log_manager->last_log_index() + 1);
    ...
}

int BallotBox::reset_pending_index(int64_t new_pending_index) {
    ...
    _pending_index = new_pending_index;
    ...
}
```

在调用 `commit_at` 函数时，只有 `logIndex>=_pending_index` 的日志才能被提交：
```cpp
// 将 index 在 [fist_log_index, last_log_index] 之间的日志的投票数加一
int BallotBox::commit_at(
        int64_t first_log_index, int64_t last_log_index, const PeerId& peer) {
    ...
    // (1) 如果在 _pending_index 之前的日志将无法被提交
    if (last_log_index < _pending_index) {
        return 0;
    }
    ...

    // (2) 在这之后的日志可以正常计算 `Quorum`
    int64_t last_committed_index = 0;
    const int64_t start_at = std::max(_pending_index, first_log_index);
    Ballot::PosHint pos_hint;
    for (int64_t log_index = start_at; log_index <= last_log_index; ++log_index) {
        Ballot& bl = _pending_meta_queue[log_index - _pending_index];
        pos_hint = bl.grant(peer, pos_hint);
        if (bl.granted()) {
            last_committed_index = log_index;
        }
    }

    if (last_committed_index == 0) {
        return 0;
    }
    ...
    _pending_index = last_committed_index + 1;
    // (3) 更新 commitIndex
    _last_committed_index.store(last_committed_index, butil::memory_order_relaxed);
    // (4) 调用 FSMCaller::do_committed 开始应用日志
    // The order doesn't matter
    _waiter->on_committed(last_committed_index);
    return 0;
}
```

这里可能忽略了一些日志复制的细节，你可以参考 [4.1 日志复制](/ch04/4.1/replicate.md)中的相关内容。

提交 no-op 日志
---

一般 Raft 实现会在节点当选 Leader 后提交一条本任期的 `no-op` 日志，而 braft 中提交的是本任期的配置日志。在节点刚成为 Leader 时会调用 `ConfigurationCtx::flush` 复制并提交配置日志：

```cpp
void NodeImpl::become_leader() {
    ...
    _conf_ctx.flush(_conf.conf, _conf.old_conf);
    ...
}

void NodeImpl::ConfigurationCtx::flush(const Configuration& conf,
                                       const Configuration& old_conf) {
    ...
    _stage = STAGE_STABLE;
    _old_peers = _new_peers;
    ...
    _node->unsafe_apply_configuration(conf, old_conf.empty() ? NULL : &old_conf,
                                      true);

}
```

`ConfigurationCtx::flush` 会调用 `unsafe_apply_configuration` 函数来做以下几件事：
```cpp
void NodeImpl::unsafe_apply_configuration(const Configuration& new_conf,
                                          const Configuration* old_conf,
                                          bool leader_start) {
    ...
    // (1) 设置日志应用后的回调函数，
    //     即该配置日志被复制并成功应用（调用 on_on_configuration_committed）后
    //     就会调用该回调函数
    ConfigurationChangeDone* configuration_change_done =
            new ConfigurationChangeDone(this, _current_term, leader_start, _leader_lease.lease_epoch());
    // Use the new_conf to deal the quorum of this very log
    _ballot_box->append_pending_task(new_conf, old_conf, configuration_change_done);

    // (2) 将配置日志追加到 LogManager，其会唤醒 Replicator 向 Follower 同步日志
    std::vector<LogEntry*> entries;
    entries.push_back(entry);
    _log_manager->append_entries(&entries,
                                 new LeaderStableClosure(
                                        NodeId(_group_id, _server_id),
                                        1u, _ballot_box));
    // (3) 将配置设为当前节点配置
    _log_manager->check_and_set_configuration(&_conf);
}
```

## on_apply
## on_configuration_committed

上面已经提到将配置日志交给 `LogManager` 进行复制，待其复制达到 `Quorum` 后，才会更新 `commitIndex`，并会调用 `FSMCaller::do_committed` 开始应用日志，参见上述的 `BallotBox::commit_at` 函数。

当然应用的日志包括之前任期的日志和本任期的配置日志。如果日志类型是配置，则调用状态机的 `on_configuration_committed`，否则回调 `on_apply`：

```cpp
void FSMCaller::do_committed(int64_t committed_index) {
    ...
    int64_t last_applied_index = _last_applied_index.load(
                                        butil::memory_order_relaxed);

    // We can tolerate the disorder of committed_index
    if (last_applied_index >= committed_index) {
        return;
    }
    ...
    IteratorImpl iter_impl(_fsm, _log_manager, &closure, first_closure_index,
                 last_applied_index, committed_index, &_applying_index);
    for (; iter_impl.is_good();) {
            ...
            // (1) 如果是配置日志，则调用状态机的 `on_configuration_committed`
            if (iter_impl.entry()->type == ENTRY_TYPE_CONFIGURATION) {
                    ...
                    _fsm->on_configuration_committed(
                            Configuration(*iter_impl.entry()->peers),
                            iter_impl.entry()->id.index);
                }
            }
            ...
            // (1.1) 其待被应用后，调用回调函数，即 `ConfigurationChangeDone::Run`
            if (iter_impl.done()) {
                iter_impl.done()->Run();
            }
            iter_impl.next();
            continue;
        }
        ...
        // (2) 如果是普通日志，则调用状态机的 `on_apply`
        Iterator iter(&iter_impl);
        _fsm->on_apply(iter);
        ...
        iter.next();
    }
    // (3) 更新 applyindex
    _last_applied_index.store(committed_index, butil::memory_order_release);
}
```

on_leader_start
---

在配置日志被应用（即调用 `on_configuration_committed`）后，其会调用其回调函数 `ConfigurationChangeDone::Run()`，在该函数中会调用状态机的 `on_leader_start` 开启 Leader 任期：

```cpp
class ConfigurationChangeDone : public Closure {
public:
    void Run() {
        ...
        // 回调用户的状态机的 on_leader_start，_term 为当前 Leader 的 Term
        if (_leader_start) {
            _node->_options.fsm->on_leader_start(_term);
        }
        ...
    }
    ...
};
```