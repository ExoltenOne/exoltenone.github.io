---
layout: post
title:  "How to trace your SQL"
date:   2016-12-12 07:00:00
category: programming
tags: [fsharp,sql]
excerpt_separator: <!--more-->
links:
  - name: The article image
    address: https://www.flickr.com/photos/therefromhere/518053737/in/photolist-6wXNND-pBDr1d-MMajZ-7peMN-4rhfzH-6e4LKh-3ey2Ag-bZiq5-4rmkEA-2cCuux-bwvXc-bwxpk-7Qe9Wu-9vQyTG-oM1on2-6EkbGp-8xvkfj-cEDSN-Ay28kU-rvdFiM-darGjC-7Qe9YA-mQamMR-8ari4m-5E9Nov
---
{% capture xmlFile %}
<?xml version="1.0" encoding="utf-16"?>
<TraceData>
  <Header>
    <TraceProvider name="Microsoft SQL Server" MajorVersion="12" MinorVersion="0" BuildNumber="4459" />
    <ServerInformation name="SKYNET\SQLEXPRESS" />
    <ProfilerUI>
      <OrderedColumns>
        <ID>27</ID>
        ...
        <ID>2</ID>
      </OrderedColumns>
      <TracedEvents>
        <Event id="14">
          <EventColumn id="1" />
          ...
          <EventColumn id="14" />
        </Event>
        ...
      </TracedEvents>
    </ProfilerUI>
  </Header>
  <Events>
    <Event id="10" name="RPC:Completed">
      <Column id="2" name="BinaryData">00000000030000003A005B006400620</Column>
      <Column id="10" name="ApplicationName">.Net SqlClient Data Provider</Column>
      <Column id="6" name="NTUserName">Krzysztof</Column>
      <Column id="14" name="StartTime">2016-11-06T21:07:10.88+01:00</Column>
      <Column id="9" name="ClientProcessID">8780</Column>
      <Column id="11" name="LoginName">Skynet\Krzysztof</Column>
      <Column id="12" name="SPID">58</Column>
      <Column id="13" name="Duration">7327</Column>
      <Column id="15" name="EndTime">2016-11-06T21:07:10.887+01:00</Column>
      <Column id="16" name="Reads">481</Column>
      <Column id="17" name="Writes">0</Column>
      <Column id="18" name="CPU">15</Column>
      <Column id="1" name="TextData">exec [dbo].[uspGetBillOfMaterials] @StartProductID=328,@CheckDate='2010-10-05 00:00:00'</Column>
    </Event>
    ...
  </Events>
</TraceData>

{% endcapture %}

<a href="{{ site.baseurl }}{{page.url}}" class="image image-full">
  <img src="{{ site.baseurl }}/images/sql_trace.jpg" alt="The image with SQL code" />
</a>

<p>
  Have you ever thought how many store procedures your code is calling? Have you tracked them to find duplicate calls? Maybe there is a place which could be optimized but going through the code without information what procedure you are looking for is difficult or at least time consuming. There is probably a many ways to do that but here you'll find a way with F#.
</p>
<!--more-->

<h4>Record your SQL</h4>
<p>
  To do that we need to have a piece of code using store procedures. In your cases, it will be your own code but here to present end-to-end scenario I will use code written intentionally for this. You can find this code on <a href="https://github.com/ExoltenOne/TheCodeManual/tree/master/SqlProfilerTraceAnalyzer">GitHub</a>. Application AdventureWorksTraceGenerator uses <a href="https://msftdbprodsamples.codeplex.com/downloads/get/880661">AdventureWorks 2014</a> database to execute procedures: [HumanResources].[uspUpdateEmployeeHireInfo], [dbo].[uspGetManagerEmployees] and [dbo].[uspGetBillOfMaterials]. The first sproc is executed three times per loop iteration, the second - one and the third - two times. The loop is set to 1000 iterations and having this in mind we expect to have the first sproc executed 3000 times, the second 1000 times and the last 2000 times.
</p>

<p>
  The next step it to record database calls using SQL Server Profiler. First, you have to run SQL Management Studio, then click Tools and SQL Server Profiler. Afterwards, you'll have to connect to your database server and set trace properties. Trace properties for this test application you could leave default. And then if you run SQL code hitting this database you'll get all these calls recorded.
  <br/>
  <img class="center" src="{{ site.baseurl }}/images/sql_profiler.png" alt="The screen from SQL Profiler tool with recorded SQL calls" />
  <br/>
  Once your application finishes hitting the database, please stop recording be clicking 'Pause Selected Trace' and then save trace (File -> Save As -> Trace XML File...) so finally you will get a file like <a href="https://github.com/ExoltenOne/TheCodeManual/blob/master/SqlProfilerTraceAnalyzer/SqlProfilerTraceAnalyzer/trace.xml">this</a>. Below you'll find part of this file just to make you easier to catch the structure of it.
</p>

{% highlight xml %}
  {{ xmlFile }}
{% endhighlight %}

<h4>Extraxt data from recorded SQL</h4>
<p>
  To achieve our goal we'll have to analyse events and for that, we use F# and XML Type provider. So first, let's create F# Console Application  and we'll name it SqlProfilerTraceAnalyzer. Next, we'll have to install F# Data and we're going to use NuGet package for that, so type 'Install-Package FSharp.Data' in Package Manager Console. Once you installed F# Data, we'll add our trace file to the project and we set 'Copy to Output Directory' to 'Copy if newer'. And that's it, now we are ready to work.

  <br/>
  Working with F# Data is so easy as it only can be. To see the power of this library we'll need this line:
  </p>

  {% highlight fsharp %}
    type Trace = XmlProvider<"trace.xml">
  {% endhighlight %}

  <p>
    This one small line is doing a lot of work. It takes our file and infers types based on the provided XML file. So now, when we want to get events we could do that in this way:
  </p>

  {% highlight fsharp %}
    let trace = Trace.Load "trace.xml"
    printfn "%A" trace.Events
  {% endhighlight %}

  <p>
    First line load the file into the type created by inferring mechanism and right in the next line we have access to all events from trace.xml file. Having that, it's a time to do something useful with them and it will be:
  </p>
  <ul class="bullet-list">
    <li>extract store procedure name from TextData,</li>
    <li>count occurrences,</li>
    <li>calculate average and total time of execution.</li>
  </ul>
  <p>
    As you can see in the file shown above we need to extract store procedure name from column 'TextData' (id = 1) and duration from 'Duration' column (id = 13). Before we start writing code for this, we have to know that registered columns set depends on the type of registered event and it means that some of the columns may not appear for some events. We could simply filter out events with a name different than 'RPC:Completed' but we deal with this in another way (just to show some F# magic). We'll use for that options! This goal we achieve by a function which for given columns and column id return 'Some value' if column is within the set and if not then 'None'.
  </p>

  {% highlight fsharp %}
    let extractOptionValue columns id =
      let result = Array.tryFind (fun (y:Trace.Column) -> y.Id = id) columns
      Option.map (fun (x:Trace.Column) -> x.String) result
  {% endhighlight %}


  <p>
  We could imagine that code above transform our Event element to these two lines:
  </p>

  {% highlight xml %}
    7327
    exec [dbo].[uspGetBillOfMaterials] @StartProductID=328,@CheckDate='2010-10-05 00:00:00'
  {% endhighlight %}

 <p>
  Well, this is not exactly what we want to get and in fact these lines are type of Option&lt;string&gt; option but we want Option&lt;string&gt;. This happens because column not always have value and F# treat that as a Option&lt;string&gt; and if we put that into Option.map function we get Option&lt;string&gt; option. First we'll focus on the easier part, on the duration. Since we know that our input is Option&lt;string&gt; option and our output should be int we are almost in home. One thing is missing here and it is pattern matching. Joining all of this together we'll get that function:
</p>

  {% highlight fsharp %}
    let getDuration (durationOption: Option<string> option) =
        match durationOption with
        | Some x ->
                    match x with
                    | Some value -> int value
                    | None -> 0
        | None -> 0
  {% endhighlight %}

<p>
  Getting procedure name is slightly more complex. First of all we want to remove square braces, but this is easy and we use standard Replace method.
</p>

  {% highlight fsharp %}
    let removeSquareBraces (name:string) =
      name.Replace("[", String.Empty).Replace("]", String.Empty)
  {% endhighlight %}

<p>
  As we can see in the recorded SQL we have keyword exec and provided parameters and these parts we want also to get rid of. With this, regular expression helps and in case when we are able to extract name we return Some(value) and otherwise None.
</p>

  {% highlight fsharp %}
    let extractSprocName execution =
        let regex = new Regex(".*exec (?<sproc>\*)");
        let result = regex.Match execution
        let group = result.Groups.["sproc"]
        if group.Success then Some(removeSquareBraces group.Value) else None
  {% endhighlight %}

<p>
  As we have a code to extract name we'll use to map store porcedure data from Option&lt;string&gt; option to Option&lt;string&gt;. And the code will be similar to a pattern matching used for duration. So once again for existing value we get Some(value) and for missing one, None.
</p>

  {% highlight fsharp %}
    let getName (nameOption: Option<string> option) =
        match nameOption with
        | Some x ->
                    match x with
                    | Some value -> extractSprocName value
                    | None -> None
        | None -> None
  {% endhighlight %}

<h4>Get all working together</h4>

<p>
  We went long road but we are almost in home. Right now we have to simply put all of these functions together and we could do that in a really nice way. This time we use pipeline operator and speaking strictly pipe forward. This operator is really helpful and it lets us keep code clean and easy to read. The operator itself is defined as:
</p>

  {% highlight fsharp %}
    let (|>) x f = f x
  {% endhighlight %}

<p>
  So instead of simply writing f x we could write x |> f and it will be the same. This is really cool when a result of one function is an input of another. It is simple but powerful and allows us to join set on methods into a chain. Doing that we get:
</p>

  {% highlight fsharp %}
    let result = trace.Events
                  |> Array.map (fun x -> x.Columns)
                  |> Array.map (fun x -> (extractOptionValue x 1, extractOptionValue x 13))
                  |> Array.map (fun (nameColumn, durationColumn) -> (getName nameColumn, getDuration durationColumn))
                  |> Array.filter (fun (name,_) -> name.IsSome)
                  |> Array.map (fun (name,duration) -> (Option.get name, duration))
                  |> Array.toSeq
                  |> Seq.groupBy (fun (name, duration) -> name)
                  |> Seq.map (fun (name, entries) ->
                                                      let count = Seq.length entries
                                                      let sum = Seq.sumBy (fun (_,duration) -> duration) entries
                                                      let avg = Seq.averageBy (fun (_,duration) -> float duration) entries
                                                      (name, count, sum, avg))
                  |> Seq.sortBy (fun (name,_,_,_) -> name)
                  |> Seq.map(fun (name,count,sum,avg) -> sprintf "\n%-60s - Count: %-5d - Sum: %-6d - Avg: %-5f" name count sum avg)
                  |> Seq.reduce (sprintf "%s%s")
  {% endhighlight %}

<h4>Some last explanations and the result</h4>
<p>
  We have everything ready but just before I'll show a result I'll explain few thus far not mentioned parts. After extracting sproc name and duration we get tuple Option&lt;string&gt; * int and we are only interested with nodes containing sproc name. In this place Array.filter appears to by handy. As we secured our code and we know that right now we have only nodes with name we could apply Array.map once again and moved from Option&lt;string&gt; * int to string * int. In our next step we need to prepare our data to be able to count the executions, calculate a sum and an average and to do that we have to group data by name. Unfortunately Array can't do that for us so we are forced to find a different collection supporting that operation. Sequence can do that and to change Array into Sequence we need to invoke Array.toSeq. Once we get count, sum and average we return them with a name as a tuple and with this we are ready to present our effort - last three lines are used only to make the result more readable. So after all we have:
</p>

{% highlight html %}
  HumanResources.uspUpdateEmployeeHireInfo                     - Count: 2000  - Sum: 775252  - Avg: 387.626000
  dbo.uspGetBillOfMaterials                                    - Count: 3000  - Sum: 1457625 - Avg: 485.875000
  dbo.uspGetManagerEmployees                                   - Count: 1000  - Sum: 1222580 - Avg: 1222.580000
  sp_reset_connection                                          - Count: 5999  - Sum: 80208   - Avg: 13.370228

{% endhighlight %}

<p>
  I'll find this code useful in my work and I hope it will be the same for you. If something seems unclear please let me know and I'll try to explain it. If you have other solution to deal with that problem let me also know. Almost forget, all code you'll find on my <a href="https://github.com/ExoltenOne/TheCodeManual">GitHub</a>.
</p>
