# High Level Approach

I approached implementing the RAFT protocol by reading the RAFT paper and incrementally implementing each each section and RPC. This required three steps: setting up the state of the replica, implementing leader election, and implementing log replication. Throughout the project, I referred to figure 2 in the RAFT paper, which specifies variables and actions to take for each step of the consensus protocol.

## Replica State

Implementing replica state involved carefully following the "State" table in figure 2 of the RAFT paper. In the constructor of the replica, I created the following class variables: `current_term`, `voted_for`, `current_role`, `current_leader`, `votes_received`, `messages`, `kvstore`, `logs`, `commit_index`, `last_applied`, `next_index`, and `match_index`. The first 5 variables are used within leader election and the last 5 variables are used within log replication. The variables `messages` and `kvstore` are used for handling RPCs/client requests and storing the state of the key-value store, respectively. To simplify the state of the replcias, I created the `Role` and `RPC` enums. The `Role` enum indicates the role for a replica in RAFT: a replica can either be a Follower, Candidate or Leader. Similaritily, the `RPC` enum indicates the type for a RPC in the protocol. These RPCs are RequestVote, RequestVote Response, AppendEntries, and AppendEntries Response.

## Leader Election

To implement leader election, I first created the election and heartbeat timeouts, with their respective helper functions, to detect when an election needs to be started or if a heartbeat needs to be sent from a leader. This involved creating the class variables `election_timeout`, initialized with the `reset_election_timeout`, and `heartbeat_timeout`, initialized to infinity. These values are reset with the `reset_election_timeout` and `reset_heartbeat_timeout` functions, which set the election timeout to the current time + a random value between 0.45 and 0.60 milliseconds and the heartbeat timeout to the current time + 0.4 milliseconds.

I then implemented the process for starting an election. This involved converting a follower to a candidate and sending RequestVote RPCs to the other replicas. Both processes are handled with the `start_election` and `request_vote` functions. The `start_election` function increments the replica's term, changes its role to candidate and votes for itself. It then calls the `request_vote` function, which sends a RequestVote RPC to the other replicas.

The final step I implemented was responding to RequestVote RPCs and handling that response. Responding to RequestVote RPCs is handled with the `receive_request_vote` function and handling that function's response is done with the `receive_vote` function. The `receive_request_vote` function votes for the source of the RPC if it's log is up to date and the replica had not voted for any other node yet. The `receive_vote` function tallies the votes and converts the replica to the leader if it receives a quorum of votes.

## Log Replication

To implement log replication, I first created the `request_append_entries` function, which implements the AppendEntires RPC. This function essentially sends every follower the log entries they have not added yet.

I then implemented receiving the AppendEntries RPC. This is done in the `receive_append_entries` fuction. This function deletes conflicting entries in the follower's log and appends new entries from the leader into the follower's log. It then updates the commit index of the follower and then applies those commits into the key-value store. Finally, the function sends a response to the leader indicating whether the operation was successful or not.

The final step I implemented was handling AppendEntries RPC responses. This is done with the `receive_append_entries_response` function This function updates the follower's match and next indexes if the operations was successful or decrements the next index and retries the AppendEntries RPC if not. It then commits the log entries into the key-value store if a quorum of followers have updated their logs.

# Challenges

The major challenge of this project was understanding the steps and nuance in the RAFT protocol. There are a lot of implementation details that are easily lost in text or diagrams of the RAFT paper that, if missed, cause major errors. There are also a significant amount of edge cases to consider, and message/RPC handling must be designed in a specific manner to handle those edge cases. A good lesson from this challenge is to always completely understand implementation details of a protocol or algorithm before starting to code. Implementing RAFT would have been a lot easier if I spent more time reading the paper and taking notes on specific processes and edge cases within the protocol.

Another challenge I encountered in this project is debugging. When running tests, it was often difficult to completely understand what part of the code was failing. This is primarily due to the interconnectivity of the RAFT nodes -- every message and state interacts with each other in multiple ways, and figuring out what state or message was causing the error required careful observation of the processes running. Therefore, to debug, I called print statements at critical points in the replica class -- sending and receiving RPCs and handling messages. This allowed me to easily see the interconnectivity between messages and states, and ultimately reveal what was causing the tests to crash.

# List of Well-Designed Properties and Features

- Leader election
  Leader election directly follows the specifications found in the RAFT paper. It uses the RequestVote RPC and heartbeat/election timeouts to start elections, vote on candidates, and declare a new leader. The implementation successfully elects a leader in one round most of the time using randomized election timeouts. In the case of a tie, the implementation gracefully starts a new election and elects a new leader. The leader election process also gracefully handles partitions, where a new leader with the most updated log is elected.
- Log replication
  Log replications also directly follows the specifications found in the RAFT paper. It uses the AppendEntries RPC to send new entries to followers. The followers in turn add those entries to its log, and sends a response to the leader. Once the leader recieves a quorum of follower success responses for a log entry, it can safely commit the log entry to the state machine, or key-value store. On ensuing AppendEntries RPCs, followers learn about the new commit and commit the new entry to their key-value store. Overall, this process successfully achieves consensus by making every follower agree to the leader's log.
- Timeouts
  There are two timeouts in this RAFT implementation: the heartbeat timeout and the election timeout. The election timeout triggers the start of the election and the hearbeat timeout triggers a heartbeat broadcast from the leader. These two timeouts elegantly work in conjunction to elect a leader and keep the leader in power until it fails. When a leader fails, there will be an election timeout due to the lack of heartbeats. This triggers an election and a new leader is selected. The leader will send heartbeats when there is a heartbeat timeout -- this is typically slightly less than the election timeout to prevent new elections. Followers will therefore not timeout and consensus will be achieved through that leader.
- Message Handling
  In this implementaiton messages are received through the function `receive_all` and handled through the functions `as_leader`, `as_candidate`, and `as_follower`. The `receive_all` function receives all messages using select and a loop. This guarantees that, if data is ready in the socket, it will be handled by the replica. This prevents cases where data is ready but not read, leading to timeouts and unexpected behavior. The individual functions for each state handle all the RPCs relevant to that state. This reduces complexity of writing code for each state and makes it more obvious what each state handles or does for each message.

# Testing

To test the RAFT implementation, I first tested every RPC and function I implemented individually to ensure that the correct values were being sent and received by the replicas. I made sure that different error and success cases were being handled correctly and the correct responses were formed. To debug, I added print statements at the beginning of every function to ensure that the states and messages were accurate.

After implementing a specific step, such as leader election or log replication, I would test my code against the provided tests. I made sure that all tests passed for that specific process before moving on. After putting everything together, I ran every test and made sure there were no outstanding errors.
