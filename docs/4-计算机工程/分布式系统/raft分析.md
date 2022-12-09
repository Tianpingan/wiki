raft 最重要的三对函数：
- `request_vote`与`handle_request_vote_reply`
- `append_entries`与`handle_append_entries_reply`
- `install_snapshot`与`handle_install_snapshot_reply`

raft 最重要的四个background函数：
- `run_background_ping`
- `run_background_election`
- `run_background_commit`
- `run_background_apply`


当然，raft对外暴露的重要函数包括：
- `raft()`，构造函数
- `start()`，开始
- `stop()`，结束
- `new_command(command cmd, int &term, int &index)`，传入新`cmd`
