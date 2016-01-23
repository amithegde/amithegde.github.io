---
layout: post
title: "Rendering Chart in LinqPad"
category: c#, LINQ, Chart, LinqPad
forreview: false
filename: "2016-1-23-rendering-chart-in-linqpad.md"
---

Often charts tell us what numbers can't tell us. Here is a quick and easy way to render charts in LinqPad:

{% highlight csharp %}

void Main()
{
	new LpChart(series: GetSeries(), chartType: SeriesChartType.Column, width: 1250).Dump();
	new LpChart(series: GetSeries(), chartType: SeriesChartType.Line, width: 1250).Dump();
	new LpChart(series: GetSeries(), chartType: SeriesChartType.Bar, width: 500).Dump();
}

private ChartSeries GetSeries()
{
	var items = Enumerable.Range(1, 25).Select(x => string.Format("item-{0}", x));

	var rand = new Random(100);
	var prices = items.Select(x => rand.NextDouble());

	return new ChartSeries { XSeries = items.ToList(), YSeries = prices.ToList() };
}

public class ChartSeries
{
	public IList<string> XSeries { get; set; }
	public IList<double> YSeries { get; set; }
}

public class LpChart
{
	public Chart Chart { get; private set; }
	public ChartArea ChartArea { get; private set; }
	public Series Series { get; private set; }
	public Bitmap Bitmap { get; private set; }

	public LpChart(ChartSeries series, SeriesChartType chartType = SeriesChartType.Column, int width = 1200)
	{
		this.Chart = new Chart();
		this.ChartArea = new ChartArea();
		this.Chart.ChartAreas.Add(this.ChartArea);

		this.Series = new Series();
		this.Series.ChartType = chartType;
		this.Series.Points.DataBindXY(series.XSeries.ToArray(), series.YSeries.ToArray());
		this.Chart.Series.Add(this.Series);

		this.Chart.Width = width;

		this.Bitmap = new Bitmap(width: this.Chart.Width, height: this.Chart.Height);
	}

	public void Dump()
	{
		this.Chart.DrawToBitmap(this.Bitmap, this.Chart.Bounds);
		this.Bitmap.Dump();
	}
}

{% endhighlight %}

Here is a [gist](https://gist.github.com/amithegde/895a2b468bb23745ce2a) with all references.