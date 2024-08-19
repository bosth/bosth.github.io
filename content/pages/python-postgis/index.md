---
title: "Python for PostGIS"
tags: [python, postgis, postgresql, geospatial]
date: 2024-08-01
---

I wrote and maintain two Python modules that make it easier to use geospatial Python from within your PostgreSQL database.

## ``plpygis``

`plpygis` is a pure Python module with no dependencies that can convert geometries between [Well-known binary](https://en.wikipedia.org/wiki/Well-known_binary) (WKB/EWKB), [Well-known Text](https://en.wikipedia.org/wiki/Well-known_text_representation_of_geometry) (WKT/EWKT) and GeoJSON representations. `plpygis` is mainly intended for use in PostgreSQL [PL/Python](https://www.postgresql.org/docs/current/plpython.html) functions to augment [PostGIS](https://postgis.net/)'s native capabilities.

{{< github repo="bosth/plpygis" >}}

A simple Python function using `plpygis` would be:

```postgresql
CREATE OR REPLACE FUNCTION swap(geom geometry)
  RETURNS geometry
AS $$
  # swaps the x and y coordinates of a point
  from plpygis import Geometry, Point
  old_point = Geometry(geom)
  new_point = Point([old_point.y, old_point.x])
  return new_point
$$ LANGUAGE plpython3u;
```

The magic happens with `Geometry(geom)`, which is automatically converts from PostGIS's `geometry` type, and the `return` statement, which automatically converts *back* to a PostGIS `geometry`.

The function above can be called with a normal SQL statement:

```postgresql
SELECT swap(geom) FROM city;
```

I spoke about **plpygis** at FOSS4G 2017 in Boston. The slides are [here](https://2017.foss4g.org/post_conference/Extending-PostGIS-with-Python.pdf) and a video of the talk is also available. I also presented an example of using `plpygis` for some real-world analysis at FOSS4G 2023 in Prizren, and you can find a write-up [here]({{< ref "ads-b" >}}).

{{< vimeo id="248099711" title="Extending PostGIS with Python" >}}

<br />

{{< button href="https://plpygis.readthedocs.io/en/latest/" >}}
Documentation
{{< /button >}}

## ``geofdw``

``geofdw`` is a collection of [PostGIS](http://postgis.net)-related [foreign data wrappers](https://wiki.postgresql.org/wiki/Foreign_data_wrappers) for [PostgreSQL](http://postgresql.org) written in Python using the [multicorn](http://multicorn.org) extension. By using a FDW, you can access spatial data through Postgres tables without having to import the data first, which can be useful for dynamic or non-tabular data available through web services.

{{< github repo="bosth/geofdw" >}}
