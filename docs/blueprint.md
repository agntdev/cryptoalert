# CryptoAlert — Bot specification

**Archetype:** custom

**Voice:** professional and concise — write every user-facing message, button label, error, and empty state in this voice.

CryptoAlert is a personal Telegram bot that allows users to create private watchlists of cryptocurrencies and receive alerts for price thresholds or percent changes. Users can manage their watchlists through inline buttons and manual entry, set quiet hours, and receive optional morning digests. The bot owner receives aggregated usage metrics.

> This is the complete contract for the bot. Implement EVERY entry point, flow, feature, integration, and edge case below. The completeness review checks the bot against this document after each build pass.

## Primary audience

- Individual crypto traders
- Crypto hobbyists
- Telegram users seeking lightweight price alerts

## Success criteria

- Users can successfully set up and receive alerts for price thresholds and percent changes
- Owner can view aggregated usage metrics including user count and most-triggered alerts
- All user settings and watchlists are persisted across sessions

## Entry points

Every feature must be reachable from the bot's command/button surface (button-first; only /start and /help are slash commands).

- **/start** (command, actor: user, command: /start) — Open the main menu with quick buttons for adding coins, viewing the watchlist, and setting preferences
- **/price** (command, actor: user, command: /price) — Check the current price of a specific coin or the entire watchlist
- **Add coin** (button, actor: user, callback: add_coin:start) — Open the coin selection menu with quick-add buttons for common coins and an option to add a custom ticker
- **My list** (button, actor: user, callback: my_list:view) — View and manage the current user's watchlist with inline buttons for each coin
- **Set quiet hours** (button, actor: user, callback: quiet_hours:start) — Configure quiet hours to suppress notifications during specific times
- **Set morning digest** (button, actor: user, callback: morning_digest:start) — Enable or disable the daily morning digest and set the delivery time

## Flows

### onboarding
_Trigger:_ /start

1. Display main menu with quick buttons for adding coins, viewing the watchlist, and setting preferences

_Data touched:_ User profile

### add_coin
_Trigger:_ add_coin:start

1. Show inline keyboard with common coins (BTC, ETH, TON) and an 'Add custom ticker' option
2. If 'Add custom ticker' is selected, prompt user to type a ticker symbol
3. Validate and add the coin to the user's watchlist

_Data touched:_ Watchlist entry

### manage_watchlist
_Trigger:_ my_list:view

1. Display the user's watchlist with inline buttons for each coin (Remove, Set price alert, Set percent alert, Disable alerts)

_Data touched:_ Watchlist entry

### price_threshold_alert
_Trigger:_ Set price alert

1. Prompt user to enter a target price and direction (above/below)
2. Confirm the alert settings in natural language

_Data touched:_ Watchlist entry

### percent_change_alert
_Trigger:_ Set percent alert

1. Prompt user to enter a percent threshold and optionally direction (up/down/both)
2. Set the period to 1 hour as default
3. Confirm the alert settings in natural language

_Data touched:_ Watchlist entry

### price_check
_Trigger:_ /price

1. If no argument is provided, return current prices and 1h changes for all coins in the watchlist
2. If a specific ticker is provided, return current price and 1h change for that coin

_Data touched:_ Watchlist entry

### morning_digest
_Trigger:_ morning_digest:start

1. Enable or disable the morning digest
2. Set the delivery time in the user's local timezone

_Data touched:_ User profile

### quiet_hours
_Trigger:_ quiet_hours:start

1. Set quiet hours start and end times in the user's local timezone
2. Configure behavior for scheduled digests during quiet hours

_Data touched:_ User profile

## Data entities

Durable data (must survive a restart) uses the toolkit's persistent store, never in-memory maps.

- **User profile** _(retention: persistent)_ — Stores user-specific settings and preferences
  - fields: Telegram user id, Display name, Timezone, Quiet hours start/end, Morning digest enabled + time, Alert cooldown/pause duration
- **Watchlist entry** _(retention: persistent)_ — Represents a cryptocurrency in the user's watchlist
  - fields: Ticker, Display name, Last known price, Enabled alerts (thresholds, percent-change settings), Per-alert cooldown state and last-trigger timestamp
- **Owner metrics** _(retention: persistent)_ — Aggregated usage statistics for the bot owner
  - fields: Total user count, Counts of fired alerts grouped by alert type and ticker

## Integrations

- **Telegram** (required) — User-facing bot chat for interactions and owner admin chat for metrics
- **Price feed** (required) — Reliable market price provider for cryptocurrency prices
- **Cron/scheduler** (optional) — Periodic percent-change checks and daily morning digests
Call external APIs against their real contract (correct endpoints, ids, params); credentials from env. Do not fake responses.

## Owner controls

- View aggregated usage metrics (user count and most-triggered alerts) in the owner's admin chat

## Notifications

- Price threshold alerts with detailed information
- Percent change alerts with detailed information
- Morning digest with watchlist prices and changes
- Error notifications for unknown tickers or typos with suggestions

## Permissions & privacy

- All user data is private and not shared with third parties
- Owner metrics are aggregated and non-identifying
- User settings and watchlists are stored securely

## Edge cases

- Price source failures with silent retries and no alerts using stale data
- Unknown or typo tickers with suggestions and explanations
- Alerts during quiet hours are suppressed or delayed
- Multiple alerts for the same coin with cooldown periods

## Required tests

- Verify that price threshold alerts are triggered and suppressed correctly with cooldown periods
- Verify that percent change alerts are triggered for 1-hour periods with the correct direction
- Verify that morning digests are sent at the correct local time and respect quiet hours
- Verify that unknown tickers are handled with suggestions and explanations

## Assumptions

- Users' timezones are determined from Telegram locale or asked once on first use
- Common coins (BTC, ETH, TON) are offered as quick-add buttons
- Percent-change period is fixed to 1 hour
- Default percent-alert direction is both (up or down)
- Default cooldown after an alert is 3 hours
- Quiet hours behavior suppresses push alerts and delays digests until quiet hours end
- /price with no argument returns the entire watchlist
- Price source failures are retried silently without sending alerts with stale data
- Unknown tickers/typos are handled with suggestions and explanations
- Owner metrics are aggregated and non-identifying
