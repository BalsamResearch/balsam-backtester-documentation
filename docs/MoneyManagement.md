# Money Management
Strategies are responsible for determining what and when to buy or sell. Money managers on the other hand are responsible for determing *how much* to buy or sell or whether to even take the trade at all.

## Built-in money managers
The backtester has several built-in money managers that implement common position sizing algorithms. These can be found in the `Balsam.MoneyManagement` namespace. Some of these include:

- `FixedFrational`: uses trailing average true range, or alternatively, distance to stop, as a risk measure to size positions. Commonly used for futures trading strategies.
- `FullyInvested`: targets a 100% notional exposure. For futures contracts, notional value will be calculated using the unadjusted close if available.
- `FixedContracts`: targets a fixed number of contracts or shares.
- `PassThrough`: uses trade quantity as defined by the strategy rather than the default behavior of assigning trade quantity from the money manager.

## Custom money managers
You can easily implement your own position sizing logic by inheriting from the abstract `MoneyManager` class and overridding the `OnEntry()` method. It is up to the user to assign a quantity, otherwise a trade will be rejected by default. Below we create a simple money manager targeting a fixed quantity as determined by the `PositionSize` property.

> [!NOTE]
> The money manager will happily process fractional quantities. To enforce an integer quantity like many real-world exchanges, use code to round, truncate, or cast to `int`. Alternatively, set the `MoneyManager.Options.Rounding` property to either `ContractRounding.Round` or `ContractRounding.Truncate`. A trade quantity of zero means the entry will be ignored.

```csharp
class FixedQuantity : MoneyManager
{
    public double PositionSize { get; set; } = 1;

    protected override void OnEntry(Trade trade)
    {
        trade.Quantity = PositionSize;
    }
}
```
Money managers are typically assigned to a strategy before running a simulation. If we wanted to always trade 100 contracts (or shares if testing equities) we could assign our new custom `FixedQuantity` money manager to a strategy like so:

```csharp
var strat = new SimpleSma();
strat.PrimarySeries = data;
strat.MoneyManager = new FixedQuantity { PositionSize = 100 };
strat.RunSimulation();
```
## The DataStore
The backtester's architecture is somewhat unique in that strategy logic is separate from money management logic. Each can run independently of the other. This is enabled by a `DataStore`, an in-memory database of trades and pricing information that can be passed from strategies to money managers. This is particularly beneficial when working with strategies that are computationally expensive. You can calculate trades once and then work on position sizing and risk control separately and much more efficiently than if you had to recalculate the signals from scratch each time. DataStores can also be merged to allow portfolio testing of multiple systems.

After we have run a simulation we can call `Save()` on the strategy's `DataStore`. This saves all the trade, pricing, and risk information to disk in a compact, binary format. You can use whatever file name you prefer. By convention, an extension of ".bds" for "Backtester DataStore" is used in this example.
```csharp
var strat = new SimpleSma();
strat.RunSimulation(data);
strat.DataStore.Save(@"c:\temp\simpleSma.bds")
```
Later when working on money management, we can load this `DataStore` from disk to run a money management simulation and get the backtest report.
```csharp
var mm = new FixedQuantity { PositionSize = 5 };
mm.DataStore = Balsam.Data.DataStore.Load(@"c:\temp\simpleSma.bds");
mm.RunSimulation();
```
We can also generate a `DataStore` by calling `Strategy.Run()` and passing in a strategy type. `Strategy.Run` expects an enumerable of `BarSeries` for the data parameter. A `BarSeriesCollection` returned from a call to `LoadAll()` or `LoadSymbols()` from a `BarServer` is commonly used. `Strategy.Run` will then start parallel backtests running on multiple threads and merge all the trades into a single `Datastore` ordered by trade date. This allows for tests across a portfolio of instruments to be run very efficiently.
```c#
var mm = new FixedQuantity { PositionSize = 5 };
mm.DataStore = Strategy.Run(typeof(SimpleSma), null, data); //multi-threaded
mm.RunSimulation();
```