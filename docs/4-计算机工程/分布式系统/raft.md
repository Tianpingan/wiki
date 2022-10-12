## 领导人选举



## 日志复制

append_entries
```C++
void RaftServer::AppendEntry(const AppendEntryArgs *args, AppendEntryReply *reply)
{
	mutex_.Lock();
	const AppendEntryArgs &arg = *args;
	int32_t  prev_i = arg.prev_index_;
	uint32_t prev_t = arg.prev_term_;
	uint32_t prev_j = 0;
	if (!running_)
		goto end;

	if (arg.term_ < term_)
		goto end;

	if ((arg.term_ > term_) || (arg.term_ == term_ && state_ == Candidate))
		BecomeFollower(arg.term_); //
	else
		RescheduleElection(); //本身就是follower，就刷新事件

	if (prev_i >= int32_t(logs_.size()))
		goto index;
	if (prev_i >= 0 && logs_[prev_i].term_ != prev_t) {
		assert(commit_ < prev_i);
		logs_.erase(logs_.begin() + prev_i, logs_.end());
		goto index;
	}

	++prev_i;
	for (; prev_i < int32_t(logs_.size()) && prev_j < arg.entries_.size(); ++prev_i, ++prev_j) {
		if (logs_[prev_i].term_ != arg.entries_[prev_j].term_) {
			assert(commit_ < prev_i);
			logs_.erase(logs_.begin() + prev_i, logs_.end());
			break;
		}
	}
	//prev_j之前是相同的
	if (prev_j < arg.entries_.size())
		logs_.insert(logs_.end(), arg.entries_.begin() + prev_j, arg.entries_.end());

	if (arg.leader_commit_ > commit_) {
		commit_ = std::min(arg.leader_commit_, int32_t(logs_.size()) - 1);
		// for (; applied_ < commit_;) apply_func_(logs_[++applied_]);
	}

index:
	reply->idx_ = logs_.size() - 1;

end:
	reply->term_ = term_;
	mutex_.Unlock();
}
```



handle_append_entries_reply
```C++
void RaftServer::ReceiveAppendEntryReply(uint32_t i, const AppendEntryReply &reply)
{
	mutex_.Lock();
	uint32_t vote = 1;
	if (!running_|| state_ != Leader)
		goto end;
	if (reply.term_ > term_) {
		BecomeFollower(reply.term_);
		goto end;
	}
	if (reply.term_ != term_)
		goto end;
	if (next_[i] == reply.idx_ + 1)
		goto end;
	next_[i] = reply.idx_ + 1;
	match_[i] = next_[i] - 1;

	if (commit_ >= reply.idx_ || logs_[reply.idx_].term_ != term_)
		goto end;
	for (uint32_t i = 0; i < peers_.size(); ++i)
		if (match_[i] >= reply.idx_)
			++vote;
	if (vote > ((peers_.size() + 1) / 2)) {
		commit_ = reply.idx_;
		// for (; applied_ < commit_;) apply_func_(logs_[++applied_]);
	}
end:
	mutex_.Unlock();
}
```


## 领导选举


