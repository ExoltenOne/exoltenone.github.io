---
layout: post
title:  "How to show your SQL"
date:   2016-12-19 21:00:00
category: programming
tags: [fsharp,sql]
excerpt_separator: <!--more-->
links:
  - name: The article image
    address: http://getrefe.com/downloads/chart/
  - name: ARGU library site
    address: http://fsprojects.github.io/Argu/index.html
  - name: F# Charting library site
    address: https://fslab.org/FSharp.Charting/index.html
  - name: My GitHub account with the code
    address: https://github.com/ExoltenOne/TheCodeManual/tree/master/SqlProfilerTraceAnalyzer/HowToShowYourSql
---

<a href="{{ site.baseurl }}{{page.url}}" class="image image-full">
  <img src="{{ site.baseurl }}/images/sql_trace_show.jpg" alt="The image with chart on a tablet" />
</a>

<p>
  <a href="{{ site.url }}{% post_url 2016-12-12-how-to-trace-your-sql %}">Last time</a> we did a good job. We tracked store procedures hitting our server and we printed some details to the console. This was good, but this is not the end of what we can do. The first version contained hard-coded file name used to extract data and we only put the result on the console. Today we'll change this hard-coded file to a parameter and we add printing result to the file and to the chart.
</p>
<!--more-->
<h4>ARGU - command line arguments</h4>
<p>
  <a href="http://fsprojects.github.io/Argu/index.html">ARGU</a> is a very useful library which makes handling command line arguments as easy as it only can be. We only need to add discriminated union representing our arguments and add line parsing input arguments to our union values. However, first things first: let's add ARGU to our project 'SqlProfilerTraceAnalyzer'. As usually, we use for that NuGet package:
</p>

{% highlight html %}
  Install-Package Argu
{% endhighlight %}

<p> Now it's time for discriminated union. We need a name parameter and a value for it. So, the name will be TraceFile and for the value we'll define a path of string type. To pass this to our application we need to execute it with '--tracefile trace.xml'. </p>

{% highlight fsharp %}
  type Arguments =
    | TraceFile of path:string
{% endhighlight %}

<p>Afterwards, we are ready to extract the file name from passed arguments. This operation, as I mentioned is a piece of cake. All what we need are the three lines below:</p>

{% highlight fsharp %}
  let parser = ArgumentParser.Create<Arguments>()
  let arg = parser.ParseCommandLine argv
  let file = arg.GetResult (<@ TraceFile @>, defaultValue = "trace.xml")
{% endhighlight %}

<p>As you can see, we not only read argument value but we also set a default value in case when parameter won't be supplied. This is really cool! Now it's time to use this value, but before we do this we should refactor our code.</p>

<h4>Small refactor</h4>

<p>Originally code analysing file was located in Program.fs file. We need to change it slightly to be able to provide a file name as a parameter. So let's move whole code to TraceAnalyzer.fs file and put it inside the module with the same name.</p>

{% highlight fsharp %}
[<AutoOpen>]
module TraceAnalyzer

open System
open System.Text.RegularExpressions
open FSharp.Data

type Trace = XmlProvider<"trace.xml">

let analyze (traceFile:string) =
  let extractOptionValue columns id =
      let result = Array.tryFind (fun (y:Trace.Column) -> y.Id = id) columns
      Option.map (fun (x:Trace.Column) -> x.String) result

  let trace = Trace.Load traceFile

  let getDuration (durationOption: Option<string> option) =
      match durationOption with
      | Some x ->
                  match x with
                  | Some value -> int value
                  | None -> 0
      | None -> 0

  let removeSquareBraces (name:string) =
      name.Replace("[", String.Empty).Replace("]", String.Empty)

  let extractSprocName execution =
      let regex = new Regex(".*exec (?<sproc>\S*)");
      let result = regex.Match execution
      let group = result.Groups.["sproc"]
      if group.Success then Some(removeSquareBraces group.Value) else None

  let getName (nameOption: Option<string> option) =
      match nameOption with
      | Some x ->
                  match x with
                  | Some value -> extractSprocName value
                  | None -> None
      | None -> None

  let result = trace.Events
                  |> Array.map (fun x -> x.Columns)
                  |> Array.map (fun x -> (extractOptionValue x 1, extractOptionValue x 13))
                  |> Array.map (fun (nameColumn, durationColumn) -> (getName nameColumn, getDuration durationColumn))
                  |> Array.filter (fun (name, _) -> name.IsSome)
                  |> Array.map (fun (name,duration) -> (Option.get name, duration))
                  |> Array.toSeq
                  |> Seq.groupBy (fun (name, duration) -> name)
                  |> Seq.map (fun (name, entries) ->
                                                      let count = Seq.length entries
                                                      let sum = Seq.sumBy (fun (_,duration) -> duration) entries
                                                      let avg = Seq.averageBy (fun (_,duration) -> float duration) entries
                                                      (name, count, sum, avg))
                  |> Seq.sortBy (fun (name,_,_,_) -> name)

  result

{% endhighlight %}

<p>Please notice that module was marked with AutoOpen attribute what means that you don't need to open it strictly and you can simply refer directly to a function defined here. All existing functions are right now inside a analyze function and this prevents from using them outside of the scope. Last thing worth to mention is removing formatting from the result. I did it because we are going to need raw data for our visualizers. After this analyze function return seq<string*int*int*float>.</p>

<h4>Console visualizer</h4>
<p>
  Our first visualizer will be simple printing to the console just to check if everything works. We just need to put our previous formatting code into a function. So, let's call it printToConsole:
</p>

{% highlight fsharp %}
let printToConsole (result:seq<string*int*int*float>) =
    result
        |> Seq.map(fun (name,count,sum,avg) -> sprintf "\n%-60s - Count: %-5d - Sum: %-7d - Avg: %-5f" name count sum avg)
        |> Seq.reduce (sprintf "%s%s")
        |> printfn "%A"
{% endhighlight %}

<p>
  And now let's invoke it:
</p>

{% highlight fsharp %}
let parser = ArgumentParser.Create<Arguments>()
let arg = parser.ParseCommandLine argv

let arguments = arg.GetAllResults()

let file = arg.GetResult (<@ TraceFile @>, defaultValue = "trace.xml")
let result = analyze file

printToConsole result
{% endhighlight %}

<p>
  Yeah! Everything works like it was written originally and we also got the same result. That is a good news.
</p>

{% highlight html %}
  HumanResources.uspUpdateEmployeeHireInfo                     - Count: 2000  - Sum: 775252  - Avg: 387.626000
  dbo.uspGetBillOfMaterials                                    - Count: 3000  - Sum: 1457625 - Avg: 485.875000
  dbo.uspGetManagerEmployees                                   - Count: 1000  - Sum: 1222580 - Avg: 1222.580000
  sp_reset_connection                                          - Count: 5999  - Sum: 80208   - Avg: 13.370228

{% endhighlight %}

<p>
  Printing to console maybe it's not the thing which you want to have always enabled. And maybe you want to have a possibility to specify this via argument '--console'. To have this working in that way we need a new Arguments value.
</p>

{% highlight fsharp %}
type Arguments =
    | TraceFile of path:string
    | Console
{% endhighlight %}

<p>
As the last step, we have to check if console argument was provided. With ARGU there are no problems with that. We get all arguments, ignore trace file one and process console one.
</p>

{% highlight fsharp %}
let parser = ArgumentParser.Create<Arguments>()
let arg = parser.ParseCommandLine argv

let arguments = arg.GetAllResults()

let file = arg.GetResult (<@ TraceFile @>, defaultValue = "trace.xml")
let result = analyze file

let matchResult argument result =
    match argument with
    | Arguments.TraceFile _ -> ()
    | Arguments.Console -> printToConsole result

let rec matchResults arguments =
                match arguments with
                | head :: tail ->
                                    matchResult head result
                                    matchResults tail
                | [] -> ()

matchResults arguments
{% endhighlight %}

<p>
  If we run our application right now then we get the same result as above. And this means that it is the time for the next step which will be the next visualizer. This time we log our result to the file.
</p>

<h4>Console visualizer</h4>
<p>
 As previously we need a new Arguments value.
</p>

{% highlight fsharp %}
type Arguments =
    | TraceFile of path:string
    | Console
    | File
{% endhighlight %}

<p>
  We also need to define function writing to the file. I'll here use the same format as we have in console visualizer but it could be whatever you want.
</p>

{% highlight fsharp %}
let printToFile  (result:seq<string*int*int*float>) =
    let toFile str =
        System.IO.File.WriteAllText("trace.txt", str)

    result
        |> Seq.map(fun (name,count,sum,avg) -> sprintf "\n%-60s - Count: %-5d - Sum: %-7d - Avg: %-5f" name count sum avg)
        |> Seq.reduce (sprintf "%s%s")
        |> toFile
{% endhighlight %}

<p>
  The last thing which we need to do to enable saving to the file is a modification of matchResult function. We need to list File value in the pattern matching.
</p>

{% highlight fsharp %}
let matchResult argument result =
    match argument with
    | Arguments.TraceFile _ -> ()
    | Arguments.Console -> printToConsole result
    | Arguments.File -> printToFile result
{% endhighlight %}

<p>
  Keep in mind that you could pass more than one argument. So if you want to redirect output to the console and to the file you need to invoke the application with '--console' and '--file'.
</p>

<h4>Chart visualizer</h4>
<p>
And last but not least - chart visualizer. This time we generate charts saved in jpeg file based on our raw data. To do that we need <a href="https://fslab.org/FSharp.Charting/index.html">F# Charting</a> library and as previously we use NuGet to install it.
</p>

{% highlight html %}
  Install-Package FSharp.Charting
{% endhighlight %}

<p>
  Assuming that we want to have a choice of what type of chart we want to create (Pie or Bar) we need to define discriminated union used later as a value of an argument.
</p>

{% highlight html %}
type TypeChart =
  | Bar
  | Pie

  type Arguments =
      | TraceFile of path:string
      | Console
      | File
      | Chart of typeChart:TypeChart
{% endhighlight %}

<p>
ARGU deals easily with this and the last effort is to create function the printToChart which takes as parameters raw data and chart type.
</p>

{% highlight html %}
let printToChart (result:seq<string*int*int*float>) (typeChart:TypeChart) =

    let selectChart typeChart (data:seq<string*float>)  =
                        match typeChart with
                        | TypeChart.Bar -> Chart.Bar data
                        | TypeChart.Pie -> Chart.Pie data :> GenericChart

    let createChart typeChart name (data:seq<string*float>) =
        let chart = data |> selectChart typeChart
        chart.ShowChart() |> ignore
        chart.SaveChartAs(name, ChartImageFormat.Jpeg)

    let createChart' = createChart typeChart

    let countData = result |> Seq.map (fun (name,count,_,_) -> (name,(float)count))

    let sumData = result |> Seq.map (fun (name,_,sum,_) -> (name,(float)sum/1000.))

    let avgData = result |> Seq.map (fun (name,_,_,avg) -> (name,(float)avg))

    createChart' "count.jpeg" countData
    createChart' "sum.jpeg" sumData
    createChart' "avg.jpeg" avgData
{% endhighlight %}

<p> And when we extend the matchResult function to support Chart argument: </p>

{% highlight html %}
let matchResult argument result =
    match argument with
    | Arguments.TraceFile _ -> ()
    | Arguments.Console -> printToConsole result
    | Arguments.File -> printToFile result
    | Arguments.Chart typeChart -> SqlProfilerTraceVisualiser.Charting.printToChart result typeChart
{% endhighlight %}

<p>
  So finally if we run the application with the argument '--chart bar' we get:
</p>

<figure>
  <img src="{{ site.baseurl }}/images/chart_avg.jpeg" alt="The chart with time average of store procedures" />
  <figcaption>Fig 1. - The chart with time average of store procedures</figcaption>
</figure>

<figure>
  <img src="{{ site.baseurl }}/images/chart_count.jpeg" alt="The chart with count of store procedures" />
  <figcaption>Fig 2. - The chart with count of store procedures</figcaption>
</figure>

<figure>
  <img src="{{ site.baseurl }}/images/chart_sum.jpeg" alt="The chart with time sum of store procedures" />
  <figcaption>Fig 3. - The chart with time sum of store procedures</figcaption>
</figure>

<h4>Final refactor</h4>
<p>
To clean the code I decided to introduce SqlProfilerTraceVisualiser library where I moved all visualisers. It could be useful with our next step when we try to build something more dynamic.
</p>

<h4>Summary</h4>
<p>
As you can see we started with the application which was only able to analyze and print results to the console. Right now we could also save it to the file and generate charts and this is not the end.
</p>
