# Skullster Economy Bot: Detailed Cog Explanation

This document explains how the bot is structured and what each cog does, with a focus on **how the Python code works**, not just what the commands do.

---

## 1. Big picture: what this bot actually is

This is a `discord.py` bot built around a **modular cog system**.

A **cog** is basically a Python class that groups related bot features together. Instead of shoving 4,000 lines into one file and crying into your keyboard, the code splits features into separate modules:

- economy
n- menu ordering
- gambling
- banking
- taxes
- reviews
- roles
- task system
- whitelist/security
- etc.

The bot stores most of its persistent data in **JSON files** under `data/`.
That means it is **file-based persistence**, not a database.

### What this means in practice

Each cog usually follows the same pattern:

1. Define file paths and constants.
2. Define helper functions like `_load()`, `_save()`, `_get_user()`.
3. Define a `commands.Cog` class.
4. Add commands and listeners inside the cog.
5. Register the cog in `setup()`.

---

## 2. `bot.py`: the core loader and traffic controller

`bot.py` is the entry point.

### What it does

- loads environment variables with `load_dotenv()`
- reads config like token, prefix, guild ID, owner ID
- creates the bot instance with `commands.Bot(...)`
- enables intents
- loads all cogs from the `COGS` list
- syncs slash commands to a home guild
- starts the bot

### Important concepts here

#### `commands.Bot(...)`
This is the central bot object. Everything hangs off this.

```py
bot = commands.Bot(
    command_prefix=PREFIX,
    intents=intents,
    owner_id=OWNER_ID,
    help_command=None,
)
```

- `command_prefix=PREFIX` means prefix commands use something like `.balance`
- `help_command=None` disables Discord.py's built-in help command so the custom help cog can replace it

#### Intents
The bot enables:
- `message_content`
- `members`
- `guilds`

Without `message_content`, prefix commands and message listeners like `on_message` would not work correctly.

#### `COGS` list
This is the list of extensions the bot loads on startup.
Each entry like `"cogs.bar_economy"` maps to a Python module.

#### `on_ready`
This fires once the bot connects successfully.
It logs status and sets a presence.

#### `on_command_error`
This is a central error handler.
It catches common Discord.py exceptions like:
- `NotOwner`
- `MissingPermissions`
- `MissingRequiredArgument`
- `BadArgument`

It deliberately ignores:
- `CommandNotFound`
- some `CheckFailure` cases

That matters because `channel_guard.py` already handles its own blocking behavior.

#### Owner commands in `bot.py`
These are maintenance tools:
- `.sync`
- `.reload`
- `.load`
- `.unload`
- `.cogs`
- `.treelist`
- `.ping`

This is standard bot-dev quality of life stuff. Because apparently restarting a whole bot for one cog edit is a hobby people choose.

---

## 3. Core architectural pattern used across the cogs

Most cogs use the same three helper shapes:

### `_load_*()`
Reads JSON from disk.

Example idea:
```py
def _load_econ() -> dict:
    if not os.path.isfile(ECONOMY_FILE):
        return {}
    with open(ECONOMY_FILE, "r") as f:
        return json.load(f)
```

### `_save_*()`
Writes JSON back to disk.

### `_get_user(...)`
Guarantees a user entry exists.

Typical pattern:
```py
if key not in data:
    data[key] = {"balance": 0, "cooldowns": {}}
```

That is important because it prevents `KeyError` when a brand-new user runs a command.

### Why this matters for learning Python

This bot is teaching you a very common beginner-to-intermediate backend pattern:

- load state
- mutate state
- save state
- respond to the user

That same pattern shows up in web apps, APIs, games, bots, and desktop apps.

---

## 4. Shared data model: `data/bar_economy.json`

This is the most important file in the project. A lot of cogs depend on it.

A user record usually contains fields like:

- `balance`
- `cooldowns`
- `debt`
- possibly savings data
- tax data
- mute data
- boosts
- other economy-related metadata

Because multiple cogs read and write the same file, this file acts like the bot's central economy ledger.

### Very important side effect

If two cogs both modify the same user at different times, they are all relying on:

- consistent JSON shape
- consistent field names
- re-saving without wiping other fields

That is why `setdefault(...)` is used a lot.
It prevents older code from exploding when new fields get added later.

---

# Cog-by-cog explanations

---

## 5. `channel_guard.py`

### Purpose
This cog restricts most commands to one specific channel and penalizes users who use commands elsewhere.

### Why it exists
Instead of letting economy/game commands flood every channel, it forces command use into a designated bot/economy channel.

### Key idea: global command check
Inside `__init__`, it registers a bot-wide check:

```py
bot.add_check(self._global_channel_check)
```

That means **every prefix command** passes through this check before it runs.

### What `_global_channel_check` does

It allows the command if:
- the message is not in the guarded guild
- the user is the owner
- the user has a bypass role
- the command is already in the allowed channel

Otherwise it:
- applies a penalty
- raises `commands.CheckFailure("wrong_channel")`

### Penalty system
`_penalize(...)` subtracts `PENALTY` from the user's balance.
If balance goes below zero, the overflow becomes **debt**.

That means:
- balance cannot stay negative
- negative overflow gets stored as `debt`

### `settle_debt(uid, earned)`
This is one of the most important helper functions in the whole project.
It is used by other cogs when users **earn money**.

It applies deductions in this order:

1. repay penalty debt
2. repay loan balance
3. apply arbitrary tax
4. apply bracket tax

Then it returns:
```py
(net_earned, total_deducted)
```

That makes this function a kind of **economy deduction pipeline**.

### Commands
- `.debt` → shows your debt
- `.debtclear @user` → owner clears debt
- `.debtcheck @user` → admin checks debt

### Python lessons in this cog

- global command checks
- shared helper functions used by multiple cogs
- mutating nested JSON safely
- converting negative balances into debt rather than storing negatives directly
- raising `CheckFailure` to block command execution

---

## 6. `bar_economy.py`

### Purpose
This is the main wallet/economy cog.
It handles routine ways users gain or move currency.

### Core commands
- `.balance` / `.bal` / `.zest`
- `.daily`
- `.beg`
- `.work`
- `.rob @user`
- `.give @user <amount>`
- `.barcfg ...` for config
- `.zestadmin ...` for admin economy actions

### Main ideas inside the code

#### Cooldowns
The cog stores timestamps in `user["cooldowns"]`.

Helper flow:
- `_check_cd(user, key, seconds)` returns remaining seconds
- `_set_cd(user, key)` stores current timestamp

That is a classic custom cooldown system. It does not rely entirely on Discord.py decorators like `@commands.cooldown`; it uses JSON-backed cooldowns instead.

### Economy actions

#### `.daily`
Adds a random amount from `DAILY_AMOUNT`.

#### `.beg`
Adds a small random amount from `BEG_AMOUNT`.

#### `.work`
Adds a random amount from `WORK_AMOUNT`, using themed flavor text from `WORK_LINES`.

#### `.rob`
Attempts to steal from another user.
Likely includes:
- success chance (`ROB_CHANCE`)
- steal percentage range (`ROB_AMOUNT`)
- failure fine (`ROB_FINE`)
- daily robbery limit (`ROB_DAILY_LIMIT`)

This teaches a useful game-design pattern: one action can either increase or decrease balance depending on RNG and rules.

#### `.give`
Transfers money from one user to another.

### Black market integration
The helper `_get_active_boost(uid, key)` checks if a user has an active boost stored in economy data.
That means `bar_economy.py` is not isolated. It is aware of `blackmarket.py` effects.

### Admin groups

#### `.barcfg`
Used to manage cooldown configuration stored in `bar_config.json`.
That means cooldowns are not hardcoded only in the file. They can be changed at runtime.

#### `.zestadmin`
Lets authorized people:
- give money
- take money
- set balance
- reset user economy or cooldown state

### Python lessons in this cog

- JSON-backed cooldown logic
- random event economy design
- command groups and subcommands
- using helper functions to avoid repeated code
- cross-cog effect checking through shared data

---

## 7. `bar_menu.py`

### Purpose
This cog manages the drink menu and drink ordering.

### Data files used
- `data/bar_menu.json`
- `data/bar_orders.json`
- `data/bar_economy.json`

### What it does conceptually

It has two separate concerns:

1. **menu browsing**
2. **buying/order processing**

### Menu browsing
`.menu` displays paginated drink information.
The helper `_build_page(...)` creates an embed for one page of drinks.

That is a common UI pattern:
- store a list of items
- slice it into pages
- format one page into an embed

### Ordering
There is a `_OrderConfirmView(discord.ui.View)` with buttons.
That means ordering is interactive, not just plain text.

Important Discord.py concept:
- `discord.ui.View` is used for buttons, selects, and interactive message components

### Order flow
Typical logic here is:
1. user chooses a drink
2. bot checks stock and cooldown
3. bot checks user balance
4. bot asks for confirmation
5. on confirm, balance is reduced and order is stored

### Economy interaction
This cog directly reads economy data and deducts currency.

### Commission
`VIV_COMMISSION = 0.15`
This suggests part of an order may go to Viv or be tracked as a staff/house cut.

### Dynamic slash command
There is a `_make_order_command(...)` function.
That means the slash command for ordering is built programmatically instead of only by decorator.

That is a slightly more advanced pattern.
It is often used when you want autocomplete or custom app-command behavior.

### Admin commands
The `.drink` command group lets bar admins:
- add drinks
- remove drinks
- edit drinks
- toggle availability
- restock
- list drink IDs

### Python lessons in this cog

- list/dict persistence
- button-based interactions
- embed pagination
- menu/inventory style data handling
- separating browsing logic from confirmation logic

---

## 8. `bar_channel.py`

### Purpose
This cog enforces stricter behavior in the dedicated bar-ordering channel.

### Difference from `channel_guard.py`
- `channel_guard.py` blocks commands used in the wrong channel
- `bar_channel.py` watches a specific channel and penalizes messages that are not appropriate there

So one is **global command gating**, the other is **local channel behavior moderation**.

### How it works
It uses `on_message`.

That means it sees raw messages before or alongside normal command handling.

### Allowed behavior
Constants show allowed items such as:
- slash: `order`
- prefix: `menu`, `balance`, `bal`, `zest`

So in that channel, users are expected to do only certain bot actions.

### Penalty escalation
This cog tracks:
- offence count
- reset windows
- usage limits

Penalty rises by `PENALTY_INCREMENT = 3` per offence.
So the punishment escalates over time rather than staying flat.

### Python lessons

- message listeners
- local behavior tracking with timestamps
- escalating penalties
- distinction between command-level checks and raw message moderation

---

## 9. `tasks.py`

### Purpose
This is the work/task system.
It adds structured ways for users to earn currency beyond `.work` and `.daily`.

### Types of tasks
The cog supports three main categories:

1. **grind** command
2. **fixed daily tasks**
3. **random claimable tasks**
4. **staff-assigned tasks**

### Key files
- `data/tasks.json`
- `data/bar_economy.json`
- `data/bar_orders.json`

### `grind`
`.grind` is a high-payout, long-cooldown work action.
It uses `GRIND_AMOUNT` and `GRIND_CD`.

### Fixed daily tasks
Examples:
- `.cleanthebar`
- `.checkinventory`
- `.polishglasses`

These likely each pay a fixed amount and can only be done once per reset period.
The private helper `_do_fixed_task(...)` probably centralizes that logic.

That is a good design decision because it avoids repeating the same code three times.

### Random tasks
There is a `RANDOM_TASK_POOL` constant and a helper `_refresh_random_pool(task_data)`.
This means the bot periodically refreshes available random jobs.

Users can:
- view them with `.taskboard`
- claim one with `.taskclaim`
- see progress with `.taskprogress`

### Event tracking
This is a smart part of the design.
The cog has `_track_event(...)` and listens for `on_command_completion`.

That means certain commands can count as task progress.
For example, a task like “place an order” or “earn Zest” can be tracked after a relevant command finishes.

This is a mini event-driven system.

### Earnings pipeline
The helper `_apply_earnings(uid, amount)` explicitly says it applies earnings through debt/loan deduction.
So tasks do **not** simply add raw balance.
They pass through the same debt/loan/tax logic as other earnings.

That is good architecture because the economy rules stay consistent.

### Staff assigned tasks
- `.taskassign @user <reward> <description>`
- `.taskcomplete @user <task_id>`

This lets staff manually create work for users and issue rewards when done.

### Python lessons

- task state machines
- reset logic
- event-driven progress tracking
- reusing shared economy settlement logic
- centralizing repeated behavior in helper methods

---

## 10. `savings.py`

### Purpose
This cog adds a bank-style savings account separate from the wallet.

### Core idea
Users have:
- a wallet balance
- a savings balance

Savings is safer or more structured, and it also earns interest.

### Commands
- `.savings`
- `.deposit <amount>`
- `.withdraw <amount>`
- `.interesttoggle`
- `.interestcheck`
- `.savingsset`

### Limits and rules
- deposit max is `40%` of wallet per operation
- deposit cooldown is `1800` seconds
- withdraw cooldown is `3600` seconds

### Interest system
There are two background loops:
- daily loop
- weekly loop

Using:
- `@tasks.loop(hours=24)`
- `@tasks.loop(hours=168)`

This is one of the first real "background job" patterns in the project.

### Why that matters
When the cog loads, those loops start running and periodically update user savings balances.

### Toggles
The cog supports enabling/disabling daily and weekly interest separately.
That makes it both a game mechanic and an admin balancing tool.

### Python lessons

- periodic tasks with `discord.ext.tasks`
- account separation inside one user record
- cooldowns for financial actions
- admin controls for economy balancing

---

## 11. `banking.py`

### Purpose
This cog handles loans and delinquency.

### Key concept
Users can request a loan if they meet requirements. Then a portion of future earnings is automatically used to repay it.

### Important constants
- `MIN_BALANCE_LOAN = 500`
- `DEFAULT_LOAN_MAX = 1500`
- `DEFAULT_REPAY_RATE = 15`

### Loan lifecycle
#### 1. request
`.loanrequest <amount>`

The user requests a loan. Requirements likely include having enough baseline balance/history.

#### 2. approve or deny
Handled by bank owner/admin commands:
- `.loanapprove`
- `.loandeny`

#### 3. active loan
Once approved, loan data is stored in `data/loans.json`.

#### 4. repayment
Repayment happens automatically in `channel_guard.settle_debt(...)`.
That means many earning actions slowly chip away at the loan.

#### 5. delinquency
If unpaid too long, the loan becomes delinquent.
That can freeze or restrict the account.

### Interest tiers
There is an `INTEREST_TIERS` constant and `_interest_rate(issued_at)` helper.
That suggests the longer a loan remains open, the more expensive or severe it becomes.

### Blocked commands
`DELINQUENT_BLOCKED_COMMANDS` means users with bad loan status lose access to certain economy features.
This is effectively a punishment mechanic.

### Background loops
This cog has several background tasks:
- `interest_loop` every 48 hours
- `delinquent_loop` every 24 hours
- `bracket_notify_loop` every 24 hours

So this cog does not only respond to commands. It also performs periodic maintenance.

### Commands
- `.loanrequest`
- `.loanapprove`
- `.loandeny`
- `.loanview`
- `.loanpay`
- `.loanrate`
- `.loanstatus`
- `.loanlist`

### Python lessons

- separate persistent file for a subsystem
- approval workflow
- automatic repayment integration with shared economy logic
- time-based state transitions like active → delinquent
- maintenance loops for background enforcement

---

## 12. `taxation.py`

### Purpose
This cog adds taxes and a shared community pot.

### Two kinds of tax

#### 1. Arbitrary tax
A custom tax assigned to a user by staff.

Example:
- tax one user at 20% for 7 days

#### 2. Bracket tax
Top 5% users are taxed automatically.

Constants:
- `TAX_BRACKET_RATE = 0.10`
- `TAX_BRACKET_THRESHOLD = 0.05`

That means top earners can pay 10% on earnings into the community pot.

### How taxes are actually applied
Taxes are not manually deducted by commands every time.
They are applied during earnings settlement:

- `apply_arb_tax(uid, earned)`
- `apply_bracket_tax(uid, earned)`

These are called from `channel_guard.settle_debt(...)`.

That is a great architectural choice because:
- every earning source behaves consistently
- tax logic stays centralized
- other cogs do not need to reimplement tax rules

### Community pot
The pot is stored in `data/community_pot.json`.
Tax revenue likely gets accumulated there.

### Commands
- `.tax set @user rate duration`
- `.tax remove @user`
- `.tax toggle @user`
- `.tax bracket`
- `.tax check [@user]`
- `.pot`
- `.pot give @user amount`
- `.pot add amount`

### Python lessons

- centralizing financial rules into pure helper functions
- time-based config for temporary taxes
- percentile/bracket logic
- separate file for a shared communal resource

---

## 13. `blackjack.py`

### Purpose
A mini-game cog where users gamble Zest against the dealer.

### Core command
- `.blackjack <bet>` or `.bj <bet>`

### What it teaches
This is not just economy mutation. It is a **stateful game session**.

### Helper functions
- `_new_deck()`
- `_card_str()`
- `_hand_str()`
- `_hand_value()`
- `_is_blackjack()`

These helpers isolate card logic from Discord UI logic.
That separation is very good practice.

### UI class: `BlackjackView`
Uses buttons:
- Hit
- Stand

That means the game is interactive and persistent over multiple clicks.

### `BlackjackGame`
This class handles the state of a single blackjack game.
Likely stores:
- player hand
- dealer hand
- bet
- whether game ended
- associated message/view

Methods like:
- `start()`
- `update()`
- `end()`

show a clear object-oriented design.

### Economy tie-in
Starting a game probably reserves or subtracts the bet, then payout logic at the end adjusts wallet balance based on win/loss/push.

### Python lessons

- object-oriented design for game state
- helper-function decomposition
- Discord button UI
- stateful interaction across multiple user actions

---

## 14. `blackmarket.py`

### Purpose
This is a rarer, more game-like item market system with drops, boosts, and resale.

### Data files
- `data/blackmarket.json`
- `data/bmarket_inventory.json`
- `data/bar_economy.json`

### Big concepts

#### rotating drops
There is a timed `drop_loop` every 2 hours.
It posts a new market item.

#### item ownership
Purchased items go into inventory.

#### listing/resale
Users can list owned items back onto the black market.

#### boosts
Some items give active boosts that affect other cogs.
For example, bar economy checks boosts via `_get_active_boost(...)`.

### Item types
- `consumable`
- `boost`
- `rare`

### Effects
The file defines `BOOST_EFFECTS` and `VALID_EFFECTS`, meaning items can have behavior beyond being collectible.

### Rich-user balancing
It includes `_is_rich(uid)` and lower cashback for very wealthy users.
That means the designer is intentionally preventing rich users from exploiting cashback too hard. Petty, but structurally sensible.

### Commands
- `.bmarket`
- `.bmarket buy [listing_id]`
- `.bmarket inventory`
- `.bmarket list <inv_id>`
- `.bmarket add ...`
- `.bmarket grant ...`
- `.bmarket drop`
- `.bmarket price ...`
- plus admin pool/listing tools

### UI
It also uses Discord UI select menus for channel selection.
That is more advanced than plain text commands.

### Python lessons

- timed content rotation
- inventories and resale markets
- item effect systems
- UI components beyond buttons
- balancing systems tied to player wealth tiers

---

## 15. `marketplace.py`

### Purpose
Player-to-player marketplace for more general listings.

### Listing types
- `item`
- `service`
- `zest`
- `custom`

### Core idea
Users can list things for sale and other users can buy them.
This is separate from the black market, which is more curated/gamey.

### Cuts
- `HOUSE_CUT = 0.10`
- `SELLER_CUT = 0.90`

So the marketplace keeps a 10% fee.

### Special handling by type

#### `item`
Must exist in black market inventory first.
The item is removed from inventory and held as listed stock.

#### `service`
Requires staff approval before going live.
This creates a moderation queue using `pending` entries.

#### `zest`
Seller must actually have the Zest being sold, and it is deducted when listed.

#### `custom`
Freeform listing.

### Listing IDs
Uses `new_id("l")` from `id_gen.py`.
So marketplace listings get unique IDs with a deterministic prefix and random suffix.

### Commands
- `.market [page]`
- `.market sell ...`
- `.market view <id>`
- `.market buy <id>`
- `.market listings`
- `.market delist <id>`
- `.market pending`
- `.market approve <id>`
- `.market deny <id>`
- `.market remove <id>`

### Python lessons

- multi-type validation logic
- approval queues
- marketplace fee distribution
- reusing inventory and ID systems from other cogs

---

## 16. `chamberlain.py`

### Purpose
This is a service-booking system, parallel to the bar ordering system.

Think of it as the bot equivalent of:
- browse services
- book a slot
- confirm booking
- pay
- notify relevant party

### Data files
- `chamberlain_menu.json`
- `chamberlain_orders.json`
- `chamberlain_pending.json`
- `chamberlain_availability.json`
- `bar_economy.json`

### Main behaviors
- display services with `.services`
- tip with `.tip`
- manage staff availability with `.availability`
- manage service definitions with `.service`
- use slash `/book` to make bookings

### Key design feature
There are two UI view classes:
- `_BookingStatusView`
- `_BookConfirmView`

That means the booking flow is interactive.

### Availability system
There is a separate availability file and helpers like `_is_available(...)`.
This suggests service booking checks slot availability before confirming.

### Commission
`COMMISSION_RATE = 0.15`
Like bar ordering, service revenue probably has a cut system.

### Why this cog matters for learning
This is a very good example of a **workflow cog**, not just a one-shot command cog.
A booking is a multi-step process:

1. browse service
2. request booking
3. check availability
4. confirm
5. deduct funds
6. save order
7. notify staff/user

### Python lessons

- workflow/stateful command design
- multi-file persistence
- UI-driven confirmation flow
- availability calendars and slot checks

---

## 17. `leaderboard.py`

### Purpose
This cog computes rankings from different economy-related sources.

### Data sources
- `bar_economy.json`
- `bar_orders.json`
- `chamberlain_orders.json`

### Types of leaderboards
From the help text and function names, it supports things like:
- balance
- orders
- services
- debt
- lifetime earned

### `zestbalance`
This command gives a detailed overview for one user, including:
- wallet balance
- debt
- lifetime earned
- number of bar orders and spend
- number of service bookings and spend
- loan information

This is not just a top-N ranking command. It is also an economy inspection tool.

### Why it matters
This cog is mostly **aggregation logic**.
It does not create new data much. It reads across multiple files and summarizes them.

That is a very common real-world programming job:
not “create system,” but “read several systems and combine them into one report.”

### Python lessons

- aggregation across files
- sorting/ranking users
- derived metrics like lifetime earned or spend totals
- reporting via embeds

---

## 18. `reviews.py`

### Purpose
This cog creates a gated review system for hotel reviews and item reviews.

### Review gating rules
Constants show requirements like:
- minimum 10 orders
- minimum 30 days in server
- hotel review cooldown: 14 days
- item review cooldown: 3 days
- item review daily cap: 1

So this is not an open spam system. It deliberately checks credibility/history.

### Moderation/filtering
It includes:
- word count minimum/maximum
- hard-blocked terms list
- text filter checks

### UI components
- `HotelReviewModal`
- `ItemReviewModal`
- `ReviewWarningView`

This means users submit reviews through Discord modals and confirmation views.

### Slash commands
This cog defines slash commands like:
- `/review hotel`
- `/review item`
- `/review delete`
- `/review hide`
- `/reviews hotel`
- `/reviews item`

Autocomplete is used for item selection.
That is a strong sign the author wanted a more polished slash UX here.

### How the logic works
Before allowing submission, `_check_eligibility(member)` verifies whether the user is eligible.
Then the modal collects the review body and rating info.

### Why this cog is educational
It combines:
- validation
- user eligibility checks
- modals
- moderation actions
- storage and retrieval

That is a fairly complete feature module.

### Python lessons

- modal-based input
- validation rules before submission
- autocomplete
- soft moderation actions like hide/delete
- timestamp-based cooldown checks

---

## 19. `roles.py`

### Purpose
This cog does two things:

1. toggle a dedicated verify role
2. create custom text commands that add/remove/toggle roles

### `.verify @user`
Staff can give or remove the configured verify role.
That is straightforward role management using Discord's API.

### Dynamic role command system
This is the more interesting part.
Staff can create commands dynamically with:
- `.rolecmd create <name> <add|remove|toggle>`

The cog then asks which role that command should affect.
It stores the command definition in `data/role_commands.json`.

Example concept:
- create command `.vip @user`
- bot stores that `.vip` should toggle the VIP role

### How it executes custom commands
It uses `on_message`.

When a message starts with the prefix:
1. strip prefix
2. get command name
3. load dynamic commands
4. if the command exists and isn't already a real registered bot command,
5. parse the mentioned user
6. add/remove/toggle the configured role

That is a great example of building your own mini command parser.

### Python lessons

- persistence-backed dynamic behavior
- prompt/reply interaction with `wait_for`
- role manipulation
- intercepting raw messages to implement custom commands

---

## 20. `help.py`

### Purpose
Custom help command.

### Why custom help exists
Default help is often ugly or too generic, and it does not understand your bot's permission tiers well.

### What this cog does
It builds different help output depending on whether the user is:
- regular user
- staff
- admin
- owner

### Main command
- `.bothelp`
- alias: `.help`

### Design style
This file is basically a big embed builder.
It is more about formatting and access-tier presentation than business logic.

### Why it matters
It acts as living documentation for the bot.
It also reveals how the author thinks about the bot's feature categories.

### Python lessons

- permission-sensitive output
- embed composition
- using one helper method to build multiple help variants

---

## 21. `econmute.py`

### Purpose
This cog can temporarily or permanently block a user from using economy-related features.

### How it works
It stores mute state in the economy file and checks whether a command should be blocked.

### `BLOCKED_COMMANDS`
This constant contains economy commands like:
- daily
- work
- beg
- grind
- rob
- give
- order
- deposit
- withdraw
- blackjack
- loanrequest
- and others

### Why this is useful
This is a moderation system specifically for the economy, not a full Discord mute.
A user can still chat, but they lose access to the economy/game loop.

### Commands
- `.econmute`
- `.econunmute`
- `.econmutecheck`

### Listener
It also hooks `on_command`, which means it can intercept economy commands before they complete.

### Python lessons

- feature-specific moderation
- command interception
- central blocklist design

---

## 22. `whitelist.py`

### Purpose
Server whitelist system.
This controls **which guilds the bot is allowed to stay in**.

### Why it exists
If this is a private or semi-private bot, the owner does not want it freely invited anywhere.

### Behavior
When the bot joins a guild:
- if guild is whitelisted, it stays
- if not, it leaves automatically

### Time-limited whitelisting
A guild can be whitelisted for N days or permanently.
The cog stores expiry information and prunes expired guilds every 6 hours.

### Commands
- `.whitelist`
- `.unwhitelist`
- `.whitelistview`
- `.servers`
- `.whitelistextend`

### Nice touch
`get_inviter(...)` tries to inspect audit logs to figure out who invited the bot.
That is useful for accountability.

### Python lessons

- guild lifecycle events
- expiry-based permissions
- background pruning tasks
- owner-only infrastructure control

---

## 23. `nya_bully.py`

### Purpose
A novelty/reactive cog that watches for a specific user's “nya” messages and reacts with escalating skull energy.
Because no serious bot project is complete without a deeply specific bit. Naturally.

### How it works
Uses `on_message`.

Checks:
- bot is enabled
- message author matches target ID
- message content matches regex pattern
- channel is one of the configured allowed nya channels

### Spam escalation
Tracks recent usage in memory via:
```py
self._nya_state
```

If the target user keeps saying “nya” within a configured time window, the bot:
- adds reactions
- escalates replies
- becomes increasingly exhausted in tone

### Commands
- `.nyatoggle`
- `.nyachannel`
- `.nyachannel add`
- `.nyachannel remove`
- `.nyachannel removeserver`
- admin config via `.nyacfg ...`

### Python lessons

- regex pattern matching
- in-memory rate tracking with `defaultdict`
- configurable message listeners
- per-guild channel configuration

---

## 24. `info.py`

### Purpose
Posts a review-information embed explaining how the review system works.

### Command
- `.reviewinfo`

### Why this exists
It decouples "explain the rules" from the review submission system itself.
That is clean design.

### Python lessons

- information-only cog
- embed-based documentation command

---

## 25. `id_gen.py`

### Purpose
Shared deterministic ID-prefix plus random suffix generator.

### How it works
- builds a constant prefix from a hash of owner name + DOB
- generates a random suffix with `os.urandom`
- optionally inserts a tag like `l` for listing IDs

Example:
- `yd06_labcd`

### Why this is clever
It creates IDs that are:
- short
- fairly unique
- consistent in format
- harder to guess than simple incremental integers

### Python lessons

- hashing with `hashlib`
- random bytes with `os.urandom`
- custom base-36 encoding

---

# Relationships between cogs

## 26. Cross-cog dependencies you should absolutely notice

This project is modular, but the cogs are **not independent islands**.
Several of them rely on each other.

### Biggest shared dependency: `bar_economy.json`
Used by many cogs:
- `bar_economy`
- `channel_guard`
- `tasks`
- `savings`
- `banking`
- `taxation`
- `blackmarket`
- `marketplace`
- `econmute`
- `leaderboard`
- likely `bar_menu` and `chamberlain`

### Most important function link: `settle_debt(...)`
Defined in `channel_guard.py`, used as the earning-adjustment pipeline.

This function connects:
- debt
- loans
- arbitrary taxes
- bracket taxes

So when you see an earning command in another cog, mentally ask:
> Does it just add raw money, or does it pass through `settle_debt`-style logic?

That question matters a lot.

### `blackmarket` → `bar_economy`
Boosts bought in the black market affect economy behavior.

### `marketplace` ↔ `blackmarket`
Marketplace item listings depend on black market inventory.

### `leaderboard` reads multiple subsystems
It aggregates from economy, bar orders, and chamberlain orders.

### `roles` uses raw `on_message`
Because dynamic commands are not registered as normal Discord.py commands, it manually parses messages.

### `reviews` uses order history
Review eligibility depends partly on actual order data.

---

# Beginner Python takeaways from this project

## 27. What this project teaches really well

### A. State management
Load JSON, modify dicts/lists, save JSON.

### B. Helper functions
Many cogs use small private helpers like `_load()`, `_save()`, `_fmt_cd()`, `_get_user()`.
That is exactly how you keep logic readable.

### C. Separation of concerns
Different files focus on different responsibilities.
That is much better than one giant bot file.

### D. Event-driven programming
Discord bots are event-driven:
- commands
- listeners
- button clicks
- modal submits
- background loops

### E. UI layers in Discord.py
You see all three major user interaction styles:
- prefix commands
- slash commands
- UI components (buttons/selects/modals)

---

# Things to watch out for as a learner

## 28. Design tradeoffs and possible weaknesses

### 1. Heavy JSON dependence
This is fine for small bots, but JSON becomes fragile as complexity grows.
A real database would eventually be better.

### 2. Shared-file write collisions
Because many cogs write the same economy file, bugs can happen if two features mutate it in conflicting ways.

### 3. Permission logic is repeated
Several cogs re-define staff ID sets and role ID sets.
That would be better centralized in a utility module.

### 4. Cross-cog imports can create coupling
Example: `channel_guard` imports tax functions, and `bar_economy` checks black market boosts.
That is workable, but over time it can get messy if not managed carefully.

### 5. Constants are embedded in code
Channel IDs, role IDs, and owner IDs appear directly in files. That works, but moving more of that to config would improve maintainability.

---

# Best order to study these files

## 29. Recommended reading order for learning

If you want to understand the project with the least suffering possible, read in this order:

1. `bot.py`
2. `bar_economy.py`
3. `channel_guard.py`
4. `tasks.py`
5. `savings.py`
6. `banking.py`
7. `taxation.py`
8. `bar_menu.py`
9. `chamberlain.py`
10. `blackjack.py`
11. `marketplace.py`
12. `blackmarket.py`
13. `leaderboard.py`
14. `reviews.py`
15. `roles.py`
16. `help.py`
17. `whitelist.py`
18. `nya_bully.py`
19. `info.py`
20. `id_gen.py`

Why this order works:
- first understand bot startup
- then understand the main economy data model
- then understand the deduction pipeline
- then move into feature subsystems
- then finish with utility/specialty cogs

---

# Final summary

## 30. What this bot is, in one sentence

This is a modular Discord economy/RP/game bot where nearly every system funnels through a shared wallet, debt, loan, tax, inventory, and order ecosystem.

## 31. What matters most for you as a Python learner

If you understand these five things, you understand most of this project:

1. how cogs are loaded and structured
2. how JSON persistence works
3. how user data is initialized and mutated safely
4. how command/listener/UI patterns differ
5. how shared helper functions connect separate subsystems

## 32. The single most important architectural lesson

This project is not really “a bunch of random commands.”
It is a network of small systems that all modify shared state.

That is the part worth studying closely.
The commands are just the shiny buttons humans press while the real machinery mutters underneath.

