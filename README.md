# HFT notes
## Trading Systems
### Does not require ultra-low latency

- It's important to note that even in trading systems that don't demand ultra-low latency, efficient and reliable execution remains important.
- Latency requirements may vary depending on the specific strategy, market conditions, and the desired trade execution speed.
1. [Positional Trading Systems](https://github.com/reshinto/hft_notes/blob/main/trading_systems/position_trading_system.md)
    - Positional trading involves holding positions in securities for longer durations, ranging from days to months or even years.
    - These systems typically analyze fundamental factors, macroeconomic indicators, or technical patterns to identify investment opportunities.
    - Since the holding period is longer, the latency requirements are less stringent compared to high-frequency trading.

2. [Swing Trading Systems](https://github.com/reshinto/hft_notes/blob/main/trading_systems/swing_trading_system.md)
    - Swing trading strategies aim to capture short-to-medium-term price swings in the market.
    - These systems analyze technical indicators, chart patterns, and market trends to identify potential entry and exit points.
    - While swing trading may involve more frequent trades than positional trading, it operates on a longer time horizon, allowing for slightly higher latency tolerance.

3. [Trend-Following Systems](https://github.com/reshinto/hft_notes/blob/main/trading_systems/trend_following_system.md)
    - Trend-following strategies aim to identify and capitalize on longer-term price trends in the market.
    - These systems use various indicators and trend analysis techniques to determine the direction and strength of a market trend.
    - The emphasis is on staying in the trade for an extended period, so ultra-low latency is not critical.

4. [Fundamental Investing Systems](https://github.com/reshinto/hft_notes/blob/main/trading_systems/fundamental_investing_system.md)
    - Fundamental investing strategies rely on analyzing a company's financials, industry trends, and other fundamental factors to make long-term investment decisions.
    - These systems often require comprehensive research and analysis rather than real-time market data, so low-latency requirements are not as important.

5. [Options or Derivatives Trading Systems](https://github.com/reshinto/hft_notes/blob/main/trading_systems/options_or_derivatives_trading_system.md)
    - Trading systems focused on options or other derivatives may not require ultra-low latency, as these markets often have longer expiration periods and less frequent trading.
    - Strategies involving options spreads, volatility trading, or hedging positions can operate with slightly higher latency requirements.

### Requies ultra-low latency

- In these trading systems, even small differences in latency can significantly impact profitability and competitiveness.
- Therefore, HFT firms invest heavily in low-latency technology infrastructure, including high-speed networks, proximity hosting, and optimized software algorithms, to minimize execution times and maintain an edge in the markets.
1. [High-Frequency Trading (HFT) Systems]()
    - HFT strategies involve executing a large number of trades within very short timeframes, often taking advantage of small price discrepancies or market inefficiencies that exist for only fractions of a second.
    - These systems rely on ultra-low latency to ensure rapid order placement, market data processing, and order execution to exploit fleeting opportunities.

2. [Market Making Systems]()
    - Market making involves providing liquidity to the market by continuously placing bid and ask quotes for a particular security.
    - Market makers aim to profit from the bid-ask spread while minimizing the risk exposure.
    - Ultra-low latency is essential for market-making systems to swiftly adjust quotes in response to changing market conditions and to ensure fast order execution when interacting with incoming orders.

3. [Arbitrage Trading Systems]()
    - Arbitrage strategies seek to profit from price discrepancies between different markets, exchanges, or instruments.
    - These price differences may exist for only short periods, requiring ultra-low latency to identify, analyze, and execute trades swiftly to exploit the arbitrage opportunity before it diminishes.

4. [Scalping Systems]()
    - Scalping strategies aim to profit from small price fluctuations by entering and exiting trades rapidly.
    - These systems typically seek to capture very short-term market movements and require ultra-low latency to enter and exit positions quickly with minimal slippage.

5. [News-Based Trading Systems]()
    - Some trading systems rely on news or event-driven strategies, where the speed of information processing is crucial.
    - These systems analyze news feeds, social media sentiment, or other data sources to identify market-moving events and execute trades based on the anticipated impact.
    - Ultra-low latency is essential to process and react to news events in real-time.

6. [Statistical Arbitrage Systems]()
    - Statistical arbitrage strategies seek to identify and exploit statistical patterns or relationships between different securities or assets.
    - These systems often involve high-frequency trading and require ultra-low latency to capitalize on the short-lived opportunities identified by the statistical models.

## Commonly used algorithms

1. Market Making Algorithms
    - Market making algorithms are used to provide liquidity to the market by continuously quoting bid and ask prices for a particular security.
    - These algorithms dynamically adjust quotes based on market conditions, order book depth, and other factors to maintain a balanced position and profit from the bid-ask spread.

2. Statistical Arbitrage Algorithms
    - Statistical arbitrage algorithms aim to exploit pricing inefficiencies or relative mispricings between related securities based on statistical models.
    - These algorithms analyze historical relationships, correlations, or deviations between assets and generate trades to capture opportunities for price convergence or divergence.

3. Momentum Trading Algorithms
    - Momentum trading algorithms identify and capitalize on short-term price trends or momentum in the market.
    - These algorithms use technical indicators, market data analysis, and pattern recognition techniques to enter positions aligned with the prevailing trend and exit when the momentum weakens.

4. Pairs Trading Algorithms
    - Pairs trading algorithms involve trading a pair of related securities simultaneously, taking advantage of their relative price movements.
    - These algorithms monitor the price spread between the two securities and execute trades when the spread deviates from its historical mean, anticipating a reversion to the mean.

5. Liquidity Detection Algorithms
    - Liquidity detection algorithms aim to identify and interact with hidden or non-displayed liquidity in the market, such as dark pools or alternative trading venues.
    - These algorithms employ sophisticated order routing and execution techniques to access and execute trades in less visible liquidity sources.

6. Smart Order Routing (SOR) Algorithms
    - SOR algorithms optimize order routing decisions by dynamically analyzing market conditions, liquidity availability, and execution costs across multiple exchanges or trading venues.
    - These algorithms split orders, route them to the most favorable venues, and balance execution speed and price improvement to achieve best execution.

7. Execution Algorithms
    - Execution algorithms are designed to optimize the execution of large orders by breaking them into smaller, manageable parts and executing them over time. 
    - These algorithms consider factors like market impact, trading volumes, and order book dynamics to minimize slippage and achieve efficient execution.

8. Scalping Algorithms
    - Scalping algorithms focus on making a large number of small-profit trades by capturing small price differentials or spreads.
    - These algorithms aim to capitalize on short-term price fluctuations and typically employ high-frequency trading techniques to execute trades rapidly.

9. Volatility Trading Algorithms
    - Volatility trading algorithms aim to profit from changes in market volatility levels.
    - These algorithms employ options strategies, volatility derivatives, or dynamic hedging techniques to take positions based on anticipated volatility movements or shifts in implied volatility levels.

10. High-Frequency Trading (HFT) Algorithms
    - HFT algorithms encompass a broad range of ultra-low latency algorithms that execute a large number of trades within extremely short timeframes.
    - These algorithms exploit small price discrepancies, market microstructure patterns, or latency arbitrage opportunities, relying on speed and technology to gain a competitive edge.

## Common structure

1. Data Ingestion and Processing
    - Connect to market data sources or APIs.
    - Receive and parse incoming market data feeds in real-time.
    - Store or update relevant market data in memory or a database.
2. Strategy Development
    - Implement mathematical models, statistical analysis, or machine learning techniques to generate trading signals.
    - Define rules and parameters for trade entry, exit, position sizing, and risk management.
    - Incorporate any proprietary algorithms or trading strategies specific to the firm's approach.
3. Order Generation
    - Translate trading signals or strategy rules into order generation logic.
    - Determine the desired order type (market, limit, etc.) and order quantity based on the trading strategy.
    - Apply any pre-trade risk checks or constraints defined by regulatory requirements or risk management rules.
4. Order Routing and Execution
    - Connect to order execution venues or trading platforms via APIs.
    - Route orders to the appropriate venues based on factors like liquidity, execution speed, or transaction costs.
    - Handle order execution confirmations and updates from the venues.
    - Implement algorithms for smart order routing, balancing tradeoff between speed and best execution.
5. Risk Management
    - Monitor positions, exposure, and overall risk metrics in real-time.
    - Apply risk limits or controls to prevent excessive loss or exposure.
    - Implement risk mitigation techniques like hedging or portfolio rebalancing.
6. Performance Monitoring and Analysis
    - Track and record trading performance metrics, including P&L (profit and loss), trade statistics, and order execution quality.
    - Conduct post-trade analysis to evaluate the effectiveness of the trading strategies.
    - Monitor latency and system performance to ensure efficient operation and identify any issues.
7. Infrastructure and Connectivity
    - Build and maintain a high-performance, low-latency infrastructure.
    - Optimize networking, server hardware, and software configurations for fast data processing and order execution.
    - Implement fault-tolerant and high-availability mechanisms to ensure system reliability.

## Common protocols

1. FIX (Financial Information eXchange) Protocol
    - FIX is a widely used protocol in the financial industry for communication between trading systems, exchanges, and brokers.
    - It is a standard messaging protocol that facilitates order routing, trade execution, and market data dissemination.
    - FIX provides a structured format for transmitting trade-related information and is commonly used in both internal and external communications.
2. REST (Representational State Transfer) API
    - REST is an architectural style used for building web services that can be accessed over the internet.
    - REST APIs use HTTP as the underlying communication protocol and are often employed for accessing external APIs or web-based services.
    - HFT firms may utilize REST APIs to connect to data providers, exchanges, or other external platforms for market data retrieval or order placement.
3. WebSockets
    - WebSockets is a communication protocol that provides full-duplex, bidirectional communication between a client and a server over a single, long-lived connection.
    - It allows for real-time data streaming and enables efficient and low-latency communication.
    - HFT firms may utilize WebSockets to receive streaming market data or execute trades in real-time.
4. Native API Protocols
    - Some proprietary trading systems or platforms provide their own custom API protocols for connectivity.
    - These protocols are specific to the system or platform being accessed and often offer optimized and low-latency communication.
    - HFT firms may integrate with these native APIs to access internal trading systems or platforms.
5. Binary Protocols
    - Binary protocols are used for high-performance and efficient data transmission.
    - They typically involve encoding data in a binary format to reduce overhead and increase transmission speed.
    - HFT firms may employ custom binary protocols for communication between internal systems or for high-speed connectivity with specific trading platforms or exchanges.
6. TCP/IP (Transmission Control Protocol/Internet Protocol)
    - TCP/IP is the fundamental protocol suite used for internet communication.
    - It provides reliable, connection-oriented communication between network devices.
    - HFT firms often utilize TCP/IP as the underlying protocol for communication with internal systems, external APIs, or data providers.
7. gRPC
    - gRPC is a modern, high-performance, open-source framework developed by Google that allows efficient communication between client and server applications.
    - It is based on the protobuf (Protocol Buffers) serialization format, which enables fast and compact data transmission.
    - usage examples
        1. Internal Communication
            - gRPC can be used for inter-process communication or communication between different components within the trading system.
            - It offers a lightweight, efficient, and high-performance communication channel, which is crucial for fast and reliable data exchange between different modules or services.
        2. Microservices Architecture
            - HFT firms may adopt a microservices architecture, where different components of the trading system are developed and deployed as separate services.
            - gRPC can be utilized as the communication protocol between these microservices, allowing them to interact efficiently and exchanging data in a performant manner.
        3. API Development
            - HFT firms may develop their own APIs for internal purposes, such as accessing proprietary trading functionalities or connecting with specific systems. gRPC can be employed to define and implement these APIs, providing a scalable and efficient communication mechanism.
        4. Integration with External Services
            - HFT firms often need to integrate with external services, such as market data providers or execution venues.
            - gRPC can be used to develop client applications that communicate with these external services, allowing for fast and reliable data retrieval, order placement, and trade execution.

## Common Architectures

- Factors such as trading strategies, market focus, regulatory requirements, and existing technology infrastructure influence the selection of the most suitable architecture for a particular trading system.
1. Monolithic Architecture
    - In a monolithic architecture, the entire trading system is built as a single, self-contained application.
    - All components, such as market data processing, order management, strategy execution, and risk management, are tightly coupled within a single codebase. 
    - While this design simplifies development and deployment, it may pose challenges in scalability and flexibility.
2. Microservices Architecture
    - Microservices architecture decomposes the trading system into a collection of loosely coupled, independently deployable services.
    - Each service focuses on a specific functionality, such as market data retrieval, order routing, risk management, or analytics.
    - These services communicate with each other using lightweight protocols (such as gRPC or REST) and can be developed, deployed, and scaled independently.
    - This design allows for better scalability, fault isolation, and flexibility in adding or updating components.
3. Event-Driven Architecture
    - Event-driven architecture focuses on asynchronous communication and processing of events.
    - In this design, various components of the trading system publish and subscribe to events, such as market data updates, order executions, or system notifications.
    - Components react to events asynchronously, enabling parallel processing and faster response times.
    - Event-driven architectures can help reduce latency and enable high throughput in HFT systems.
4. Distributed Computing Architecture
    - Distributed computing architectures leverage multiple nodes or machines working together to process data and execute trades.
    - This design enables parallel processing, load balancing, and fault tolerance.
    - Distributed architectures often involve clusters of servers or cloud-based infrastructure to handle the high volumes of data and compute-intensive tasks in HFT systems.
5. High-Performance Computing (HPC) Architecture
    - HPC architectures utilize specialized hardware, such as FPGA (Field-Programmable Gate Array) or GPU (Graphics Processing Unit), to achieve ultra-low latency and high-performance computing capabilities.
    - These architectures offload computationally intensive tasks, such as market data processing or complex algorithmic calculations, to hardware accelerators, resulting in faster processing speeds and reduced latencies.
6. Hybrid Architectures
    - HFT systems may adopt hybrid architectures that combine multiple design patterns and technologies.
    - For example, a hybrid architecture may leverage microservices for flexibility and scalability while incorporating event-driven messaging for fast and asynchronous communication between components.
