# Nav2 Behavior Tree Navigation Robustness

This fork highlights a focused improvement to Nav2 Behavior Tree navigation failure handling. The work hardens active-goal preemption, improves terminal Behavior Tree diagnostics, propagates clearer action failure information, and adds regression coverage for the affected paths.

## Highlights

- Protects an active `NavigateThroughPoses` goal when a replacement goal cannot be transformed.
- Rejects invalid replacement goals without aborting the current valid navigation request.
- Flushes final Behavior Tree status transitions for complete introspection logs.
- Fails fast when navigation BT action nodes are missing required goal inputs.
- Propagates action error codes and messages through BT blackboard outputs.
- Adds recovery retry diagnostics for primary failure, recovery attempt, retry, exhaustion, and recovery failure paths.
- Adds unit and system coverage for preemption, missing inputs, and action failure propagation.

## Change Map

```mermaid
flowchart LR
  Client["Navigation Client"] --> BTNav["nav2_bt_navigator"]
  BTNav --> Server["BtActionServer"]
  Server --> Engine["BehaviorTreeEngine"]
  Engine --> Tree["Behavior Tree"]

  Tree --> NavToPose["NavigateToPoseAction"]
  Tree --> NavThrough["NavigateThroughPosesAction"]
  Tree --> FollowPath["FollowPathAction"]
  Tree --> Recovery["RecoveryNode"]

  BTNav --> TF["TF Transform Validation"]
  Server --> Logger["BT Status Logger"]
  NavToPose --> Blackboard["Error Code / Message Outputs"]
  NavThrough --> Blackboard
  FollowPath --> Blackboard
  Recovery --> Diagnostics["Recovery Retry Diagnostics"]
```

## Active Goal Protection

The preemption path now validates a replacement `NavigateThroughPoses` goal before accepting it. If the replacement poses cannot be transformed, only the pending goal is terminated and the current navigation goal continues.

```mermaid
sequenceDiagram
  autonumber
  participant Client
  participant ActionServer as BtActionServer
  participant Navigator as NavigateThroughPosesNavigator
  participant TF as TF Buffer
  participant Current as Current Goal

  Client->>ActionServer: Send replacement goal
  ActionServer->>Navigator: onPreempt(pending goal)
  Navigator->>TF: Validate replacement poses
  TF-->>Navigator: Transform unavailable
  Navigator->>ActionServer: terminatePendingGoal()
  Navigator-->>Current: Continue current goal
  ActionServer-->>Client: Replacement goal terminated
```

## Preemption Decision Flow

```mermaid
flowchart TD
  A["Preempt requested"] --> B{"Pending goal available?"}
  B -- "No" --> C["Throttle warning<br/>ignore this loop"]
  B -- "Yes" --> D{"Same BT or default BT?"}
  D -- "No" --> E["Reject replacement<br/>different tree requested"]
  D -- "Yes" --> F{"Replacement poses transform?"}
  F -- "Yes" --> G["Accept pending goal"]
  F -- "No" --> H["Terminate pending goal"]
  H --> I["Keep current goal active"]
```

## Behavior Tree Status Handling

Terminal tree states are flushed after execution completes, which keeps the final `SUCCESS` or `FAILURE` transition visible to subscribers.

```mermaid
stateDiagram-v2
  [*] --> Running: Tick tree
  Running --> Running: Loop callback
  Running --> Success: Root SUCCESS
  Running --> Failure: Root FAILURE
  Running --> Failure: Unexpected IDLE
  Success --> Flush: Flush terminal status
  Failure --> Flush: Flush terminal status
  Flush --> Halt: Halt actions
  Halt --> [*]
```

## Missing Input Guard

Navigation action BT nodes now stop before sending an action goal when required input ports are missing. The node returns `FAILURE` and writes an explicit error code and message.

```mermaid
flowchart LR
  Tick["BT node tick"] --> Input{"Required goal input present?"}
  Input -- "Yes" --> Send["Send action goal"]
  Input -- "No" --> Stop["Do not send goal"]
  Stop --> Code["Set UNKNOWN error code"]
  Stop --> Message["Set explanatory error message"]
  Code --> Failure["Return FAILURE"]
  Message --> Failure
```

## Recovery Diagnostics

`RecoveryNode` now logs important retry transitions without changing BT XML or plugin interfaces.

```mermaid
flowchart TD
  Start["Tick primary child"] --> Primary{"Primary status"}
  Primary -- "SUCCESS" --> Done["Return SUCCESS"]
  Primary -- "RUNNING" --> Running["Return RUNNING"]
  Primary -- "FAILURE" --> Retries{"Retries remaining?"}
  Retries -- "No" --> Exhausted["Log retry exhaustion<br/>return FAILURE"]
  Retries -- "Yes" --> TryRecovery["Log recovery attempt<br/>tick recovery child"]
  TryRecovery --> RecoveryStatus{"Recovery status"}
  RecoveryStatus -- "SUCCESS" --> RetryPrimary["Log recovery success<br/>retry primary"]
  RecoveryStatus -- "RUNNING" --> RecoveryRunning["Return RUNNING"]
  RecoveryStatus -- "FAILURE" --> RecoveryFailed["Log recovery failure<br/>return FAILURE"]
  RetryPrimary --> Start
```

## Failure Propagation

```mermaid
flowchart TB
  ActionServer["Action server result"] --> BTNode["BT action node"]
  BTNode --> Status{"Action status"}
  Status -- "Succeeded" --> Success["BT SUCCESS"]
  Status -- "Aborted" --> ErrorOutputs["Copy error code and message"]
  ErrorOutputs --> Failure["BT FAILURE"]
  Failure --> Navigator["Navigator action result"]
  Navigator --> Operator["Clearer runtime diagnostics"]
```

## Test Coverage

```mermaid
flowchart TB
  Coverage["Regression coverage"] --> Unit["BT action unit tests"]
  Coverage --> System["NavigateThroughPoses system test"]

  Unit --> NTP["NavigateToPose missing goal input"]
  Unit --> NTHP["NavigateThroughPoses missing goals input"]
  Unit --> FPFailure["FollowPath failure code and message propagation"]
  Unit --> FPReplace["FollowPath replacement while active"]

  System --> BadTF["Bad-TF replacement preemption"]
  BadTF --> Expected["Replacement terminates<br/>current goal does not abort"]
```

## Code References

Core runtime changes:

- [`BtActionServer` preempt guard and terminal status flush](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_behavior_tree/include/nav2_behavior_tree/bt_action_server_impl.hpp)
- [`BehaviorTreeEngine` terminal failure logging](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_behavior_tree/src/behavior_tree_engine.cpp)
- [`RecoveryNode` retry diagnostics](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_behavior_tree/plugins/control/recovery_node.cpp)
- [`RecoveryNode` logger member](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_behavior_tree/include/nav2_behavior_tree/plugins/control/recovery_node.hpp)
- [`NavigateToPoseAction` missing-input guard](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_behavior_tree/plugins/action/navigate_to_pose_action.cpp)
- [`NavigateThroughPosesAction` missing-input guard](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_behavior_tree/plugins/action/navigate_through_poses_action.cpp)
- [`NavigateToPose` TF warning throttling](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_bt_navigator/src/navigators/navigate_to_pose.cpp)
- [`NavigateThroughPoses` bad replacement goal handling](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_bt_navigator/src/navigators/navigate_through_poses.cpp)

Tests and documentation:

- [`FollowPath` action failure and replacement tests](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_behavior_tree/test/plugins/action/test_follow_path_action.cpp)
- [`NavigateToPoseAction` missing-input test](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_behavior_tree/test/plugins/action/test_navigate_to_pose_action.cpp)
- [`NavigateThroughPosesAction` missing-input test](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_behavior_tree/test/plugins/action/test_navigate_through_poses_action.cpp)
- [`NavigateThroughPoses` bad-TF preemption system test](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_system_tests/src/system/nav_through_poses_tester_node.py)
- [`BT navigation failure audit`](https://github.com/Rpirayesh/navigation2/blob/bt-navigation-failure-robustness/nav2_bt_navigator/doc/bt_navigation_failure_audit.md)

## Compatibility

The implementation preserves existing ROS actions, services, topics, parameters, Behavior Tree XML ports, and public BT plugin base-class APIs.

## Upstream Contribution

Draft pull request: https://github.com/ros-navigation/navigation2/pull/6245
