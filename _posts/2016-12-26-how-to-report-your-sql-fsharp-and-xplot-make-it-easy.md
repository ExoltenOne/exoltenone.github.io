---
layout: post
title:  "How to report your SQL - FSharp and XPlot make it easy"
date:   2016-12-26 07:30:00
category: programming
tags: [fsharp,sql]
excerpt_separator: <!--more-->
links:
  - name: The article image
    address: https://www.pexels.com/photo/black-samsung-tablet-computer-106344/
  - name: XPlot library site
    address: https://tahahachana.github.io/XPlot/
  - name: XPlot hello world by Leif Battermann (done for previous XPlot version)
    address: http://blog.leifbattermann.de/2015/11/07/xplot-hello-world/
  - name: Materialize CSS library site
    address: http://materializecss.com/
  - name: My GitHub account with the code
    address: https://github.com/ExoltenOne/TheCodeManual/tree/master/SqlProfilerTraceAnalyzer/HowToShowYourSql
---

<a href="{{ site.baseurl }}{{page.url}}" class="image image-full">
  <img src="{{ site.baseurl }}/images/report_sql.jpeg" alt="The image with chart on a tablet and a coffee" />
</a>

<p>
  It's a time to go further. <a href="{{ site.url }}{% post_url 2016-12-19-how-to-show-your-sql %}">Recently</a> we modified SqlProfilerTraceAnalyzer by adding different ways of showing recorded trace. Today we'll do something even better - we'll generate an HTML page with charts showing count, the total and the average time for our SQL. After that, we'll add a table containing information about our store procedures. Last time we used <a href="https://fslab.org/FSharp.Charting/">F# Charting</a>, but today we'll take another F# library, and it will be <a href="https://tahahachana.github.io/XPlot/">XPlot</a>.
  </p>
<!--more-->

<h4>Introduction to XPlot</h4>
<blockquote cite="http://stackoverflow.com">
  <p>XPlot is a cross-platform data visualization package for the F# programming language powered by popular JavaScript charting libraries <a href="https://developers.google.com/chart/">Google Charts</a> and <a href="https://plot.ly/">Plotly</a>. The library provides a complete mapping for the configuration options of the underlying libraries and so you get a nice F# interface that gives you access to the full power of Google Charts and Plotly. The XPlot library can be used interactively from F# Interactive, but charts can equally easy be embedded in F# applications and in HTML reports.</p>
</blockquote>
<p>
  <cite>from <a href="https://tahahachana.github.io/XPlot/">XPlot</a> page.</cite>
</p>

<h4>Preaparation of application for XPlot</h4>
<p>
  We'll extend our application by adding new visualizer using XPlot, but before that we need to have a way to tell our application to use this new visualizer by passing proper command line argument. As you could remember, we have to add first new value to our discriminated union. There won't be a surprise - we'll call it Xplot. And to use it we'll have to execute the application with '--xplot'.
</p>

{% highlight fsharp %}
type Arguments =
    | TraceFile of path:string
    | Console
    | File
    | Chart of typeChart:TypeChart
    | Xplot

with
    interface IArgParserTemplate with
        member s.Usage =
            match s with
            | TraceFile _ -> "specify a trace file"
            | Console _ -> "set output to console"
            | Chart _ -> "set output to chart"
            | File _ -> "set output to file"
            | Xplot _ -> "set output to HTML file with chart"
{% endhighlight %}

<p>
    Once we have ARGU configured, we need to modify the matching function.
</p>

{% highlight fsharp %}
let matchResult argument result =
  match argument with
  | Arguments.TraceFile _ -> ()
  | Arguments.Console -> printToConsole result
  | Arguments.File -> printToFile result
  | Arguments.Chart typeChart -> printToChart result typeChart
  | Arguments.Xplot -> printToXplot result
{% endhighlight %}

<p>
  As you may noticed I added printToXplot function which doesn't exist yet. And this is the right time to create it!
</p>

<h4>XPlot visualizer</h4>
<p>
  It is a time to leave SqlProfilerTraceAnalyzer project and move to SqlProfilerTraceVisualiser where we'll define printToXplot function. But first, we'll have to install XPlot library, and as always we use a NuGet package for that. XPlot is a part of a bigger library called FsLab, and we could use it as a whole but NuGet contains it split into smaller packages like GoogleCharts, Plotly, GoogleCharts.WPF etc. We'll use GoogleCharts so, please execute below command in Package Console Manager under SqlProfilerTraceVisualiser project.
</p>

{% highlight html %}
  Install-Package XPlot.GoogleCharts
{% endhighlight %}

<h5>HTML template</h5>
<p>
    The output will be an HTML file, and we need to define a structure of it. It will be a simple page where at the top you'll find a header, just below a table, and in next row - charts. To make formatting easier, we'll use <a href="http://materializecss.com/">Materialize CSS</a>. We'll add the file (let's name it XplotChart.html) to the SqlProfilerTraceVisualiser project as embedded resource.
</p>

{% highlight html %}
  <!DOCTYPE html>
  <html xmlns="http://www.w3.org/1999/xhtml">
  <head>
      <title>Trace report</title>

      <link href="http://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet">
      <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/materialize/0.97.8/css/materialize.min.css">
      <!--<link rel="stylesheet" href="ghpages-materialize.css">-->

      <style>
          body {
              background-color: #fcfcfc;
          }
          .header {
               color: #ee6e73;
               font-weight: 300;
          }
          .chart > div {
              width: auto !important;
              height: auto !important;
          }
      </style>
  </head>
  <body>
  <div class="container">
      <h3 class="header center">Trace report</h3>
      <div class="row">
          <div class="col l12 m12 chart">
              {{ table }}
          </div>
      </div>
      <div class="row">
          <div class="col l6 m12 chart">
              {{ chartCount }}
          </div>
          <div class="col l6 m12 chart">
              {{ chartSum }}
          </div>
          <div class="col l12 m12 chart">
              {{ chartAvg }}
          </div>
      </div>
  </div>

      <script type="text/javascript" src="https://code.jquery.com/jquery-2.1.1.min.js"></script>
      <script src="https://cdnjs.cloudflare.com/ajax/libs/materialize/0.97.8/js/materialize.min.js"></script>
  </body>
  </html>

{% endhighlight %}

<p>
  Afterwards, we need a function to read our template. This task is easy, and it'll take only a few lines:
</p>

{% highlight fsharp %}
let readEmbededFile fileName =
  let assembly = Assembly.GetExecutingAssembly()

  use stream = assembly.GetManifestResourceStream(fileName)
  use reader = new StreamReader(stream)
  let file = reader.ReadToEnd()
  file
{% endhighlight %}

<h5>Charts</h5>
<p>
  We cannot jump to charts without preparing our data. printToXplot function gets as input seq<string*int*int*float> and we need to transform our data into three sequences as it was with F# Charting visualizer. The first one for a number of executions, the second for total and the third average time. We'll achieve that by using Map function.
</p>

{% highlight fsharp %}
let countData = result |> Seq.map (fun (name,count,_,_) -> (name,(float)count))

let sumData = result |> Seq.map (fun (name,_,sum,_) -> (name,(float)sum/1000.))

let avgData = result |> Seq.map (fun (name,_,_,avg) -> (name,(float)avg))
{% endhighlight %}

<p>
  Building charts with XPlot is a simple task. When we have already data we need to define the type of chart by selecting interesting for us function (in our case it will be Chart.Bar), next provide options like size, orientation, title, background color and in the last step define labels. Be aware that to use this you'll need to open XPlot.GoogleCharts namespace.
</p>

{% highlight fsharp %}
let createChart title legend (data:seq<string*float>) =
    let bColor = new BackgroundColor();
    bColor.fill <- "#fcfcfc";

    let options =
        Options(
            title = title,
            orientation = "horizontal",
            width = 640,
            height = 480,
            colors = [| "#42A5F5" |],
            backgroundColor = bColor
        )

    [data]
        |> Chart.Bar
        |> Chart.WithOptions options
        |> Chart.WithLabels [legend]
{% endhighlight %}

<p>
  Once we have the function for creating a chart, we'll create them for our data.
</p>

{% highlight fsharp %}
let countChart = createChart "Store procedures" "Count" countData
let totalTimeChart = createChart "Store procedures - Total time" "Time [ms]" sumData
let avgTimeChart = createChart "Store procedures - Average time" "Time [us]" avgData
{% endhighlight %}

<p>
  As next we replace placeholders in the template with an HTML code generated by charts. For that purpose, we use the GetInlineHtml method.
</p>

{% highlight fsharp %}
  let replace (placeholder:string) (chart:GoogleChart) (format:string) =
    format.Replace(placeholder, chart.GetInlineHtml())

  let file = readEmbededFile "XplotChart.html"
  let newFile = file
                  |> replace "{{ chartCount }}" countChart
                  |> replace "{{ chartSum }}" totalTimeChart
                  |> replace "{{ chartAvg }}" avgTimeChart
{% endhighlight %}

<p>
  Before we write our new file, we need to change the template slightly. This is a consequence of using the GetInlineHtml method which assumes that page is correctly set up for using Google Charts. It could be avoided in some cases just by using the GetHtml method but this one generates HTML code for the whole page, and it is not a good solution for us. The modification is small, and it is reduced to script tags.
</p>

{% highlight html %}
<script type="text/javascript" src="https://www.google.com/jsapi"></script>
<script type="text/javascript">
  google.load("visualization", "1", { packages: ["corechart", "table"] });
</script>
{% endhighlight %}

<p>
  Now we can save our file.
</p>

{% highlight fsharp %}
let writeToFile fileName (content:string) =
    use stream = new FileStream(fileName, FileMode.Create)
    use writer = new StreamWriter(stream)
    writer.Write(content)

writeToFile "chartReport.html" newFile
{% endhighlight %}

<p>
  If we run right now our application with '--xplot' we'll get a page with three charts, but on the beginning, we'll decide to put also a table. And this a right time to do this.
</p>

<h5>Table</h5>

<p>
  The table doesn't differ from the chart and in fact this is kind of chart. So to create one we need a function Chart.Table and we also need to define options and labels.
</p>

{% highlight fsharp %}
let table = [countData; sumData; avgData]
              |> Chart.Table
              |> Chart.WithOptions(Options(showRowNumber = true, width = 1280))
              |> Chart.WithLabels [ "Store procedure name"; "Count"; "Total time [ms]"; "Average time [us]" ]
{% endhighlight %}

<p>
  With the table in our hands, we should replace the placeholder in the template.
</p>

{% highlight fsharp %}
let newFile = file
                |> replace "{{ table }}" table
                |> replace "{{ chartCount }}" countChart
                |> replace "{{ chartSum }}" totalTimeChart
                |> replace "{{ chartAvg }}" avgTimeChart
{% endhighlight %}

<p>
  We're almost ready to get our page. It won't work right now because a table chart needs a table package, so we'll have to add it to our script tags.
</p>

{% highlight html %}
<script type="text/javascript" src="https://www.google.com/jsapi"></script>
<script type="text/javascript">
    google.load("visualization", "1", { packages: ["corechart", "table"] });
</script>
{% endhighlight %}

<h4>The result</h4>
<p>
  After executing the application, we'll get a nicely formatted report. If you wish to see a not only the screenshot but whole page, you'll find it <a href="{{ site.url }}/examples/chartReport.html">here</a>.
</p>

<figure class="article-image">
  <img src="{{ site.baseurl }}/images/trace_report.png" alt="The screenshot with a table and charts " />
  <figcaption>Fig 1. - The screenshot with a table and charts generated by XPlot.</figcaption>
</figure>

<h4>Summary</h4>
<p>
  F# and XPlot gave great combination. We started with an XML file and after all, we changed this into attractively formatted HTML page. Of course feel free to adapt the code to your needs and if you have any doubts - comments are for you.
</p>
