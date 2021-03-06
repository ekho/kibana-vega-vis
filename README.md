# Vega visualization plugin for Kibana

> Build [Vega](https://vega.github.io/vega/examples/) and [VegaLite](https://vega.github.io/vega-lite/examples/) data visualizations into Kibana, either standalone, or on top of a map.

# Watch a short introduction video
[![Leaflet Vega layer demo](https://i.ytimg.com/vi_webp/lQGCipY3th8/maxresdefault.webp)](https://www.youtube.com/watch?v=lQGCipY3th8)


# Quick Demo

* Use a direct download link from the [releases page](https://github.com/nyurik/kibana-vega-vis/releases/latest). _For more info, see [Kibana plugin installation instructions](https://www.elastic.co/guide/en/kibana/current/_installing_plugins.html)_
```
bin/kibana-plugin install \
   https://github.com/nyurik/kibana-vega-vis/releases/download/<latest_version>.zip
```
* In Kibana, choose Visualize, and add Vega visualization.
* You should immediatelly see a default graph
* Try changing `mark` from `line` to `point`, `area`, `bar`, `circle`, `square`, ... (see [docs](https://vega.github.io/vega-lite/docs/mark.html#mark-def))
* Try other [Vega](https://vega.github.io/vega/examples/) or [VegaLite](https://vega.github.io/vega-lite/examples/) visualizations. You may need to make URLs absolute, e.g. replace `"url": "data/world-110m.json"` with `"url": "https://vega.github.io/editor/data/world-110m.json"`. (see [notes below](#Using Vega and VegaLite examples))
* Using [makelogs util](https://www.npmjs.com/package/makelogs), generate some logstash data and try [logstash examples](public/examples/logstash), including the [map example](public/examples/logstash/logstash-geosrc-map.json)

# Vega with a map
Kibana's default map can be used as a base of the Vega graph. To enable, the graph must specify `type=map` in the host configuration:

```yaml
{
  "_hostConfig": {
    "type": "map",

    // Initial map position
    "latitude": 40.7,      // default 0
    "longitude": -74,      // default 0
    "zoom": 7,             // default 2
    "mapStyle": "default", // defaults to "default", but can also be false to disable base layer
    "minZoom": 5,          // default 0
    "maxZoom": 13,         // defaults to the maximum for the given style, or 25 when base is disabled
    "zoomControl": false,  // defaults to true, shows +/- buttons to zoom in/out

    // When false, repaints on each move frame. Makes the graph slower when moving the map
    "delayRepaint": true, // default true
  },
  /* the rest of Vega JSON */
}
```

This plugin will automatically inject a projection called `"projection"`. Use it to calculate positioning of all geo-aware marks. Additionally, you may use `latitude`, `longitude`, and `zoom` signals. These signals can be used in the graph, or can be updated to modify the positioning of the map.

# Querying ElasticSearch
By default, Vega's [data](https://vega.github.io/vega/docs/data/) element can use embedded and external data with a `"url"` parameter. Kibana plugin adds support for the direct ElasticSearch queries by overloading the "url"` value.

Here is an example of an ES query that gets data from `logstash-*` index, filtering by 

```yaml
{
  "data": [
    {
      "name": "myEsDataSource",
      "url": {
        // Index name
        "index": "logstash-*",

        // Use current dashboard search string and time range filter with the "@timestamp" field.
        // Set value to true to ignore the time filter
        "%context_query%": "@timestamp",

        // TIP: request can be copied from the debug view of another visualizer
        // You can try this query in Kibana Dev tools (hardcode or remove the `%...%` values first)
        "body": {
          // When aggegating, do not return individual documents that match the query
          "size": 0,

          // Data aggegation...
          "aggs": {
            // Name of the aggegation - your Vega graph will use it to parse the results
            "hist": {
              "date_histogram": {
                "field": "@timestamp",
                "interval": "2h",
                "time_zone": "America/New_York",
                "min_doc_count": 1
              }
            }
          }
        }
      },

      // This is a useful trick to access just the list of aggregation results named "hist"
      //
      "format": {
        "type": "json",
        "property": "aggregations.hist.buckets"
      },

    }
  ],
  ...
}
```

As a result, "myEsDataSource" will be a list of objects. Note that `"key"` is a unix timestamp, and can be used without conversions by the Vega date expressions.
```yaml
[
    {
      "key_as_string": "2017-06-13T04:00:00.000-04:00",
      "key": 1497340800000,
      "doc_count": 6
    },
    {
      "key_as_string": "2017-06-13T06:00:00.000-04:00",
      "key": 1497348000000,
      "doc_count": 14
    },
    ...
]
```

Query may be specified with individual range and dashboard context as well. This query is equivalent to `"%context_query%": "@timestamp"`, except that the timerange is shifted back by 10 minutes:

```yaml
{
  "data": [
    {
      "name": "myEsDataSource",
      "url": {
        // Index name
        "index": "logstash-*",

        "body": {
          "query": {
            "bool": {
              "must": [
                // This string will be replaced with the auto-generated "MUST" clause
                "%dashboard_context-must_clause%",

                "range": {
                  // apply timefilter (upper right corner) to the @timestamp variable
                  "@timestamp": {
                    // "%timefilter%" will be replaced with the current
                    // values of the time filter (from the upper right corner)
                    "%timefilter%": true
    
                    // Only work with %timefilter%
                    // Shift the current timefilter by 10 units back
                    "shift": 10,
                    
                    // supports week, day (default), hour, minute, second.
                    "unit": "minute"
                  }
                }
              ],
              "must_not": [
                // This string will be replaced with the auto-generated "MUST-NOT" clause
                "%dashboard_context-must_not_clause%"
              ]
            }
          },
     ...
```


# Vega vs VegaLite
VegaLite is a simplified version of Vega, useful to quickly get started, but has a number of limitations.  VegaLite is automatically converted into Vega before rendering. Compare [logstash-simple_line-vega](public/examples/logstash/logstash-simple_line-vega.json) and [logstash-simple_line-vegalite](public/examples/logstash/logstash-simple_line-vegalite.json) (both use the same ElasticSearch logstash data). You may use [this editor](https://vega.github.io/editor/) to convert VegaLite into Vega.

# Debugging

## Browser Debugging console
Use browser debugging tools (e.g. F12 or Ctrl+Shift+J in Chrome) to inspect the `VEGA_DEBUG` variable:
 * `view` - access to the Vega View object. See [Vega Debugging Guide](https://vega.github.io/vega/docs/api/debugging/) on how to inspect data and signals at runtime. For VegaLite, `VEGA_DEBUG.view.data('source_0')` would get the main dataset. For Vega, it uses the data name as defined in your Vega spec.
 * `spec` - Vega JSON specification after some modifications by this plugin. In case of VegaLite, this is the output of the VegaLite compiler. 
 * `vlspec` - If this is a VegaLite graph, JSON specification of the graph before VegaLite compilation.

## Data
If you are using ElasticSearch query, make sure your resulting data is what you expected. The easiest way to view it is by using "networking" tab in the browser debugging tools (e.g. F12).  Modify the graph slightly so that it makes a search request, and view the response from the server.  Another approach is to use [Kibana Dev Tools](https://www.elastic.co/guide/en/kibana/current/console-kibana.html) tab - place the index name into the first line: `GET <INDEX_NAME>/_search`, and add your query as the following lines (just the value of the `"query"` field)

If you need to share your graph with someone, you may want to copy the raw data response to [gist.github.com](https://gist.github.com/), possibly with a `.json` extension, use the `[raw]` button, and use that url directly in your graph.

# Notes

### Useful Links
* [Editor](https://vega.github.io/editor/) - includes examples for Vega & VegaLite, but does not support any Kibana-Plugin-specific features like ElasticSearch requests and interactive base maps.
* VegaLite [Tutorials](https://vega.github.io/vega-lite/tutorials/getting_started.html), [docs](https://vega.github.io/vega-lite/docs/), and [examples](https://vega.github.io/vega-lite/examples/)
* Vega [Tutorial](https://vega.github.io/vega/tutorials/), [docs](https://vega.github.io/vega/docs/), [examples](https://vega.github.io/vega/examples/)

### Using Vega and VegaLite examples
When using [Vega](https://vega.github.io/vega/examples/) and [VegaLite](https://vega.github.io/vega-lite/examples/) examples, you may need to modify the "data" section to use absolute URL. For example, replace `"url": "data/world-110m.json"` with `"url": "https://vega.github.io/editor/data/world-110m.json"`. Also, regular Vega examples use `"autosize": "pad"` layout model, whereas Kibana plugin uses `fit`. Remove all `autosize`, `width`, and `height` values.  See [sizing and positioning](#sizing-and-positioning) below.

### Additional configuration options
These options are specific to this plugin. They control how plugin interprets your Vega spec. [Map support](#vega-with-a-map) has additional configuration options.
```yaml
{
  "_hostConfig": {
    // Placement of the Vega-defined signal bindings.
    // Can be `left`, `right`, `top`, or `bottom` (default).
    "controlsLocation": "top",
    // Can be `vertical` or `horizontal` (default).
    "controlsDirection": "vertical",
    // If true, hides most of Vega and VegaLite warnings
    "hideWarnings": true,
  },
  /* the rest of Vega JSON */
}
```

### Sizing and positioning
##### Vega and VegaLite
By default, Kibana Vega graphs will use `autosize = { type: 'fit', contains: 'padding' }` layout model for Vega and VegaLite graphs. The `fit` model uses all available space, ignores `width` and `height` values, but respects the padding values. You may override this behaviour by specifying a different `autosize` value.

##### Vega on a map
All Vega graphs will ignore `autosize`, `width`, `height`, and `padding` values, using `fit` model with zero padding.


## Development

See the [kibana contributing guide](https://github.com/elastic/kibana/blob/master/CONTRIBUTING.md) for instructions setting up your development environment. Once you have completed that, use the following npm tasks.

  - `npm start`

    Start kibana and have it include this plugin

  - `npm run build`

    Build a distributable archive

  - `npm run test:browser`

    Run the browser tests in a real web browser

  - `npm run test:server`

    Run the server tests using mocha

For more information about any of these commands run `npm run ${task} -- --help`.
