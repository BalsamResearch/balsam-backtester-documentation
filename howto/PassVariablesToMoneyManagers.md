## Pass variables to a money manager
Sometimes it's helpful to be able to pass variables from a strategy to a money manager for later use in position sizing or other analytics. A `VariableDictionary` is used throughout the backtester to pass arbitrary data back and forth. `VariableDictionary` implements `IDictionary<string, object>` so it acts like a traditional dictionary with a string for the key. Objects can also be accessed by ordinal position in the order in which they were added.

There are two ways to pass variables from a `Strategy` to a `MoneyManager`. The first is via `StrategyVariables`. This property can be accessed from within a strategy to set arbitrary objects at the strategy level for later use in a money manager. The second and more common approach is to attach variables to a position when it is opened in a strategy. 

Let's once again revisit the `SimpleSma` example strategy to make these modifications. Notice in `OnStrategyStart()` we add "some very important info for later" to `StrategyVariables` using "MyMessage" as the key. Then in `OnPositionOpening()`, we call `position.Variables.Add()` to add the current 21 day ATR reading to a newly opened position. Note how we use `.Last` to refer to the most recent value.

```csharp
class SimpleSma : Strategy
{
    [Optimize(20, 80, 10)]
    public int ShortLength { get; set; } = 50;

    [Optimize(120, 240, 20)]
    public int LongLength { get; set; } = 200;

    [Optimize(3, 5, 0.5, includeNaN: true)]
    public double StopCoefficient { get; set; } = 3;

    protected override void OnStrategyStart()
    {
        Col1 = Sma(Close, ShortLength);
        Col2 = Sma(Close, LongLength);
        Col3 = Atr(21);

        StrategyVariables.Add("MyMessage", "Some very important info for later.");

        Plot(Col1, 0, Color.Blue);
        Plot(Col2, 0, Color.Red);
        Plot(PlotInstruction.PlotStops);
        Plot(Stochastic(10), 1, Color.Purple, paneSize: 33f);
    }

    protected override void OnBarClose()
    {
        if (CrossAbove(Col1, Col2).Last)
        {
            Buy();
        }
        else if (Col1.Last < Col2.Last)
        {
            Sell();
        }

        ExitOnStop();
    }

    protected override void OnPositionOpening(Position position)
    {
        base.OnPositionOpening(position);
        position.Stop = position.Entries.AvgGrossPrice - Col3.Last * StopCoefficient;
        position.Variables.Add("Atr", Col3.Last);
    }
}
```
Later within a money manager we can recover strategy level variables by calling `GetStrategyVariables` and specifying the strategy name. Variables attached to an opening position can be accessed via the `Variables` property of a `Trade` object in the `OnEntry()` method. In this example, we (arbitrarily) reject trade entries with an Atr reading greater than 5.

```csharp
class CustomMoneyManager : MoneyManager
{
    protected override void OnStrategyStart()
    {
        base.OnStrategyStart();

        var sv = GetStrategyVariables("SimpleSma");
        var myMessage = sv.GetValueAsString(0);
        Console.WriteLine(myMessage);
    }

    protected override void OnEntry(Trade trade)
    {
        if (trade.Variables.GetValueAsDouble("Atr") > 5)
        {
            trade.Cancel();
        }
        else
        {
            trade.Quantity = 10;
        }
    }
}
```
