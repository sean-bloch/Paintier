# Paintier
Linq to IPC over Kestrel Named Pipes to wrap Avanlonia/Skia canvas API

# Sample
For example, run this in LinqPad
```csharp
await using var p = Paintier.Connect();
var s = p.Shapes();

var steps = Enumerable.Range(0, 52).Select(i => i * 5).ToArray();   // 0,5,...,255
const double pitch = 6, radius = 2.5, gap = 8;                       // circle spacing, size, space between slices
double slice = 52 * pitch + gap;

// one tuple per circle: slice index (blue), red column, green row
var cells =
    from bi in Enumerable.Range(0, 52)                // blue slice index
    from gi in Enumerable.Range(0, 52)                // green -> row
    from ri in Enumerable.Range(0, 52)                // red   -> column
    select (bi, gi, ri);

// 13 slices per tile row, 4 tile rows -> 4 batches of 35,152 ops (limit is 100,000 per request)
var results = new List<ShapeResult>();
bool first = true;
foreach (var tileRow in cells.GroupBy(c => c.bi / 13))
{
    var b = s.Batch();
    if (first) { b.Clear(); first = false; }          // wipe the canvas once, in the first batch

    foreach (var (bi, gi, ri) in tileRow)
    {
        double x = (bi % 13) * slice + ri * pitch;
        double y = (bi / 13) * slice + gi * pitch;
        b.Circle(x, y, radius, PaintColor.Rgb(steps[ri], steps[gi], steps[bi]));
    }
    results.Add(await b.Flush());
}

results.Select(r => new { r.Ok, r.Count, r.Requests, r.Milliseconds, r.Error }).Dump();
```

Avalonia canvas in another host process draws this:
<img width="1323" height="458" alt="image" src="https://github.com/user-attachments/assets/bac8f4fe-6176-4d1c-8a6e-784833ad3377" />


