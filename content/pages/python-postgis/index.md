---
title: "Python for PostGIS"
tags: [python, postgis, postgresql, geospatial]
---

I authored two Python modules that make it easier to use geospatial Python from within your PostgreSQL database.

## ``plpygis``

``plpygis`` is a Python conveter to and from the PostGIS geometry type, WKB, EWKB, GeoJSON and Shapely geometries and additionally supports `__geo_interface__`. ``plpygis`` is intended for use in PL/Python functions, making the entire Python geospatial ecosystem available in SQL queries.

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

I spoke about **plpygis** at FOSS4G 2017 in Boston. The slides are [here](https://2017.foss4g.org/post_conference/Extending-PostGIS-with-Python.pdf) and a video of the talk is also available.

{{< vimeo 248099711 >}}

<br />

{{< button href="https://plpygis.readthedocs.io/en/latest/" target="_blank" >}}
Documentation
{{< /button >}}

## ``geofdw``

``geofdw`` is a collection of [PostGIS](http://postgis.net)-related [foreign data wrappers](https://wiki.postgresql.org/wiki/Foreign_data_wrappers) for [PostgreSQL](http://postgresql.org) written in Python using the [multicorn](http://multicorn.org) extension. By using a FDW, you can access spatial data through Postgres tables without having to import the data first, which can be useful for dynamic or non-tabular data available through web services.

{{< github repo="bosth/geofdw" >}}
