# ![Balsam](https://raw.githubusercontent.com/BalsamResearch/.github/main/images/BalsamLogo_24x24.png) Balsam Research Backtester Documentation
Welcome to the Balsam Research backtester documentation site!

## What is it?

The Backtester is a suite of tools designed for rapid prototyping and testing of systematic trading strategies. It can run simple event studies to quantify market "edges" all the way to full blown multi-currency, multi-timeframe, multi-strategy portfolio simulations. Written in C#, it can be called from any [.NET language](https://dotnet.microsoft.com/en-us/languages). It can also serve as a general toolbox underlying many market related applications.

## Design philosophy

Many backtesters, particularly those written in R or Python, are often vector-based. For example, you might have an array containing a discrete signal (e.g. +1, 0, -1 for long, flat or short respectively) or a continuous variable (a z-score for example). This array can then be multiplied against market returns to calculate trading strategy performance. Vector based simulations run extremely fast.

Another common architecture is event driven. Here you write code that responds to events like the market opening, a trade being printed, or the completion of a bar. An event driven approach mimics the real world and is particularly useful for real-time trading; however, it can sometimes be tedious for research where you typically want to iterate as fast as possible without worrying about how you might actually deploy a model in production. An event driven backtester typically requires data to be added at the completion of each new interval (e.g. the close of each new bar) and then recalculating indicators. A vector based backtester on the other hand can calculate an indicator across the full dataset in a single call.

The backtester employs a hybrid approach. Indicators are calculated in advance at the start of the simulation. Then the backtester steps through the data by date, calling various events. This approach allows for simplicity, while at the same time allowing you to respond to events like you would in real-time (e.g. opening or closing a position). The Backtester is also designed to hide as much complexity as possible to speed strategy development while still allowing the user access to low-level classes and functionality for fine-grained control if and when needed.

So what does a strategy look like in practice?

### A simple moving average strategy

```c#
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

After a cursory glance at this code, even if you are not a C# expert, you can probably guess this is a long-only simple moving average strategy.  If the 50 period SMA of closing prices is above the 200 period SMA, we buy; otherwise we sell (sell does not mean sell short in this context.)  The `Strategy` class abstracts away a lot of the details so you can focus on trading logic without having to worry about indicator calculations, lining up dates, or keeping track of entries and exits.

## An important prerequisite

There are many trading software packages promising point and click system design and testing. These can be an easy way to get started, but inevitably you will get boxed in. While you don't have to be a professional programmer by any means to use the backtester, if the simple moving average system above made no sense whatsoever, this is probably *not* the tool for you. *A key prerequisite is the willingness to write and debug code.* Specifically, you will need a basic understanding of at least one .NET language, with C# being the obvious choice given the examples in the documentation were written in C#. Your journey will be far more rewarding and certainly less frustrating if you understand basic syntax and a few key concepts like classes, namespaces, methods and properties.

## Getting started
Check out the [docs](./docs/README.md) to get started.