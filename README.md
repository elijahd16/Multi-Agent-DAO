# Multi-Agent DAO System

A decentralized autonomous organization (DAO) built on Solana blockchain, powered by four specialized AI agents that manage governance, strategy, treasury, and user profiles. The system uses a shared-memory architecture for inter-agent communication and supports user interaction through Discord.

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Agents](#agents)
  - [ProposalAgent (Pion)](#proposalagent-pion)
  - [StrategyAgent (Kron)](#strategyagent-kron)
  - [TreasuryAgent (Vela)](#treasuryagent-vela)
  - [UserProfileAgent (Nova)](#userprofileagent-nova)
- [Core Components](#core-components)
  - [MemoryManager & ExtendedMemoryManager](#memorymanager--extendedmemorymanager)
  - [MemorySyncManager](#memorysyncmanager)
  - [MessageBroker](#messagebroker)
  - [Subscription Registry](#subscription-registry)
  - [BaseAgent and Shared Types](#baseagent-and-shared-types)
- [Scripts / Entry Points](#scripts--entry-points)
  - [startPion.ts](#startpionts-proposalagent)
  - [startKron.ts](#startkronts-strategyagent)
  - [startVela.ts](#startvelets-treasuryagent)
  - [startNova.ts](#startnovats-userprofileagent)
- [Communication Flow](#communication-flow)
- [How to Run Locally](#how-to-run-locally)
- [Directory Structure](#directory-structure)
- [License](#license)

## Overview

This system extends a DAO with specialized AI agents that communicate through a shared-memory, event-driven architecture:

- Users interact with agents via Discord channels using natural language commands
- Agents register for specific event types via a central subscription registry
- Updates are broadcast to all subscribed agents through a message broker
- Agents acquire distributed locks when needed for critical operations
- Treasury assets are managed using a Solana wallet in a Trusted Execution Environment
- A reputation-based voting system gives users influence proportional to their contributions

The four primary agents in this system are ProposalAgent (Pion), StrategyAgent (Kron), TreasuryAgent (Vela), and UserProfileAgent (Nova).

## Architecture
Here is a high-level architectural diagram:

```
                           +----------------+
                           |  Pion (ProposalAgent)  |
                           |  - Proposals & Voting  |
        +----------------->|  - Closes Votes        |
        |                  +----------^-------------+
        |                             |
        |                             |
        |                  +----------|-------------+
        |                  | Kron (StrategyAgent)   |
        |   +--------------| - Execute & Manage     |
        |   |              |   Trading Strategies   |
        |   |              +----------^-------------+
        |   |                         |
        v   |                         |
+----------------+                    |             +------------------+
|  Vela (TreasuryAgent)  | <----------+------------| Nova (UserProfileAgent) |
| - Manages Treasury     |                         | - User Profiles         |
| - Swaps/Deposits       |                         | - Reputation, etc.      |
| - Transfers            |                         +----------^--------------+
+------------------------+                                    |
                                                              
                    (Shared Memory, Cross-process Sync, and Event Broadcasting)

    MemoryManager / ExtendedMemoryManager  --  MemorySyncManager  --  MessageBroker
```

Key points:

- Each agent is an instance of the BaseAgent class or a specialized extension.
- MemoryManager + MessageBroker + MemorySyncManager provide a robust communication and storage system.
- Agents can be run in separate processes, each connecting to the same database or via cross-process messages.
- The subscription registry ensures each agent only receives relevant memory updates.

## Agents

### ProposalAgent (Pion)
File: packages/plugin-solana/src/agents/proposal/ProposalAgent.ts

Responsibilities:

- Creating new proposals (proposal and proposal_created memory).
- Interpreting proposal text, validating input, scheduling votes.
- Tracking votes (vote_cast memory) and closing proposals after a deadline or user request.
- Executing proposals when quorum is reached (e.g., parameter changes or governance actions).
- Monitors proposals for final status updates, creating proposal_execution_result or proposal_status_changed.

Key Features:

- Proposal Lifecycle: open → pending_execution → executing → executed or failed.
- Vote Handling: auto-detect user votes from chat commands or emoji reactions.
- Locking: uses DistributedLock to avoid concurrent updates on the same proposal.

### StrategyAgent (Kron)
File: packages/plugin-solana/src/agents/strategy/StrategyAgent.ts

Responsibilities:

- Managing advanced trading strategies for tokens, e.g.:
  - Take-profit (TP) levels, stop-loss (SL), trailing stop, DCA, grids, rebalancing.
- Position tracking: opens or updates positions based on user instructions or proposals.
- Strategy Execution: triggers token swaps (via the treasury) when conditions are met (e.g., price threshold).
- Cross-Process updates: listens to strategy_execution_request, price_update, position_update, and more.

Key Features:

- Integration with SwapService for actual token swaps.
- Flexible to handle multiple strategy types (TP, SL, trailing, etc.).
- Subscribes to real-time or scheduled events to check conditions (time-based, price-based, etc.).

### TreasuryAgent (Vela)
File: packages/plugin-solana/src/agents/treasury/TreasuryAgent.ts

Responsibilities:

- Core Treasury operations: deposit, transfer, token swaps, wallet management.
- Maintains on-chain Solana wallet keypairs, monitors balances, handles deposit verifications.
- Receives direct user commands (!register, !deposit, !verify, !swap, !balance).
- Serves as the bridge between on-chain actions and the rest of the DAO.

Key Features:

- Uses SwapService for token swaps.
- WalletProvider & TokenProvider to query token data, fetch balances, or sign transactions.
- Subscribes to deposit events (deposit_received, pending_deposit) to finalize them.
- Works closely with Pion for proposals that require treasury movements.

### UserProfileAgent (Nova)
File: packages/plugin-solana/src/agents/user/UserProfileAgent.ts (not fully shown in the snippet but implied in startNova.ts)

Responsibilities:

- Manages user profiles, reputations, preferences, tasks, and other conversation-based data.
- Could handle user-level permission checks, roles (admin, moderator, user).
- Provides an entry point for normal user queries, interaction logging, and extended conversation flows.

Key Features:

- Subscribes to all user message memory (user_message) to update user context.
- Potentially handles reputation changes, awarding tokens or adjusting privileges.
- Merges "chatbot" style conversation with user registration details in the DAO context.

## Core Components

### MemoryManager & ExtendedMemoryManager
Files:

- packages/plugin-solana/src/shared/memory/BaseMemoryManager.ts
- packages/plugin-solana/src/shared/utils/runtime.ts (where ExtendedMemoryManager is constructed)

Function:

- Provides read/write operations for storing Memory objects in a database.
- Can handle "versioned" memory types (like proposals) and unique memory constraints.
- Agents rely on it to createMemory, getMemories, subscribeToMemory, etc.
- ExtendedMemoryManager is a specialized overlay that enriches the base memory manager with additional logic like locks, concurrency checks, or advanced search.

### MemorySyncManager
File: packages/plugin-solana/src/shared/memory/MemorySyncManager.ts

Function:

- Synchronizes memory changes across multiple processes.
- Listens for create/update/delete events and replicates them, ensuring all agent processes share consistent state.
- Uses inter-process messaging (process.send in Node.js) or a custom bus mechanism.

### MessageBroker
File: packages/plugin-solana/src/shared/MessageBroker.ts

Function:

- A local event emitter that broadcasts memory events to in-process subscribers.
- Agents subscribe to memory types (e.g., "proposal_created"), and the MessageBroker routes relevant events.
- Works in tandem with MemorySyncManager for cross-process broadcasting.

### Subscription Registry
File:

- packages/plugin-solana/src/shared/types/subscriptionRegistry.ts
- packages/plugin-solana/src/shared/types/memory-subscriptions.ts

Function:

- Enumerates which memory types each agent must subscribe to. E.g., the ProposalAgent must subscribe to vote_cast, proposal_created, etc.
- Ensures an agent only handles the relevant memory events, thereby simplifying the logic.

### BaseAgent and Shared Types
File: packages/plugin-solana/src/shared/BaseAgent.ts

Key Elements:

- BaseAgent is the abstract class providing common functionality:
  - Lifecycle methods: initialize(), shutdown().
  - Transaction helpers: withTransaction().
  - Subscriptions, cross-process event hooking.
- AgentMessage, BaseContent, Room IDs, DistributedLock interfaces are also defined in types/base.ts.

## Scripts / Entry Points
There are four main script files, each starting one of the specialized agents:

### startPion.ts (ProposalAgent)
- Path: packages/plugin-solana/src/startPion.ts
- Bootstraps the ProposalAgent with a configuration (e.g., DB connection, memory manager).
- Reads .env.pion for environment overrides.
- Once up, the agent listens for proposal memory events and user requests (like !propose, !vote, !close).

### startKron.ts (StrategyAgent)
- Path: packages/plugin-solana/src/startKron.ts
- Bootstraps the StrategyAgent for advanced trading logic.
- Reads .env.kron for environment overrides.
- After startup, it manages strategy creation, monitors price updates, and triggers token swaps automatically.

### startVela.ts (TreasuryAgent)
- Path: packages/plugin-solana/src/startVela.ts
- Bootstraps the TreasuryAgent to handle all treasury-related commands (!deposit, !verify, !balance, etc.).
- Connects to a Solana node (from SOLANA_RPC_URL) and to a DB for memory storage.
- Maintains the treasury's on-chain wallet using environment key config.

### startNova.ts (UserProfileAgent)
- Path: packages/plugin-solana/src/startNova.ts
- Bootstraps the "Nova" agent that handles user profiles.
- Handles user messages, conversation context, possibly includes reputation updates.
- Reads .env.nova for environment overrides.

## Communication Flow
1. User enters commands (e.g., !propose, !deposit) in a chat interface.
2. A Client Interface (Discord or Direct) captures the message and stores it as a Memory of type user_message.
3. MemoryManager triggers a local event → MessageBroker broadcasts → The agent responsible for that memory type picks it up.
4. E.g., if the message is a deposit command, TreasuryAgent sees it and processes it.
5. The agent performs logic (maybe a swap, a deposit verification, or schedule a proposal).
6. Any changes are stored as new Memory records (proposal_created, vote_cast, strategy_execution_request, etc.).
7. The MemorySyncManager replicates the event to other processes if in multi-process mode.
8. Other agents listen to relevant memory types, e.g., the StrategyAgent might see a proposal_passed memory, or the ProposalAgent sees a vote_cast.

## How to Run Locally
### Install dependencies
```bash
npm install
```

### Set environment variables
- Copy .env.example to .env and fill in Solana or DB credentials.
- Optionally create specialized .env.pion, .env.kron, .env.vela, .env.nova for each agent.

### Run an agent
Example: to start the TreasuryAgent (Vela):
```bash
ts-node packages/plugin-solana/src/startVela.ts
```
or the ProposalAgent (Pion):
```bash
ts-node packages/plugin-solana/src/startPion.ts
```

### Multi-process
Simply run each script in a separate terminal (or a process manager). The MemorySyncManager will keep them in sync if MULTI_PROCESS=true.

## Directory Structure
```
packages/plugin-solana/src/
├── agents/
│   ├── proposal/
│   │   └── ProposalAgent.ts      # Pion
│   ├── strategy/
│   │   └── StrategyAgent.ts      # Kron
│   ├── treasury/
│   │   └── TreasuryAgent.ts      # Vela
│   └── user/
│       └── UserProfileAgent.ts   # Nova
├── shared/
│   ├── memory/
│   │   ├── BaseMemoryManager.ts
│   │   ├── MemorySyncManager.ts
│   │   └── ...
│   ├── types/
│   │   ├── base.ts
│   │   ├── memory-subscriptions.ts
│   │   ├── subscriptionRegistry.ts
│   │   └── ...
│   ├── MessageBroker.ts
│   └── ...
├── startPion.ts       # Boot script for ProposalAgent
├── startKron.ts       # Boot script for StrategyAgent
├── startVela.ts       # Boot script for TreasuryAgent
├── startNova.ts       # Boot script for UserProfileAgent
└── ...
```

## License
This plugin is released under the MIT License (or whichever license your repository uses). See LICENSE for details.



