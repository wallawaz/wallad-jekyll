---
layout: post
title: "Steam D3 Chart"
date: 2016-10-08
---


<div class="post-link">
  <a href="http://benwallad.com:8000/steam">tldr: View Chart</a>
</div>
<br />
I've been intruiged with [D3](https://d3js.org/) for a while now, toying with its powerful charting applications. Since I enjoy keeping up with gaming trends I thought it would be cool to use D3 to chart the most popular current games grouped by genre. By aggregating the data by genre, we can clearly see which genres are most popular. Since [steam](http://store.steampowered.com) dominates the PC gaming market, I decided that the most popular games currently being played on steam would be the basis of my project.

Steam itself does disclose top games by player count, but only for its top 100 games. I wanted to include more than 100 games in my calculation, so I looked elsewhere for steam player stats.
I ended up finding the website [steamdb](https://steamdb.info/), which has a wealth of information on all steam titles.
In particular, their */graph* endpoint contains the "Top games by current player count" which contains the current player count for over 7,000 games on steam. This is exactly what I was looking for. Additionally, steamdb lists each game's *APPID* which was a key datapoint for obtaining more data for each title.

Using [Requests](http://docs.python-requests.org/en/master/) and [BeautifulSoup4](https://www.crummy.com/software/BeautifulSoup/), I wrote a scraper that grabs *APPID* and *player_count* stats from this page and updates a SQLite database with this info.

Next, I needed to grab the genres that each of these *APPIDS* belong to in order to rank the most popular genres. The Steam API [endpoint](http://store.steampowered.com/) returns all relevant game metadata given an *APPID* parameter. Thus, I expanded my scraper to get the genres for each *APPID* from the Steam API. The Steam API is rate-limited, thus I  needed to delay my requests to the acceptible threshold in order to adhere to steam's acceptible terms of use.

---
<br />
With sufficient data, I started building the back-end application to fuel D3. I chose the [Flask](http://flask.pocoo.org/) micro framework to create a simple web application to query SQLite and serve the aggregated data to the front-end webpage. Since D3 is capable of making asynchronous JSON requests to grab input data, all responses returned from Flask were encoded in JSON.

The main query that fuels the D3 bar graph is a GROUP BY query that joins [steam_genres], [steam_apps_genres], and [steam_apps] to find the aggregate SUM of current_players by genre.

```sql

    select
        a.id as genre_id,
        a.genre,
        sum(c.peak_count) as player_count
    from
        steam_genres a join steam_apps_genres b
        on a.id  = b.genre_id
        join steam_apps c
        on b.steam_id = c.id
    group by
        a.genre
    having
        sum(peak_count) >= 1000
    order by
        a.genre

```
<sub>_I ended up limiting the genres displayed to only those with more than 1,000 players to reduce clutter_.</sub>

This query is executed on page load via the **/list** flask endpoint.
```javascript

    d3.json("/list", function(error, json) {
        for (var i in json.genres) {
            i = parseInt(i);
            totals.push(parseInt(json.genres[i][2]));
            initialData.push(json.genres[i]);
        }
        var results = loadGraph();

```
The data returned is **binded** to a D3 **selection** of "rect". Effectively creating a svg rectangle on the page for each genre returned:
### loadGraph()
```javascript

    var bars = svg.selectAll("rect")
        .data(initialData)
        .enter()
        .append("rect")
        .attr("x", function(d, i) {
            return xScale(d[1]);
        })
        .attr("y", function(d) {
            return h - margin["bottom"] - margin["top"] -  yScale(d[2]); // each data element is an array of [genre_id, genre_name, count]
        })
        .attr("width", xScale.rangeBand() - barPadding)
        .attr("height", function(d) {
            return yScale(d[2]);
        })
        .attr("fill", function(d, i) {
                return colors(i);
        })
        .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

```

Next, for more interactivity, I wanted to be able to see the top 10 games being played for a particular genre by clicking on a bar within the graph. This was accomplished via the D3 **selection.on** event listener:
```javascript

    bars.on("click", function(d) {

        // fade out any existing table
        fadeInOut("out", "#steamDetailsBox", 400);

        var genre_id = d[0];  //grab the genre_id of the data element.
        var data = [];
        var myColor = d3.select(this).style("fill");
        d3.json("/top_games/" + genre_id, function(error, json) {
            for (var i in json.top_games) {
                data.push(json.top_games[parseInt(i)]);
            }

```
#### The **/top_games/** endpoint is supplied with the genre_id via the D3 json request
```python

    @app.route("/top_games/<genre_id>")
    def top_games(genre_id):
        genre_id = int(genre_id)

        query = queries.genre_top_games.format(genre_id=genre_id)

        result = {
            "top_games": []
        }

        # get the genre_name to display on top of the table
        genre_name = []
        genre_name_query = "select genre from steam_genres where id = ?"
        with cursor_execute(steam.dbh,
                            genre_name_query,
                            params=[genre_id]) as cursor:
            for row in cursor:
                genre_name.append(row[0])

        genre_name = genre_name[0]
        result["genre_name"] = genre_name

        with cursor_execute(steam.dbh, query) as cursor:
            for rank, row in enumerate(cursor):
                obj = {
                    "rank": rank + 1,
                    "id": row[0],
                    "game": row[1],
                    "players": row[2],
                }
                result["top_games"].append(obj)

        return json.dumps(result)

```
With the top_games response, I then loaded these 10 elements within the JSON response into a table on the main page.

---
<br />
Finally, I thought it would be cool to be able to dynamically load a specific game's metadata on the page. However I did not want to have to store all steam metadata within the application. So, I created another endpoint to ping the steam api for a given steam_id as needed after page load.

```python

    @app.route("/details/<id>")
    def details(id):

        # handling api counter client side
        id = str(id)
        url = steam.steam_url + id
        response = requests.get(url)
        if response.status_code != 200:
            return json.dumps({
                "error": str(response)
            })
        response = response.json()
        response = response[id]

        try:
            data = response[u"data"]
        except KeyError:
            return json.dumps(response)

        platforms = "platforms" in data
        metacritic = "metacritic" in data
        header_image = "header_image" in data

        out = {}
        if platforms:
            out["platforms"] = data["platforms"]
        if metacritic:
            out["metacritic"] = data["metacritic"]
        if header_image:
            out["header_image"] = data["header_image"]

        return json.dumps(out)

```
<sub>_I chose to display the datapoints: **platforms** (windows/mac/linux), **metacritic** (score and url), and **header_image**._</sub>

With a successful response, elements within the main page are then loaded with metadata via D3 selections.
### steamDetails()
```javascript

    d3.json(url, function(error, json) {
        if (error) {
            console.log(error);
        }
        else {

            // grab exact jpg image
            var u = json["header_image"];
            var ending = u.match("\.jpg.*")[0];
            var header_url = u.replace(ending, ".jpg");

            // select the div to append insert into
            var detailsBox = d3.select("#steamDetailsBox");

            // remove any existing metadata we may have previously appended
            detailsBox.select("#headerImage").selectAll("img").remove();
            detailsBox.select("#platforms").selectAll("i").remove();
            detailsBox.select("#metacritic").selectAll("a").remove();

            // append header_url into div
            var headerImageContent = detailsBox.select("#headerImage")
                .append("img")
                .attr("src", header_url)
                .classed("image-responsive", true)
                .style("width", "90%")
                .style("border", "2px");

            // etc, etc.
```
<br />

Overall, I enjoyed working with D3 in this project. I like the concept of **binding** your data to a DOM selection, so you always know that a particular element is tied to a given dataset. Additionally, after data has been bound to the selection, futher attributes of the selection can be defined with the bound data through annonymous functions. For example, the height of each bar is defined by applying the yScale to the value of each data element in the selection:
```javascript

       .attr("height", function(d) {

          //d[0] == genre_id
          //d[1] == genre name
          return yScale(d[2]);
      })
```

All source code is posted on [github](https://github.com/wallawaz/steam_graph) if you would like to view more.

[Steam genre popularity](http://benwallad.com:8000/steam)

