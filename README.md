# ArcPy Tips & Tricks

## Section 1: Efficient Geoprocessing Tools

* ArcPy imports are time consuming in manual scripts (~30s), but every geoprocessing service has access to its own shared ArcPy instance maintained by the server, negating this overhead. If you import a custom toolbox or geoprocessing tool from a running script, that initial import will be accessible to all future instances of the script until you stop and restart the service. This has several implications:

1. If that imported toolbox is generating a server token on instantiation and isn’t set up to create a new one on expiry you’re only ever going to have the initial token, and your service will throw access errors once that token times out, e.g. 30 minutes after start-up. New runs of your script aren't freshly generating a token, because from their perspective one already exists - the first run generated it. Be sure to check whether tokens are valid, even if the script should be requesting a new one by default.

2. Check if a toolbox/tool exists in the current ArcPy environment before importing it or you’ll end up with multiple copies of the same toolbox stacked up in your shared environment, with a new one added every time your tool is run.

* Creating an in-memory feature class is dramatically faster than creating one (from slowest to fastest): remotely on another server, locally on the same machine, or within the tool’s scratch geodatabase (created fresh by the server for every new toolbox instance). Storing data in memory does come with risks: A large enough dataset can max out your available memory, resulting in the toolbox crashing or a huge increase in running time as the server begins paging out data to the hard disk.

* Cursors should always be passed a minimal list of fields. If you don’t need the geometry be sure to exclude ‘SHAPE@’ from your field list. If you only need a small subset of features ensure you pass an appropriate WHERE clause.

* You can select a subset of features in a feature layer with a where clause. If you don’t have the information to reduce the size of your feature set with a WHERE clause when instantiating a cursor you can do so with a selection once you’ve iterated through your features and built a list of features that need to be accessed in the future.

* Geometry objects (those returned from the 'SHAPE@' field of a feature) are a rich, useful class to work with. They come with an enormous set of built-in spatial functions, allowing you to perform analysis tasks like clipping and intersecting entirely in memory. These operations can be much faster than running the equivalent tool on a feature class, and if you're already iterating through rows with a cursor there's comparatively little overhead in performing geometry operations as you traverse.

* In one workflow I had to erase a set of small polygons from a larger polygon, an operation we had to repeat hundreds of thousands of times a day. Using the erase tool on a feature class was prohibitively slow, so we decided to minimise the work it did by pre-checking geometry objects ourelves. This pre-check took our operation from an average of 25 seconds to 5.

1. With search cursors access the large polygon and the set of smaller polygons that need to be erased from it.
2. For each smaller polygon, check if it overlaps the large one by running 'smallerPolygonGeometry.distanceTo(largerPolygonGeometry)'. If this returns zero they touch or intersect - store a reference to that polygon.
3. Initialise a new search cursor on the set of smaller polygons with the list generated in 2. as a WHERE clause.
4. Perform the erase using only those polygons that touch the target.

``` Py
    # Improve the speed of the Erase operation by only including polygons which touch the dissolved management block.
    with arcpy.da.SearchCursor(shape_to_erase_from, "SHAPE@") as erase_cursor:
        for erase_target in erase_cursor:
            erase_target_geometry = erase_target[0]
        with arcpy.da.SearchCursor(shapes_to_erase, ["OBJECTID", "SHAPE@"]) as shapes_to_erase_cursor:
            shapes_that_touch_the_target = []
            for shape_to_erase in shapes_to_erase_cursor:
                geometry_to_erase = shape_to_erase[1]
                if geometry_to_erase.distanceTo(shape_to_erase_from) == 0:
                    shapes_that_touch_the_target.append(shape_to_erase[0])  # Store a reference to the row's OBJECTID or GLOBALID if present.

            erase_query = '"OBJECTID" IN ({0})'.format(
                ", ".join(map(str, shapes_that_touch_the_target)) or "''")
            arcpy.SelectLayerByAttribute_management(
                shapes_to_erase, "NEW_SELECTION", shapes_that_touch_the_target)

    if len(shapes_that_touch_the_target) > 0:
        arcpy.Erase_analysis(shape_to_erase_from,
                             shapes_that_touch_the_target, output_feature_class)
```
