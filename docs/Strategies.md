# Strategies
With a few important building blocks out of the way, let's revisit the simple moving average system from the [Introduction](..\README.md#a-simple-moving-average-strategy) to understand a few key concepts.

```csharp
using Balsam;

class SimpleSma : Strategy
{
    protected override void OnStrategyStart()
    {
        Col1 = Sma(Close, 50);
        Col2 = Sma(Close, 200);
    }

    protected override void OnBarClose()
    {
        if (Col1.Last > Col2.Last)
        {
            Buy();
        }
        else
        {
            Sell();
        }
    }
}
```
### Strategy code discussion
* On the first line we import the namespace `Balsam`. This is the primary namespace containing key classes for creating and working with strategies. Other important namespaces include `Balsam.DataServers` and `Balsam.MoneyManagement`.

* Next we create a class called `SimpleSma` which inherits from Strategy, the abstract base class containing core backtesting functionality. All strategies must inherit from `Strategy` or a derivative of it.

* Within the new `SimpleSma` class, we override the virtual method `OnStrategyStart()`. This is where strategy initialization is typically performed. Here we calculate two simple moving averages on lines 7 and 8 and assign them to the built-in `TimeSeries` variables `Col1` and `Col2`. Col is short for column. Think of it just like a spreadsheet column. For convenience, there are 20 predefined column variables. You can also declare your own `TimeSeries` variables within the class if you prefer and it will work just the same.

* Finally, we override the `OnBarClose()` method. Like the name suggests, this is called after the close of every new bar and is typically where trading logic is implemented. Here we reference the moving average values that were assigned to the `Col1` and `Col2` variables in `OnStrategyStart()`. `Col1.Last` is an alias for `Col1[0]`. Using `Last` can make code a little easier to understand at a glance. But you might be wondering, if `Last` is equivalent to `[0]`, aren't we always referring to the same observation? If we were working with simple arrays the answer would be yes; however, `TimeSeries` have some additional functionality under the hood which brings up a key point you must understand when coding strategies.

## Concurrency

The backtester automatically maintains concurrency, or the notion that all data is aligned by date.  In `OnStrategyStart()`, `TimeSeries` act like simple arrays which can be indexed from zero to count - 1.  But once the simulation starts running, this behavior changes. Indexing into a `TimeSeries` within the `OnBarClose()` method using `.Last` or `[0]` works ***relative to the current date that is being tested.***

Here's a concrete example of this concept in action:

```csharp
class ConcurrencyExample : Strategy
{
    protected override void OnStrategyStart()
    {
        //create some dummy data
        Col1 = new TimeSeries()
            {
                { new DateTime(2024, 1, 1), 1 },
                { new DateTime(2024, 1, 2), 2 },
                { new DateTime(2024, 1, 3), 3 },
                { new DateTime(2024, 1, 4), 4 },
                { new DateTime(2025, 1, 5), 5 }
            };

        Console.WriteLine($"Indexing in OnStrategyStart() is {Col1.Indexing}.");
        Console.WriteLine($"Date      Value");
        for (int x = 0; x < Col1.Count; x++)
        {
            Console.WriteLine($"{Col1.Date[x]:d}    {Col1[x]}");
        }

        PrimarySeries = Col1.ToBarSeries();
    }

    protected override void OnBarClose()
    {
        if (IsFirstBar)
        {
            Console.WriteLine($"Indexing in OnBarClose() is {Col1.Indexing}.");
            Console.WriteLine("Date      Last [0]  Prev [1]");
        }

        Console.WriteLine($"{CurrentDate:d}    {Col1.Last}   {Col1[0]}     {Col1.Previous}   {Col1[1]}");
    }
}
```
When running the code, we'll see the following printed to the console:

```
Indexing in OnStrategyStart() is Absolute.
Date      Value
1/1/2024    1
1/2/2024    2
1/3/2024    3
1/4/2024    4
1/5/2025    5
Indexing in OnBarClose() is RelativeToCurrent.
Date      Last [0]  Prev [1]
1/2/2024    2   2     1   1
1/3/2024    3   3     2   2
1/4/2024    4   4     3   3
1/5/2025    5   5     4   4
```

We added five values to the pre-defined `TimeSeries` variable `Col1` and then looped through those values from 0 to count - 1. Note how the behavior of `Col1[0]` changes when called from within `OnBarClose`. Here `[0]` always refers to the latest value available relative to the current date being tested *not* the first value in the series. You can also see the equivalence of `Col1.Last` and `Col1[0]` (the current bar being tested) as well as `Col1.Previous` and `Col1[1]` (one bar prior to the current bar).

## Setback
The eagle-eyed among you may have noticed that we are missing the observation for 1/1/2024 in `OnBarClose()`. This is due to the `Setback` property which allows for an additional period of initialization before `OnBarClose` is called for the first time. By default, the backtester uses a `Setback` of 1 so that we can always refer to the previous bar without encountering an error. You can modify this behavior by changing the `Setback` property from within `OnStrategyStart()` as needed.

## MaxBarsBack and FirstValidDate

Now let's modify this code further by assigning a 2 period SMA to the `Col2` variable:

```csharp
class ConcurrencyExample2 : Strategy
{
    protected override void OnStrategyStart()
    {
        Col1 = new TimeSeries()
            {
                {new DateTime(2024, 1, 1), 1},
                {new DateTime(2024, 1, 2), 2},
                {new DateTime(2024, 1, 3), 3},
                {new DateTime(2024, 1, 4), 4},
                {new DateTime(2025, 1, 5), 5}
            };

        Col2 = Sma(Col1, 2);  //A simple moving average of length 2 isn't valid until at least 2 bars
        Setback = 1;  //this is the default setback value

        Console.WriteLine($"Indexing in OnStrategyStart() is {Col1.Indexing}.");
        Console.WriteLine($"Date      Value");
        for (int x = 0; x < Col1.Count; x++)
        {
            Console.WriteLine($"{Col1.Date[x]:d}    {Col1[x]}");
        }

        PrimarySeries = Col1.ToBarSeries();
    }

    protected override void OnBarClose()
    {
        if (IsFirstBar)
        {
            Console.WriteLine($"Indexing in OnBarClose() {Col1.Indexing}.");
            Console.WriteLine("Date      Last [0]  Prev [1]  Sma Sma[1]");
        }

        Console.WriteLine($"{CurrentDate:d}    {Col1.Last}   {Col1[0]}     {Col1.Previous}   {Col1[1]}   {Col2.Last} {Col2.Previous}");
    }
}
```

If we re-run we get the following output:

```text
Indexing in OnStrategyStart() is Absolute.
Date      Value
1/1/2024    1
1/2/2024    2
1/3/2024    3
1/4/2024    4
1/5/2025    5
Indexing in OnBarClose() is RelativeToCurrent.
Date      Last [0]  Prev [1]  Sma Sma[1]
1/3/2024    3   3     2   2   2.5 1.5
1/4/2024    4   4     3   3   3.5 2.5
1/5/2025    5   5     4   4   4.5 3.5
```
Note how in `OnBarClose` we have "lost" another observation after adding the two period moving average. This is because `Sma(Col1, 2)` requires two observations to initialize the calculation.  With the default `Setback` of 1 to allow for referencing the prior bar, we therefore start calling `OnBarClose` on day 3.

`TimeSeries` expose a property called `MaxBarsBack` which indicates the index at which valid data first becomes available. For a two period moving average it takes periods 0 and 1 to calculate a value, so `MaxBarsBack` would be 1. A related property is `FirstValidDate`, which returns the first date on which a `TimeSeries` is valid (in this case it would return 1/2/2024.)

The backtester looks across all indicators initialized in `OnStrategyStart()` to determine how much data is needed to "warmup" the system. This generally requires no intervention by the user unless you want to index further back in time than one period. For example, if you wanted to reference the value of the 200 period Sma 10 bars ago you would need to change the `Setback` to 10. However you wanted to also include a 50 period Sma and reference it back ten periods from the current bar, this would be fine since the 200 period Sma requires even more data to initialize than the 60 periods required to reference a 50 period Sma 10 bars ago. In most cases the default `Setback` of 1 will be fine.

## Mixing periodicities

Concurrency works automatically and across periodicities so you can mix and match daily and weekly data and the backtester will ensure you are always using the latest value that *could have been known at that point in time*. In the example below, we buy if the close of the most recent bar is below its 5 period SMA and the *weekly* SMA is greater than the prior week's reading. If today is Thursday, `Col2.Last`, which contains weekly data, would refer to the prior Friday (i.e. four trading days ago) and `Col2.Previous` would be *two* Fridays ago. If today is Friday, `Close.Last`, `Col1.Last`, and `Col2.Last` would all share Friday's date since the week just ended. You can observe concurrency in action by setting a breakpoint in `OnBarClose()` and examining the `CurrentDate` property of `Col1` (daily) and `Col2` (weekly) as the backtester calls each date in the `PrimarySeries` in succession.
```csharp
class MixedPeriodicities : Strategy
{
    protected override void OnStrategyStart()
    {
        var weekly = PrimarySeries.ToWeekly();

        Col1 = Sma(Close, 5);
        Col2 = Sma(weekly.Close, 52);
    }

    protected override void OnBarClose()
    {
        if (Close.Last < Col1.Last && Col2.Last > Col2.Previous)
        {
            Buy();
        }
    }
}

```
### Order sides
Equity traders already think in terms of four different order sides: `Buy`/`Sell`, and `SellShort`/`Cover`. Futures traders generally only think in terms of `Buy`/`Sell` since short selling is an integral part of the futures market (for every long there is naturally an offsetting short). The backtester operates more like an equity trader, and indeed, it can keep track of Reg-T margin requirements and debit/credit interest on cash and short balances for equity strategies. The backtester does require the use of four different order sides to keep track of long and short positions separately even when testing futures contracts.

Given the four different order sides, this means a call to `SellShort` will be ignored if you currently have an open long position. You would need to explictly close this long position first by calling `Sell` before issuing an opening `SellShort` order. Alternatively, you can set the `AutoReverseOnOppositeEntry` property to true and the backtester will do that work for you. Methods used to close a position like `Sell` or `Cover` can safely be called when there is an opposite position or no position at all. If the orders aren't appropriate for the current position, they will be ignored. This reduces the need for writing extra logic around order placement.

## Order placement
So far our examples have only used `Buy()` and `Sell()` which generate market orders for execution on the open of the next bar. The other key order methods include:

- `BuyLimit`
- `BuyStop`
- `BuyClose` 

> [!CAUTION]
> `BuyClose` generates a market order for execution on the current bar's close for backtesting purposes only. By making calculations and trading on a price that is only known *after* the market has actually closed, you could introduce a subtle look-ahead bias. The backtester has safeguards built-in to prevent look-ahead bias but this is one area where the benefits for testing outweigh the risks as long as you are cognizant of the tradeoff and risks.

Of course there are corresponding order methods for `Sell`, `SellShort`, and `Cover` as well. All these methods are convenience methods that create and submit the appropriate order object under the hood. If for some reason you need more control you can always submit orders directly. For example, below we check if there is a long position and then submit a sellstop order directly to the `SubmitOrder` method.
```csharp
if (MarketPosition == PositionSide.Long)
{
    SubmitOrder(new StopOrder(OrderSide.Sell, CurrentPosition.Quantity, Symbol, CurrentPosition.Stop) { SignalName = "Protective stop" });
}
```
But there is rarely a need to do this. After all, abstracting away the underlying complexity to speed the testing process is a major part of the backtester's appeal.

### Time in force
All orders have a default `TimeInForce` of `Day`. In the context of the backtester, this means orders are good for the next bar, whether that bar is daily, weekly or even one minute. For efficiency in backtesting, it's generally advisable to use the built-in convenience methods which use the default `TimeInForce`. If you prefer, you can create and manage orders with other `TimeInForce` settings but it would then be incumbent upon you to manage these orders. For example, if you wanted to manually submit stop orders with a `TimeInForce` of `GTC`, to cancel them you would need to call `CancelOrder` like so:


```C#
var pending = GetPendingOrders().OfType<StopOrder>().Where(x => x.Side == OrderSide.Sell);

foreach (var sellStop in pending)
{
    CancelOrder(sellStop);
}
```