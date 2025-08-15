## Convert to different timeframes
### Compression
The `Balsam.Compression` namespace contains classes designed to compress or convert data to different timeframes and even different bar types. The `BarSeries` class itself exposes the following convenience methods which call common compression routines using their default values:
- `ToIntraday()` converts intraday data to higher timeframes. For example, to convert one minute bars to hourly, call `ToIntraday(60)`.
- `ToDaily()` converts intraday data to daily bars ending at a specified time of day.
- `ToWeekly()` converts daily data to weekly assuming a Friday week end.
- `ToMonthly()` converts data to monthly bars.

For more fine-grained control, call the `Compress()` method and pass in a compressor object. In the [EventStudy](../docs/EventStudies.md#a-more-realistic-example) example, we forced the weekly bars to use a Friday date even when Friday was a holiday by setting the `UseFixedDates` property of the `WeeklyCompression` object to true.
```c#
.Compress(new WeeklyCompression() { UseFixedDates = true });
```
Custom compression algorithms can be created by implementing `ICompressionProvider` or by inheriting from the abstract base class `CompressionProvider` and implementing the abstract `OnCompress` method.

### Conversion
Other classes within this namespace transform data in other ways that can be helpful. For example, `SessionCompression` can be used to extract the day session from 24 hour trading by passing in the start and end times for regular trading hours. A new BarSeries will be returned containing only bars within the specified time range which could then be compressed further to create daily bars if desired.
```csharp
BinaryBarServer server = new BinaryBarServer(@"c:\data\intraday");
var data = server.LoadSymbol("@ES");
//assuming regular trading hours from 8:30AM CT to 3:15PM CT.
var rth = data.Compress(new SessionCompression(new TimeSpan(8, 30, 0), new TimeSpan(15, 15, 0)));
var daily = rth.ToDaily(new TimeSpan(15, 15, 0));
```