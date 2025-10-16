# Financial Information eXchange (FIX)
- It is the world’s most widely used open messaging standard for electronic trading.
- It defines how trading systems talk to each other—using tag=value fields separated by a special delimiter (the SOH control character)—so orders, executions, allocations, and status updates can flow reliably and be audited later.
- FIX is governed and maintained by the FIX Trading Community, a non-profit standards body.
- It’s used by buy-side firms, brokers (sell-side), trading venues, and even regulators, across asset classes.  ￼

- FIX is built in two layers:
  - a session layer (keeps a durable, sequenced conversation alive with heartbeats, sequence numbers, and recovery)
  - an application layer (the business messages like NewOrderSingle and ExecutionReport).
    - The canonical TagValue encoding uses ASCII fields delimited by SOH (0x01), and includes a computed BodyLength (9) and CheckSum (10) for integrity.  ￼

- Why FIX?
  - It’s vendor-neutral, battle-tested, and interoperable.
  - It enables deterministic replay and audit from text logs, which matters for compliance (e.g., SEC/FINRA, MiFID II) and incident post-mortems.
  - Even though ultra-low-latency venues often expose binary protocols (e.g., ITCH/OUCH/SBE), the industry continues to use FIX for order routing, broker connectivity, and post-trade workflows because it standardizes complex business semantics across many parties.
 
## What is FIX?
**Protocol**: agreed rules for how messages are structured and exchanged.

**TagValue**: tag=value pairs delimited by SOH (ASCII 0x01).

**Session**: long-lived, sequenced conversation between two endpoints.

**Application message** business message (e.g., orders, fills).

**Technical (HFT/quant-dev) definition**
- FIX is an open standard messaging protocol that defines both a session layer (durable, sequenced transport with recovery via Resend) and an application layer (business semantics).
- The session keeps a continuous MsgSeqNum (34) stream with Heartbeats (35=0), TestRequest (35=1) for liveness checks, and Resend Request (35=2)/Sequence Reset (35=4) for recovery.
- The application layer covers order entry (35=D), execution reporting (35=8), cancels (35=F), replace (35=G), market data (35=W/X), allocations, and more.
- The canonical encoding is TagValue with SOH delimiters, BodyLength (9), and CheckSum (10).
- Examples:
  1. A broker Order Management Systems (OMs) speaks FIX 4.4 to a buy-side Execution Management System (EMS) for order routing and drop copy.
  2. An exchange gateway accepts FIX-order entry while native matching runs on an ultra-fast binary protocol; FIX remains the client-facing standard.
 
**Layman definition**
- Think of FIX as the common language that banks, funds, and exchanges use to trade.
- It sets rules for how to talk (session layer) and what to say (application layer).
- Messages are like forms with numbered boxes (tags).
- The forms are sent in a strict order, with keep-alive pings and receipt numbers, so nobody loses a message.
- Examples:
  1. Your fund’s system sends a “buy 100 shares” form to your broker using FIX.
  2. The broker replies with “order accepted” and later “filled 50, then 50” forms.

**ELI5 definition**
- FIX is a walkie-talkie language for trading. Each sentence has label=value parts.
- The walkie-talkies send “are you there?” pings and count each message so missing messages can be re-sent.
- Examples:
  1. “Buy candy 100 pieces” → “Okay, half done” → “All done.”
  2. If a message is missed, the other kid says, “Please repeat #5–#6!”

## Why FIX?
**Interoperability**: connect once, trade with many.

**Determinism**: sequenced, replayable logs.

**Governance**: maintained by an industry body (FIX Trading Community).

**Technical (HFT/quant-dev) definition**
- Open, vendor-neutral: FIX removes proprietary lock-in across buy-/sell-side, venues, and vendors.  ￼
- Operational maturity: Session semantics (heartbeats, resends, sequence state) are standardized, so engines interoperate predictably.  ￼
- Audit & replay: Plain-text FIX logs plus sequence discipline provide a deterministic reconstruction of the trading day—useful for regulatory record-keeping (SEC 17a-4; MiFID II Article 16).
- Examples:
  1. After an outage, teams replay FIX logs to rebuild order state exactly.
  2. Compliance validates fills against FIX logs when investigating client disputes.
  
**Layman definition**
- Works everywhere: Many firms already “speak” FIX, so connecting is easier.
- Clear records: FIX messages are like numbered receipts; you can replay what happened.
- Examples:
  1. Switching brokers is easier because both support FIX.
  2. Regulators can check FIX logs to see every step of a trade.

**ELI5 definition**
- FIX is like standard Lego bricks—they fit with everyone’s sets.
- You can rebuild what happened, piece by piece, using the numbered bricks.

## When is FIX used?
**Pre-trade**: IOIs, quotes, RFQ.

**Trade**: orders, executions.

**Post-trade**: allocations, confirmations, clearing.

**Drop copy**: real-time copies of fills and order events to risk/compliance.

**Technical (HFT/quant-dev) definition**
- FIX began in pre-trade/trade for equities and now spans post-trade (allocations/confirmations) and multiple asset classes.
- Drop copy sessions deliver real-time execution copies for risk and surveillance.
- Examples:
  1. Pre-trade: Buy-side requests quotes from dealers over FIX.
  2. Post-trade: Buyside sends AllocationInstruction to the broker; broker returns confirms. (Message names per FIX application layer.)

**Layman definition**
- Before trading, firms ask for prices using FIX.
- During trading, they send orders and get fills.
- After trading, they split the trade across accounts and confirm details.
- A drop copy is like “CC-ing” your compliance team in real time.
- Examples:
  1. A fund splits a big 10k-share trade into smaller fills and allocates to many clients.
  2. Compliance watches drop copy to catch suspicious activity live.

**ELI5 definition**
- Ask price → buy → tell the teacher what you bought and for whom.
- A drop copy is the teacher’s copy of every note you pass.

## Where is FIX used?
**Buy-side**: asset managers, hedge funds.

**Sell-side**: brokers/dealers.

**OMS/EMS**: order/execution management systems.

**Venues**: exchanges, ATS/dark pools.

**Clearing/Risk**: CCPs, risk hubs.

**Technical (HFT/quant-dev) definition**
- FIX links buy-side EMS/OMS to brokers (sell-side), brokers to exchanges/ATS, and to clearing/risk systems.
- It’s common in on-prem, co-lo, or cloud deployments and typically offers separate cert/UAT vs prod sessions for onboarding and testing. 
- Examples:
  1. OMS → Broker (routing) and Broker → Exchange (venue access).
  2. Venue → Firm drop copy to a surveillance hub; allocations sent post-trade.

**Layman definition**
- FIX is the wiring between the fund’s system, the broker, and the exchange.
- Firms also use it to test in a sandbox before they go live.
- Examples:
  1. Practice account (UAT) first; then real trading (prod).
  2. Real-time copies go to a watchdog system.

**ELI5 definition**
- FIX is the roads that connect homes (funds), shops (brokers), and malls (exchanges).
- There’s a practice road before you take the highway for real.

## How FIX works?
**Heartbeat**: keep-alive ping message (35=0).

**TestRequest**: “please prove you’re alive” (35=1).

**Resend Request**: ask for missing sequence range (35=2).

**Logon/Logout**: start/stop the session (35=A/5).

**Technical (HFT/quant-dev) definition**
- Session Layer: Provides reliable, ordered, recoverable messaging across one or more TCP connections.
  - Uses MsgSeqNum (34) to detect gaps; Resend (2)/SequenceReset (4) resolve them; HeartBtInt (108) and TestRequest (1) monitor liveness; SendingTime (52) checks time drift; Logon (A) negotiates options (e.g., ResetSeqNumFlag (141)), Logout (5) ends.  ￼
- Application Layer: Business messages like NewOrderSingle (D), ExecutionReport (8), OrderCancelRequest (F), Cancel/Replace (G), MarketDataSnapshot (W)/IncrementalRefresh (X).  ￼
- Encoding: TagValue with SOH delimiter (0x01), BodyLength (9) counts bytes between tags 35 and 10 exclusive, and CheckSum (10) is the modulo-256 sum of all bytes. 
- Examples:
  1. If you miss messages 105–110, you send Resend(2) 7=105,16=110 and peer re-transmits with PossDupFlag(43)=Y.
  2. A gateway sends TestRequest(1) if nothing arrives for N×HeartBtInt, expecting a Heartbeat(0) echoing TestReqID(112). 

**Layman definition**
- The session is like a phone call that counts each sentence.
  - If numbers skip, you ask to repeat the missing ones.
- The application is the content (buy, fill, cancel).
- Messages have a length and a checksum so you know nothing got garbled.
- Examples:
  - “I said sentence #8 before #9—please resend #8.”
  - “Say ‘BANANA’ back to prove you’re still there.” (TestRequest/Heartbeat)

**ELI5 definition**
- Session = the call; Application = the conversation.
- If you miss a number, ask your friend to repeat that number.
- ASCII sequence: (session keepalive)

  |Initiator       |   |       Acceptor|
  |---------       | - |      ---------|
  |Logon (A)  |`------------>`|  Logon (A)|
  |Heartbeat (0) |`<-------->`| Heartbeat (0)|
  |TestRequest (1) |`------->`|  (no msgs)|
  |Heartbeat (0,112=ID) |`<---`| (echo)|
  |Resend (2: 105-110) |`--->`|  SequenceReset/Gaps + replays|
  |Logout (5)   |`<--------->`|  Logout (5)|

## Message Anatomy and Core Tags
Common Tags Examples

|Tag|Name|Meaning|Example|
|---|----|-------|-------|
|8|BeginString|FIX version indicator|FIX.4.4|
|9|BodyLength|Byte count (excl. 8/9 and 10)|###|
|35|MsgType|Message type code|D (NewOrderSingle)|
|34|MsgSeqNum|Sequence number|123|
|49|SenderCompID|Sender ID|BUYFIRM|
|56|TargetCompID|Recipient ID|BROKER1|
|52|SendingTime|UTC timestamp|20251017-01:00:00.000|
|11|ClOrdID|Client order ID|CLORD12345|
|55|Symbol|Ticker/instrument|AAPL|
|54|Side|1=Buy, 2=Sell|1|
|38|OrderQty|Order quantity|100|
|40|OrdType|2=Limit, 1=Market|2|
|44|Price|Limit price|183.50|
|59|TimeInForce|0=Day, 3=IOC…|0|
|150|ExecType|Why this ExecReport was sent|1 (Partial fill, 4.2)|
|39|OrdStatus|Current order status|1 (Partially filled)|
|10|CheckSum|Mod-256 sum, 3 digits|###|

[Definitions per FIX spec/dictionaries](https://www.fixtrading.org/standards/tagvalue-online/)

## End-to-End Order Flow Examples

### Session Handshake
#### Logon / Heartbeat / TestRequest / Resend
- Client → Server: Logon (35=A)
  - 35=A Logon; 98=0 no encryption; 108=HeartBtInt(seconds); 141=ResetSeqNumFlag; 553/554 optional creds.

  ```
  8=FIX.4.4|9=###|35=A|49=BUYFIRM|56=BROKER1|34=1|52=20251017-01:00:00.000|
  98=0|108=30|141=Y|553=user123|554=secret|10=###|
  ```

#### Heartbeat / TestRequest / Resend flow
- Server → Client: Logon (35=A) (mirrors 108, confirms session).

  ```
  # Server issues a TestRequest (no traffic seen)
  8=FIX.4.4|9=###|35=1|49=BROKER1|56=BUYFIRM|34=2|52=20251017-01:05:00.000|112=PING1|10=###|
  
  # Client must reply with Heartbeat echoing 112
  8=FIX.4.4|9=###|35=0|49=BUYFIRM|56=BROKER1|34=3|52=20251017-01:05:00.050|112=PING1|10=###|
  
  # Client detects gap (missed #5–#7) and requests resend
  8=FIX.4.4|9=###|35=2|49=BUYFIRM|56=BROKER1|34=8|52=20251017-01:05:01.000|7=5|16=7|10=###|
  
  # Server replays originals with PossDupFlag(43)=Y or uses SequenceReset(4) with GapFillFlag(123)=Y
  ```

### New Order → Partial Fill → Full Fill
- Client → Broker: NewOrderSingle (35=D)
  - 11 client ID; 54 Buy; 40 Limit; 59 Day; 60 TransactTime; 44 price.

  ```
  8=FIX.4.4|9=###|35=D|49=BUYFIRM|56=BROKER1|34=11|52=20251017-01:06:00.000|
  11=CLORD12345|55=AAPL|54=1|38=100|40=2|44=183.50|59=0|60=20251017-01:06:00.000|10=###|
  ```

- Broker → Client: ExecutionReport (35=8) — New (Ack)
  - 150=0 New; 39=0 New; 151 LeavesQty=100; 14 CumQty=0; 6 AvgPx=0.
 
  ```
  8=FIX.4.4|9=###|35=8|49=BROKER1|56=BUYFIRM|34=12|52=20251017-01:06:00.020|
  37=BRK123|11=CLORD12345|17=EX1|150=0|39=0|55=AAPL|54=1|38=100|151=100|14=0|6=0|10=###|
  ```

- Broker → Client: ExecutionReport — Partial fill
  - 150=1 Partial fill (FIX 4.2/4.4 semantics); 39=1 Partially filled; 32 LastQty; 31 LastPx; 151 Leaves=50; 14 Cum=50.
  
  ```
  8=FIX.4.4|9=###|35=8|49=BROKER1|56=BUYFIRM|34=13|52=20251017-01:06:00.500|
  37=BRK123|11=CLORD12345|17=EX2|150=1|39=1|55=AAPL|54=1|38=100|151=50|14=50|6=183.50|32=50|31=183.50|10=###|
  ```

- Broker → Client: ExecutionReport — Full fill
  - 150=2 Fill; 39=2 Filled; Leaves=0; Cum=100; AvgPx=183.50. (ExecType/OrdStatus semantics.)

  ```
  8=FIX.4.4|9=###|35=8|49=BROKER1|56=BUYFIRM|34=14|52=20251017-01:06:01.000|
  37=BRK123|11=CLORD12345|17=EX3|150=2|39=2|55=AAPL|54=1|38=100|151=0|14=100|6=183.50|32=50|31=183.50|10=###|
  ```

### Cancel & Replace
- Client → Broker: OrderCancelRequest (35=F)
  - 41=OrigClOrdID points to the order you’re canceling.

  ```
  8=FIX.4.4|9=###|35=F|49=BUYFIRM|56=BROKER1|34=21|52=20251017-01:07:00.000|
  11=CLORD12345CXL|41=CLORD12345|55=AAPL|54=1|10=###|
  ```

- Broker → Client: ExecReport Pending Cancel → Canceled
  - ExecType=Pending Cancel/Cancelled; OrdStatus mirrors current state.
  ```
  ...|35=8|150=6|39=6|11=CLORD12345CXL|41=CLORD12345|...   # Pending Cancel
  ...|35=8|150=4|39=4|11=CLORD12345CXL|41=CLORD12345|...   # Canceled
  ```

- Client → Broker: Cancel/Replace (Amend) (35=G)
  - New ClOrdID; OrigClOrdID points to last accepted ID; replaces qty/price.
  - Typical flow: Pending Replace → Replace or Cancel Reject (for failure).
  
  ```
  8=FIX.4.4|9=###|35=G|49=BUYFIRM|56=BROKER1|34=22|52=20251017-01:07:05.000|
  11=CLORD12345R2|41=CLORD12345|55=AAPL|54=1|38=200|44=183.40|10=###|
  ```

### (Bonus) Market Data Snapshot (35=W)
- 268 = NoMDEntries; each entry has 269 (0=bid,1=ask), 270 (price), 271 (size).

```
8=FIX.4.4|9=###|35=W|49=VENUE1|56=BUYFIRM|34=51|52=20251017-01:00:00.000|
55=AAPL|268=2|269=0|270=183.45|271=500|269=1|270=183.55|271=400|10=###|
```

## FIX vs REST vs Raw Sockets vs gRPC

|Aspect|FIX (TagValue sessions)|REST (HTTP/1.1)|Raw sockets (custom)|gRPC (HTTP/2+Protobuf)|
|------|-----------------------|---------------|--------------------|----------------------|
|Latency|Text; adequate for routing. Not venue-native speed.|Moderate; HTTP/1.1, headers; no streaming by default.|Can be very low if binary & tailored.|Low; HTTP/2 multiplexing + binary Protobuf. 
|Framing/Streaming|Message-framed; session heartbeats.|Request/response; streaming uncommon.|Whatever you build.|Full duplex streaming over HTTP/2. |
|Statefulness|Stateful session with sequence & resend.|Stateless requests.|You define it.|Long-lived streams with flow control. |
|Back-pressure|App-level (throttles, rejects).|App-level only.|Custom.|Built-in via HTTP/2 flow control. |
|Schema evolution|FIX dictionaries/“FIX Latest”; extensible tags.|Versioned JSON; ad hoc.|Your responsibility.|Protobuf schemas; strong typing. |
|Interoperability|Industry standard; ubiquitous support.|Universal tooling; not trading-specific.|N/A.|Good within an org; less common x-firm. |
|Compliance/Audit|Deterministic text logs; replayable.|Server logs; not sequenced by default.|Custom.|Logs exist, but business replay cross-org is uncommon. |
|Operational maturity|Engines, cert/UAT, drop copy norms.|Mature web ops.|Varies.|Mature inside microservices.|
|Typical use|Broker/exchange routing, post-trade, drop copy.|Internal tools, portals, reports.|Venue-native/binary feeds.|Internal microservices, streaming analytics.  |

**When FIX beats REST/gRPC (examples):**
1. Sequenced recovery: After a blip you must Resend exactly 105–110 with original sequence and PossDupFlag; REST/gRPC don’t define this cross-firm pattern.  ￼
2. Cross-firm audit: A regulator expects FIX logs with 34/52/10 ensuring ordered, checksummed records.  ￼

**When REST/gRPC beat FIX (examples):**
1. Internal microservices: typed contracts, auto-gen stubs, back-pressure, streaming analytics → gRPC wins internally.  ￼
2. Non-streaming workflows: self-service admin APIs, reporting, file drop → REST is simpler to build and consume.

## Gotchas & Pitfalls
- Clock drift → SendingTime(52) checks may fail; keep NTP/PTP tight.  ￼
- Sequence gaps → Resend storms; throttle, rate-limit, and prefer gap-fills where appropriate.  ￼
- Checksum/length errors (10/9) → reject loops; ensure exact SOH and byte counts.  ￼
- Day-change policies → sessions often reset daily; automate clean Logon/Logout and sequence resets.  ￼
- Ordering vs idempotency → Replays set PossDupFlag(43)=Y; your app must dedupe.  ￼
- Environment split → cert/UAT vs prod configs drift; keep Orchestra/ROE in source control.
