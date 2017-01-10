---
layout: post
title:  "IT emigration to the USA - FSharp and D3.js shows you where people goes"
date:   2017-01-10 03:00:00
category: programming
tags: [fsharp,d3js]
excerpt_separator: <!--more-->
links:
  - name: The article image
    address: https://mrprintables.com/printable-map-of-the-united-states.html
  - name: United States Department of Labor
    address: https://www.foreignlaborcert.doleta.gov/performancedata.cfm#dis
  - name: Free World Cities Database
    address: https://www.maxmind.com/en/free-world-cities-database
  - name: D3.js library page
    address: https://d3js.org/
  - name: My GitHub account with the code
    address: https://github.com/ExoltenOne/TheCodeManual/tree/master/WorkInUS/WorkInUS
---

<a href="{{ site.baseurl }}{{page.url}}" class="image image-full">
  <img src="{{ site.baseurl }}/images/usa.png" alt="The USA map." />
</a>

<p>
  I heard many times about people going to the USA to live and work there. For some people, it stills revealed as the best plan for life and the best country to live. However, it is not an easy thing. A company must to request for you and organise a visa, mostly H-1B and pay for it. This process, by itself nature takes some time and requires you and your future company to pay an attention. It is intriguing and this was an impulse for my small research. I was wondering where people mostly goes and today we create a map showing us that. As usually, FSharp helps us with that, but this time we orchestrate it with D3.js library.
</p>

<!--more-->
<h4>Data</h4>
<p>
  First we need to get data and we find them on United States Department of Labor <a href="https://www.foreignlaborcert.doleta.gov/performancedata.cfm#dis">page</a>. You will find on this page information for all visa programs and if you are considering a relocation to the USA, it will be a good start. Going back to our application, we need information about H-1B program and at the time of writing this post the most recent data are in <a href="https://www.foreignlaborcert.doleta.gov/docs/py2015q4/H-1B_Disclosure_Data_FY15_Q4.xlsx">H-1B_Disclosure_Data_FY15_Q4.xlsx</a> file. Watch out, the file is big, it weight 143 MB. The file contains a lot of columns, but we need only a few - these mentioned below.
</p>

| Column        | Meaning       |
| ------------- |:-------------:|
| CASE_STATUS     | A request status |
| EMPLOYMENT_START_DATE     | Beginning date of employment      |
| SOC_CODE | Occupational code    |
| WORKSITE_STATE | State information of the foreign worker's intended area of employment   |
| WORKSITE_CITY | City information of the foreign worker's intended area of employment     |
| WAGE_RATE_OF_PAY | Rate of pay offered    |
| WAGE_UNIT_OF_PAY | Unit of pay.  Valid values include “Hour", "Week", "Bi-Weekly", "Month", or "Year"       |

<h4>Preparing data</h4>
<p>
  Before we jump into coding let's define requirements for data:
</p>
<ul>
  <li>
    only accepted request must be taken into consideration - CASE_STATUS = CERTIFIED,
  </li>
  <li>
    we care for most recent request = EMPLOYMENT_START_DATE in 2015,
  </li>
  <li>
    only computer science and math - SOC_CODE starts with 15,
  </li>
  <li>
    we ignore dependent territories like Puerto Rico, Guam etc. - WORKSITE_STATE different than PR, GU , VI , MP.
  </li>
</ul>

<p>
  We just pointed out how we plan to deal with data, but we haven't described what exactly want we show on the map. And this is key information because we need that to know what we want to extract from the excel file. So, we gonna create the USA map with marked cities as a circle which radius depends on the number of offers, and when we mouse enter a circle, we show a tooltip with a city name, a number of offers and average salary.
</p>

<p>
  Now, we are ready to write code. As first, we have to install ExcellProvider and as always we'll do that using NuGet package by typing 'Install-Package ExcelProvider' in Package Manager Console. Next, we need to create a type representing the excel file, get rows and filter them out accordingly to the specification above.
</p>

{% highlight fsharp %}
  open FSharp.ExcelProvider

  type Visa = ExcelFile<"H-1B_Disclosure_Data_FY15_Q4.xlsx">

  type VisaGroupRequest = { CityName:String; Region:String; Salary:decimal; OffersNumber:int  }

  [<EntryPoint>]
  let main argv =

      let visa = new Visa()

      let starts date limitDate =
          match DateTime.TryParse(date) with
          | (true, parsedDate) -> parsedDate > limitDate
          | _ -> false

      let startsIn2015 date = starts date (new DateTime(2015,1,1))

      let isComputerScieneOrMath (code:String) =
          code.StartsWith "15"

      let excludedRegions = [| "PR";"GU";"VI";"MP" |]

      let visaCities = visa.Data
                      |> Seq.filter (fun item -> item.CASE_STATUS = "CERTIFIED")
                      |> Seq.filter (fun item -> startsIn2015 item.EMPLOYMENT_START_DATE)
                      |> Seq.filter (fun item -> isComputerScieneOrMath item.SOC_CODE)
                      |> Seq.filter (fun item -> not (Array.contains item.WORKSITE_STATE excludedRegions))

      printfn "%A" (Seq.length visaCities)

      0 // return an integer exit code
{% endhighlight %}

<p>
  Our next step is the extraction. We need to leave only city name, state code, a number of offers and average salary. Only the last one is not a simple mapping. We could have different salary rate like year, hour or week. In the application, we limit it only to year and hour. However, we still have to map salary from one rate to another. As base rate I prefer year, and to get a yearly salary from hour rate we need to multiple it by  <a  href="https://www.opm.gov/policy-data-oversight/pay-leave/pay-administration/fact-sheets/computing-hourly-rates-of-pay-using-the-2087-hour-divisor/">2087</a>.
</p>

{% highlight fsharp %}
  let getNumber text =
      let regex = new Regex("^\d*") // ^\d*(\.\d*)?
      let result = regex.Match(text)
      match result.Success with
      | true -> Some((decimal)result.Value)
      | false -> None

  let calculateSalary (paymentRate:String) paymentUnit =
      match paymentUnit with
      | "Year" -> getNumber paymentRate
      | "Hour" -> match getNumber paymentRate with
                  | Some value -> Some(value * 2087m)
                  | None -> None
      | _ -> None

  let visaCities = visa.Data
            |> Seq.filter (fun item -> item.CASE_STATUS = "CERTIFIED")
            |> Seq.filter (fun item -> startsIn2015 item.EMPLOYMENT_START_DATE)
            |> Seq.filter (fun item -> isComputerScieneOrMath item.SOC_CODE)
            |> Seq.filter (fun item -> not (Array.contains item.WORKSITE_STATE excludedRegions))
            |> Seq.map (fun item -> ( item.WORKSITE_CITY, item.WORKSITE_STATE, calculateSalary item.WAGE_RATE_OF_PAY item.WAGE_UNIT_OF_PAY))
            |> Seq.filter (fun (_,_,salary) -> salary.IsSome)
            |> Seq.map (fun (name, state, salary) -> (name.ToLower(), state, salary.Value))
            |> Seq.groupBy (fun (name,state,_) -> (name,state))
            |> Seq.map (fun (cityAndState, groups) -> {
                                                           CityName = fst cityAndState;
                                                           Region = snd cityAndState;
                                                           Salary = Seq.averageBy (fun (_,_,salary) -> salary) groups;
                                                           OffersNumber = Seq.length groups })
{% endhighlight %}

<h5>Cities localization</h5>
<p>
  To display cities in a proper place on the map we need to know city's latitude and longitude, but with <a href="https://www.maxmind.com/en/free-world-cities-database">Free World Cities Database project</a> it's easy. To load this file, we need the CsvTypeProvider. In order to get it FSharp.Data library must be installed ('Install-Package FSharp.Data'). We are only interested in the US cities and make running faster I prepared a file containing only those cities.
</p>

{% highlight fsharp %}

  type City = { Name:String; Region:String; Latitude:decimal; Longitude:decimal }

  let cities = Cities.Load("worldcitiespopus.txt").Rows
              |> Seq.map (fun item -> ({ City.Name = item.City; Region = item.Region; Latitude = item.Latitude; Longitude = item.Longitude}))
{% endhighlight %}

<p>
  As you can see both types contains a city name and region, and we used these values to join them both.
</p>

{% highlight fsharp %}
  let findCity (visaRequest:VisaGroupRequest) =
        let result = cities |> Seq.tryFind (fun item -> item.Name = visaRequest.CityName && item.Region = visaRequest.Region)
        match result with
            | Some value -> Some(value)
            | _ -> None

    let visaCitiesLocalization = visaCities
                                    |> Seq.map (fun city -> (city, city |> findCity))
                                    |> Seq.filter (fun city -> (snd city).IsSome)
                                    |> Seq.map (fun (request,city) -> {
                                                                           Name = CultureInfo.CurrentCulture.TextInfo.ToTitleCase request.CityName;
                                                                           Region = request.Region;
                                                                           Salary = request.Salary;
                                                                           OffersNumber = request.OffersNumber;
                                                                           Longitude = city.Value.Longitude;
                                                                           Latitude = city.Value.Latitude})
{% endhighlight %}

<p>
  And, at the end, we'll have to write this collection to JSON file. We use for that the most popular JSON library in .NET - Newtonsoft. To add it to our project, we need to type 'Install-Package Newtonsoft.Json'.
</p>

{% highlight fsharp %}
  let writeToFile fileName (content:string) =
      use stream = new FileStream(fileName, FileMode.Create)
      use writer = new StreamWriter(stream)
      writer.Write(content)

  JsonConvert.SerializeObject visaCitiesLocalization |> writeToFile "us-cities.json"
{% endhighlight %}

<h4>Displaying data</h4>

<p>
   In the listing below I put the code used to generate the map.
</p>

{% highlight html %}
  <html>
  <head>
      <script src="https://cdnjs.cloudflare.com/ajax/libs/d3/3.5.8/d3.min.js" charset="utf-8"></script>
      <style>
          /*Removed to keep listing smaller*/
      </style>
  </head>
  <body>

  <h1 class="header">H-1B Visa accepted requests in Computer Science and Math area (2015)</h1>
  <!--To test it locally run python -m SimpleHTTPServer-->
  <script>

      var svgMapId = "svg-map";

      //Width and height
      var w = 1280;
      var h = 650;

      //Define map projection
      var projection = d3.geo.albersUsa()
          .translate([w / 2, h / 2])
          .scale([w]);

      var drawCities = function(svg) {
          d3.json("us-cities.json", function(data) {

              svg.selectAll("circle")
                  .data(data)
                  .enter()
                  .append("circle")
                  .attr("cx", function(d) {
                      return projection([d.Longitude, d.Latitude])[0];
                  })
                  .attr("cy", function(d) {
                      return projection([d.Longitude, d.Latitude])[1];
                  })
                  .attr("r", function(d) {
                      return Math.sqrt(d.OffersNumber) / 3;
                  })
                  .attr("class", "bubble")
                  .append("title")
                  .text(function(d) {
                      return d.Name + " Number of offers: " + d.OffersNumber + " Avg salary: " + d.Salary.toFixed(2);
                  });
          });
      };

      var drawMap = function() {

          //Define path generator
          var path = d3.geo.path()
              .projection(projection);

          //Create SVG element
          var svg = d3.select("body")
              .append("svg")
              .attr("width", w)
              .attr("height", h)
              .attr("class", "center")
              .attr("id", svgMapId);

          //Load in GeoJSON data
          d3.json("us-states.json", function(json) {

              //Bind data and create one path per GeoJSON feature
              svg.selectAll("path")
                  .data(json.features)
                  .enter()
                  .append("path")
                  .attr("class", "border border--state")
                  .attr("d", path)
                  .style("fill", "#ccc");

              drawCities(svg);

          });
      };

      drawMap();

  </script>
  </body>
  </html>
{% endhighlight %}

<p>
  And here you have the result and <a href="{{ site.url }}/examples/visaCitiesMap.html">the online version</a>:
</p>

<figure class="article-image">
  <img src="{{ site.baseurl }}/images/usa_visa_cities.png" alt="The USA map with foreign workers employment cities" />
  <figcaption>Fig 1. - The USA map with foreign workers employment cities.</figcaption>
</figure>
