# Order Books

## What is an order book?
- it is a live list of buy and sell orders.
- It shows prices and sizes people want to trade.
- It updates many times per second in real markets.

|Level|Definition|Other Definition|Typical Use|
|-----|----------|----------------|-----------|
|L1|Shows only the best bid and best ask with sizes. <br>Best bid = highest buy price. <br>Best ask = lowest sell price.|Top-of-Book <br> Quotes <br> Best-Bid-Offer (BBO)|for pricing, display, and routing checks <br>e.g.: show best price; check National Best Bid and Offer (NBBO) before routing a retail order.|
|L2|Shows aggregated depth at each price level. <br>You see multiple prices up and down from the best. <br>Each price shows total queued size.|Depth-of-Book (aggregated) <br>Market-By-Price (MBP) <br>Aggregated Depth <br>Depth-Of-Market (DOM) <br>Order Book View|for depth, imbalance, and short-term pressure <br>It shows how much size sits near the top|
|L3|Shows every order in the queue at each price. <br>You see FIFO order and queue position. <br>You can track adds, partial fills, and cancels.|Order-by-Order (per-order) <br>Market-By-Order (MBO) <br>Full Depth|for queue math, microstructure research, and market making <br>You can estimate queue position and follow each order’s life.|

**L1**
```
BIDS: [10.00 x 80]
ASKS: [10.10 x 10]
```
- Best bid and best ask are the 2 numbers that defines L1
- If a buyer trades at 10.10 for 10, the ask vanishes.
- New ask becomes the next best ask level.
- Spread
  - Best ask minus best bid

**L2**

|Price|Bid Size|Ask Size|
|-----|--------|--------|
|10.12|0|40|
|10.11|0|20|
|10.10|0|10|
|10.00|80|0|
|9.99|200|0|

- L2 totals at each price level
- L2 does not show individual orders
- Imbalance
  - Compare total bid vs total ask near top.
  - If bids `>>` asks, price may tilt up short-term.
  - Layman: More buyers near top can push price up.
  - ELI5: More kids wanting candy may raise the price.
  - why watch imbalance?
    - It hints at short-term pressure.

**L3**

```
Front → [A1:30] [A2:20] [A3:10] ← Back
```

- L3 reveals each individual order and its position
- The earliest order at that price gets to trade first at that price
- FIFO at each price
  - Layman: Earlier orders trade before later ones.
  - ELI5: First kid gets candy first.
- Queue position changes
  - New add at 10.10 enters the back.
  - Partial fill reduces front order first.
  - Cancel removes that order from its position.
    - Layman: People join back, leave front, or step out.
    - ELI5: Kids line up, move forward, or go home.

**Multi-order-book**

- Single venue
  - 1 exchange's book only
  - Layman: One store’s price board.
  - ELI5: One classroom’s candy line.
- NBBO (consolidated)
  - best across venues
  - Layman: Compare stores and pick best buy/sell.
  - ELI5: Check all classrooms and pick the best.
  - example

    |Venue|Best Bid|Best Ask|
    |-----|--------|--------|
    |X|9.99×200|10.03×50|
    |Y|10.00×30|10.02×60|
    |NBBO|10.00|10.02|

- Latency & stale quotes
  - Prices change fast.
  - Slow data can be stale and risky.
  - Layman: Late info can make you buy too high.
  - ELI5: If you hear late, the candy is gone.
- Cross-venue arbitrage (high level)
  - Buy cheap on one venue, sell higher on another.
  - Constraints: fees, latency, size, rules.
  - Layman: Profit from price differences carefully.
  - ELI5: Buy in one classroom, sell in another.
- Where multi-book logic lives
  - `Ingest → normalize → per-venue books → consolidation`
  - Strategy consumes NBBO and venue books.

### Core terms

**Order**
- Instruction to buy/sell with price and size.
- Layman: A note saying “I’ll pay X for Y units.”
- ELI5: A ticket: “I want 3 apples for $2 each.”

**Bid / Ask**
- Bid = buy price. Ask = sell price.
- Layman: Buyer’s offer vs seller’s offer.
- ELI5: “I’ll pay $2” vs “I’ll take $2.10.”

**Price / Quantity (Size)**
- Price per unit; size is count.
- Layman: How much per item; how many items.
- ELI5: $2 per apple; 3 apples.

**Tick**
- Minimum price step.
- Layman: The smallest allowed price change.
- ELI5: You can jump only one small stair.

**Spread**
- `Ask − Bid (must be ≥ 0)`.
- Layman: Gap between best seller and best buyer.
- ELI5: The space between two kids’ offers.

**Depth**
- Total size available at prices.
- Layman: How much you can buy/sell at each price.
- ELI5: How many apples in each spot in line.

**Liquidity**
- How easily you can trade size near the price.
- Layman: Can you trade fast without moving the price a lot?
- ELI5: Can you get many apples without pushing kids aside?

### Priority model

**Price-time priority**
- Better price wins. Earlier time wins.
- Layman: Highest buyer first; lowest seller first; earlier first.
- ELI5: The front of the line goes first.

**Partial fills**
- Trades can match part of an order.
- Layman: You might sell some now, rest later.
- ELI5: You get two apples now, one later.

**Cancels**
- Remove remaining size from the book.
- Layman: You leave the line before trading.
- ELI5: A kid steps out of the line.

**Queue position**
- Your place in FIFO at that price.
- Layman: Closer to the front means faster fills.
- ELI5: Front kids get candy first.

### Multi-venue & NBBO

**Per-venue book**
- L1/L2/L3 for one exchange (venue).

**NBBO**
- National Best Bid and Offer across venues.
- NBBO bid = max(best bids), NBBO ask = min(best asks).
- Layman: Look across all markets and pick best two.
- ELI5: Pick tallest buyer and shortest seller across rooms.

## Order Book Strategies

**Market making**
- Quote both sides, earn the spread.
- Manage inventory and adverse selection risk.
- Layman: Be a shop; buy low, sell slightly higher.
- ELI5: Stand in both lines and trade both ways.

**Imbalance / momentum reading**
- Heavy bids vs asks can hint short-term moves.
- Layman: Many buyers near top can nudge prices up.
- ELI5: More kids wanting candy pushes price up.

**Queue games**
- Earlier position increases fill chance.
- Last in line may never trade.
- Layman: Being early matters a lot.
- ELI5: Front kids get candy; last may miss out.

**Cross-venue arbitrage**
- Exploit price gaps with speed and risk control.
- Layman: Buy cheaper here, sell higher there.
- ELI5: Two rooms, two prices, quick swap.

**Illegal tactics (strictly not allowed)**
- Spoofing (fake orders), layering, manipulation.
- Markets enforce surveillance and penalties.
- Layman: Do not bluff the market with lies.
- ELI5: Don’t pretend to want candy to trick others.

## Performance

**Why performance matters**
- Latency decides who trades first.
- Throughput handles many events per second.
- Determinism eases testing and replay.

**Pipeline: Parse → book update → metrics → log/emit**
- Layman: Read messages, update lines, measure, record.
- ELI5: Hear, change lines, count, write it down.

**Time vs Space**
- Time: aim for O(1) updates on hot path.
- Space: use arrays/maps with cache-friendly layout.

**Optimization levers**
- SoA layout; price-indexed arrays; preallocation.
- Predictable branches; batched I/O; minimal copies.
- Deterministic ordering and fixed seeds.
- Layman: Prepare memory, avoid surprises, write in chunks.

**Why C++**
- Control memory/layout, inlining, and syscalls.
- Great compilers and profiling tools.
- Typical: single-thread hot path + lock-free ingest.
- Layman: Fine control for speed and stability.
