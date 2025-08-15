## Sync data
Sometimes it's helpful to align dates across series, particularly when working outside of the backtester which automatically handles different periodicities and missing dates with its concurrency management features. To align dates with another series, use the `.Sync()` method. A simple example with some dummy data will hopefully make this clear. Notice how the dummy `TimeSeries` in the code below each have three observations but the dates only overlap on two of them.
```csharp
var a = new TimeSeries()
{
    { new DateTime(2024, 1, 1), 1},
    { new DateTime(2024, 1, 2), 2},
    { new DateTime(2024, 1, 4), 4 }
};
a.Name = "TimeSeries A";

var b = new TimeSeries()
{
    { new DateTime(2024, 1, 1), 10},
    { new DateTime(2024, 1, 3), 30},
    { new DateTime(2024, 1, 4), 40 }
};
b.Name = "TimeSeries B";

a.Print();
b.Print();

var synced = a.Sync(b);
synced.Name = "A synced to B";
synced.Print();

synced = b.Sync(a);
synced.Name = "B synced to A";
synced.Print();
 ```
 In the output below, notice how syncing uses dates from the series we passed into the `Sync` method while preserving the values of the original series on which `Sync` was called. If a date is missing from the original series, the previous value is carried forward by default.
 ```
TimeSeries A
Count: 3 MaxBarsBack: 0
1/1/2024 to 1/4/2024
   0 1/1/2024 1
   1 1/2/2024 2
                <- note A is missing the 1/3/2024 observation
   2 1/4/2024 4
_______________________
TimeSeries B
Count: 3 MaxBarsBack: 0
1/1/2024 to 1/4/2024
   0 1/1/2024 10
                 <- note B is missing the 1/2/2024 observation
   1 1/3/2024 30
   2 1/4/2024 40
_______________________
A synced to B
Count: 3 MaxBarsBack: 0
1/1/2024 to 1/4/2024
   0 1/1/2024 1
   1 1/3/2024 2 <- A's prior value from 1/2/2024 is carried forward to 1/3/2024
   2 1/4/2024 4
_______________________
B synced to A
Count: 3 MaxBarsBack: 0
1/1/2024 to 1/4/2024
   0 1/1/2024 10
   1 1/2/2024 10 <- B's prior value from 1/1/2024 is carried forward to 1/2/2024
   2 1/4/2024 40
 ```
 
If we call `Sync` passing in a `SyncOption` value, we can change the default behavior to fill missing dates with NaN or the default value of the underlying data type.
```csharp
synced = a.Sync(b, SyncOption.NaN);
synced.Name = "A synced to B with SyncOption.NaN";
synced.Print();

synced = b.Sync(a, SyncOption.ZeroOrDefault);
synced.Name = "B synced to A with SyncOption.ZeroOrDefault";
synced.Print();
```
```
A synced to B with SyncOption.NaN
Count: 3 MaxBarsBack: 0
1/1/2024 to 1/4/2024
   0 1/1/2024 1
   1 1/3/2024 NaN <- missing date filled with double.NaN
   2 1/4/2024 4
___________________________________________
B synced to A with SyncOption.ZeroOrDefault
Count: 3 MaxBarsBack: 0
1/1/2024 to 1/4/2024
   0 1/1/2024 10
   1 1/2/2024 0  <- missing date filled with zero, the default for numeric types
   2 1/4/2024 40
```