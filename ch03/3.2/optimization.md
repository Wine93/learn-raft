概览
===

braft 在实现 Leader 选举的时候做了一些优化，这些优化点归纳起来主要为了实现以下几个目的：

* **快速**：减少选举时间，让集群尽快产生 Leader（如超时时间随机化、Wakeup Candidate）
* **稳定**：当集群中有 Leader 时，尽可能保持稳定，减少没必要的选主（如 PreVote、Follower Lease）
* **分区问题**：解决出现分区时造成的各种问题（如 PreVote、Follower Lease、Check Quorum、Leader Lease）

具体来说，braft 对于分区场景做了很详细的优化：

| 分区场景                        | 造成问题                                                                       | 优化方案                   |
|:--------------------------------|:-------------------------------------------------------------------------------|:---------------------------|
| Follower 被隔离于对称网络分区   | `term` 增加，重新加入集群会打断 Leader，造成没必要的选举                       | PreVote                    |
| Follower 被隔离于非对称网络分区 | Follower 选举成功，推高集群其他成员的 `term`，间接打断 Leader，造成没必要选举  | Follower Lease             |
| Leader 被隔离于对称网络分区     | 集群会产生多个 Leader，客户端需要重试；可能会产生 `Stale Read`，破坏线性一致性 | Check Quorum、Leader Lease |
| Leader 被隔离于非对称网络分区   | Leader 永远无法写入，也不会产生新 Leader，客户端会不断重试                     | Check Quorum               |

优化 1：超时时间随机化
===

瓜分选票
---

如果多个成员在等待 `election_timeout_ms` 后同时触发选举，可能会造成选票被瓜分，导致无法选出 Leader，从而需要触发下一轮选举。为此，可以将 `election_timeout_ms` 随机化，减少选票被瓜分的可能性。但是在极端情况下，多个成员可能拥有相同的 `election_timeout_ms`，仍会造成选票被瓜分，为此 braft 将 `vote_timeout_ms` 也进行了随机化。双层的随机化可以很大程度降低选票被瓜分的可能性。

> **vote_timeout_ms:**
>
> 如果成员在该超时时间内没有得到足够多的选票，将变为 Follower 并重新发起选举，无需等待 `election_timeout_ms`

![图 3.5  相同的选举超时时间](image/3.5.png)

具体实现
---

随机函数：
```cpp
// FLAGS_raft_max_election_delay_ms 默认为 1000
inline int random_timeout(int timeout_ms) {
    int32_t delta = std::min(timeout_ms, FLAGS_raft_max_election_delay_ms);
    return butil::fast_rand_in(timeout_ms, timeout_ms + delta);
}
```

`ElectionTimer` 相关逻辑:

```cpp
// timeout_ms：1000
// 产生的随机时间为：[1000,2000]
int ElectionTimer::adjust_timeout_ms(int timeout_ms) {
    return random_timeout(timeout_ms);
}

void ElectionTimer::run() {
    _node->handle_election_timeout();
}

// Timer 超时后的处理函数
void NodeImpl::handle_election_timeout() {
    ...
    return pre_vote(&lck, triggered);
}
```

`VoteTimer` 相关逻辑:
```cpp
// timeout_ms：2000
// 产生的随机时间为：[2000,3000]
int VoteTimer::adjust_timeout_ms(int timeout_ms) {
    return random_timeout(timeout_ms);
}

void VoteTimer::run() {
    _node->handle_vote_timeout();
}

// Timer 超时后的处理函数
void NodeImpl::handle_vote_timeout() {
    ...
    // 该参数作用可参见 issue: https://github.com/baidu/braft/issues/86
    if (FLAGS_raft_step_down_when_vote_timedout) {  // 默认为 true
        ...
        step_down(_current_term, false, status);  // 先降为 Follower
        pre_vote(&lck, false);  // 再进行 PreVote
    }
    ...
}
```

优化 2：Wakeup Candidate
===

Leader 正常退出
---

当一个 Leader 正常退出时，它会选择一个日志最长的 Follower，向其发送 `TimeoutNow` 请求，而收到该请求的 Follower 无需等待超时，会立马变为 Candidate 进行选举（无需 `PreVote`）。这样可以缩短集群中没有 Leader 的时间，增加系统的可用性。特别地，在日常运维中，升级服务版本需要暂停 Leader 所在服务时，该优化可以在无形中帮助我们。

具体实现
---

Leader 服务正常退出，会向 Follower 发送 `TimeoutNow` 请求：

```cpp
void NodeImpl::shutdown(Closure* done) {  // 服务正常退出会调用 shutdown
    ...
    step_down(_current_term, _state == STATE_LEADER, status);
    ...
}

void NodeImpl::step_down(const int64_t term, bool wakeup_a_candidate,
                         const butil::Status& status) {
    ...
    // (1) 调用状态机的 `on_leader_stop`
    if (_state == STATE_LEADER) {
        _fsm_caller->on_leader_stop(status);
    }
    ...
    // (2) 转变为 Follower
    _state = STATE_FOLLOWER;
    ...
    // (3) 选择一个日志最长的 Follower 作为 Candidate，向其发送 TimeoutNowRequest
    if (wakeup_a_candidate) {
        _replicator_group.stop_all_and_find_the_next_candidate(
                                            &_waking_candidate, _conf);
        Replicator::send_timeout_now_and_stop(
                _waking_candidate, _options.election_timeout_ms);
    }
    ...
}
```

Follower 收到 `TimeoutNow` 请求，会调用 `handle_timeout_now_request` 进行处理：

```cpp
void NodeImpl::handle_timeout_now_request(brpc::Controller* controller,
                                          const TimeoutNowRequest* request,
                                          TimeoutNowResponse* response,
                                          google::protobuf::Closure* done) {
    ...
    response->set_success(true);
    elect_self(...);  // 调用 elect_self 函数转变为 Candidate，并发起选举
    ...
}
```

优化 3：PreVote
===

对称网络分区
---

![图 3.6  Follower 被隔离于对称网路分区](image/3.6.png)

我们考虑上图中在网络分区中发生的一种现象：

* (a)：节点 `S1` 被选为 `term 1` 的 Leader，正常向 `S2,S3` 发送心跳
* (b)：发生网络分区后，节点 `S2` 收不到 Leader `S1` 的心跳，在等待 `election_timeout_ms` 后触发选举，进行选举时会将角色转变为 Candidate，并将自身的 `term` 加一后广播 `RequestVote` 请求；然而由于分区收不到足够的选票，在 `vote_timeout_ms` 后宣布选举失败从而触发新一轮的选举；不断的选举致使 `S2` 的 `term` 不断增大
* (c)：当网络恢复后，Leader `S1` 得知 `S2` 的 `term` 比其大，将会自动降为 Follower，从而触发了重新选主。值得一提的是，Leader `S1` 有多种渠道得知 `S2` 的 `term` 比其大，因为节点之间所有的 RPC 请求与响应都会带上自身的 `term`，所以可能是 `S2` 选举发出的 `RequestVote` 请求，也可能是 `S2` 响应 `S1` 的心跳请求，这取决于第一个发往 Leader `S1` 的 RPC 包

从上面可以看到，当节点 `S2` 重新回归到集群时，由于其 `term` 比 Leader 大，致使 Leader 降为 Follower，从而触发重新选主。而 `S2` 大概率不会赢得选举，最终的 Leader 可能还是 `S1`，因为在节点 `S2` 被隔离的这段时间，`S1,S3` 有新的日志写入，导致 `S1,S3` 的日志都要比 `S2` 新，所以这么看，这是一次没有必要的选举。

为了解决这一问题，Raft 在正式请求投票前引入了 `PreVote` 阶段，该阶段不会增加自身 `term`，并且节点需要在 `PreVote` 获得足够多的选票才能正式进入 `RequestVote` 阶段。

PreVote
---

关于 `PreVote` 的作用以及其与 `RequestVote` 之间的区别，我们已经在[<3.1 选举流程>](/ch03/3.1/election.md)中介绍过了，参见[投票规则](/ch03/3.1/election.md#tou-piao-gui-ze)。

具体实现
---

具体实现我们也在[<3.1 选举流程>](/ch03/3.1/election.md)中已经详细解析过了，参见：

* [第一阶段：PreVote](/ch03/3.1/election.md#jie-duan-yi-prevote)
* [第二阶段：RequestVote](/ch03/3.1/election.md#jie-duan-er-requestvote)

优化 4：Follower Lease
===

上面我们提到了 `PreVote` 优化可以阻止在对称网络分区情况下，节点重新加入集群干扰集群的场景，下面会介绍在非对称网络下，`PreVote` 无法阻止的一些场景。

非对称网络分区
---

下图描述这些场景都有一些前提：

* 图中灰色节点因分区或其他原因收不到心跳而发起选举
* 在发起选举的短时间内，各成员拥有相同的日志，此时上层无写入

![图 3.7  一些 PreVote 无法阻止的干扰集群场景](image/3.7.png)

**场景 (a)：**

* 节点 `S1` 为 `term 1` 的 Leader；集群出现非对称网络分区，只有 `S1` 与 `S2` 之间无法通信
* `S2` 因分区收不到 Leader `S1` 的心跳而触发选举；`S2` 在 `PreVote` 阶段获得 `S2,S3` 的同意
* `S2` 将 `term` 变为 2，并在 `ReuquestVote` 阶段获得 `S2,S3` 同意被选为 `term 2` 的 Leader
* Leader `S2` 向 `S3` 发送心跳，致使 `S3` 将自身 `term` 变为 2
* `S1` 发现 `S3` 的 `term` 比自己高，遂降为 Follower，从而触发 `S1,S3` 这个分区的重新选举

**场景 (b)：**

该场景主要描述的是通过配置变更后，被移除的节点干扰集群的场景，由于这里涉及配置变更相关的流程，所以该场景我们将在[<6.1 配置变更>](/ch06/6.1/configuration_change.md)中详细介绍，参见[干扰集群](/ch06/6.1/configuration_change.md#gan-rao-ji-qun)。

**场景 (c)：**

* 节点 `S1` 为 `term 1` 的 Leader
* 由于网络间歇丢包，导致 `S2` 未能在 `election_timeout_ms` 时间内收到心跳，从而触发选举；`S2` 在 `PreVote` 阶段收到 `S2,S3` 的同意
* `S2` 将 `term` 变为 2，并在 `ReuquestVote` 阶段获得 `S2,S3` 同意被选为 `term 2` 的 Leader
* `S1` 因得知集群内有节点 `term` 比它高，遂降为 Follower

从以上三个场景可以看出，其实这些选举都是没必要的，因为保持原来的 Leader 依然可以让集群正常工作。

Follower Lease
---

为了解决这个问题，braft 引入了 `Follower Lease` 的特性。当 Follower 认为 Leader 还存活的时候，在 `election_timeout_ms` 时间内就不会投票给别的节点。Follower 每次收到 Leader 的心跳或 `AppendEntries` 请求就会更新 Lease，该 Lease 有效时间区间如下：

![图 3.8  Follower Lease 有效时间区间](image/3.8.png)

> **关于 max_clock_drift**
>
> 需要注意的是，Follower Lease 其实是下面提到的 Leader Lease 特性的一部分。之所以引入 `max_clock_drift`，主要是考虑时钟漂移的问题，关于这一点我们将在下面 Leader Lease 一节中详细介绍。单就 Follower Lease 这个特性来说，加不加该偏移值都不会影响其正确性。

转移 Leader
---

当我们手动转移 Leader 时，希望集群内能尽快产生新 Leader，而 Follower Lease 的存在可能导致一些节点不会投票给新的候选人，直至 Lease 过期。 为此 braft 在实现时做了一个优化，其大致流程如下：

* 当 Leader 转移领导权给候选人时，候选人会立即发起选举（跳过 `PreVote`）
* 当候选人收到老 Leader 的投票后，会在 `RequestVote` 的请求中带上该 Leader 的 Id（即 `disrupted_leader`），并对那些因 Lease 而拒绝的节点重发 `RequestVote` 请求
* 所有节点在收到 `RequestVote` 请求后，如果发现请求中带有 `disrupted_leader` 并且`disrupted_leader` 等于之前 Follower Lease 跟随的 Leader，则将当前 Lease 作废并投赞成票

总的来说，如果候选人的领导权是由老 Leader 交接而来，其他跟随老 Leader 的节点就可以放弃 Lease 安心投票，让集群尽快产生 Leader。具体细节以及相关实现见 [PR#262](https://github.com/baidu/braft/pull/262)。

具体实现
---

在节点启动时会初始化 Lease：

```cpp
// 在节点启动时初始化 Lease：
int NodeImpl::init(const NodeOptions& options) {
    ...
    _follower_lease.init(options.election_timeout_ms, options.max_clock_drift_ms);
   ...
}

void FollowerLease::init(int64_t election_timeout_ms, int64_t max_clock_drift_ms) {
    _election_timeout_ms = election_timeout_ms;
    _max_clock_drift_ms = max_clock_drift_ms;

    // When the node restart, we are not sure when the lease will be expired actually,
    // so just be conservative.
    _last_leader_timestamp = butil::monotonic_time_ms();
}
```

在收到 Leader 的心跳或者 `AppendEntries` 请求时更新 Lease：

```cpp
void NodeImpl::handle_append_entries_request(brpc::Controller* cntl,
                                             const AppendEntriesRequest* request,
                                             AppendEntriesResponse* response,
                                             google::protobuf::Closure* done,
                                             bool from_append_entries_cache) {
    ...
    if (!from_append_entries_cache) {
        // Requests from cache already updated timestamp
        _follower_lease.renew(_leader_id);
    }
    ...
}

// 更新 last_leader_timestamp
void FollowerLease::renew(const PeerId& leader_id) {
    _last_leader = leader_id;
    _last_leader_timestamp = butil::monotonic_time_ms();
}
```

如果还在租约内，代表 Leader 还存活着，Follower 就不会给其他节点投赞成票，包括 `PreVote` 和 `RequestVote`：

```cpp
// 处理 PreVote 请求
int NodeImpl::handle_pre_vote_request(const RequestVoteRequest* request,
                                      RequestVoteResponse* response) {
    ...
    bool rejected_by_lease = false;
    int64_t votable_time = _follower_lease.votable_time_from_now();
    if (grantable) {
        rejected_by_lease = (votable_time > 0);  // 大于 0 代表还在租约内，则拒绝投票
    }
    ...
    response->set_rejected_by_lease(rejected_by_lease);
    ...
}

// 处理 RequestVote 请求
int NodeImpl::handle_request_vote_request(const RequestVoteRequest* request,
                                          RequestVoteResponse* response) {
    // 如果当前候选人是老 Leader 转移过来，我们可以放弃 Lease，投赞成票
    PeerId disrupted_leader_id;
    if (_state == STATE_FOLLOWER &&
            request->has_disrupted_leader() &&
            _current_term == request->disrupted_leader().term() &&
            0 == disrupted_leader_id.parse(request->disrupted_leader().peer_id()) &&
            _leader_id == disrupted_leader_id) {
        // The candidate has already disrupted the old leader, we
        // can expire the lease safely.
        _follower_lease.expire();
    }
    ...
    bool rejected_by_lease = false;
    ...
    int64_t votable_time = _follower_lease.votable_time_from_now();
    // if the vote is rejected by lease, tell the candidate
    if (votable_time > 0) {  // 大于 0 代表还在租约内，则拒绝投票
        rejected_by_lease = log_is_ok;
        break;
    }
    ...
    response->set_rejected_by_lease(rejected_by_lease);
    ...
}

/*
 * Lease 剩余的时间
 *  =0: 不在 Lease 内，可以投票
 *  >0: 还在 Lease 内，不允许投票
 */
int64_t FollowerLease::votable_time_from_now() {
    ...
    int64_t now = butil::monotonic_time_ms();
    int64_t votable_timestamp = _last_leader_timestamp + _election_timeout_ms +
                                _max_clock_drift_ms;
    if (now >= votable_timestamp) {
        return 0;
    }
    return votable_timestamp - now;
}

// 放弃 Lease
void FollowerLease::expire() {
    _last_leader_timestamp = 0;
}
```

优化 5：Check Quorum
===

前面我们讨论了 Follower 被隔离于网络分区时出现的问题以及相关的优化，现在我们讨论下 Leader 被隔离于网络分区下的场景。

网络分区
---

![图 3.9  Leader 被隔离于网路分区](image/3.9.png)

**场景 (a)：**

* 集群出现非对称网络分区：Leader `S1` 可以向 Follower `S2,S3` 发送请求，但是却接受不到 Follower 的响应
* Follower `S2,S3` 可以一直接收到心跳，所以不会发起选举
* 由于 Leader `S1` 向 Follower `S2,S3` 发送的复制日志请求，一直收不到 `ACK`，导致日志永远无法被 commit；从而导致 Leader 一直不断的重试复制（需要注意的是，日志复制是没有超时的）
* 客户端因写入操作超时，不断发起重试

从上面可以看出，这种场景下，集群将永远无法写入数据，这是比较危险的。

**场景 (b)：**

* 节点 `S1` 为 `term 1` 的 Leader，`S2,S3` 为 Follower
* 集群出现对称网络分区，Leader 与 Follower 之间的网络不通
* Follower `S3` 因收不到 Leader 的心跳发起选举，被 `S2,S3` 选为 `term 2` 的 Leader
* 客户端在老 Leader `S1` 上写入超时；刷新 Leader，向新 Leader `S3` 发起重试，写入成功；

从以上流程可以看出，该场景会导致客户端的重试；如果客户端比较多的话，需要全部都刷新一遍 Leader。

Check Quorum
---

为了解决上述的提到的问题，Raft 引入了 `Check Quorum` 的机制，当节点成为 Leader 后，每隔 `election_timeout_ms` 时间内检查 Follower 的存活情况，如果发现少于 `Quorum` 节点存活，Leader 将主动降为 Follower。

具体实现
---

当节点成为 Leader 时，会启动 `StepdownTimer`，该 Timer 每个 `election_timeout_ms` 运行运行一次：
```cpp
// 成为 Leader 后启动 StepdownTimer：
void NodeImpl::become_leader() {
    ...
    _stepdown_timer.start();
}

// Timer 的超时时间是在 init 函数中初始化的
int NodeImpl::init(const NodeOptions& options) {
    ...
    CHECK_EQ(0, _stepdown_timer.init(this, options.election_timeout_ms));
    ...
}
```

Timer 超时后，会调用 `check_dead_nodes` 检测节点的存活情况，如果少于一半的节点存活，则主动降为 Follower：

```cpp
void StepdownTimer::run() {
    _node->handle_stepdown_timeout();
}

void NodeImpl::handle_stepdown_timeout() {
    ...
    int64_t now = butil::monotonic_time_ms();
    check_dead_nodes(_conf.conf, now);
    ...
}

void NodeImpl::check_dead_nodes(const Configuration& conf, int64_t now_ms) {
    std::vector<PeerId> peers;
    conf.list_peers(&peers);
    size_t alive_count = 0;
    Configuration dead_nodes;  // for easily print

    // (1) 统计存活的节点
    for (size_t i = 0; i < peers.size(); i++) {
        if (peers[i] == _server_id) {
            ++alive_count;
            continue;
        }

        // 获取每个节点最后一次发送 RPC 的时间戳，进行对比
        // 注意 last_rpc_send_timestamp 是 Leader 发出去的时间戳，
        // 但只有当节点返回响应时才会更新，见下面代码片段
        if (now_ms - _replicator_group.last_rpc_send_timestamp(peers[i])
                <= _options.election_timeout_ms) {
            ++alive_count;
            continue;
        }
        dead_nodes.add_peer(peers[i]);
    }

    // (2) 若存活的节点超过一半，则通过检测
    if (alive_count >= peers.size() / 2 + 1) {
        return;
    }

    // (3) 否则 step_down 成 Follower
    butil::Status status;
    status.set_error(ERAFTTIMEDOUT, "Majority of the group dies");
    step_down(_current_term, false, status);
}
```

更新节点的 `last_rpc_send_timestamp`:
```cpp
// 发送心跳：
void Replicator::_send_empty_entries(bool is_heartbeat) {
    ...
    // 发送的时候保存当时的时间戳： butil::monotonic_time_ms()
    // 返回响应的时候回调 _on_heartbeat_returned 才更新
    google::protobuf::Closure* done = brpc::NewCallback(
                is_heartbeat ? _on_heartbeat_returned : _on_rpc_returned,
                _id.value, cntl.get(), request.get(), response.get(),
                butil::monotonic_time_ms());
    ...
    RaftService_Stub stub(&_sending_channel);
    stub.append_entries(cntl.release(), request.release(),
                        response.release(), done);
}

// 心跳响应：参数 rpc_send_time 为上述创建 Closure 时保存
void Replicator::_on_heartbeat_returned(
        ReplicatorId id, brpc::Controller* cntl,
        AppendEntriesRequest* request,
        AppendEntriesResponse* response,
        int64_t rpc_send_time) {
    ...
    // 更新 last_rpc_send_timestamp
    r->_update_last_rpc_send_timestamp(rpc_send_time);
    ...
}
```

优化 6：Leader Lease
===

违背线性一致性
---

![图 3.10  分区导致系统违背线性一致性](image/3.10.png)

我们考虑上图所示的场景：

* 节点 `S1` 被 `S2,S3` 选为 `term 1` 的 `Leader`
* 由于集群发生网络分区，`S3` 收不到 `S1` 的心跳，发起选举，被 `S2,S3` 选为 `term 2` 的 Leader
* 此时 `Client 1` 往新 Leader `S3` 写入数据（`x=1`），并返回成功
* 在此之后，`Client 2` 从老 Leader `S1` 读取 `x` 的值，得到的将是 `0`

从上面可以看到，`Client 1` 在 `Client 2` 写入后读到的依旧是老数据，这违背了线性一致性。当然，上面提到的 `Check Quorum` 机制可以减少老 Leader `S1` 存在的时间，但是不能完全避免。

> 值得一提的是，对于某些数据不共享的系统，即对某块数据的读写只由单个客户端负责的模型，可以不用考虑该场景；因为其读写都会在同一个 Leader 上，所以不会违背线性一致性。

线性一致性读
---

产生 `Stale Read` 的原因是产生分区后，负责读取的 Leader 已经不是最新的 Leader 了。为了解决上面提到的 `Stale Read`，Raft 提出了以下 3 种读取方式：

> 注意：下图方案图中标记的延时在计算时都忽略了 `read local` 这个步骤，因为这取决于具体的状态机实现

**方案一：Raft Log Read**

![图 3.11  Raft Log Read](image/3.11.png)

Leader 接收到读取请求后，将读请求的 Log 走一遍日志的复制，待其被提交后，apply 的时候读取状态机的数据返回给客户端。显然，这种读取方式非常的低效。

**方案二：ReadIndex Read**

![图 3.12  ReadIndex Read](image/3.12.png)

Leader 接收到读取请求后：

1. 将当前的 `commitIndex` 记为 `readIndex`
2. 向 Follower 发送心跳，收到 `Quorum` 成功响应可以确定自己依旧是 Leader
3. Leader 等待自己的状态机执行，直到 `applyIndex>=readIndex`
4. Leader 执行读取请求，将结果返回给客户端

当然，基于 `ReadIndex Read` 可以实现 `Follower Read`：

1. Follower 收到读取请求后，向 Leader 发送请求获取 Leader 的 `readIndex`
2. Leader 重走上面的步骤 1,2，并将 `readIndex` 返回给 Follower
3. Follower 等待自己的状态机执行，直到 `applyIndex>=readIndex`
4. Follower 执行读取请求，将结果返回给客户端

可以看到 `ReadIndex Read` 比 `Raft Log Read` 拥有更低的时延。

**方案三：Lease Read**

![图 3.13  Lease Read](image/3.13.png)

我们上面提到，违背线性一致性的原因是集群分区后产生了多主，当前 Leader 已不是最新的 Leader。而 braft 提供的 `Leader Lease` 可以确保在租约内，集群中只有当前一个主。基于其实现的读取方式，称为 `Lease Read`。只要 Leader 判断其还在租约内，可以直接读取状态机向客户端返回结果。

我们总结以下上述提到的 3 种方案：

| 方案           | 时延                     | braft 是否实现 |
|:---------------|:-------------------------|:---------------|
| Raft Log Read  | 高时延                   | 是             |
| ReadIndex Read | 低时延                   | 否             |
| Lease Read     | 时延最低（非 100% 可靠） | 是             |

Leader Lease
---

![图 3.14  3 副本场景下 Leader Lease 有效时间区间](image/3.14.png)

`Leader Lease` 的实现原理是基于一个共同承诺，超半数节点共同承诺在收到 Leader RPC 之后的 `election_timeout_ms` 时间内不再参与投票，这保证了在这段时间内集群内不会产生新的 Leader，正如上图所示。

虽然 Leader 和 Follower 计算超时时间都是用的本地时钟，但是由于时钟漂移问题的存在，即各个节点时钟跑的快慢（振幅）不一样，导致 Follower 提前跑完了 `election_timeout_ms`，从而投票给了别人，让集群在分区场景下产生了多个 Leader。为了防止这种现象，`Follower Lease` 的时间加了一个 `max_clock_drift`，其等于 `election_timeout_ms`（默认为 1 秒），这样即使 Follower 时钟跑得稍微快了些也没有关系。但是针对机器之间时钟振幅相差很大的情况，仍然无法解决。总的来说，Follower 的时钟跑慢点没问题，但是跑快点就可能违背以上的承诺，要想 `Leader Lease` 正常工作，得确保各个节点之间的时钟漂移在一定误差内。

`Leader Lease` 的实现必须依赖我们上述提到的 [Follower Lease](#follower-lease)，其实在 braft 中，`Follower Lease` 正是属于 `Leader Lease` 功能的一部分。

具体实现
---

braft 提供了 `is_leader_lease_valid` 和 `get_leader_lease_status` 这 2 个接口，用户需要在读之前调用这 2 个接口来判断当前 Leader 是否仍在租期内。`get_leader_lease_status` 函数首先会判断一下当前的租约状态，其可能是以下几种：

* `DISABLED`：未开启 `Leader Lease` 特性；该特性默认关闭，用户需要将 `FLAGS_raft_enable_leader_lease` 设置为 `true` 来开启该特性
* `EXPIRED`：当前 Leader 已经主动 `step_down` 了，不再是 Leader 了
* `NOT_READY`：当前节点是 Leader，但是不保证它的数据是最新的；这会出现在节点已经当选 Leader，但还在回放数据，并未调用 `on_leader_start`；亦或在节点 `step_down` 时
* `VALID`：确定当前 Leader 是集群内唯一的 Leader
* `SUSPECT`：存疑的；当前租约已经过期了，需要 Leader 根据与各 Follower 最后通信时间来重新续约。braft 并不是后台线程自动续约的，而是需要 Leader 在用户调用接口时主动续约，但其实这 2 种实现的租约区间是一样的，因为 Leader 一直记录着与各 Follower 最后的通信时间，这些通信时间就是租约的另一种存在形式，在需要续租的时候将其转换成对应租约即可，参见以上图 3.14。

```cpp
bool NodeImpl::is_leader_lease_valid() {
    LeaderLeaseStatus lease_status;
    get_leader_lease_status(&lease_status);
    return lease_status.state == LEASE_VALID;
}

void NodeImpl::get_leader_lease_status(LeaderLeaseStatus* lease_status) {
    // Fast path for leader to lease check
    LeaderLease::LeaseInfo internal_info;
    // (1) 先获取下当前 Lease 的状态，如果是明确的状态可直接返回给用户
    //     如果租约已经过期了，则根据记录的各 Follower 最后通信时间来重新续约
    _leader_lease.get_lease_info(&internal_info);
    switch (internal_info.state) {
        case LeaderLease::DISABLED:
            lease_status->state = LEASE_DISABLED;
            return;
        case LeaderLease::EXPIRED:
            lease_status->state = LEASE_EXPIRED;
            return;
        case LeaderLease::NOT_READY:
            lease_status->state = LEASE_NOT_READY;
            return;
        case LeaderLease::VALID:
            lease_status->term = internal_info.term;
            lease_status->lease_epoch = internal_info.lease_epoch;
            lease_status->state = LEASE_VALID;
            return;
        case LeaderLease::SUSPECT:
            // Need do heavy check to judge if a lease still valid.
            break;
    }

    // (2) 当前已经不再是 Leader，直接返回 `EXPIRED`
    if (_state != STATE_LEADER) {
        lease_status->state = LEASE_EXPIRED;
        return;
    }

    // (3) 根据记录的各 Follower 最后通信时间来重新续约
    int64_t last_active_timestamp = last_leader_active_timestamp();
    _leader_lease.renew(last_active_timestamp);

    // (4) 重新获取租约状态
    _leader_lease.get_lease_info(&internal_info);
    if (internal_info.state != LeaderLease::VALID && internal_info.state != LeaderLease::DISABLED) {
        butil::Status status;
        status.set_error(ERAFTTIMEDOUT, "Leader lease expired");
        step_down(_current_term, false, status);
        lease_status->state = LEASE_EXPIRED;
    } else if (internal_info.state == LeaderLease::VALID) {
        lease_status->term = internal_info.term;
        lease_status->lease_epoch = internal_info.lease_epoch;
        lease_status->state = LEASE_VALID;
    } else {
        lease_status->state = LEASE_DISABLED;
    }
}
```

`get_leader_lease_status` 会调用 `get_lease_info` 获取当前租约的状态：
```cpp
void LeaderLease::get_lease_info(LeaseInfo* lease_info) {
    lease_info->term = 0;
    lease_info->lease_epoch = 0;
    if (!FLAGS_raft_enable_leader_lease) {  // (1) DISABLED
        lease_info->state = LeaderLease::DISABLED;
        return;
    }

    BAIDU_SCOPED_LOCK(_mutex);
    if (_term == 0) {  // (2) EXPIRED
        lease_info->state = LeaderLease::EXPIRED;
        return;
    }
    if (_last_active_timestamp == 0) {  // (3) NOT_READY
        lease_info->state = LeaderLease::NOT_READY;
        return;
    }
    if (butil::monotonic_time_ms() < _last_active_timestamp + _election_timeout_ms) {  // (4) VALID
        lease_info->term = _term;
        lease_info->lease_epoch = _lease_epoch;
        lease_info->state = LeaderLease::VALID;
    } else {  // SUSPECT
        lease_info->state = LeaderLease::SUSPECT;
    }
}
```

租约状态更新（1）：节点启时会创建 `LeaderLease` 并调用 `LeaderLease::init` 进行初始化
```cpp
int NodeImpl::init(const NodeOptions& options) {
    ...
    _leader_lease.init(options.election_timeout_ms);
    ...
}

class LeaderLease {
 public:
 ...
 LeaderLease()
        : _election_timeout_ms(0)
        , _last_active_timestamp(0)
        , _term(0)
        , _lease_epoch(0)
    {}
...
}

void LeaderLease::init(int64_t election_timeout_ms) {
    _election_timeout_ms = election_timeout_ms;
}
```

租约状态更新（2）：刚成为 Leader 时
```cpp
void NodeImpl::become_leader() {
    ...
    _leader_lease.on_leader_start(_current_term);
    ...
    // 状态机的 on_leader_start 回调在这之后
}

void LeaderLease::on_leader_start(int64_t term) {
    ...
    ++_lease_epoch;
    _term = term;
    _last_active_timestamp = 0;
}
```

租约状态更新（3）：当 Leader `step_down` 时
```cpp
void NodeImpl::step_down(const int64_t term, bool wakeup_a_candidate,
                         const butil::Status& status) {
    ...
    if (_state == STATE_LEADER) {
        _leader_lease.on_leader_stop();
    }
    ...
}

void LeaderLease::on_leader_stop() {
    ...
    _last_active_timestamp = 0;
    _term = 0;
}
```

租约状态更新（4）：在 `get_leader_lease_status` 中调用 `last_leader_active_timestamp` 获得租约有效区间中开始的时间戳，并调用 `LeaderLease::renew` 进行续约：

```cpp
void NodeImpl::get_leader_lease_status(LeaderLeaseStatus* lease_status) {
    ...
    int64_t last_active_timestamp = last_leader_active_timestamp();
    _leader_lease.renew(last_active_timestamp);
    ...
}
```

先看续约函数：
```cpp
void LeaderLease::renew(int64_t last_active_timestamp) {
    ...
    _last_active_timestamp = last_active_timestamp;
}
```

`last_leader_active_timestamp` 函数会计算在达到 `Quorum` 响应的这批节点中，最早向其发送 RPC 对应的时间戳：
```cpp
int64_t NodeImpl::last_leader_active_timestamp(const Configuration& conf) {
    std::vector<PeerId> peers;
    conf.list_peers(&peers);
    std::vector<int64_t> last_rpc_send_timestamps;
    LastActiveTimestampCompare compare;
    for (size_t i = 0; i < peers.size(); i++) {
        if (peers[i] == _server_id) {
            continue;
        }

        // (1) 获取对应 Follower 的最后一次 RPC 发送时间戳，并将其放入数组中
        //     注意 last_rpc_send_timestamp 记录的是 Leader 发出去的时间戳，
        //     但是只有当节点返回响应时该值才有效，如果节点未响应，获取该值将得到 0
        int64_t timestamp = _replicator_group.last_rpc_send_timestamp(peers[i]);
        last_rpc_send_timestamps.push_back(timestamp);
        // (2) 根据发出去的时间戳构建最小堆
        std::push_heap(last_rpc_send_timestamps.begin(), last_rpc_send_timestamps.end(), compare);
        if (last_rpc_send_timestamps.size() > peers.size() / 2) {
            // (3) 每次扔掉一个最早发出去的 RPC 时间戳，保持堆里面的数量始终为 `Quorum`-1
            //     之所以保持 `Quorum` - 1，是因为 Leader 本身也是一个节点
            std::pop_heap(last_rpc_send_timestamps.begin(), last_rpc_send_timestamps.end(), compare);
            last_rpc_send_timestamps.pop_back();
        }
    }
    // Only one peer in the group.
    if (last_rpc_send_timestamps.empty()) {
        return butil::monotonic_time_ms();
    }

    // (4) 返回堆里最小的时间戳
    //     之所以要返回最早的，是因为先发送 RPC 的，会先开始 Follower Lease，
    //     当然其也会先过期，违背承诺，投票给其他节点
    std::pop_heap(last_rpc_send_timestamps.begin(), last_rpc_send_timestamps.end(), compare);
    return last_rpc_send_timestamps.back();
}

// 比较函数
struct LastActiveTimestampCompare {
    bool operator()(const int64_t& a, const int64_t& b) {
        return a > b;
    }
};
```

参考
===

* [分布式一致性 Raft 与 JRaft](https://www.sofastack.tech/projects/sofa-jraft/consistency-raft-jraft/)
* [关于 DDIA 上对 Raft 协议的这种极端场景的描述，要如何理解？](https://www.zhihu.com/question/483967518)
* [Symmetric network partitioning](https://github.com/baidu/braft/blob/master/docs/cn/raft_protocol.md#symmetric-network-partitioning)
* [Raft在网络分区时leader选举的一个疑问？](https://www.zhihu.com/question/302761390)
* [Raft 必备的优化手段（一）：Leader Election 篇](https://zhuanlan.zhihu.com/p/639480562)
* [Raft 笔记(四) – Leader election](https://youjiali1995.github.io/raft/etcd-raft-leader-election/)
* [raft: implement leader steps down #3866](https://github.com/etcd-io/etcd/issues/3866)
* [共识协议优质资料汇总（paxos，raft）](https://zhuanlan.zhihu.com/p/628681520)
* [关于leader_lease续租时机](https://github.com/baidu/braft/issues/202)
* [TiKV 功能介绍 – Lease Read](https://cn.pingcap.com/blog//lease-read/)
* [CAP理论中的P到底是个什么意思？](https://www.zhihu.com/question/54105974)
* [TiDB 新特性漫谈：从 Follower Read 说起](https://cn.pingcap.com/blog/follower-read-the-new-features-of-tidb/#Follower_Read)
* [raft 成员变更](https://zhuanlan.zhihu.com/p/606448818)